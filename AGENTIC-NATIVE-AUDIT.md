# Quarkus-Native Audit: `agentic` module

All findings verified against actual source via IntelliJ IDE semantic index.
Project: `quarkus-langchain4j`, module: `agentic/`
Date: 2026-06-02

---

## How to read this document

Each finding has:
- **Verified source** — file:line (or JAR decompile) where the issue was confirmed
- **Impact** 1–5 (5 = every Quarkus agent user is affected)
- **Effort** 1–5 (1 = trivial one-liner, 5 = requires upstream API change in `langchain4j-agentic`)
- **API change** — No / Additive / Yes-breaking
- **Risk** — Low / Med / High (of the fix itself)

---

## Section A — Parallel Alternatives (architectural pattern)

These are the highest-priority findings. They represent areas where the `langchain4j-agentic`
library invented its own version of something Quarkus already provides natively, creating a
parallel, inferior system that degrades the entire Quarkus user experience.

---

### A-1 · Supplier static methods reinvent CDI injection

**Verified source**: `AgenticProcessor.java:135–349` — eight supplier annotations
(`@ChatModelSupplier`, `@ToolsSupplier`, `@ChatMemorySupplier`, `@ChatMemoryProviderSupplier`,
`@ContentRetrieverSupplier`, `@RetrievalAugmentorSupplier`, `@AgentListenerSupplier`,
`@ToolProviderSupplier`) are all validated to be `static`, no-args methods at build time.

The agentic module invented a parallel dependency injection system using static interface methods:

| Supplier annotation | CDI equivalent | What the supplier loses |
|---|---|---|
| `@ChatModelSupplier static ChatModel m()` | `@Inject ChatModel` | Scoping, interception, qualifiers, test alternatives |
| `@ToolsSupplier static Object[] tools()` | `@Inject Instance<Object>` filtered by `@Tool` | Dynamic discovery, CDI lifecycle |
| `@ChatMemorySupplier static ChatMemory m()` | `@Inject ChatMemory` | `@RequestScoped` per-session |
| `@ChatMemoryProviderSupplier static ChatMemory forId(Object id)` | `@Inject ChatMemoryProvider` | Transactions, injection |
| `@ContentRetrieverSupplier static ContentRetriever r()` | `@Inject ContentRetriever` | Qualifiers, testability |
| `@RetrievalAugmentorSupplier static RetrievalAugmentor r()` | `@Inject RetrievalAugmentor` | Same |
| `@AgentListenerSupplier static AgentListener l()` | CDI `Instance<AgentListener>` | Auto-discovery, metrics, OTel |
| `@ToolProviderSupplier static ToolProvider p()` | `@Inject ToolProvider` or `Instance<ToolProvider>` | Dynamic registration |

The `@CdiBean` annotation + `CdiChatSupplierParameterResolver` is a narrow workaround that only
works for `@ChatModelSupplier` parameters and only resolves by type (no qualifier support).

**Impact**: 5 — root architectural problem; causes most other CDI findings below
**Effort**: 4 (CDI auto-wire path requires no upstream changes; full removal requires upstream SPI change)
**API change**: Additive (static methods remain valid)
**Risk**: Med

---

### A-2 · Guardrails: complete parallel absence — the entire CDI guardrail system is unreachable

**Verified source**: `ide_search_text` for "guardrail" in `agentic/` → **zero hits**.

The `quarkus-langchain4j-core` module has a rich, production-grade CDI guardrail system:

| Core module guardrail | Purpose | Applied via |
|---|---|---|
| `InputGuardrail` | Validate/rewrite the prompt before the LLM call | `@InputGuardrails({...})` on AI service method |
| `OutputGuardrail` | Validate/reprompt the LLM response; retry | `@OutputGuardrails({...})` on AI service method |
| `ToolInputGuardrail` | Validate tool call arguments before execution | `@ToolInputGuardrails({...})` on `@Tool` method |
| `ToolOutputGuardrail` | Validate/modify tool results before returning | `@ToolOutputGuardrails({...})` on `@Tool` method |

All are CDI beans (`@ApplicationScoped`, `@RequestScoped`) with injection, metrics, CDI events,
chain semantics, reprompting, streaming support, and full test coverage.

**Agents get none of this.** There is no way to:
- Validate or sanitise inputs to an agent call
- Validate the output of a sub-agent before passing it to the next agent
- Reprompt an agent when its output fails a business rule
- Apply security/PII filters at agent boundaries

The `@ErrorHandler` static method is a crude partial substitute — fires only after a failure,
cannot modify inputs, cannot reprompt, cannot access CDI beans.

**Quarkus-native fix**: `AgenticProcessor` needs `@BuildStep` support to detect
`@InputGuardrails`/`@OutputGuardrails` on agent interfaces and wire them via
`QuarkusAgenticContextConsumer` into the `AgentConfigurator`. The guardrail CDI discovery
and execution infrastructure already exists in `GuardrailsSupport`.

