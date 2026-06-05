# C2 — CDI-Native Agents Design

**Chapter:** 2 of 8  
**Audit findings:** S-1, S-2, O-3, A-1 (partial)  
**Branch:** `c2-cdi-native-agents`

---

## Goal

Agent configuration no longer requires static supplier methods. CDI beans of
`ContentRetriever`, `ChatMemory`, `ChatMemoryProvider`, `RetrievalAugmentor`,
and `ToolProvider` are automatically wired into any agent that does not declare
a static supplier for that type. `AgentListener` CDI beans are auto-discovered
and applied to all agents globally. The `@CdiBean` parameter resolver gains
qualifier support.

---

## Architecture

### Build-time detection, runtime wiring

All detection and validation happens at build time in `AgenticProcessor`.
Runtime wiring in `QuarkusAgenticContextConsumer` is a simple switch over
pre-computed data — no `Arc.container()` calls, no reflection, no bean
discovery at runtime.

### Two wiring modes

| Mode | Supplier types | Behavior |
|------|---------------|----------|
| **Fallback** | ContentRetriever, ChatMemory, ChatMemoryProvider, RetrievalAugmentor, ToolProvider | CDI bean is wired only when the agent declares no static supplier for that type AND exactly one CDI bean of that type exists |
| **Additive** | AgentListener | CDI beans are wired into ALL agents, composing with any per-agent `@AgentListenerSupplier` via `ComposedAgentListener` |

### Interaction with upstream

The upstream `DeclarativeUtil.configureAgent()` processes static suppliers
first, then calls the Quarkus configurator consumer (`QuarkusAgenticContextConsumer`)
last. For fallback types, the consumer only fires for types in `cdiResolvedSuppliers`
— which excludes types where a static supplier exists. No conflict possible.

