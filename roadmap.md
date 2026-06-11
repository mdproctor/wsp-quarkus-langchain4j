# langchain4j Ecosystem Convergence Roadmap

**Date:** 2026-06-11

Three Java projects integrate langchain4j with different framework targets: **langchain4j** (core, framework-agnostic), **langchain4j-cdi** (portable CDI for Jakarta EE servers), and **quarkus-langchain4j** (Quarkus-native with build-time processing). All three wrap the same upstream runtime. Today they diverge in annotations, wiring behaviour, and configuration — unnecessarily. Users who learn one project's API hit friction when touching another. Contributors who add a feature in one project must reinvent it in the others.

This roadmap identifies upstream changes — framework-agnostic by design — that let all three projects share annotations, SPIs, and contracts. The goal is one developer mental model across the ecosystem, with each framework delivering the best integration its platform allows.

---

## Where We Diverge Today

### Annotations

langchain4j-cdi maintains 11 agent topology annotations (`@RegisterSimpleAgent`, `@RegisterSequenceAgent`, `@RegisterLoopAgent`, etc.) because upstream's annotations lack attributes for scope, guardrails, and listener attachment. quarkus-langchain4j uses upstream's annotations directly (`@Agent`, `@SequenceAgentService`, `@LoopAgentService`) and adds framework-specific wiring at build time.

**Root cause:** upstream annotations are CDI-unaware. langchain4j-cdi's response was to fork the annotations. quarkus-langchain4j's was to add a build-time layer. Both are workarounds for the same gap.

### Component Resolution

Three different ways to say "use this bean":

| Project | Pattern | Example |
|---------|---------|---------|
| langchain4j | Static supplier methods | `@ChatModelSupplier static ChatModel model() { ... }` |
| langchain4j-cdi | Named string references | `chatModelName = "my-model"` |
| quarkus-langchain4j | Supplier class + sentinel markers | `chatLanguageModelSupplier = BeanChatLanguageModelSupplier.class` |