**Impact**: 5 — safety and correctness feature, not optional for production use
**Effort**: 3
**API change**: No (additive — same `@InputGuardrails` annotation users already know)
**Risk**: Low

---

### A-3 · `@AgentListenerSupplier` reinvents CDI `Event<T>` / `@Observes`

**Verified source**: `AgenticProcessor.java:150–163` — `@AgentListenerSupplier` is a static factory.
`AgenticRecorder.java:99` — no CDI `Event<T>` firing anywhere. The core module fires
`AiServiceStartedEvent`, `ToolGuardrailExecutedEvent`, etc. via injected `Event<T>`.

`AgentListener` (`dev.langchain4j.agentic.observability.AgentListener`) has rich lifecycle hooks:
`beforeAgentInvocation`, `afterAgentInvocation`, `onAgentInvocationError`,
`beforeAgentToolExecution`, `afterAgentToolExecution`, `afterAgenticScopeCreated`,
`beforeAgenticScopeDestroyed`.

But because it must be wired via a static factory method, a listener cannot be a CDI bean,
cannot inject `MeterRegistry` or `Tracer`, and cannot be globally observed across all agents.
Two separate observability tracks now exist in the same library: CDI events (core) vs
`AgentListener` (agentic). Users must use both to observe the full execution.

**Quarkus-native fix**: CDI auto-discovery of `AgentListener` beans via `Arc.container().select(AgentListener.class)` in `QuarkusAgenticContextConsumer`. Upstream: fire CDI events from `AgentMonitor`.

**Impact**: 4
**Effort**: 3 (CDI auto-discovery requires no upstream change; CDI event firing requires upstream)
**API change**: No (additive)
**Risk**: Low

---

### A-4 · `@ActivationCondition` static predicate reinvents CDI routing

**Verified source**: `AgenticProcessor.java:135–148` — `@ActivationCondition` validated as a
static boolean method; `DeclarativeUtil` calls it via reflection to decide which sub-agent handles a request.

This is a runtime conditional dispatch system invented in-library. CDI already solves this:
- `@Alternative` + `@Priority` for deployment-time selection between implementations
- `@LookupIfProperty` / `@IfBuildProperty` for config-driven selection
- `Instance<T>` programmatic lookup for runtime predicate-based selection with full CDI lifecycle

The static method constraint means activation conditions cannot access `SecurityIdentity`,
`@RequestScoped` state, injected CDI beans (e.g., a feature-flag service), or be overridden
in tests via `@Alternative`.

**Impact**: 3
**Effort**: 4 (requires upstream support for CDI bean-based conditions)
**API change**: Additive
**Risk**: Low (upstream PR needed)

---

### A-5 · `PlannerAgent` prompt uses Mustache `{{}}` in a Qute pipeline — escape hack is fragile

**Verified source**: `PlannerAgentPromptTemplateContentFilterProvider.java` — string-replaces
`{'name', 'description', ...}` with `\{'name', 'description', ...}` (Qute escape). This exists
because `PlannerAgent.plan()` uses `@SystemMessage` containing `{{agents}}`, `{{request}}` (Mustache)
and literal `{'name': ...}` JSON-like syntax (which Qute would misparse as a template expression).

`PlannerAgentTest.java` is explicitly a canary test that fails if the upstream prompt changes,
alerting maintainers to update the escape string. This is a documented maintenance burden.

Two template syntaxes are now active simultaneously:
- Mustache `{{variable}}` — used by `langchain4j-agentic` prompts
- Qute `{variable}` — used by all other `@SystemMessage`/`@UserMessage` processing in Quarkus

**Consequence**: Any upstream change to the PlannerAgent prompt that adds new `{...}` literals
will silently corrupt the prompt unless caught by the narrow canary test. The canary only checks
one specific literal pattern.

**Quarkus-native fix**: Upstream should either switch to Qute-compatible syntax or provide a
pluggable template engine SPI. Short-term: extend the content filter to handle all `{...}` patterns
comprehensively, not just the one currently known.

**Impact**: 3
**Effort**: 3 (root fix requires upstream; short-term filter extension is effort 2)
**API change**: No
**Risk**: Med (silent prompt corruption)

---

### A-6 · A2A HTTP uses `JdkA2AHttpClient` — bypasses Vert.x, no config, no tracing

**Verified source**: `A2AAgentShouldWorkTest.java:64` shows `@A2AClientAgent(a2aServerUrl = "http://localhost:8080")`.
Decompile of `langchain4j-agentic` JAR: `DefaultA2AClientBuilder` uses `JdkA2AHttpClient` — a raw
`java.net.http.HttpClient` (`HTTP_2`, `followRedirects=NORMAL`). No Vert.x, no REST Client,
no Quarkus HTTP client instrumentation.

Compare: MCP transport in `quarkus-langchain4j` explicitly uses Vert.x `HttpClient`
(`QuarkusStreamableHttpMcpTransport`). A2A is inconsistent with MCP.