For AgentListener, `AgentBuilder.listener()` composes via `ComposedAgentListener`,
so CDI listeners add alongside any `@AgentListenerSupplier`.

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
public enum CdiSupplierType {
    CONTENT_RETRIEVER,
    CHAT_MEMORY,
    CHAT_MEMORY_PROVIDER,
    RETRIEVAL_AUGMENTOR,
    TOOL_PROVIDER
}
```

AgentListener is not in this enum — it uses a separate path (always additive,
`Instance<AgentListener>` injection point on every agent).

### Shared mutable state removal

`AgenticRecorder.agentsWithMcpToolBox` (static `Set<String>`) is deleted.
Its data moves into `AiAgentCreateInfo.hasMcpToolBox` — each agent carries
its own flag.

---

## Build-time Detection (`AgenticProcessor`)

### Supplier auto-wiring (fallback types)

In the `cdiSupport` build step, for each detected agent:

1. For each supplier type in `CdiSupplierType`:
   - Check if the agent interface (including transitive interfaces per protocol
     `PP-20260604-transitive-interface-traversal`) declares a static method with
     the corresponding supplier annotation
   - If yes → skip (static supplier wins)
   - If no → query `BeanDiscoveryFinishedBuildItem` for CDI beans of the matching
     Java type
     - Exactly one bean → add to `cdiResolvedSuppliers`, add `addInjectionPoint()`
       to synthetic bean configurator, mark bean as unremovable
     - Multiple beans → skip (no auto-wire, no error)
     - Zero beans → skip

2. Store `cdiResolvedSuppliers` in `AiAgentCreateInfo`

### AgentListener auto-discovery

1. Query `BeanDiscoveryFinishedBuildItem` for `AgentListener` CDI beans
2. If any exist:
   - Add `Instance<AgentListener>` injection point to EVERY agent's synthetic
     bean configurator (same pattern as existing `Instance<ToolProvider>`)
   - Mark all `AgentListener` beans as unremovable
3. The `Instance<AgentListener>` injection point is sufficient — at runtime,
   if no `AgentListener` beans exist, the `Instance` is empty and the loop
   is a no-op. No separate flag needed.

### MCP ToolProvider (generalized)

`hasMcpToolBox` moves from a static `Set<String>` into `AiAgentCreateInfo`.
The `Instance<ToolProvider>` injection point is already added for every agent.
At runtime, the MCP-specific ToolProvider wiring is conditional on `hasMcpToolBox`.

---

## Runtime Wiring (`QuarkusAgenticContextConsumer`)

```java
@Override
public void accept(DeclarativeAgentCreationContext agenticContext) {
    AgentBuilder<?, ?> builder = agenticContext.agentBuilder();

    // Fallback CDI suppliers (per-agent, only when no static supplier)
    for (CdiSupplierType type : aiAgentCreateInfo.cdiResolvedSuppliers()) {
        switch (type) {
            case CONTENT_RETRIEVER -> builder.contentRetriever(
                cdiContext.getInjectedReference(ContentRetriever.class));
            case CHAT_MEMORY -> builder.chatMemory(
                cdiContext.getInjectedReference(ChatMemory.class));
            case CHAT_MEMORY_PROVIDER -> builder.chatMemoryProvider(
                cdiContext.getInjectedReference(ChatMemoryProvider.class));
            case RETRIEVAL_AUGMENTOR -> builder.retrievalAugmentor(
                cdiContext.getInjectedReference(RetrievalAugmentor.class));
            case TOOL_PROVIDER -> builder.toolProvider(
                cdiContext.getInjectedReference(ToolProvider.class));
        }
    }

    // MCP ToolProvider (additive — kept alongside any CDI ToolProvider)
    if (aiAgentCreateInfo.hasMcpToolBox()) {
        Instance<ToolProvider> mcpToolProvider =
            cdiContext.getInjectedReference(TOOL_PROVIDER_TYPE_LITERAL);
        if (mcpToolProvider.isResolvable()) {
            builder.toolProvider(mcpToolProvider.get());
        }
    }

    // AgentListener CDI beans (global, additive — always applied)
    Instance<AgentListener> listeners =
        cdiContext.getInjectedReference(AGENT_LISTENER_TYPE_LITERAL);
    for (AgentListener listener : listeners) {
        builder.listener(listener);
    }
}
```

No `Arc.container()` calls. All references come from `SyntheticCreationalContext`
injection points declared at build time.

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
| `agentic/runtime/.../AgenticRecorder.java` | Remove `agentsWithMcpToolBox` static field; update `QuarkusAgenticContextConsumer` to handle all supplier types and AgentListener |
| `agentic/runtime/.../CdiChatSupplierParameterResolver.java` | Add qualifier extraction |
| `agentic/deployment/.../AgenticProcessor.java` | Add CDI bean detection in `cdiSupport`, add injection points, update `AiAgentCreateInfo` construction |
| `agentic/deployment/src/test/java/...` | New test classes (see Test Strategy) |

---

## Test Strategy

### Auto-wiring tests (S-1) — one test class per supplier type

- Agent with NO static supplier + one CDI bean → bean is wired
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

All tests are `@QuarkusTest` with isolated agent interfaces and CDI beans
(static inner classes or test-package classes).

---

## Protocol Compliance

- **transitive-interface-traversal**: Build-time supplier annotation scanning
  must walk the full interface hierarchy via `ValidationUtil.transitiveInterfaces()`
- **build-time-warning-precision**: No new warnings introduced in C2
- **upstream-contribution-framing**: No upstream changes required — all work is
  Quarkus-side CDI wiring using existing `AgentBuilder` API

---

## Out of Scope

- Per-agent CDI bean association via qualifiers (e.g., `@ForAgent(MyAgent.class)`) —
  not needed until multi-agent systems with per-agent retrievers are common
- `leafAgentClassNames` static field cleanup — C3 (Parallel Safety) concern
- Dev-mode monitoring state cleanup — orthogonal to CDI wiring
