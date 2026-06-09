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

Use existing upstream extension points to intercept and override values. File upstream issues for proper SPIs **before implementation starts**. Remove the workarounds when upstream lands.

Three extension points:
1. `AgenticServices.setWorkflowAgentsBuilder()` — public API, replace with config-aware wrapper
2. `A2AService.Provider.a2aService` — package-private, set via reflection (temporary until upstream adds a public setter)
3. `A2AHttpClientProvider` — ServiceLoader SPI with priority ordering, implement Vert.x provider

## Config Structure

Two separate `@ConfigRoot` interfaces. `AgenticRuntimeConfig` owns the agent config namespace with per-agent overrides. `DevUiConfig` is extracted to its own `@ConfigRoot` — it serves a different purpose (build-time dev tooling) and coupling it to the agent runtime config namespace would collide with `@WithParentName` on the per-agent map.

### AgenticRuntimeConfig

```java
@ConfigRoot(phase = ConfigPhase.RUN_TIME)
@ConfigMapping(prefix = "quarkus.langchain4j.agent")
public interface AgenticRuntimeConfig {

    Optional<Integer> defaultMaxIterations();

    @WithParentName
    @ConfigDocMapKey("agent-name")
    Map<String, AgentConfig> namedConfig();

    interface AgentConfig {
        Optional<Integer> maxIterations();
        Optional<String> a2aServerUrl();
        Optional<Duration> timeout();
    }
}
```

### AgenticDevUiConfig (separate @ConfigRoot)

```java
@ConfigRoot(phase = ConfigPhase.RUN_TIME)
@ConfigMapping(prefix = "quarkus.langchain4j.agent.dev-ui")
public interface AgenticDevUiConfig {

    @WithDefault("true")
    boolean eagerInit();
}
```

**Why separate:** `@WithParentName` on the per-agent map flattens entries directly under `quarkus.langchain4j.agent`. If `DevUiConfig` were a subgroup of the same `@ConfigRoot`, SmallRye Config would interpret `dev-ui` as an agent name. Moving it to its own `@ConfigRoot` with an explicit `prefix` eliminates the collision.

**L-5 (eager init config gate):** Already resolved by the existing `DevUiConfig.eagerInit()`. No additional config work needed — the `@ConfigRoot` extraction preserves this property.

### AgentConfig scope

`AgentConfig` includes `timeout()` alongside `maxIterations()` and `a2aServerUrl()`. These cover the configurable properties needed across agent types: `@LoopAgent`, `@PlannerBased`, and `@SupervisorAgent` all use `maxIterations`; `@A2AClientAgent` uses `a2aServerUrl`; `timeout` is flagged by audit L-1. The interface will be extended if C7/C8 surface additional properties, but the known set is covered now.

### Config key resolution

The config key is **per agentic method**, not per interface. An agent interface can have multiple agentic methods with different annotations and different names:

```java
interface MyAgent {
    @LoopAgent(name = "inner-loop", maxIterations = 5)
    AgenticScope loopStep(AgenticScope scope);

    @A2AClientAgent(name = "remote", a2aServerUrl = "http://...")
    AgenticScope remoteStep(AgenticScope scope);
}
```

The config key is: the `name` attribute from the upstream annotation if set, otherwise the method name kebab-cased (e.g. `generateStory` → `generate-story`).

Usage:
```properties
quarkus.langchain4j.agent.default-max-iterations=10
quarkus.langchain4j.agent."inner-loop".max-iterations=20
quarkus.langchain4j.agent."remote".a2a-server-url=https://prod.example.com
```

## Component 1 — ConfigAwareWorkflowAgentsBuilder

**Purpose:** Override `maxIterations` for loop agents from Quarkus config.

**Registration:** `AgenticRecorder.registerWorkflowAgentsBuilder()` at `@RuntimeInit` calls `AgenticServices.setWorkflowAgentsBuilder()`.

**Config key threading:** The recorder creates the wrapper with access to `AgenticRuntimeConfig` and a `Map<String, String>` of `methodSignature → configKey`, populated at build time from `DetectedAiAgentBuildItem` annotation scanning. This avoids the need for a static registry — the wrapper has everything it needs at construction time.

**Behaviour:** Wraps the default `WorkflowAgentsBuilderImpl`. All builder factory methods delegate to the real implementation. `loopBuilder(Class<T>)` wraps the returned `LoopAgentService` in a `ConfigAwareLoopBuilder` that:

1. Receives the `Class<T>` argument from the `loopBuilder(Class<T>)` call
2. Uses it to look up the config key from the method-to-key map
3. On `build()`, calls `loopService.maxIterations(configValue)` **after** the annotation-derived value is already set (the constructor's `configureLoop()` runs first, then the wrapper's `build()` override replaces `maxIterations` with the config value)

The override timing matters: `LoopAgentServiceImpl`'s constructor reads `@LoopAgent.maxIterations()` via `configureLoop()`. The wrapper calls `maxIterations(int)` — a builder-style setter that replaces the field — before delegating to the real `build()`. Precedence: named config > `defaultMaxIterations` > annotation value.

**Temporary:** Removed when upstream provides workflow-level `AgentConfigurator`.

## Component 2 — ConfigAwareA2AService

**Purpose:** Override A2A server URLs from Quarkus config, including config expression resolution in annotation values.