Consequences:
- Blocks a Vert.x event-loop thread if called from a reactive context (blocking I/O)
- No connection pooling governed by Quarkus/Vert.x
- No TLS/mTLS configuration via `quarkus.tls.*`
- No OTel HTTP client instrumentation — A2A calls invisible in traces
- No circuit-breaking via SmallRye Fault Tolerance
- URL is hardcoded in annotation — not overridable via `application.properties`

**Quarkus-native fix**: Provide a `VertxA2AHttpClient` implementing `A2AHttpClient` registered at build time in `AgenticProcessor`. Support `quarkus.langchain4j.agent.a2a.<name>.url` config property resolved at `RUNTIME_INIT`.

**Impact**: 4
**Effort**: 2
**API change**: No (additive)
**Risk**: Low

---

### A-7 · `AgenticScopeStore` SPI exists but has no Quarkus implementation

**Verified source**: `AgenticScopeStore` SPI in `langchain4j-agentic` JAR has `save`, `load`,
`delete`, `getAllKeys`. `DefaultAgenticScope` has `Kind` enum (`EPHEMERAL`, `REGISTERED`, `PERSISTENT`)
and a `serializableCopy()` method. The Quarkus agentic module provides **zero implementations**.
Confirmed: no file in the project implements or references this SPI beyond detection code.

Compare: `ChatMemoryStore` in the core module has multiple Quarkus implementations
(Redis, Infinispan, JPA) registered via CDI.

**Consequence**: Stateful multi-agent workflows (`PERSISTENT` scope kind) silently fall back to
in-memory only. Multi-node deployments lose state across pod restarts. Any production stateful
agentic workflow is broken by default.

**Quarkus-native fix**: New sub-modules `quarkus-langchain4j-agentic-infinispan` and
`quarkus-langchain4j-agentic-redis`, each providing an `@ApplicationScoped AgenticScopeStore`.
The core `AgenticProcessor` should detect a CDI `AgenticScopeStore` bean at build time and wire
it at `RUNTIME_INIT` via the recorder.

**Impact**: 5
**Effort**: 4 (new sub-modules; the SPI exists, the work is the Quarkus implementations)
**API change**: No (additive)
**Risk**: Low

---

## Section B — Concurrency & Context Propagation

### C-1 · Default parallel executor loses CDI context, OTel spans, and security context

**Verified source**: `PlannerBasedInvocationHandler.parallelExecution()` in `langchain4j-agentic-1.15.1-beta25-sources.jar` — calls `CompletableFuture.supplyAsync()` with `DefaultExecutorProvider.getDefaultExecutorService()` which returns `Executors.newVirtualThreadPerTaskExecutor()` on Java 21+. Not a `ManagedExecutor`.

Every `@ParallelAgent` without an explicit `@ParallelExecutor` loses CDI context
(`@RequestScoped` beans in tools throw `ContextNotActiveException`), OTel span propagation,
and `SecurityIdentity` propagation to sub-threads.

**Fix**: Add a `ManagedExecutor` injection point to the synthetic bean configurator and pass it
as the default executor when no `@ParallelExecutor` is declared.

**Impact**: 5 | **Effort**: 3 | **API change**: No | **Risk**: Med

---

### C-2 · `LangChain4jManaged.CURRENT` `ThreadLocal` not propagated into parallel sub-agent threads

**Verified source**: `LangChain4jManaged.CURRENT` is `new ThreadLocal()` in `langchain4j-core-1.15.1.jar`. `AgentInvocationHandler.invoke()` reads from this to get `AgenticScope`. On worker threads spawned by `parallelExecution()` the value is null — scope is silently isolated.

**Fix**: Wrap the executor with `ThreadContext.builder().propagated(ThreadContext.ALL_REMAINING).build().currentContextExecutor(exec)` (SmallRye Context Propagation is transitively on classpath).

**Impact**: 4 | **Effort**: 2 | **API change**: No | **Risk**: Low

---

### C-3 · CDI request context not activated on parallel agent worker threads

**Verified source**: `AgenticRecorder.java:81` — no request context lifecycle anywhere. `AgentMeterRegistryTest` manually calls `Arc.container().requestContext().activate()`, proving the team knows the pattern but hasn't applied it systematically.

**Fix**: Wrap the executor with context-propagating code that activates/terminates a request context per task. Subsumed by C-1 fix if `ManagedExecutor` is used.

**Impact**: 4 | **Effort**: 2 | **API change**: No | **Risk**: Low

---

### C-4 · `@ParallelExecutor` static factory cannot inject CDI beans (e.g. `ManagedExecutor`)

**Verified source**: `AgenticProcessor.java:287` enforces `static`, no-args, returns `Executor`. The `@CdiBean` mechanism only exists for `@ChatModelSupplier`.

**Fix**: Extend `@CdiBean` support to `@ParallelExecutor` parameters, mirroring `CdiChatSupplierParameterResolver`.

