# C6 — Configurable Agents Design

**Date:** 2026-06-09 (revised 2026-06-09)
**Chapter:** 6 of 8 — Configurable Agents
**Layer:** L5 (Configuration)
**Status:** Approved (revised after review)

---

## Problem

Agent parameters are hardcoded in upstream annotations with no runtime override mechanism:

- `@LoopAgent(maxIterations=5)` — tuning parameter, should vary per environment
- `@SupervisorAgent(maxAgentsInvocations=10)` — same concern
- `@A2AClientAgent(a2aServerUrl="http://...")` — environment-specific, requires recompilation to change
- A2A HTTP transport uses JDK `HttpClient`, not Quarkus-native Vert.x `WebClient`

Upstream code reads annotation values directly and consumes them immediately. The `AgentConfigurator` callback only covers `AgentBuilder` (inner AiService level), not workflow builders (`LoopAgentService`, `SupervisorAgentService`) or A2A builders. There is no interception point for DI frameworks to substitute config-resolved values.

### Pre-existing L-5 fragments

Some L-5 work already landed in earlier chapters:
- `AgenticRuntimeConfig` exists with `defaultMaxIterations` (defined but unwired) and `devUi.eagerInit`
- The `quarkus.langchain4j.agent` prefix is established

C6 delivers the full L-5 layer: per-agent config namespace, A2A URL externalisation, config expression resolution, and Vert.x transport.

## Strategy

Use existing upstream extension points to intercept and override values. File upstream issues first for proper SPIs. Remove workarounds when upstream lands.

Three extension points:
1. `AgenticServices.setWorkflowAgentsBuilder()` — public API, replace with config-aware wrapper (covers loop, sequence, parallel, conditional agents)
2. `A2AService.Provider.a2aService` — package-private, set via reflection (temporary until upstream adds a public setter)
3. `A2AHttpClientProvider` — ServiceLoader SPI with priority ordering, implement Vert.x provider

**Not covered by `WorkflowAgentsBuilder`:** `SupervisorAgentService` is created directly via `new SupervisorAgentServiceImpl<>(...)` in `buildSupervisorAgent()`, not through the builder SPI. Config for supervisor `maxAgentsInvocations` is declared in the config interface but cannot be wired until upstream provides a supervisor builder SPI or the workflow-level `AgentConfigurator`.

## Upstream Issues (filed before implementation)

File on `langchain4j/langchain4j` before C6 implementation begins:

1. **Workflow-level AgentConfigurator** — widen `AgentConfigurator` to fire for all builder types (`LoopAgentService`, `SupervisorAgentService`, `A2AClientBuilder`, etc.), not just `AgentBuilder`. Framework-agnostic — Spring Boot and Micronaut have the same override need.

2. **Public `A2AService.setA2AService()` setter** — matches existing `AgenticServices.setWorkflowAgentsBuilder()` pattern. Trivial change, removes reflection hack.

## Config Structure

Two separate `@ConfigRoot` interfaces — agent runtime config and dev UI config are distinct concerns.

### AgenticRuntimeConfig (agent runtime)

```java
@ConfigRoot(phase = ConfigPhase.RUN_TIME)
@ConfigMapping(prefix = "quarkus.langchain4j.agent")
public interface AgenticRuntimeConfig {

    /** Global default max iterations for loop agents. */
    Optional<Integer> defaultMaxIterations();

    /** Per-agent configuration, keyed by agent name. */
    @WithParentName
    @ConfigDocMapKey("agent-name")
    Map<String, AgentConfig> namedConfig();

    interface AgentConfig {
        /** Override maxIterations for a @LoopAgent. */
        Optional<Integer> maxIterations();

        /**
         * Override maxAgentsInvocations for a @SupervisorAgent.
         * NOTE: declared but not wired in C6 — requires upstream
         * supervisor builder SPI or workflow-level AgentConfigurator.
         */
        Optional<Integer> maxAgentsInvocations();

        /** Override the A2A server URL for an @A2AClientAgent. */
        Optional<String> a2aServerUrl();
    }
}
```

### AgenticDevUiConfig (dev tooling — separate concern)

```java
@ConfigRoot(phase = ConfigPhase.RUN_TIME)
@ConfigMapping(prefix = "quarkus.langchain4j.agent.dev-ui")
public interface AgenticDevUiConfig {

    /**
     * Whether to eagerly initialise root agents at startup in dev mode.
     * Set to false in CI environments to avoid unnecessary agent startup latency.
     */
    @WithDefault("true")
    boolean eagerInit();
}
```

