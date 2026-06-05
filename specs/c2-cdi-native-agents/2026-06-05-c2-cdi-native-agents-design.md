# C2 — CDI-Native Agents Design

**Chapter:** 2 of 8  
**Audit findings:** S-1, S-2, O-3, A-1 (partial — see A-1 Scope below)  
**Branch:** `c2-cdi-native-agents`

---

## Goal

Agent configuration no longer requires static supplier methods. CDI beans of
`ContentRetriever`, `ChatMemory`, `ChatMemoryProvider`, and
`RetrievalAugmentor` are automatically wired into any agent that does not
declare a static supplier for that type. `AgentListener` CDI beans are
auto-discovered and applied to all agents globally. The `@CdiBean` parameter
resolver gains qualifier support.

---

## A-1 Scope

A-1 covers 8 supplier annotations. This chapter addresses:

| Supplier annotation | In scope | Mechanism |
|---------------------|----------|-----------|
| `@ChatModelSupplier` | Existing (C1) | `ChatModelInfo.FromBeanWithName` path — unchanged |
| `@ContentRetrieverSupplier` | Yes | CDI fallback auto-wiring |
| `@ChatMemorySupplier` | Yes | CDI fallback auto-wiring |
| `@ChatMemoryProviderSupplier` | Yes | CDI fallback auto-wiring |
| `@RetrievalAugmentorSupplier` | Yes | CDI fallback auto-wiring |
| `@ToolProviderSupplier` | No | MCP module registers its own `ToolProvider` CDI bean — auto-wiring would double-wire or cause ambiguity (see ToolProvider Decision below) |
| `@AgentListenerSupplier` | Yes | CDI additive auto-discovery (global) |
| `@ToolsSupplier` | Deferred | Returns `Object[]` — requires `@Tool`-annotated bean scanning, different pattern; aligns with core module's `@RegisterAiService` tool discovery |
| `@StreamingChatModelSupplier` | Deferred | Streaming variant of ChatModel; same build-time pattern but needs streaming-specific injection point plumbing |

### ToolProvider Decision

`ToolProvider` is excluded from the CDI fallback enum because:

1. **MCP type collision.** The MCP deployment processor registers its own
   `ToolProvider` CDI bean. This occupies the same CDI type space as any
   user-defined `ToolProvider` bean. CDI fallback auto-wiring would either
   double-wire the MCP bean (if it's the only `ToolProvider`) or cause
   ambiguity (if a user bean also exists).

2. **Append semantics make fallback meaningless.** `AgentBuilder.toolProvider()`
   appends to a list — there's no single-value slot to "fall back" into. The
   fallback concept (wire when no static supplier exists) only makes sense for
   overwrite-semantic setters.

3. **Existing path is sufficient.** MCP ToolProviders already have their own
   wiring path via `hasMcpToolBox` + `Instance<ToolProvider>`. User
   ToolProviders can use `@ToolProviderSupplier` static methods or the
   `@CdiBean` parameter resolver.

---

## Architecture

### Build-time detection, runtime wiring

All detection and validation happens at build time in `AgenticProcessor`.
Runtime wiring in `QuarkusAgenticContextConsumer` is a simple switch over
pre-computed data — no `Arc.container()` calls in the auto-wiring path.
(The S-2 `@CdiBean` parameter resolver is a separate, opt-in path that
does use `Arc.container()` inside the upstream `ChatSupplierParameterResolver`
SPI.)

### Two wiring modes

| Mode | Supplier types | Behavior |
|------|---------------|----------|
| **Fallback** | ContentRetriever, ChatMemory, ChatMemoryProvider, RetrievalAugmentor | CDI bean is wired only when the agent declares no static supplier for that type AND exactly one CDI bean of that type is resolvable with `@Default` qualifier |
| **Additive** | AgentListener | CDI beans are wired into ALL agents, composing with any per-agent `@AgentListenerSupplier` via `ComposedAgentListener` |

### Shared-instance constraint

All agents without a static supplier for a given fallback type receive the
**same CDI bean instance** (unless the bean is `@Dependent`, in which case
each agent receives a separate instance). If two agents need different
`ContentRetriever` instances, one of them must use a static
`@ContentRetrieverSupplier` method (or the `@CdiBean` parameter resolver
with qualifiers once upstream extends the SPI to non-chat suppliers).
Per-agent CDI bean association via qualifiers (e.g.,
`@ForAgent(MyAgent.class)`) is out of scope for C2.

### Upstream coupling

C2 depends on `DeclarativeUtil.configureAgent()` calling the
`AgentConfigurator` consumer **after** all static supplier processing
(line 149 of `DeclarativeUtil.java` in `langchain4j-agentic-1.15.1-beta25`).
If upstream reorders or adds a "set defaults after configurator" step, CDI
beans could silently overwrite static suppliers for overwrite-semantic types
(`contentRetriever`, `chatMemory`, `chatMemoryProvider`, `retrievalAugmentor`).
`listener` is immune — it uses compose semantics.

### Why not a unified wiring descriptor

The spec uses separate concerns in `AiAgentCreateInfo` (`cdiResolvedSuppliers`,
`hasMcpToolBox`, `ChatModelInfo`) rather than one unified descriptor. The three
wiring modes have genuinely different semantics — upstream-controlled (static
suppliers), build-time-detected (CDI fallback), and additive-special-case (MCP).
A unified descriptor would need internal branches for each mode, adding
complexity without simplifying the runtime logic. The flat record with a few
fields is clearer.

---

## Data Model

### `AiAgentCreateInfo` expansion

```java
public record AiAgentCreateInfo(
    String agentClassName,
    ChatModelInfo chatModelInfo,
    boolean hasInterceptorBindings,
    Set<CdiSupplierType> cdiResolvedSuppliers,
    boolean hasMcpToolBox
) { ... }
```

### `CdiSupplierType` enum

```java
/**
 * Supplier types eligible for CDI fallback auto-wiring when no static
 * supplier is declared on the agent interface.
 *
 * <p>All types in this enum have overwrite-semantic setters on
 * {@code AgentBuilder} (simple field assignment). The build-time check
 * ensures CDI beans are only wired when no static supplier exists,
 * preventing silent overwrites.
 *
 * <p>{@code ToolProvider} is excluded — its append semantics and MCP type
 * collision make fallback auto-wiring inappropriate (see spec: ToolProvider
 * Decision).
 *
 * <p>{@code AgentListener} is excluded — it uses a separate additive path
 * ({@code Instance<AgentListener>} injection on every agent).
 *
 * <p>This enum is coupled to {@code langchain4j-agentic}'s supplier
 * annotation set. If upstream adds new supplier annotations, this enum
 * must be updated in both runtime and deployment modules. The set has been
 * stable across several betas; the enum provides exhaustive switch coverage
 * and type safety, which outweighs the maintenance cost of manual updates.
 */
public enum CdiSupplierType {
    CONTENT_RETRIEVER,
    CHAT_MEMORY,
    CHAT_MEMORY_PROVIDER,
    RETRIEVAL_AUGMENTOR
}
```

### Shared mutable state removal

`AgenticRecorder.agentsWithMcpToolBox` (static `Set<String>`) is deleted.
Its data moves into `AiAgentCreateInfo.hasMcpToolBox` — each agent carries
its own flag.

---

## Build-time Detection (`AgenticProcessor`)

### Build step ordering

The `cdiSupport` step already consumes `InterceptorResolverBuildItem`, which
runs after bean discovery. `BeanDiscoveryFinishedBuildItem` is produced
earlier (after bean discovery, before validation), while
`InterceptorResolverBuildItem` is produced during validation. Adding
`BeanDiscoveryFinishedBuildItem` as a parameter does not change the step's
execution ordering — the step already runs after both are available. The step
continues to produce `SyntheticBeanBuildItem`, which is consumed by Arc's
internal synthetic bean registration step that runs later. No existing build
step depends on agent synthetic beans being available before
`BeanDiscoveryFinishedBuildItem`.

### Bean resolution semantics

"Exactly one CDI bean" means: exactly one bean resolvable with the `@Default`
qualifier. This is queried via `BeanDiscoveryFinishedBuildItem.beanStream()`
filtered by bean type and `@Default` qualifier.

Examples:
- Single bean, no explicit qualifiers → gets `@Default` implicitly → auto-wired
- Two `@Named` beans, no `@Default` → no bean with `@Default` → skip
- One `@Default` + one `@Named` → exactly one `@Default` → auto-wire the `@Default` one
- Two beans with `@Default` → multiple → skip

### CDI scope validation

Agents are `@ApplicationScoped` synthetic beans created during `@RuntimeInit` —
no request context is active. Any bean resolved via `getInjectedReference()` at
that point must be available at application scope (`@ApplicationScoped`,
`@Singleton`, or `@Dependent`).

At build time, after determining a bean is eligible for auto-wiring, validate
its scope. If the bean is `@RequestScoped` or `@SessionScoped`:
- Emit a **build-time error**: "CDI bean of type `<Type>` is `@RequestScoped`
  and cannot be auto-wired into agent `<AgentName>`. Agent synthetic beans are
  created at application startup when no request context is active. Use
  `@ApplicationScoped` or provide the bean via a static supplier method."

This applies to all four fallback types AND `AgentListener`.

### Supplier auto-wiring (fallback types)

In the `cdiSupport` build step, for each detected agent:

1. For each supplier type in `CdiSupplierType`:
   - Check if the agent interface (including transitive interfaces per protocol
     `PP-20260604-transitive-interface-traversal`) declares a static method with
     the corresponding supplier annotation
   - If yes → skip (static supplier wins)
   - If no → query `BeanDiscoveryFinishedBuildItem` for CDI beans of the matching
     Java type, resolvable with `@Default` qualifier
     - Exactly one bean →
       - Validate scope (see CDI scope validation above)
       - Add to `cdiResolvedSuppliers`
       - Add `addInjectionPoint()` with `Instance<T>` type to synthetic bean
         configurator
       - Mark bean as unremovable
     - Multiple beans → log INFO: "Multiple `<Type>` CDI beans found but agent
       `<AgentName>` declares no static supplier — auto-wiring skipped. Use a
       static `@<SupplierAnnotation>` method to select explicitly."
     - Zero beans → skip

2. Store `cdiResolvedSuppliers` in `AiAgentCreateInfo`

### AgentListener auto-discovery

1. Add `Instance<AgentListener>` injection point to EVERY agent's synthetic bean
   configurator **unconditionally**. When no `AgentListener` beans exist, the
   `Instance` is empty and the runtime iteration is a no-op.
2. Mark all `AgentListener` CDI beans as unremovable (if any exist)
3. Validate scope of each `AgentListener` bean (see CDI scope validation)

### MCP ToolProvider

`hasMcpToolBox` moves from a static `Set<String>` into `AiAgentCreateInfo`.
The `Instance<ToolProvider>` injection point is already added for every agent
(unchanged from the existing code). At runtime, the MCP-specific ToolProvider
wiring is conditional on `hasMcpToolBox`.

---

## Runtime Wiring (`QuarkusAgenticContextConsumer`)

All injection uses `Instance<T>` with `isResolvable()` checks — consistent with
the existing `ToolProvider` pattern and defensive against bean removal by later
build steps. Each fallback type has a pre-built `TypeLiteral` constant.

```java
private static final TypeLiteral<Instance<ContentRetriever>>
    CONTENT_RETRIEVER_INSTANCE = new TypeLiteral<>() {};
private static final TypeLiteral<Instance<ChatMemory>>
    CHAT_MEMORY_INSTANCE = new TypeLiteral<>() {};
private static final TypeLiteral<Instance<ChatMemoryProvider>>
    CHAT_MEMORY_PROVIDER_INSTANCE = new TypeLiteral<>() {};
private static final TypeLiteral<Instance<RetrievalAugmentor>>
    RETRIEVAL_AUGMENTOR_INSTANCE = new TypeLiteral<>() {};
private static final TypeLiteral<Instance<AgentListener>>
    AGENT_LISTENER_INSTANCE = new TypeLiteral<>() {};

@Override
public void accept(DeclarativeAgentCreationContext agenticContext) {
    AgentBuilder<?, ?> builder = agenticContext.agentBuilder();

    // Fallback CDI suppliers (per-agent, only when no static supplier)
    for (CdiSupplierType type : aiAgentCreateInfo.cdiResolvedSuppliers()) {
        switch (type) {
            case CONTENT_RETRIEVER -> {
                Instance<ContentRetriever> i = cdiContext
                    .getInjectedReference(CONTENT_RETRIEVER_INSTANCE);
                if (i.isResolvable()) builder.contentRetriever(i.get());
            }
            case CHAT_MEMORY -> {
                Instance<ChatMemory> i = cdiContext
                    .getInjectedReference(CHAT_MEMORY_INSTANCE);
                if (i.isResolvable()) builder.chatMemory(i.get());
            }
            case CHAT_MEMORY_PROVIDER -> {
                Instance<ChatMemoryProvider> i = cdiContext
                    .getInjectedReference(CHAT_MEMORY_PROVIDER_INSTANCE);
                if (i.isResolvable()) builder.chatMemoryProvider(i.get());
            }
            case RETRIEVAL_AUGMENTOR -> {
                Instance<RetrievalAugmentor> i = cdiContext
                    .getInjectedReference(RETRIEVAL_AUGMENTOR_INSTANCE);
                if (i.isResolvable()) builder.retrievalAugmentor(i.get());
            }
        }
    }

    // MCP ToolProvider (additive — unchanged from existing code)
    if (aiAgentCreateInfo.hasMcpToolBox()) {
        Instance<ToolProvider> mcpToolProvider =
            cdiContext.getInjectedReference(TOOL_PROVIDER_TYPE_LITERAL);
        if (mcpToolProvider.isResolvable()) {
            builder.toolProvider(mcpToolProvider.get());
        }
    }

    // AgentListener CDI beans (global, additive — always applied)
    Instance<AgentListener> listeners =
        cdiContext.getInjectedReference(AGENT_LISTENER_INSTANCE);
    for (AgentListener listener : listeners) {
        builder.listener(listener);
    }
}
```

No `Arc.container()` calls in the auto-wiring path. All references come from
`SyntheticCreationalContext` injection points declared at build time.

---

## S-2 Fix: `CdiChatSupplierParameterResolver` Qualifier Support

```java
public Object resolve(Context context) {
    Parameter parameter = context.parameter();
    Annotation[] qualifiers = Arrays.stream(parameter.getAnnotations())
        .filter(ann -> !ann.annotationType().equals(CdiBean.class))
        .filter(ann -> ann.annotationType().isAnnotationPresent(
            jakarta.inject.Qualifier.class))
        .toArray(Annotation[]::new);
    return Arc.container().select(parameter.getType(), qualifiers).get();
}
```

This enables `@CdiBean @ModelName("gpt4") ChatModel model` on
`@ChatModelSupplier` parameters. The `@CdiBean` marker is excluded;
CDI qualifier annotations are extracted and passed to `Arc.container().select()`.

This is a runtime `Arc.container()` call, but it's inside the upstream
`ChatSupplierParameterResolver` SPI — opt-in via `@CdiBean`, bounded scope.
Separate from the build-time auto-wiring path.

---

## Files Changed

| File | Change |
|------|--------|
| `agentic/runtime/.../AiAgentCreateInfo.java` | Add `cdiResolvedSuppliers`, `hasMcpToolBox` fields |
| `agentic/runtime/.../CdiSupplierType.java` | New enum |
| `agentic/runtime/.../AgenticRecorder.java` | Remove `agentsWithMcpToolBox` static field; update `QuarkusAgenticContextConsumer` to handle all supplier types and AgentListener via `Instance<T>` with pre-built `TypeLiteral` constants |
| `agentic/runtime/.../CdiChatSupplierParameterResolver.java` | Add qualifier extraction |
| `agentic/deployment/.../AgenticProcessor.java` | Add `BeanDiscoveryFinishedBuildItem` consumption, CDI bean detection, scope validation, injection points, `AiAgentCreateInfo` construction update |
| `agentic/deployment/src/test/java/...` | New test classes (see Test Strategy) |

---

## Test Strategy

### Auto-wiring tests (S-1) — one test class per supplier type

- Agent with NO static supplier + one `@Default` CDI bean → bean is wired
- Agent WITH static supplier + CDI bean → static supplier wins
- Agent with NO static supplier + NO CDI bean → agent works (no injection)
- Multiple CDI beans + no static supplier → no auto-wiring (neither bean wired)

### AgentListener auto-discovery tests (O-3)

- One `@ApplicationScoped AgentListener` → receives lifecycle callbacks
- Multiple `AgentListener` CDI beans → ALL receive callbacks
- CDI listener + `@AgentListenerSupplier` → both fire (additive)
- No `AgentListener` CDI beans → agents work normally

### Qualifier resolution tests (S-2)

- `@CdiBean @ModelName("x")` → correct bean selected
- `@CdiBean` without qualifier + single bean → resolution succeeds
- `@CdiBean` without qualifier + multiple beans → `AmbiguousResolutionException`

### Coexistence and regression tests

- `hasMcpToolBox` migration: existing MCP tests pass after moving from
  static `Set<String>` to `AiAgentCreateInfo`
- ChatModel wiring regression: existing `ChatModelInfo.FromBeanWithName` path
  works with expanded `AiAgentCreateInfo` record
- Mixed supplier modes: agent with `@ChatModelSupplier` (static) +
  no `@ContentRetrieverSupplier` + CDI `ContentRetriever` bean → auto-wire
  fires only for retriever

All tests are `@QuarkusTest` with isolated agent interfaces and CDI beans
(static inner classes or test-package classes).

---

## Protocol Compliance

- **transitive-interface-traversal**: Build-time supplier annotation scanning
  must walk the full interface hierarchy via `ValidationUtil.transitiveInterfaces()`
- **build-time-warning-precision**: INFO log on multiple-bean ambiguity fires
  only when auto-wiring is blocked, not on valid multi-bean configurations
- **upstream-contribution-framing**: No upstream changes required — all work is
  Quarkus-side CDI wiring using existing `AgentBuilder` API

---

## Post-C2 Housekeeping

- **ARC42STORIES.MD line 405** is stale: says
  "`Arc.container().select(AgentListener.class)` in `QuarkusAgenticContextConsumer`"
  — the actual implementation uses build-time injection points +
  `SyntheticCreationalContext.getInjectedReference()`. Update when C2 lands.

---

## PR2 — Generalized ParameterResolver

PR1 (above) ships Quarkus-side auto-wiring with no upstream changes. PR2
files an upstream PR to `langchain4j-agentic` and follows up with a Quarkus
extension once upstream merges. These are independent — PR1 does not depend
on PR2.

### Problem

`ChatSupplierParameterResolver` enables pluggable parameter injection for
`@ChatModelSupplier` methods — but ONLY for chat model suppliers. All other
supplier annotations enforce zero parameters (or a single required parameter
like `Object memoryId` for `@ChatMemoryProviderSupplier`). This means:

- `@ContentRetrieverSupplier` methods can't receive CDI/Spring beans
- `@AgentListenerSupplier` methods can't inject `MeterRegistry` or `Tracer`
- `@ParallelExecutor` methods can't inject `ManagedExecutor`
- Users resort to `Arc.container().select(...)` or
  `applicationContext.getBean(...)` inside static methods

The same friction applies to all 9 zero-parameter supplier types plus
`@ParallelExecutor`.

### Upstream PR (PR2a): Generalized ParameterResolver

**Scope:** Generalize `ChatSupplierParameterResolver` to all supplier
annotations.

**Change:** Introduce a `SupplierParameterResolver` SPI (or rename the
existing `ChatSupplierParameterResolver`) applicable to all annotated static
supplier methods. For each annotation type:

1. Keep required parameters validated (e.g., `Object memoryId` for
   `@ChatMemoryProviderSupplier`)
2. Allow additional parameters resolved by the `SupplierParameterResolver`
   chain
3. Resolver chain is pluggable — frameworks register their own resolvers

**Affected annotations (12):**

| Annotation | Required params | Additional params via resolver |
|---|---|---|
| `@ChatModelSupplier` | (existing — unchanged) | Already supported |
| `@StreamingChatModelSupplier` | (existing — unchanged) | Already supported |
| `@ContentRetrieverSupplier` | None | New |
| `@ChatMemorySupplier` | None | New |
| `@ChatMemoryProviderSupplier` | `Object memoryId` | New (alongside memoryId) |
| `@RetrievalAugmentorSupplier` | None | New |
| `@ToolProviderSupplier` | None | New |
| `@ToolsSupplier` | None | New |
| `@PlannerSupplier` | None | New |
| `@AgentListenerSupplier` | None | New |
| `@McpClientSupplier` | None | New |
| `@ParallelExecutor` | None | New |

**Upstream framing:** "The `ChatSupplierParameterResolver` pattern has proven
valuable for dynamic model selection — DI frameworks and custom resolvers can
inject managed instances into `@ChatModelSupplier` methods. This PR
generalizes the same pattern to all supplier annotations. Spring users get
`@SpringBean`-annotated parameters, Quarkus users get `@CdiBean`, and plain
Java users can register custom resolvers. Required parameters (`memoryId`,
etc.) remain validated. No breaking changes — `ChatSupplierParameterResolver`
becomes a type alias for backward compatibility."

**Backward compatibility:** Existing zero-parameter supplier methods continue
to work unchanged. The parameter validation relaxes from "exactly zero" to
"zero or more, with additional params resolved by the resolver chain."

### Quarkus follow-up (PR2b): CdiParameterResolver

After upstream merges PR2a:

1. Rename `CdiChatSupplierParameterResolver` → `CdiParameterResolver`
2. Register it for all supplier types (not just `@ChatModelSupplier`)
3. `@CdiBean` + qualifiers work on any supplier method:

```java
@ContentRetrieverSupplier
static ContentRetriever retriever(
        @CdiBean @Named("vectorDb") ContentRetriever r) {
    return r;
}
```

4. Per-agent bean selection becomes possible without auto-wiring changes —
   the static method is a one-liner pass-through that declares which
   qualified bean to use

**Interaction with PR1 auto-wiring:** PR1's auto-wiring handles the common
case (one default bean, zero ceremony). PR2b handles the power case
(multiple beans, per-agent selection via qualifiers). They compose — auto-
wiring fires when no static supplier exists; the `@CdiBean` resolver fires
when a static supplier IS present and has `@CdiBean` parameters.

### Future scope (C7): Workflow control annotations

The same ParameterResolver pattern applies to workflow control annotations.
Once PR2a's infrastructure exists in upstream, extending it to these
annotations is a validation-constraint relaxation — not a new SPI:

| Annotation | Required params | Benefit of @CdiBean params |
|---|---|---|
| `@ErrorHandler` | `ErrorContext` | DLQ injection, alerting, retry budget |
| `@ActivationCondition` | `@V` params or `AgenticScope` | Feature-flag-driven routing |
| `@ExitCondition` | `@LoopCounter` | Metric/threshold-driven loop exit |
| `@Output` | (varies) | DI in output assembly |

These would be a separate upstream PR filed during C7 work.

### Related: ExecutorProvider SPI (C3 scope)

Not part of C2, but noted here for completeness.
`DefaultExecutorProvider` is a utility class in `langchain4j-core` with no
SPI — frameworks cannot replace the default parallel executor globally. A
separate upstream PR during C3 would introduce an `AgentExecutorProvider`
interface discoverable via ServiceLoader, allowing Quarkus to register a
`ManagedExecutor`-based implementation and Spring to register a
`TaskExecutor` wrapper. This is independent of PR2.

---

## Out of Scope

- **`@ToolProviderSupplier` CDI auto-wiring** — excluded due to MCP type
  collision and append semantics (see ToolProvider Decision). ToolProviders
  wire via MCP path, static `@ToolProviderSupplier`, or `@CdiBean` parameter
  resolver. PR2b extends `@CdiBean` to `@ToolProviderSupplier` params.
- **`@ToolsSupplier` CDI auto-wiring** — returns `Object[]`, requires scanning
  for `@Tool`-annotated CDI beans. Different pattern from single-typed suppliers.
  The core module's `@RegisterAiService` already does this; aligning the agentic
  module would be a separate effort. (PR2b enables `@CdiBean` on
  `@ToolsSupplier` params as a narrower alternative.)
- **`@StreamingChatModelSupplier` CDI auto-wiring** — streaming variant of
  `ChatModel`. Same build-time pattern but needs streaming-specific injection
  point plumbing. Deferred to when streaming agent support is prioritized.
- **Per-agent CDI bean association via qualifiers** — addressed by PR2b:
  `@CdiBean @Named("x")` on supplier method parameters enables per-agent
  bean selection after upstream merges the generalized ParameterResolver.
- **`leafAgentClassNames` static field cleanup** — C3 (Parallel Safety) concern.
- **Dev-mode monitoring state cleanup** — orthogonal to CDI wiring.