**Impact**: 3 | **Effort**: 3 | **API change**: No (additive) | **Risk**: Low

---

### C-5 · `HumanInTheLoop` blocks the calling thread — unsafe on Vert.x event loop

**Verified source**: `HumanInTheLoopAgentShouldWorkTest.java:68-74` — `futureResult.get()` with no timeout. Blocks the Vert.x event loop if called from a reactive handler.

**Fix**: Provide a `QuarkusHumanInTheLoopContext` helper using `vertx.executeBlocking()`. Document `@Blocking` requirement.

**Impact**: 3 | **Effort**: 3 | **API change**: No (additive helper) | **Risk**: Med

---

### C-6 · Sequential `future.get()` loop in `parallelExecution()` suboptimal

**Verified source**: `PlannerBasedInvocationHandler.parallelExecution()` — sequential for-loop of `.get()` calls waits on submission order, not completion order.

**Fix**: Replace with `CompletableFuture.allOf(tasks.toArray(...)).get()`.

**Impact**: 2 | **Effort**: 1 | **API change**: No | **Risk**: Low

---

### C-7 · `loadClassSafe()` uses `Thread.currentThread().getContextClassLoader()` — fragile

**Verified source**: `AgenticRecorder.java:196, :69` — TCCL may not be the deployment classloader on Vert.x worker threads or virtual threads.

**Fix**: Replace with `AgenticRecorder.class.getClassLoader()`.

**Impact**: 2 | **Effort**: 1 | **API change**: No | **Risk**: Low

---

## Section C — CDI Integration

### S-1 · `QuarkusAgenticContextConsumer` only injects `ToolProvider` — all other suppliers are CDI-blind

**Verified source**: `AgenticRecorder.java:196-213` — only injects `Instance<ToolProvider>` for MCP toolbox. `ContentRetriever`, `RetrievalAugmentor`, `ChatMemory`, `ChatMemoryProvider`, `AgentListener` have no CDI injection path.

**Fix**: For agents without a static supplier method for a given type, detect CDI beans of that type at build time via `BeanDiscoveryFinishedBuildItem` and inject via `SyntheticCreationalContext.getInjectedReference`.

**Impact**: 5 | **Effort**: 3 | **API change**: Additive | **Risk**: Med

---

### S-2 · `@CdiBean` resolver does type-only lookup — breaks with multiple beans of same type

**Verified source**: `CdiChatSupplierParameterResolver.java:14-16` — `Arc.container().select(context.parameter().getType()).get()`. No qualifier extraction. Two `ChatModel` beans → `AmbiguousResolutionException` at runtime, not build time.

**Fix**: Extract qualifier annotations from parameter (excluding `@CdiBean`) and pass to `Arc.container().select(type, qualifiers)`.

**Impact**: 3 | **Effort**: 2 | **API change**: No | **Risk**: Low

---

### S-3 · `markCdiBeanParametersAsUnremovable` only guards `@ChatModelSupplier` — latent bug

**Verified source**: `AgenticProcessor.java:438-452` — only iterates `item.getChatModelSupplier()`. Any extension of `@CdiBean` to other suppliers will silently produce `UnsatisfiedResolutionException`.

**Fix**: Expand the loop to cover all supplier annotation types.

**Impact**: 2 | **Effort**: 1 | **API change**: No | **Risk**: Low (becomes High if S-1 ships without this)

---

### S-4 · `ChatSupplierParameterResolver` SPI is intentionally chat-only — upstream structural blocker

**Verified source**: `DeclarativeUtil.class` in `langchain4j-agentic-1.15.1-beta25` — `addChatSupplierParameterResolver` exists only for the chat model path. All other suppliers enforce zero parameters.

Full `@CdiBean` support for non-chat suppliers requires an upstream `DeclarativeUtil` API extension.

**Impact**: 4 | **Effort**: 5 (upstream) | **API change**: No | **Risk**: Med

---

## Section D — Observability

### O-1 · No OTel spans for agent invocations

**Verified source**: Zero occurrences of `Span`, `WithSpan`, `Tracer`, `opentelemetry` in `agentic/`. A supervisor with 3 sub-agents making 2 LLM calls each produces 6 flat sibling spans with no parent agent span — traces are useless for debugging.

**Fix**: `AgentSpanListener` CDI bean injecting `Tracer`, wired via CDI auto-discovery (O-3). Conditioned on `quarkus-opentelemetry` being present.

**Impact**: 5 | **Effort**: 2 (after O-3) | **API change**: No | **Risk**: Low

---

### O-2 · No agent-level Micrometer metrics

**Verified source**: Only model-level metrics exist (`gen_ai.client.token.usage`, `gen_ai.client.operation.duration` via `MetricsChatModelListener`). No per-agent invocation counter, error rate, latency timer, or tool execution counter.

**Fix**: `AgentMetricsListener` CDI bean injecting `MeterRegistry`, auto-discovered (O-3). Emit `gen_ai.agent.invocations`, `gen_ai.agent.duration`, `gen_ai.agent.errors`, `gen_ai.agent.tool.executions`.

