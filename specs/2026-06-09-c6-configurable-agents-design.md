# C6 — Configurable Agents Design

**Date:** 2026-06-09
**Chapter:** 6 of 8 — Configurable Agents
**Layer:** L5 (Configuration)
**Status:** Approved

---

## Problem

Agent parameters are hardcoded in upstream annotations with no runtime override mechanism:

- `@LoopAgent(maxIterations=5)` — tuning parameter, should vary per environment
- `@A2AClientAgent(a2aServerUrl="http://...")` — environment-specific, requires recompilation to change
- A2A HTTP transport uses JDK `HttpClient`, not Quarkus-native Vert.x `WebClient`

Upstream code reads annotation values directly and consumes them immediately. The `AgentConfigurator` callback only covers `AgentBuilder` (inner AiService level), not workflow builders (`LoopAgentService`) or A2A builders. There is no interception point for DI frameworks to substitute config-resolved values.

## Strategy

Use existing upstream extension points to intercept and override values. File upstream issues for proper SPIs. Remove the workarounds when upstream lands.

Three extension points:
1. `AgenticServices.setWorkflowAgentsBuilder()` — public API, replace with config-aware wrapper
2. `A2AService.Provider.a2aService` — package-private, set via reflection (temporary until upstream adds a public setter)
3. `A2AHttpClientProvider` — ServiceLoader SPI with priority ordering, implement Vert.x provider

## Config Structure

Expand `AgenticRuntimeConfig` using the standard Quarkus per-named-bean pattern (`@WithParentName` + `Map<String, ConfigGroup>` + `@ConfigDocMapKey`).

```java
@ConfigRoot(phase = ConfigPhase.RUN_TIME)
@ConfigMapping(prefix = "quarkus.langchain4j.agent")
public interface AgenticRuntimeConfig {

    Optional<Integer> defaultMaxIterations();

    @WithParentName
    @ConfigDocMapKey("agent-name")
    Map<String, AgentConfig> namedConfig();

    DevUiConfig devUi();

    interface AgentConfig {
        Optional<Integer> maxIterations();
        Optional<String> a2aServerUrl();
    }

    interface DevUiConfig {
        @WithDefault("true")
        boolean eagerInit();
    }
}
```

Usage:
```properties
quarkus.langchain4j.agent.default-max-iterations=10
quarkus.langchain4j.agent."story-loop".max-iterations=20
quarkus.langchain4j.agent."remote-writer".a2a-server-url=https://prod.example.com
```

**Config key:** The resolved agent name — the `name` attribute from the upstream annotation if set, otherwise the method name kebab-cased (e.g. `generateStory` → `generate-story`). Extracted at build time, validated for uniqueness.

## Component 1 — ConfigAwareWorkflowAgentsBuilder

**Purpose:** Override `maxIterations` for loop agents from Quarkus config.

**Registration:** `AgenticRecorder.registerWorkflowAgentsBuilder()` at `@RuntimeInit` calls `AgenticServices.setWorkflowAgentsBuilder()`.

**Behaviour:** Wraps the default `WorkflowAgentsBuilderImpl`. All builder factory methods delegate to the real implementation. `loopBuilder()` wraps the returned `LoopAgentService` in a `ConfigAwareLoopBuilder` that, on `build()`, resolves `maxIterations` from `AgenticRuntimeConfig.namedConfig()` (falling back to `defaultMaxIterations()`, then the annotation value).

**Temporary:** Removed when upstream provides workflow-level `AgentConfigurator`.

## Component 2 — ConfigAwareA2AService

**Purpose:** Override A2A server URLs from Quarkus config.

**Registration:** `AgenticRecorder.registerConfigAwareA2AService()` at `@RuntimeInit`. Grabs the current `A2AService.get()`, wraps it, and sets `A2AService.Provider.a2aService` via reflection (field is package-private, no public setter exists).

**Behaviour:** Wraps `DefaultA2AService`. Intercepts `a2aBuilder(url, class)` — resolves the URL from `AgenticRuntimeConfig.namedConfig()` for the matching agent, falls back to the annotation-provided URL.

