# C4 — Observable Agents Design Spec

**Date:** 2026-06-07
**Branch:** c4-observable-agents
**Audit refs:** O-1, O-2, O-4, O-5, O-6, D-4
**Depends on:** C2 (AgentListener CDI auto-discovery), C3 (ManagedExecutor context propagation)

---

## Upstream Context

Upstream `langchain4j-agentic` provides the `AgentListener` SPI and `AgentMonitor` (in-memory execution tree + HTML report generator) but zero production observability implementations — no OTel spans, no metrics, no health checks, no event bridge. This chapter is purely a Quarkus integration layer: CDI `AgentListener` beans that plug into the upstream SPI with Quarkus-native observability.

Tool execution spans are deliberately excluded from the agent OTel listener. The core module's `ToolSpanWrapper` already creates `langchain4j.tools.*` spans at the AI service level, and these nest correctly under agent spans via OTel context propagation (C3's `ManagedExecutor` wiring). Duplicating tool spans from `AgentListener.beforeAgentToolExecution`/`afterAgentToolExecution` would produce double spans in every trace.

---

## Architecture

Four conditional CDI beans registered via `AdditionalBeanBuildItem` in a new `AgenticObservabilityProcessor`, plus two targeted fixes in existing classes.

| Bean | Type | Condition | Module |
|------|------|-----------|--------|
| `AgentSpanListener` | `AgentListener` | `Capability.OPENTELEMETRY_TRACER` present | runtime |
| `AgentMetricsListener` | `AgentListener` | `MetricsCapabilityBuildItem` with Micrometer | runtime |
| `AgentCdiEventListener` | `AgentListener` | Unconditional | runtime |
| `AgentHealthCheck` | `@Readiness HealthCheck` | `Capability.SMALLRYE_HEALTH` present | runtime |

All listener beans set `inheritedBySubagents() → true` so that a single registration on the root agent propagates observability to the entire agent tree.

**Why `AdditionalBeanBuildItem` (not `SyntheticBeanBuildItem`):** All four beans are regular CDI classes with `@Inject` — they need the CDI container to resolve `Tracer`, `MeterRegistry`, `Event<>`, and `HealthCheckResponse`. Synthetic beans don't support injection. The listener beans are auto-discovered by the existing C2 `Instance<AgentListener>` injection point on every agent — no special wiring beyond being registered as CDI beans.

**Why a separate `AgenticObservabilityProcessor`:** `AgenticProcessor` is already 1100+ lines. The Quarkus extension guide recommends separating build steps by concern into dedicated processor classes — build step classes have no shared state, and fine-grained steps enable better pruning by the build system.

---

## O-1: OTel Spans — `AgentSpanListener`

`@ApplicationScoped` CDI bean injecting `Tracer`.

### Span lifecycle

- `beforeAgentInvocation` — starts a span named `langchain4j.agent.<agentName>`, makes it current via `span.makeCurrent()`, stores the `Span` + `Scope` pair in a `ConcurrentHashMap<String, SpanScope>` keyed by agent ID
- `afterAgentInvocation` — retrieves the entry, records token usage attributes if available, closes the scope, ends the span
- `onAgentInvocationError` — retrieves the entry, calls `span.recordException(error)`, sets error status, closes the scope, ends the span

### Span attributes (OTel GenAI semantic conventions)

| Attribute | Value | When |
|-----------|-------|------|
| `gen_ai.operation.name` | `"agent_invocation"` | Always |
| `gen_ai.agent.name` | `AgentRequest.agentName()` | Always |
| `gen_ai.agent.id` | `AgentRequest.agentId()` | Always |
| `gen_ai.usage.input_tokens` | From `AgentResponse.chatResponse().tokenUsage()` | On completion, if present |
| `gen_ai.usage.output_tokens` | From `AgentResponse.chatResponse().tokenUsage()` | On completion, if present |

### Nesting