**Impact**: 4 | **Effort**: 2 (after O-3) | **API change**: No | **Risk**: Low

---

### O-3 · `AgentListener` beans not CDI auto-discovered — prerequisite for O-1 and O-2

**Verified source**: `AgenticRecorder.java:99` — no `Arc.container().select(AgentListener.class)`. Listeners must be static factories per agent interface.

**Fix**: In `QuarkusAgenticContextConsumer.accept()`, call `Arc.container().select(AgentListener.class)` and add all resolved instances to `agentBuilder`. Add matching `addInjectionPoint` in synthetic bean configurator.

**Impact**: 4 | **Effort**: 2 | **API change**: No (additive) | **Risk**: Low

---

### O-4 · No MicroProfile Health readiness probe for agent availability

**Verified source**: Zero `HealthCheck`, `Readiness`, `Liveness` references in `agentic/`.

**Fix**: `@Readiness HealthCheck` CDI bean conditioned on `quarkus-smallrye-health`, using `eagerlyInitRootAgents` logic.

**Impact**: 3 | **Effort**: 2 | **API change**: No | **Risk**: Low

---

### O-5 · `DevAgentMonitorHolder` static mutable state survives hot-reload

**Verified source**: `DevAgentMonitorHolder.java:9-10` — `static final CopyOnWriteArrayList`. Fast restarts that skip agentic build steps will not re-invoke `reset()`. Stale monitors accumulate.

**Fix**: Register a `ShutdownContext` callback from the recorder to call `DevAgentMonitorHolder::reset`.

**Impact**: 3 | **Effort**: 2 | **API change**: No | **Risk**: Low

---

### O-6 · Dev UI topology/execution pages render opaque HTML blobs

**Verified source**: `AgenticJsonRpcService.java:38-64` — `HtmlReportGenerator.generateTopology(...)` returns raw HTML. Cannot be filtered, sorted, or themed with Quarkus Dev UI components.

**Fix**: Return structured JSON from JSON-RPC methods; rewrite Qwc components using Vaadin grid.

**Impact**: 3 | **Effort**: 3 | **API change**: No | **Risk**: Low

---

## Section E — Configuration & Lifecycle

### L-1 · No `@ConfigMapping` for the agentic module

**Verified source**: Zero `ConfigRoot`, `ConfigMapping`, `@ConfigProperty` in `agentic/`. No `quarkus.langchain4j.agent.*` property namespace exists.

**Fix**: Add `@ConfigMapping(prefix = "quarkus.langchain4j.agent")` covering: `Optional<Integer> defaultMaxIterations()`, `Optional<Duration> defaultTimeout()`, `boolean scopePersistenceEnabled()`, `Map<String, AgentConfig> agents()`.

**Impact**: 4 | **Effort**: 1 | **API change**: No (additive) | **Risk**: Low

---

### L-2 · `A2AClientAgent.a2aServerUrl` is hardcoded in the annotation

**Verified source**: `A2AAgentShouldWorkTest.java:64` — `@A2AClientAgent(a2aServerUrl = "http://localhost:8080")`. No config expression, no runtime override.

**Fix**: Resolve config expression strings in the attribute at `RUNTIME_INIT` via `ConfigProvider.getConfig().getValue(...)`. Support `quarkus.langchain4j.agent.a2a.<name>.url`.

**Impact**: 5 | **Effort**: 3 | **API change**: No (additive) | **Risk**: Med

---

### L-3 · `LoopAgent.maxIterations` not overridable via config

**Verified source**: `Agents.java:318` — `@LoopAgent(..., maxIterations = 5, ...)`. Annotation-only, recompile required to change.

**Fix**: Requires L-1. Then read `agents.<name>.maxIterations` at `RUNTIME_INIT` and apply via `AgenticServices.AgentConfigurator` (may require upstream API).

**Impact**: 3 | **Effort**: 4 | **API change**: No | **Risk**: Med

---

### L-4 · `AgenticScopeStore` SPI has no CDI integration — persistent scope silently in-memory only

(See A-7 for full detail.)

**Impact**: 5 | **Effort**: 4 | **API change**: No | **Risk**: Low

---

### L-5 · Eager agent initialisation only in dev mode; no config gate

**Verified source**: `AgenticDevUIProcessor.java:96-108` — `eagerlyInitRootAgents` unconditional in dev mode. No knob to disable for CI.

**Fix**: Add `quarkus.langchain4j.agent.dev-ui.eager-init=true` under L-1 config mapping.

**Impact**: 2 | **Effort**: 2 | **API change**: No | **Risk**: Low

---

## Section F — Fault Tolerance

### F-1 · `@Retry` and `@CircuitBreaker` work but are untested and undocumented