### Usage

```properties
# Global default
quarkus.langchain4j.agent.default-max-iterations=10

# Per-agent overrides (keyed by resolved agent name)
quarkus.langchain4j.agent."story-loop".max-iterations=20
quarkus.langchain4j.agent."remote-writer".a2a-server-url=https://prod.example.com

# Dev UI (separate namespace, no collision)
quarkus.langchain4j.agent.dev-ui.eager-init=false
```

### Config key resolution

The config key is the **resolved agent name** — the `name` attribute from the upstream annotation if set, otherwise the method name kebab-cased (e.g. `generateStory` → `generate-story`). Extracted at build time. Uniqueness validated across all agentic methods across all interfaces in the application — two interfaces with `@LoopAgent(name="story")` on different methods fail validation.

## Component 1 — ConfigAwareWorkflowAgentsBuilder (maxIterations override)

**Purpose:** Override `maxIterations` for loop agents from Quarkus config.

**Registration:** `AgenticRecorder.registerWorkflowAgentsBuilder(Map<String, String> classNameToConfigKey)` at `@RuntimeInit` calls `AgenticServices.setWorkflowAgentsBuilder()`. The `classNameToConfigKey` map is populated at build time from `DetectedAiAgentBuildItem` — no static registry.

**Timing (explicit):**
1. Our `loopBuilder(Class<T> agentServiceClass)` creates a `ConfigAwareLoopBuilder` wrapping the real `LoopAgentService`, storing the class and config reference
2. Upstream's `buildLoopAgent()` calls `.maxIterations(annotation.maxIterations())` on our wrapper → delegated to the real builder (annotation value set)
3. Upstream calls `.build()` on our wrapper → our wrapper calls `delegate.maxIterations(configValue)` to override, then calls `delegate.build()`

The override works because `maxIterations()` is a builder-style setter — calling it again replaces the previous value. The `Class<T> agentServiceClass` parameter received in `loopBuilder()` is the original agent interface class (not a proxy), which is used to look up the config key from the threaded map.

**Config resolution order:**
1. Per-agent named config: `quarkus.langchain4j.agent."story-loop".max-iterations`
2. Global default: `quarkus.langchain4j.agent.default-max-iterations`
3. Annotation value (upstream default)

**Temporary:** Removed when upstream provides workflow-level `AgentConfigurator`. Javadoc and code comments reference the upstream issue number.

```
// TEMPORARY WORKAROUND — will be removed when upstream provides
// workflow-level AgentConfigurator (see langchain4j/langchain4j#NNNN).
```

## Component 2 — ConfigAwareA2AService (a2aServerUrl override + expression resolution)

**Purpose:** Override A2A server URLs from Quarkus config, with config expression support.

**Registration:** `AgenticRecorder.registerConfigAwareA2AService(Map<String, String> classNameToConfigKey)` at `@RuntimeInit`. Grabs the current `A2AService.get()`, wraps it, sets `A2AService.Provider.a2aService` via reflection (field is package-private, no public setter exists). The `classNameToConfigKey` map is the same one passed to the workflow builder.

**URL resolution order (highest to lowest priority):**
1. Per-agent named config: `quarkus.langchain4j.agent."remote-writer".a2a-server-url`
2. Config expression in annotation: `@A2AClientAgent(a2aServerUrl = "${remote.agent.url}")` → resolved via `ConfigProvider.getConfig().getValue("remote.agent.url", String.class)`
3. Raw annotation value: `@A2AClientAgent(a2aServerUrl = "http://localhost:8080")`

Config expression detection: check if the annotation value matches `${...}` pattern, resolve via SmallRye Config. This enables `@A2AClientAgent(a2aServerUrl = "${remote.agent.url}")` with `remote.agent.url=https://prod.example.com` in `application.properties`.

**Temporary:** Reflection replaced by public setter when upstream merges `A2AService.setA2AService()`. Entire wrapper removed when upstream provides workflow-level `AgentConfigurator`. Both reference upstream issue numbers.

## Component 3 — VertxA2AHttpClientProvider (Vert.x HTTP transport) — separate PR

**Independent of Components 1 and 2.** Shares no code, config, or build-time wiring. Delivered as a separate PR within C6 to reduce review blast radius.