Making the span current ensures child agent invocations and tool executions (via core's `ToolSpanWrapper`) automatically become children in the trace. `ManagedExecutor` from C3 propagates OTel context to parallel worker threads.

### Thread safety

`ConcurrentHashMap<String, SpanScope>` keyed by agent ID. Parallel sub-agents run on different threads with distinct agent IDs — no contention.

`SpanScope` is a simple record: `record SpanScope(Span span, Scope scope) {}`.

---

## O-2: Micrometer Metrics — `AgentMetricsListener`

`@ApplicationScoped` CDI bean injecting `MeterRegistry`.

### Metrics

| Metric | Type | Tags | When |
|--------|------|------|------|
| `gen_ai.agent.invocations` | Counter | `gen_ai.agent.name`, `error.type` | `afterAgentInvocation` (error.type=none) and `onAgentInvocationError` (error.type=exception class name) |
| `gen_ai.agent.duration` | Timer | `gen_ai.agent.name`, `error.type` | `afterAgentInvocation` and `onAgentInvocationError` |
| `gen_ai.agent.tool.executions` | Counter | `gen_ai.agent.name`, `gen_ai.tool.name` | `afterAgentToolExecution` |

### Duration tracking

`beforeAgentInvocation` stores `System.nanoTime()` in a `ConcurrentHashMap<String, Long>` keyed by agent ID. Completion/error callbacks compute the delta and record it.

### Meter providers

Use `Meter.MeterProvider<Counter>` and `Meter.MeterProvider<Timer>` with `withRegistry(Metrics.globalRegistry)` — same pattern as `MetricsChatModelListener` in core. Creates providers once at construction time, stamps tags per invocation.

### Tag consistency with core

Core uses `gen_ai.client.*` for model-level metrics with tags like `gen_ai.request.model`, `ai_service.class_name`. Agent metrics use `gen_ai.agent.*` — different dimensions (agent vs model), no tag key collision. `error.type` is shared across both for alerting consistency.

---

## D-4: CDI Events — `AgentCdiEventListener`

`@ApplicationScoped` CDI bean injecting `Event<AgentStartedEvent>`, `Event<AgentCompletedEvent>`, `Event<AgentErrorEvent>`.

### Event types (new records in `agentic/runtime`)

| Event class | Fired from | Fields |
|-------------|-----------|--------|
| `AgentStartedEvent` | `beforeAgentInvocation` | `agentName`, `agentId`, `inputs` |
| `AgentCompletedEvent` | `afterAgentInvocation` | `agentName`, `agentId`, `inputs`, `output`, `durationNanos`, `tokenUsage` (optional) |
| `AgentErrorEvent` | `onAgentInvocationError` | `agentName`, `agentId`, `inputs`, `error` |

### Design decisions

- **`Event.fire()` (synchronous), not `fireAsync()`** — observers run on the calling thread, same as core's `AiServiceListenerAdapter`. Synchronous delivery means observers see the full CDI context (request scope, security, OTel span). Garden entry `GE-20260523-bd68ba` documents that `@ObservesAsync` silently loses OTel span context. Users who need async dispatch can declare `@ObservesAsync` on their observer method.

- **Simple records, no upstream event hierarchy** — the core uses `AiServiceEvent` from `langchain4j-observability-api`, but there is no equivalent upstream agent event API. Our events are Quarkus-side CDI events — simple records wrapping data from `AgentRequest`/`AgentResponse`/`AgentInvocationError`.

- **No qualifier annotation** — core uses `@AiServiceSelector` to filter by AI service class. Agent events carry agent name and ID in the payload. Users filter in the observer method body. A qualifier can be added later if demand arises.

- **Duration on `AgentCompletedEvent`** — the listener tracks start time in `beforeAgentInvocation` (same concurrent map pattern as the other listeners) and includes duration in the completed event so observers don't have to correlate start/end events.

---

## O-4: Health Check — `AgentHealthCheck`

`@ApplicationScoped @Readiness HealthCheck` CDI bean.

### What it checks

At readiness probe time, attempts to resolve each root agent CDI bean via `Arc.container().select(agentClass)`. If all resolve without exception: UP. If any throws: DOWN with agent name and exception message in the response data.

### Root agent class names

Passed from the build step via a recorder method that stores them in a static field on the health check (same pattern as `AgenticRecorder.setLeafAgentClassNames()`).

### Scope

`@Readiness` only, not `@Liveness`. Agent initialization failure is a deployment problem, not a runtime health signal. Runtime failures (LLM provider down) are handled by metrics/alerting on `gen_ai.agent.invocations` error rate.

---

## O-5: DevAgentMonitorHolder Hot-Reload Fix

Targeted fix in `AgenticRecorder.enableDevModeMonitoring()`.

### Problem

`DevAgentMonitorHolder` uses `static final CopyOnWriteArrayList` fields. On hot-reload, if agentic build steps don't re-run (agent interfaces unchanged), `reset()` is never called. Stale monitors accumulate.

### Fix

Add `ShutdownContext` parameter to the recorder method. Register a callback:

```java
void enableDevModeMonitoring(Set<String> rootAgentClassNames, ShutdownContext shutdownContext) {
    // ... existing logic ...
    shutdownContext.addShutdownTask(DevAgentMonitorHolder::reset);
}
```

`ShutdownContext` fires on every hot-reload cycle before the new application starts, guaranteeing a clean slate.

---

## O-6: Dev UI Structured JSON

### Backend

Replace HTML-returning methods in `AgenticJsonRpcService` with structured JSON:

- `getTopologyJson(int index)` → `JsonObject` tree: `{name, type, description, outputKey, subAgents: [...], tools: [...]}` — walks the `AgentInstance` hierarchy
- `getExecutionReportJson(int index)` → `JsonObject`: `{executions: [{memoryId, status, topLevel: {agentName, startTime, duration, tokenCount, inputs, output, toolExecutions: [...], nestedInvocations: [...]}}]}` — serializes `MonitoredExecution`/`AgentInvocation` trees recursively

Old HTML methods (`getTopologyHtml`, `getExecutionReportHtml`) are removed.

### Frontend

- `qwc-agents-topology.js` — replace `<iframe>` with Vaadin tree grid rendering topology JSON. Columns: agent name, type, description, outputKey.
- `qwc-agents-executions.js` — replace `<iframe>` with Vaadin grid showing execution timeline. Expandable rows for nested invocations. Columns: agent name, status, duration, token count, iteration index. Error rows highlighted.

---

## Build-Time Wiring — `AgenticObservabilityProcessor`

New class in `agentic/deployment`.

### Build steps

**Step 1: `registerObservabilityListeners`**
- Parameters: `Capabilities`, `Optional<MetricsCapabilityBuildItem>`, `BuildProducer<AdditionalBeanBuildItem>`
- If `Capability.OPENTELEMETRY_TRACER` present → register `AgentSpanListener` (unremovable)
- If `MetricsCapabilityBuildItem` present with Micrometer → register `AgentMetricsListener` (unremovable)
- Unconditionally → register `AgentCdiEventListener` (unremovable)

**Step 2: `registerHealthCheck`**
- Parameters: `Capabilities`, `BuildProducer<AdditionalBeanBuildItem>`
- If `Capability.SMALLRYE_HEALTH` present → register `AgentHealthCheck` (unremovable)

### Maven dependencies (runtime module)

Optional dependencies added to `agentic/runtime/pom.xml`:
- `io.quarkus:quarkus-opentelemetry` — `<optional>true</optional>`
- `io.quarkus:quarkus-micrometer` — `<optional>true</optional>`
- `io.quarkus:quarkus-smallrye-health` — `<optional>true</optional>`

Already test-scope in deployment module — no changes needed there.

---

## Testing Strategy

| Test class | Validates | Key dependencies |
|------------|-----------|------------------|
| `AgentSpanListenerTest` | Nested OTel spans with correct attributes; sub-agent span parenting; error span recording | `quarkus-opentelemetry`, `InMemorySpanExporter` |
| `AgentMetricsListenerTest` | Invocation counter, duration timer, error tags, tool execution counter | `quarkus-micrometer`, `SimpleMeterRegistry` |
| `AgentCdiEventListenerTest` | All 3 event types delivered to `@Observes` with correct payload | None (CDI always present) |
| `AgentHealthCheckTest` | Reports UP when agents initialize | `quarkus-smallrye-health` |
| `DevAgentMonitorHolderResetTest` | `ShutdownContext` callback clears stale monitors | Dev mode |
| `ObservabilityAbsentTest` | App starts cleanly with no OTel/Micrometer/Health — no errors | Negative test, no observability deps |

### Patterns

- OTel: `InMemorySpanExporter` with `schedule.delay=PT0.001S` (from `ParallelOtelPropagationTest`)
- Metrics: `SimpleMeterRegistry` added to `Metrics.globalRegistry` (from `AgentMeterRegistryTest`)
- CDI events: `@ApplicationScoped` capture bean with `CopyOnWriteArrayList` — synchronous `fire()` so no async timing issues
- All tests use inline `ChatModel` (no WireMock needed for observability tests)
- Dev UI backend: unit test calling JSON-RPC method and asserting JSON structure

---

## Files Changed (Summary)

### New files (runtime)
- `AgentSpanListener.java`
- `AgentMetricsListener.java`
- `AgentCdiEventListener.java`
- `AgentStartedEvent.java`
- `AgentCompletedEvent.java`
- `AgentErrorEvent.java`
- `AgentHealthCheck.java`

### New files (deployment)
- `AgenticObservabilityProcessor.java`

### Modified files
- `AgenticRecorder.java` — `ShutdownContext` parameter on `enableDevModeMonitoring`
- `AgenticJsonRpcService.java` — HTML methods replaced with JSON methods
- `DevAgentMonitorHolder.java` — no change (reset already exists, just needs to be called)
- `agentic/runtime/pom.xml` — optional deps for OTel, Micrometer, Health
- `qwc-agents-topology.js` — Vaadin tree grid replacing iframe
- `qwc-agents-executions.js` — Vaadin grid replacing iframe

### New test files
- `AgentSpanListenerTest.java`
- `AgentMetricsListenerTest.java`
- `AgentCdiEventListenerTest.java`
- `AgentHealthCheckTest.java`
- `DevAgentMonitorHolderResetTest.java`
- `ObservabilityAbsentTest.java`