**Verified source**: `hasAnyInterceptorBindings()` (`AgenticProcessor.java:505`) is generic — detects any interceptor binding. `FaultToleranceTest.java` proves interception works for `@Timeout`. Zero tests for `@Retry`, `@CircuitBreaker`, `@Fallback`, `@Bulkhead`.

**Fix**: Add test cases. No code change required.

**Impact**: 3 | **Effort**: 1 | **API change**: No | **Risk**: Low

---

### F-2 · `@Retry` on stateful agents silently corrupts `AgenticScope` state

**Verified source**: `AgenticScope` carries shared execution state. Core module has `ChatMemoryFlushStrategy.DEFERRED` (`ChatMemoryFlushStrategy.java:10-13`) to make `@Retry` safe — no equivalent for `AgenticScope`. A retry re-executes from scratch while scope already contains stale partial results from the first attempt.

**Fix**: `AgenticScopeCheckpoint` mechanism: snapshot scope state before each agent execution; restore on retry. Requires upstream API cooperation with `AgenticScopeRegistry`.

**Impact**: 5 (silent data corruption) | **Effort**: 4 | **API change**: No | **Risk**: High (not fixing is the risk)

---

### F-3 · `@Fallback(fallbackMethod=...)` silently fails at runtime on agent interfaces

**Verified source**: Agents are `java.lang.reflect.Proxy` instances — fallback method name resolution reflects on the proxy class, causing `FaultToleranceDefinitionException` at runtime. No build-time detection exists.

**Fix**: Build-time check in `AgenticProcessor.validate()` detecting `@Fallback(fallbackMethod=...)` on agent interfaces, directing users to `FallbackHandler<T>`.

**Impact**: 4 | **Effort**: 2 | **API change**: No | **Risk**: Low

---

### F-4 · `@Retry(retryOn=...)` interacts badly with `AgentInvocationException` wrapping

**Verified source**: `FaultToleranceTest.java:95-96` — exceptions bubble up as `AgentInvocationException` wrapping the root cause. Provider-specific exceptions (e.g. `RateLimitException`) cannot be targeted by `retryOn`.

**Fix**: Build-time warning when `@Retry` detected on an agent method without `abortOn` listing known non-retryable subtypes. Document wrapping behaviour.

**Impact**: 4 | **Effort**: 2 | **API change**: No | **Risk**: Med

---

### F-5 · `@Transactional` + agentic scope state can diverge silently on retry

**Verified source**: Zero `@Transactional` in `agentic/`. `AgenticScope` is not a JTA resource. A tool writing to JPA inside an agent + `@Transactional` + `@Retry` = partial commit + stale scope state on retry with no rollback.

**Fix**: Build-time warning when `@Transactional` co-exists with any agentic annotation on the same method.

**Impact**: 4 | **Effort**: 1 | **API change**: No | **Risk**: High (not fixing is the risk)

---

### F-6 · `@ErrorHandler` is static-only — cannot inject CDI beans during error recovery

**Verified source**: `AgenticProcessor.java:228` calls `validateStaticMethod()`. No CDI injection possible into error handlers. Users needing a retry budget counter or dead-letter queue must use `Arc.container()` in the static method body.

**Fix**: Requires upstream langchain4j API change to support instance method error handlers, or a CDI adapter pattern.

**Impact**: 3 | **Effort**: 4 | **API change**: Yes | **Risk**: Med

---

### F-7 · `hasAnyInterceptorBindings` uses `declaredAnnotations()` — misses inherited bindings

**Verified source**: `AgenticProcessor.java:505` — `agent.getIface().declaredAnnotations()`. Misses `@Timeout` inherited from a base interface.

**Fix**: Change to `agent.getIface().annotations()` for interface scan.

**Impact**: 2 | **Effort**: 1 | **API change**: No | **Risk**: Low

---

## Section G — Dev UI, Security & Native Image

### D-1 · `jsonRpcProvider` build step not gated on dev mode — registered in all profiles

**Verified source**: `AgenticDevUIProcessor.java:91-94` — no `onlyIf = IsDevelopment.class`. Other three build steps in the same class have this guard. `AgenticJsonRpcService` (with reflection-based agent invocation) is live in production.

**Fix**: Add `(onlyIf = IsDevelopment.class)`. One annotation.

**Impact**: 4 | **Effort**: 1 | **API change**: No | **Risk**: Low

---

### D-2 · `invokeAgent` JSON-RPC is unauthenticated arbitrary reflection — RCE vector

**Verified source**: `AgenticJsonRpcService.java:67-97` — accepts `agentClassName` as a plain string from the browser, calls `Class.forName(agentClassName)`, invokes `targetMethod.invoke(agent, args)`. No allow-list check against known agent classes captured at build time.

**Fix**: Allow-list `agentClassName` against class names from `addBuildTimeData("agents", agentInfos)`.

**Impact**: 5 | **Effort**: 2 | **API change**: No | **Risk**: High (not fixing is the risk)

---

### D-3 · No `ReflectiveClassBuildItem` for agent interfaces — native image gap