**Temporary:** Reflection replaced by public setter when upstream adds `A2AService.setA2AService()`. Entire wrapper removed when upstream provides workflow-level `AgentConfigurator`.

## Component 3 — VertxA2AHttpClientProvider

**Purpose:** Replace JDK `HttpClient` with Vert.x `WebClient` for A2A HTTP transport.

**Registration:** ServiceLoader via `META-INF/services/org.a2aproject.sdk.client.http.A2AHttpClientProvider`.

**Implementation:** `VertxA2AHttpClientProvider` returns priority 100 (wins over `JdkA2AHttpClientProvider` at priority 0). Creates `VertxA2AHttpClient` using the CDI-managed `Vertx` instance.

`VertxA2AHttpClient` implements the `A2AHttpClient` interface:
- `GetBuilder` — `webClient.getAbs(url).send()`
- `PostBuilder` — `webClient.postAbs(url).sendJson(body)`, plus `postAsyncSSE()` for SSE streaming
- `DeleteBuilder` — `webClient.deleteAbs(url).send()`

**Permanent:** Uses the upstream-blessed SPI. No workaround.

## Component 4 — Build-time Processing and Agent Name Registry

**Build-time (`AgenticProcessor`):** New `@BuildStep` after agent detection. For each `DetectedAiAgentBuildItem`, extract the config key from the root method's annotation `name` attribute (or fall back to method name).

**Validations:**
- Duplicate config keys across agents → build failure with both interface names
- Config references non-existent agent name → build-time warning

**Runtime (`AgentConfigKeyRegistry`):** Static holder populated by the recorder. Maps agent class name → config key. Used by `ConfigAwareWorkflowAgentsBuilder` and `ConfigAwareA2AService` to resolve which `AgentConfig` entry applies.

```java
public final class AgentConfigKeyRegistry {
    private static volatile Map<String, String> classNameToConfigKey = Map.of();

    public static Optional<String> configKeyFor(Class<?> agentClass) {
        return Optional.ofNullable(classNameToConfigKey.get(agentClass.getName()));
    }
}
```

## Upstream Issues

File on `langchain4j/langchain4j`:

1. **Workflow-level AgentConfigurator** — widen `AgentConfigurator` to fire for all builder types (`LoopAgentService`, `A2AClientBuilder`, etc.), not just `AgentBuilder`. Framework-agnostic — Spring Boot and Micronaut have the same override need.

2. **Public `A2AService.setA2AService()` setter** — matches existing `AgenticServices.setWorkflowAgentsBuilder()` pattern. Trivial change.

## Permanence Summary

| Component | Permanent? | Removal trigger |
|-----------|-----------|-----------------|
| `AgenticRuntimeConfig` expansion | Yes | — |
| `AgentConfig` interface | Yes | — |
| `AgentConfigKeyRegistry` | Yes | — |
| Build-time name extraction + validation | Yes | — |
| `ConfigAwareWorkflowAgentsBuilder` | **Temporary** | Upstream workflow-level AgentConfigurator |
| `ConfigAwareLoopBuilder` | **Temporary** | Upstream workflow-level AgentConfigurator |
| `ConfigAwareA2AService` | **Temporary** | Upstream workflow-level AgentConfigurator |
| Reflection on `A2AService.Provider` | **Temporary** | Upstream `setA2AService()` setter |
| `VertxA2AHttpClientProvider` | Yes | — |
| `VertxA2AHttpClient` | Yes | — |

## Testing Strategy

1. **Config resolution tests** — verify per-agent config overrides annotation defaults for maxIterations and a2aServerUrl
2. **Default fallback tests** — verify annotation values are used when no config is present
3. **Global default tests** — verify `defaultMaxIterations` applies when no per-agent config exists
4. **Build-time validation tests** — duplicate config key detection, unknown config key warning
5. **Vert.x HTTP client tests** — GET/POST/DELETE operations, SSE streaming
6. **Integration test** — full agent with config-overridden maxIterations executes the correct number of iterations