**Purpose:** Replace JDK `HttpClient` with Vert.x `WebClient` for A2A HTTP transport.

**Registration:** ServiceLoader via `META-INF/services/org.a2aproject.sdk.client.http.A2AHttpClientProvider`. Native image: `ServiceProviderBuildItem` registered in the deployment processor.

**Implementation:**

`VertxA2AHttpClientProvider`:
- Constructor does nothing (provider is instantiated during `A2AHttpClientFactory` static init, before CDI is available)
- `create()` resolves `Vertx` from `Arc.container().instance(Vertx.class).get()` — called later at agent creation time when CDI IS wired
- `priority()` returns 100 (wins over `JdkA2AHttpClientProvider` at priority 0)

`VertxA2AHttpClient` implements `A2AHttpClient` using Vert.x `WebClient`:
- `GetBuilder` — `webClient.getAbs(url).send()`
- `PostBuilder` — `webClient.postAbs(url).sendJson(body)`, plus `postAsyncSSE()` for SSE streaming
- `DeleteBuilder` — `webClient.deleteAbs(url).send()`

Addresses ARC42STORIES item "A2A calls block Vert.x event loop" — Vert.x WebClient is non-blocking by design.

**Permanent.** Uses the upstream-blessed SPI. No workaround. No temporary markers.

## Component 4 — Build-time Processing

**Agent name extraction (`AgenticProcessor`):** New `@BuildStep` after agent detection. For each `DetectedAiAgentBuildItem`, extract the config key from the root agentic method's annotation `name` attribute. If `name` is empty/default, derive from method name, kebab-cased.

**Validations:**
- **Duplicate config keys** — across ALL agentic methods across ALL interfaces. Two interfaces with methods resolving to the same config key → build failure naming both interfaces and methods
- **Config references non-existent agent** — build-time warning if `quarkus.langchain4j.agent."foo".max-iterations=10` doesn't match any detected agent name

**Data threading:** The `Map<String, String>` of `className → configKey` is passed directly to the recorder methods that create the wrappers. Both `registerWorkflowAgentsBuilder()` and `registerConfigAwareA2AService()` receive the same map. No static registry.

**Native image:** `ServiceProviderBuildItem` for `VertxA2AHttpClientProvider` registered in the deployment processor.

## Permanence Summary

| Component | Permanent? | Removal trigger |
|-----------|-----------|-----------------|
| `AgenticRuntimeConfig` expansion | Yes | — |
| `AgenticDevUiConfig` (separated) | Yes | — |
| `AgentConfig` interface | Yes | — |
| Build-time name extraction + validation | Yes | — |
| `ConfigAwareWorkflowAgentsBuilder` | **Temporary** | Upstream workflow-level AgentConfigurator |
| `ConfigAwareLoopBuilder` | **Temporary** | Upstream workflow-level AgentConfigurator |
| `ConfigAwareA2AService` + reflection | **Temporary** | Upstream `setA2AService()` + AgentConfigurator |
| `VertxA2AHttpClientProvider` | Yes | — |
| `VertxA2AHttpClient` | Yes | — |
| `ServiceProviderBuildItem` registration | Yes | — |

## PR Structure

Two PRs within C6:

1. **Config overrides** — Components 1, 2, 4 + config structure + tests
2. **Vert.x A2A transport** — Component 3 + tests (independent, separate review)

Both PRs reference the upstream issues in their descriptions. PR 1 includes a "Temporary workarounds" section listing each wrapper, the upstream issue it depends on, and the removal path.

## Testing Strategy

1. **Config resolution tests** — verify per-agent config overrides annotation defaults for maxIterations and a2aServerUrl
2. **Default fallback tests** — verify annotation values are used when no config is present
3. **Global default tests** — verify `defaultMaxIterations` applies when no per-agent config exists
4. **Config expression tests** — verify `${...}` patterns in a2aServerUrl annotations resolve from config
5. **Resolution priority tests** — named config > expression > raw annotation value
6. **Build-time validation tests** — duplicate config key detection, unknown config key warning
7. **Vert.x HTTP client tests** — GET/POST/DELETE operations, SSE streaming
8. **Integration test** — full agent with config-overridden maxIterations executes the correct number of iterations
9. **DevUiConfig separation test** — verify dev-ui config works independently at the new prefix