**Verified source**: `AgenticProcessor.nativeSupport()` produces `NativeImageProxyDefinitionBuildItem` but zero `ReflectiveClassBuildItem`. `AgenticRecorder.loadClassSafe()` uses `Class.forName` requiring reflective access in native.

**Fix**: In `AgenticProcessor.nativeSupport`, produce `ReflectiveClassBuildItem.builder(ifaceName).constructors(true).methods(true).build()` for each agent interface.

**Impact**: 3 | **Effort**: 2 | **API change**: No | **Risk**: Med (native image only)

---

### D-4 · No CDI events for agent lifecycle

**Verified source**: Zero `Event<` or `@Observes` in `agentic/`. Core module fires `AiServiceStartedEvent`, `AiServiceCompletedEvent`, etc. Agents emit nothing — no way to observe delegation, errors, or completions globally.

**Fix**: Define `AgentStartedEvent`, `AgentCompletedEvent`, `AgentDelegatedEvent`; fire via `Event<>` injected into the recorder.

**Impact**: 4 | **Effort**: 4 | **API change**: Additive | **Risk**: Low

---

### D-5 · `DetectedAiAgentBuildItem` undocumented as extension point

**Verified source**: Produced by `AgenticProcessor.detectAgents`; consumed only within the agentic module. No external extension can discover agents at build time.

**Fix**: Publish a stable `AgentMetadataBuildItem` with string-based public API. Document as supported extension point.

**Impact**: 3 | **Effort**: 2 | **API change**: No (additive) | **Risk**: Low

---

### D-6 · No security integration — `@RolesAllowed` on agents untested and undocumented

**Verified source**: Zero `RolesAllowed`, `SecurityIdentity`, `@Authenticated` in `agentic/`. CDI interceptors technically work (agents are `@ApplicationScoped` beans), but nothing is tested or documented. Sub-agents do not propagate security context.

**Fix**: Add `@RolesAllowed` test case. Document. For sub-agent security context propagation, ensure `SecurityIdentity` is in context-propagated scope (ties to C-1/C-3 fix).

**Impact**: 3 | **Effort**: 2 | **API change**: No | **Risk**: Low

---

## Master Priority Table

| ID | Finding | Theme | Impact | Effort | API | Risk | Priority |
|----|---------|-------|--------|--------|-----|------|----------|
| A-7 | `AgenticScopeStore` has no Quarkus impl — persistent state silently broken | Parallel | 5 | 4 | No | Low | **Critical** |
| A-2 | Guardrails completely absent | Parallel | 5 | 3 | No | Low | **Critical** |
| F-2 | `@Retry` silently corrupts `AgenticScope` state | FT | 5 | 4 | No | High | **Critical** |
| D-2 | `invokeAgent` unauthenticated reflection (RCE) | DevUI | 5 | 2 | No | High | **Critical** |
| F-5 | `@Transactional` + scope diverge silently | FT | 4 | 1 | No | High | **Fix now** |
| C-1 | Default parallel executor loses CDI/OTel/Security | Concur | 5 | 3 | No | Med | **Fix now** |
| A-1 | Supplier pattern reinvents CDI injection | Parallel | 5 | 4 | Add | Med | **Fix now** |
| A-6 | A2A uses raw JDK HTTP — blocks Vert.x, no config, no tracing | Parallel | 4 | 2 | No | Low | **Fix now** |
| O-1 | No OTel spans per agent invocation | Obs | 5 | 2† | No | Low | High |
| L-2 | `A2AClientAgent` URL hardcoded in annotation | Cfg | 5 | 3 | No | Med | High |
| C-2 | `ThreadLocal` not propagated to parallel threads | Concur | 4 | 2 | No | Low | High |
| C-3 | CDI request context missing on worker threads | Concur | 4 | 2 | No | Low | High |
| O-2 | No agent-level Micrometer metrics | Obs | 4 | 2† | No | Low | High |
| O-3 | `AgentListener` not CDI auto-discovered | Obs | 4 | 2 | No | Low | High |
| D-1 | `jsonRpcProvider` live in production | DevUI | 4 | 1 | No | Low | High |
| F-3 | `@Fallback(fallbackMethod)` silent runtime failure | FT | 4 | 2 | No | Low | High |
| F-4 | `@Retry(retryOn)` + exception wrapping unsafe | FT | 4 | 2 | No | Med | High |
| L-1 | No `@ConfigMapping` for agent module | Cfg | 4 | 1 | No | Low | High |
| S-1 | Other suppliers CDI-blind — retriever/memory/listener | CDI | 5 | 3 | Add | Med | High |
| D-4 | No CDI events for agent lifecycle | DevUI | 4 | 4 | Add | Low | Med |
| A-3 | `AgentListener` static supplier — parallel to CDI events | Parallel | 4 | 3 | No | Low | Med |
| A-4 | `@ActivationCondition` static predicate — parallel to CDI routing | Parallel | 3 | 4 | Add | Low | Med |
| A-5 | PlannerAgent Mustache in Qute pipeline — escape hack fragile | Parallel | 3 | 3 | No | Med | Med |
| C-4 | `@ParallelExecutor` can't inject CDI beans | Concur | 3 | 3 | No | Low | Med |
| C-5 | `HumanInTheLoop` blocks Vert.x event loop | Concur | 3 | 3 | No | Med | Med |
| O-4 | No MicroProfile Health probe for agents | Obs | 3 | 2 | No | Low | Med |
| O-5 | `DevAgentMonitorHolder` static state / hot-reload | Obs | 3 | 2 | No | Low | Med |
| O-6 | Dev UI renders raw HTML blobs | Obs | 3 | 3 | No | Low | Med |
| F-1 | `@Retry`/`@CircuitBreaker` untested | FT | 3 | 1 | No | Low | Med |
| F-6 | `@ErrorHandler` static-only, no CDI | FT | 3 | 4 | Yes | Med | Med |
| L-3 | `maxIterations` not config-overridable | Cfg | 3 | 4 | No | Med | Med |
| D-3 | No `ReflectiveClassBuildItem` for native | DevUI | 3 | 2 | No | Med | Med |
| D-5 | `DetectedAiAgentBuildItem` undocumented API | DevUI | 3 | 2 | No | Low | Med |
| D-6 | Security: `@RolesAllowed` untested/undocumented | DevUI | 3 | 2 | No | Low | Med |
| S-2 | `@CdiBean` resolver lacks qualifier support | CDI | 3 | 2 | No | Low | Med |
| L-4 | `AgenticScopeStore` no CDI integration | Cfg | 5 | 4 | No | Low | Med |
| F-7 | `declaredAnnotations()` misses inherited bindings | FT | 2 | 1 | No | Low | Quick win |
| C-6 | Sequential `future.get()` loop suboptimal | Concur | 2 | 1 | No | Low | Quick win |
| C-7 | `loadClassSafe` uses TCCL — fragile | Concur | 2 | 1 | No | Low | Quick win |
| L-5 | Eager init has no config disable | Cfg | 2 | 2 | No | Low | Low |
| S-3 | `markCdiBeanParametersAsUnremovable` too narrow | CDI | 2 | 1 | No | Low | Prereq |
| S-4 | `ChatSupplierParameterResolver` upstream blocker | CDI | 4 | 5 | No | Med | Upstream PR |

