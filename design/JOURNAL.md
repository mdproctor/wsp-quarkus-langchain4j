# Design Journal — c4-observable-agents

## 2026-06-07 — Architecture and implementation

### Key decisions

**AdditionalBeanBuildItem over SyntheticBeanBuildItem** — all listener beans need CDI injection (`Tracer`, `MeterRegistry`, `Event<>`). Synthetic beans don't support injection. Confirmed by Quarkus extension guide research: `AdditionalBeanBuildItem` for regular CDI classes, `SyntheticBeanBuildItem` for data-carrying beans initialized via recorder.

**Separate AgenticObservabilityProcessor** — `AgenticProcessor` is 1100+ lines. Quarkus guide says build step classes have no shared state and should be separated by concern. Follows the core `ListenersProcessor` precedent.

**Tool spans excluded from AgentSpanListener** — core `ToolSpanWrapper` already creates `langchain4j.tools.*` spans. Adding tool spans from `AgentListener.beforeAgentToolExecution`/`afterAgentToolExecution` would produce duplicates. Agent spans and tool spans nest correctly via OTel context propagation (C3's `ManagedExecutor`).

**Synchronous CDI events (`fire()`, not `fireAsync()`)** — garden entry GE-20260523-bd68ba documents that `@ObservesAsync` silently loses OTel span context. Synchronous delivery preserves the full CDI/OTel/security context.

**HealthBuildItem pattern (not AdditionalBeanBuildItem for health)** — MCP module uses `HealthBuildItem` from `quarkus-smallrye-health-deployment-spi`. This integrates with Quarkus health infrastructure correctly, including readiness classification.

### Deviations from plan during implementation

- `EventCapture` test bean needed getter methods instead of direct field access — CDI client proxies don't delegate field access
- Tests required extending `OpenAiBaseTest` — build step unconditionally requests a default ChatModel bean
- `AgentHealthCheck` initially omitted `@Readiness` to avoid CDI qualifier mismatch on `@Inject` — fixed in review by using `Arc.container().select()` with `Readiness.Literal.INSTANCE` in the test
- `AgentInstance.subagents()` (lowercase a) — plan had `subAgents()`