**Registration:** `AgenticRecorder.registerConfigAwareA2AService()` at `@RuntimeInit`. Grabs the current `A2AService.get()`, wraps it, and sets `A2AService.Provider.a2aService` via reflection (field is package-private, no public setter exists).

**Behaviour:** Wraps `DefaultA2AService`. Intercepts `a2aBuilder(url, class)`:

1. Resolves the config key for the matching agent method from the method-to-key map
2. If `AgenticRuntimeConfig.namedConfig()` has an entry with `a2aServerUrl` present → use that URL
3. Otherwise, check if the annotation-provided URL contains `${...}` config expressions → resolve via `ConfigProvider.getConfig().getValue(...)`
4. Otherwise, use the annotation-provided URL as-is

Config expression resolution in annotations is a usability win: users can write `@A2AClientAgent(a2aServerUrl = "${remote.agent.url}")` and the value resolves at runtime. Java annotations only accept compile-time constant strings, but `"${remote.agent.url}"` is a valid string literal — the `ConfigAwareA2AService` detects the `${...}` pattern and resolves it.

**Temporary:** Reflection replaced by public setter when upstream adds `A2AService.setA2AService()`. Entire wrapper removed when upstream provides workflow-level `AgentConfigurator`.

## Component 3 — VertxA2AHttpClientProvider

**Scope note:** This component is functionally independent of Components 1 and 2. It shares no code, no config, and no build-time wiring with the config override work. It replaces the HTTP transport regardless of whether config overrides are active. Consider delivering as a separate PR within C6 to reduce review blast radius.

**Purpose:** Replace JDK `HttpClient` with Vert.x `WebClient` for A2A HTTP transport.

**Registration:** ServiceLoader via `META-INF/services/org.a2aproject.sdk.client.http.A2AHttpClientProvider`.

**Implementation:** `VertxA2AHttpClientProvider` returns priority 100 (wins over `JdkA2AHttpClientProvider` at priority 0). Creates `VertxA2AHttpClient` using the CDI-managed `Vertx` instance.

**CDI timing:** `A2AHttpClientProvider` is loaded via ServiceLoader, which runs outside CDI context. `A2AHttpClientFactory.create()` may be called during static initialization. The provider must **defer `Vertx` resolution to first use**, not construction:

```java
public class VertxA2AHttpClientProvider implements A2AHttpClientProvider {
    @Override
    public A2AHttpClient create() {
        // Vertx resolved lazily on first createGet/createPost/createDelete call
        return new VertxA2AHttpClient();
    }
    // ...
}
```

`VertxA2AHttpClient` resolves `Vertx` via `Arc.container().instance(Vertx.class).get()` on first HTTP operation, not in its constructor.

`VertxA2AHttpClient` implements the `A2AHttpClient` interface:
- `GetBuilder` — `webClient.getAbs(url).send()`
- `PostBuilder` — `webClient.postAbs(url).sendJson(body)`, plus `postAsyncSSE()` for SSE streaming
- `DeleteBuilder` — `webClient.deleteAbs(url).send()`

**Native image:** `AgenticProcessor` must produce a build item to register the `META-INF/services/org.a2aproject.sdk.client.http.A2AHttpClientProvider` file for GraalVM service provider discovery. Use `ServiceProviderBuildItem` or `NativeImageResourceBuildItem`.

**Permanent:** Uses the upstream-blessed SPI. No workaround.

## Component 4 — Build-time Processing and Agent Name Registry

**Build-time (`AgenticProcessor`):** New `@BuildStep` after agent detection. For each `DetectedAiAgentBuildItem`, iterate agentic methods and extract the config key from each method's annotation `name` attribute (or fall back to method name kebab-cased).

**Validations:**
- Duplicate config keys across agentic methods across all interfaces → build failure naming both methods
- Config references non-existent agent name → build-time warning

**Runtime wiring:** The method-to-config-key map is passed to the recorder, which threads it into `ConfigAwareWorkflowAgentsBuilder` and `ConfigAwareA2AService` at construction time. No static registry class needed — the wrappers receive the map directly.

## Upstream Issues

File on `langchain4j/langchain4j` **before implementation starts**:

1. **Workflow-level AgentConfigurator** — widen `AgentConfigurator` to fire for all builder types (`LoopAgentService`, `A2AClientBuilder`, etc.), not just `AgentBuilder`. Framework-agnostic — Spring Boot and Micronaut have the same override need.

2. **Public `A2AService.setA2AService()` setter** — matches existing `AgenticServices.setWorkflowAgentsBuilder()` pattern. Trivial change.

## Permanence Summary

| Component | Permanent? | Removal trigger |
|-----------|-----------|-----------------|
| `AgenticRuntimeConfig` expansion | Yes | — |
| `AgenticDevUiConfig` (separate ConfigRoot) | Yes | — |
| `AgentConfig` interface | Yes | — |
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
4. **Config expression tests** — verify `${...}` expressions in `@A2AClientAgent(a2aServerUrl=...)` resolve at runtime
5. **Build-time validation tests** — duplicate config key detection, unknown config key warning
6. **Vert.x HTTP client tests** — GET/POST/DELETE operations, SSE streaming
7. **Vert.x lazy init test** — verify `VertxA2AHttpClient` works when ServiceLoader fires before CDI container is ready
8. **Integration test** — full agent with config-overridden maxIterations executes the correct number of iterations