†requires O-3 first

---

## Quick Wins (effort 1, low risk — ship in one PR)

1. **F-7** — Change `declaredAnnotations()` to `annotations()` in `hasAnyInterceptorBindings` (one word)
2. **C-6** — Replace sequential `future.get()` loop with `CompletableFuture.allOf()`
3. **C-7** — Replace `Thread.currentThread().getContextClassLoader()` with `AgenticRecorder.class.getClassLoader()`
4. **D-1** — Add `(onlyIf = IsDevelopment.class)` to `jsonRpcProvider` build step (one annotation)
5. **F-5** — Add build-time warning for `@Transactional` + agentic annotation co-occurrence (one `@BuildStep`)
6. **F-1** — Add `@Retry` and `@CircuitBreaker` test cases to `FaultToleranceTest`
7. **S-3** — Expand `markCdiBeanParametersAsUnremovable` loop to cover all supplier types (one loop expansion)
8. **L-1** — Add `@ConfigMapping(prefix = "quarkus.langchain4j.agent")` skeleton

---

## Recommended sequencing

**Phase 1 — Safety (one sprint):**
All quick wins above + D-2 (allow-list check in `invokeAgent`) + F-3 (build-time `@Fallback` validation) + F-4 (build-time `@Retry` warning)

**Phase 2 — Observability (one sprint):**
O-3 → O-1 → O-2 (CDI listener discovery unblocks OTel + metrics in dependency order)
O-4 (health check), O-5 (hot-reload), D-4 (CDI events)

**Phase 3 — Concurrency (one sprint):**
C-1 + C-2 + C-3 (all share the same `ManagedExecutor` fix)
A-6 (Vert.x HTTP for A2A)

**Phase 4 — CDI Integration (one sprint):**
S-1 (CDI auto-wire for other supplier types)
S-2 (qualifier support in `@CdiBean` resolver)
L-2 (A2A URL config)

**Phase 5 — Guardrails (one sprint):**
A-2 — wire core module guardrail infrastructure to agent boundaries

**Phase 6 — Persistent state (two sprints):**
A-7 — `quarkus-langchain4j-agentic-infinispan` + `quarkus-langchain4j-agentic-redis` modules
L-4 — CDI detection of `AgenticScopeStore` beans

**Requires upstream PR to langchain4j-agentic:**
F-2 (scope checkpoint for `@Retry`), S-4 (general `SupplierParameterResolver` SPI),
F-6 (non-static error handlers), A-4 (CDI-based activation conditions),
A-5 (Qute-compatible template syntax in PlannerAgent)