**Root cause:** upstream started with static methods (no DI). Each integration worked around it differently. The `SupplierParameterResolver` SPI (#5394, merged) is the seed of convergence but only covers supplier parameters, not all component types.

### Guardrails

langchain4j-cdi has guardrails on `@RegisterAIService` and `@RegisterSimpleAgent`. quarkus-langchain4j has guardrails on agent interfaces via separate annotations (`@AgentInputGuardrails`/`@AgentOutputGuardrails`). Upstream's annotations have no guardrail attributes at all — despite the guardrail interfaces existing in upstream's runtime.

### Observability

langchain4j-cdi attaches listeners via `agentListenerName = "..."` (string-based bean lookup). quarkus-langchain4j auto-discovers `AgentListener` beans at build time. Upstream's annotations have no listener attribute.

### Scope

langchain4j-cdi exposes `scope = RequestScoped.class` on every annotation. quarkus-langchain4j generates `@ApplicationScoped` synthetic beans with no override mechanism on the annotation. Upstream's annotations have no scope concept.

### Configuration

`quarkus.langchain4j.agent.myAgent.maxIterations` vs `dev.langchain4j.cdi.plugin.myAgent.config.maxIterations`. Different namespaces, different property names, different override mechanisms for the same underlying settings.

---

## Convergence Roadmap

Each item is a framework-agnostic upstream proposal. The "Enables" column shows what both CDI integrations gain if upstream adopts it.

| # | Upstream Proposal | Status | Enables | Priority | Scale | Complexity |
|---|-------------------|--------|---------|----------|-------|------------|
| 1 | **Annotation extension points** — add `scope`, `inputGuardrails`, `outputGuardrails`, `agentListener` to `@Agent` annotations | New | Both integrations use upstream annotations instead of forking. One annotation set across the ecosystem. | High | M | Med |
| 2 | **Component resolution SPI** — generalise `SupplierParameterResolver` into a full `ComponentResolver` covering all component types | Seed merged (#5394) | All component types resolved via DI — not just supplier params. Eliminates supplier markers and string-name workarounds. | High | M | Med |
| 3 | **Guardrail attributes on AI service annotations** — add `inputGuardrails()`/`outputGuardrails()` as `Class<?>[]` | New | Consistent guardrail declaration on both agents and AI services across all frameworks. | High | S | Low |
| 4 | **AgentListener attribute** — add `Class<? extends AgentListener> agentListener()` to `@Agent` | New | Standard listener attachment point. Frameworks resolve via DI; auto-discovery remains a framework enhancement. | Med | S | Low |
| 5 | **Config property glossary** — document canonical property names for overridable attributes (`maxIterations`, `maxAgentInvocations`, etc.) | New | Both integrations use the same leaf property names. Only the framework prefix differs. | Med | S | Low |
| 6 | **`@ParallelExecutor` DI params** | Filed (#5378) | CDI-injected parameters on parallel executors. | Med | S | Low |
| 7 | **Widen `AgentConfigurator` to workflow builders** | Filed (#5399) | Config overlay on loops, supervisors, A2A — not just agents. | Med | S | Low |
| 8 | **`A2AService.setA2AService()` setter** | Filed (#5400) | CDI lifecycle integration for A2A services. | Med | S | Low |

### Item 1 — Annotation Extension Points

The central proposal. Add optional attributes to upstream's `@Agent`, `@SequenceAgentService`, `@LoopAgentService`, etc.:

```java
@Agent(
    name = "myAgent",
    scope = RequestScoped.class,
    inputGuardrails = {RateLimitGuardrail.class, ContentFilterGuardrail.class},
    outputGuardrails = {PiiRedactionGuardrail.class},
    agentListener = MetricsAgentListener.class
)
```

**No CDI imports in upstream.** `scope` is `Class<? extends Annotation>` (from `java.lang.annotation`) defaulting to `Annotation.class` ("let the framework decide"). Guardrails and listener are `Class<?>` arrays. Upstream stores class references; framework integrations read and resolve them via their DI container. This is the same pattern as `tools = {MyTool.class}` — which already works this way.

This eliminates the reason langchain4j-cdi forked the annotations. If upstream's annotations carry scope, guardrails, and listener attributes, maintaining 11 separate annotations becomes unnecessary.

### Item 2 — Component Resolution SPI

`SupplierParameterResolver` (#5394, merged) proved the pattern: upstream defines an SPI, frameworks register resolvers. Generalise this to cover all component types:

```java
public interface ComponentResolver {
    <T> T resolve(Class<T> type, String name);
    boolean canResolve(Class<?> type);
}
```

Upstream's default: `ServiceLoader` + no-arg constructor. CDI integration: `CDI.current().select(type)`. Quarkus: build-time synthetic bean injection. Spring: `ApplicationContext.getBean(type)`.

One SPI replaces: quarkus-langchain4j's 15+ supplier marker classes, langchain4j-cdi's string-based named-bean lookup, and upstream's static supplier methods. All three resolve components the same way — only the resolver implementation differs.

### Item 3 — Guardrail Attributes

Subset of item 1, broken out because it can ship independently. The guardrail interfaces (`InputGuardrail`, `OutputGuardrail`) already exist in upstream. This just adds the declaration point on annotations. `Class<?>[]` arrays — frameworks resolve via DI.

### Item 4 — AgentListener Attribute

Another subset of item 1. Standard attachment point for observability. `Class<? extends AgentListener>` on `@Agent`. Frameworks resolve the class; auto-discovery of all `AgentListener` beans (what quarkus-langchain4j does today) remains a framework-specific enhancement on top.

### Item 5 — Config Property Glossary

Not an SPI — documentation only. Upstream publishes a table of canonical property names:

| Annotation | Attribute | Canonical property name |
|------------|-----------|------------------------|
| `@LoopAgentService` | `maxIterations` | `max-iterations` |
| `@SupervisorAgentService` | `maxAgentsInvocations` | `max-agent-invocations` |
| `@Agent` | `description` | `description` |

Framework integrations use these as their leaf keys. The developer writes `quarkus.langchain4j.agent.myAgent.max-iterations=5` or `dev.langchain4j.cdi.plugin.myAgent.config.max-iterations=5` — the suffix is the same, only the prefix differs.

---

## Convergence Limits

Structural barriers prevent full unification in some areas. These are real — but they don't excuse incoherence. Where the implementation must differ, the vocabulary, mental model, and developer experience should still feel like one team designed it.

### Limit 1: Config namespace — framework-owned

`quarkus.langchain4j.*` vs `dev.langchain4j.cdi.plugin.*`. Each framework dictates its namespace prefix.

**Coherence despite the limit:** The property name glossary (roadmap item 5) aligns the leaf keys. A developer who knows `max-iterations` is the property finds it under whatever prefix their framework uses. The glossary is a convention, not enforcement — but a convention that both projects follow looks designed, not accidental.

### Limit 2: Build-time vs. runtime validation

Quarkus validates at build time (`DeploymentException`). Portable CDI validates at runtime (or warns and skips). A portable CDI extension doesn't have a build phase.

**Coherence despite the limit:** The *what* should be consistent even when the *when* differs. Both integrations should validate the same things: missing agents referenced in sub-agent lists, guardrail classes that don't implement the guardrail interface, circular agent dependencies. Error messages should use the same terminology and reference the same annotation attributes. A developer who sees "Agent 'stepB' referenced in @SequenceAgentService is not registered" should get the same message on Quarkus (at build time) and portable CDI (at startup) — only the timing differs.

### Limit 3: Error handling philosophy — fail-fast vs. warn-and-skip

langchain4j-cdi skips unresolvable sub-agents with a WARNING. Quarkus throws. These reflect different priorities: portability tolerance vs. correctness guarantees.

**Coherence despite the limit:** Upstream can define the contract without dictating enforcement: "A framework integration MUST resolve all referenced sub-agents. If resolution fails, the integration SHOULD fail with a descriptive error. Implementations MAY fall back to a degraded mode with a WARNING, but this is not the recommended default." Shared contract, framework-appropriate strictness. Over time, gravity pulls toward fail-fast.

### Limit 4: Thread context propagation — no portable CDI equivalent

Quarkus uses SmallRye Context Propagation for parallel agent execution. Portable CDI has `ManagedExecutorService` (Jakarta Concurrency) but support varies across servers.

**Coherence despite the limit:** The observable behaviour should be the same: CDI request context, security context, and trace context are available on parallel agent worker threads. The mechanism differs, but the developer expectation is identical. Upstream can document the contract: "Framework integrations MUST propagate the calling thread's request context to parallel agent execution threads." How each framework delivers this is its own concern.

### Limit 5: GraalVM native image

langchain4j-cdi's no-arg constructor fallback for tools and guardrails requires reflection that doesn't work in native without explicit registration. Quarkus eliminates this path at build time.

**Coherence despite the limit:** If both integrations converge on "resolve via DI container" as the primary path (roadmap item 2), the no-arg constructor fallback becomes a portable-CDI-only escape hatch — not a divergent design. Document it as a fallback that is not available in all deployment modes.

### Limit 6: Dev UI / Dev Services — Quarkus-only value

No portable CDI equivalent. These are additive Quarkus platform features that don't create annotation or wiring inconsistency. No coherence action needed.

### Design Principle

**Share the contract, vary the mechanism.** Upstream documents what should happen — scope semantics, validation rules, context propagation, property names. Framework integrations implement the contract using their platform's capabilities. The developer mental model is one system; the runtime behaviour is the best each platform can deliver.

Where langchain4j-cdi currently diverges most is not in mechanisms (that's expected) but in *vocabulary* — different annotation names, different attribute names, different property names for the same concepts. That's the avoidable inconsistency. Converging on upstream's vocabulary and upstream's contracts is achievable even where the mechanisms must differ.

---

## What Convergence Creates

**For users:** One set of annotations across all Java frameworks. Documentation, tutorials, and examples are portable. A developer who learns `@Agent` on Quarkus uses the same annotation on WildFly.

**For the ecosystem:** Community contributions — new topologies, guardrails, listeners — land in upstream and work everywhere. Third-party integrations (Spring, Micronaut) can use the same SPIs.

**For upstream:** langchain4j becomes genuinely framework-pluggable, not just "works without a framework." Framework-specific quirks are handled by framework integrations through shared SPIs, not upstream workarounds.

**For langchain4j-cdi:** Upstream annotations now have the attributes that drove the fork (scope, guardrails, listener). The resolution SPI means CDI bean lookup works through upstream's mechanism — no custom annotations needed. Continuing to maintain 11 forked annotations becomes tech debt, not differentiation.

---

## Foundation Already in Place

| Upstream Issue | Status | Convergence role |
|----------------|--------|------------------|
| #5394 (SupplierParameterResolver) | **Merged** | Seed for item 2 — proves the resolution SPI pattern |
| #5376 (DefaultExecutorProvider) | Open | Parallel execution pluggability |
| #5378 (@ParallelExecutor DI params) | Open | Item 6 — CDI-injected parallel executor params |
| #5399 (Widen AgentConfigurator) | Open | Item 7 — config overlay on all workflow builders |
| #5400 (A2AService setter) | Open | Item 8 — CDI lifecycle for A2A |

---

## References

- langchain4j-cdi: https://github.com/langchain4j/langchain4j-cdi
- Fit-gap analysis: `specs/2026-06-08-langchain4j-cdi-fitgap.md`
- Convergence spec: `specs/2026-06-11-langchain4j-cdi-analysis-design.md`
- quarkus-langchain4j agentic PR chain: #2534 → #2555 → #2544 → #2550
- @RegisterAiService simplification: #2572
