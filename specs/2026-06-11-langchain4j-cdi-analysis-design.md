# langchain4j Ecosystem Convergence — Design Spec

**Date:** 2026-06-11
**Type:** Cross-project convergence analysis + roadmap
**Output:** `/Users/mdproctor/claude/public/quarkus-langchain4j/roadmap.md`
**Builds on:** `specs/2026-06-08-langchain4j-cdi-fitgap.md` (positioning — complementary, not competing)

---

## Goal

Produce a convergence roadmap for the langchain4j ecosystem's three Java integration projects. Map where they currently diverge, identify upstream (langchain4j core) changes that would let all three align on shared annotations and wiring behaviours, and prioritise concrete proposals.

The strategic bet: if langchain4j (upstream) and quarkus-langchain4j converge on shared SPIs and annotation attributes — with Mario already open to improvements — langchain4j-cdi faces a choice: adopt the shared approach or maintain a divergent fork. Convergence creates gravity.

## The Three Projects

| Project | Role | Approach |
|---------|------|----------|
| **langchain4j** (core) | Framework-agnostic AI framework | Defines annotations (`@Agent`, `@SequenceAgentService`, etc.) and runtime. No CDI awareness. |
| **langchain4j-cdi** | Portable CDI integration | Reinvents 11 topology annotations with CDI attributes. Runtime wiring. Targets all Jakarta EE servers. |
| **quarkus-langchain4j** | Quarkus-native integration | Wraps upstream annotations. Build-time validation, CDI wiring, config overlay. GraalVM native. |

## Scope

- All three projects' annotation surfaces, resolution SPIs, and wiring behaviours
- Focus on upstream changes that are framework-agnostic (not Quarkus-specific) but enable CDI integrations
- Covers: component resolution, agent topology, guardrails, observability, configuration, tools
- Existing upstream contributions (#5394 merged, #5376/#5378/#5399/#5400 filed) as foundation

## Non-goals

- Not asking langchain4j-cdi to change — this is a convergence roadmap that makes the shared path more attractive than the divergent one
- Not Quarkus-specific proposals — every upstream proposal must be framed as framework-agnostic
- Not prescriptive on timeline — prioritised but not scheduled

---

## Part 1: Divergence Map

Where the three projects differ today and what drives the divergence.

### 1. Agent Topology Annotations

**The problem:** langchain4j-cdi reinvented 11 annotations because upstream's are CDI-unaware.

| Concern | langchain4j (upstream) | langchain4j-cdi | quarkus-langchain4j |
|---------|----------------------|-----------------|---------------------|
| Annotations | `@Agent`, `@SequenceAgentService`, `@LoopAgentService`, etc. | `@RegisterSimpleAgent`, `@RegisterSequenceAgent`, `@RegisterLoopAgent`, etc. (11 total) | Uses upstream's annotations directly |
| Agent name | `@Agent(name = "...")` | `@RegisterSimpleAgent(name = "...")` | `@Agent(name = "...")` |
| Scope | Not on annotation | `scope = RequestScoped.class` on every annotation | Build-time processor controls scope |
| Guardrails | Not on annotation | `inputGuardrails`/`outputGuardrails` on `@RegisterSimpleAgent` only | Separate `@AgentInputGuardrails`/`@AgentOutputGuardrails` annotations (agentic module) |
| Listener | Not on annotation | `agentListenerName = "..."` | Build-time auto-discovery of `AgentListener` beans |
| Sub-agents | Class-based references | `subAgentNames = {"a", "b"}` (string-based) | Class-based, build-time validated |

**Root cause:** upstream annotations lack extension points for CDI concerns. langchain4j-cdi's response was to fork; quarkus-langchain4j's was to add a build-time layer. If upstream added optional CDI-friendly attributes (scope hint, guardrail classes, listener class), both integrations could use the same annotations.

**Convergence proposal:** Add optional attributes to upstream annotations that CDI integrations can read. Not CDI-specific — just `Class<?>` arrays and metadata that any framework can interpret. See roadmap item 1.

### 2. Component Resolution

**The problem:** Three different ways to say "use this bean."

| Pattern | langchain4j | langchain4j-cdi | quarkus-langchain4j |
|---------|-------------|-----------------|---------------------|
| Current | `Supplier<T>` static methods | `chatModelName = "my-model"` (named string) | `chatLanguageModelSupplier = MySupplier.class` (supplier class + sentinel markers) |
| Planned | `SupplierParameterResolver` SPI (#5394 merged) | — | #2572: direct bean-class `Class<? extends T>` |

**Root cause:** upstream started with static supplier methods (no DI). quarkus-langchain4j wrapped them in marker classes. langchain4j-cdi bypassed them entirely with name strings. All three are workarounds for the same missing feature: a framework-agnostic way to say "resolve this component via your DI container."

**Convergence proposal:** Generalise `SupplierParameterResolver` (already merged) into a full resolution SPI that covers all component types — not just supplier parameters. A CDI integration registers a resolver; upstream calls it. No CDI imports in upstream code. See roadmap item 2.

### 3. Guardrails

**The problem:** Guardrails exist in langchain4j-cdi's annotations but not in upstream's.

| | langchain4j | langchain4j-cdi | quarkus-langchain4j |
|--|-------------|-----------------|---------------------|
| On AI services | Not on annotation | `inputGuardrails`/`outputGuardrails` on `@RegisterAIService` | Not on `@RegisterAiService` |
| On agents | Not on annotation | On `@RegisterSimpleAgent` only | `@AgentInputGuardrails`/`@AgentOutputGuardrails` on all agent types |
| Interface | `InputGuardrail`/`OutputGuardrail` (in upstream runtime) | Same interfaces | Same interfaces |

**Root cause:** The guardrail interfaces exist in upstream. The annotation attributes to declare them don't. Both CDI integrations added their own — inconsistently.

**Convergence proposal:** Add `inputGuardrails()` and `outputGuardrails()` as `Class<?>[]` attributes to upstream's `@Agent` annotation (and `@RegisterAiService` equivalent if one exists). Framework integrations resolve the classes via their DI container. See roadmap item 3.

### 4. Observability

**The problem:** Three different observability integration points.

| | langchain4j | langchain4j-cdi | quarkus-langchain4j |
|--|-------------|-----------------|---------------------|
| Agent-level | `AgentListener` interface | `agentListenerName` (string-based bean lookup) | Build-time auto-discovery + `Instance<AgentListener>` |
| LLM-level | `ChatModelListener` | `SpanChatModelListener` (OTel) | Quarkus OTel integration |
| Metrics | — | — | Micrometer metrics + CDI events (C4 PR #2550) |

**Root cause:** `AgentListener` is in upstream but there's no annotation attribute to declare one. langchain4j-cdi added a string attribute; quarkus-langchain4j uses CDI auto-discovery. Both work but neither is standardised.

**Convergence proposal:** Add `agentListener()` as a `Class<?>` attribute to upstream's `@Agent` annotation. Framework integrations resolve it; auto-discovery remains a framework-specific enhancement. See roadmap item 4.

### 5. Configuration

**The problem:** Each project has its own config namespace and resolution.

| | langchain4j | langchain4j-cdi | quarkus-langchain4j |
|--|-------------|-----------------|---------------------|
| Namespace | — | `dev.langchain4j.cdi.plugin.*` | `quarkus.langchain4j.*` |
| Mechanism | — | `LLMConfig` SPI + `ExpressionResolver` SPI | Quarkus Config + typed config classes |
| Expression resolution | — | `${mp.config}` and `#{jakarta.el}` in annotation attributes | Not in annotations; config resolved in recorders |

**Root cause:** Configuration is inherently framework-specific. But the pattern of overriding annotation attributes from config is shared — both integrations do it, differently.

**Convergence proposal:** This is the one area where convergence is less clear. Upstream doesn't need a config SPI — that's the framework's job. What upstream could provide is a `@Configurable` marker or naming convention that tells framework integrations "this attribute is overridable from config." See roadmap item 5 (evaluate).

### 6. Tool Handling

**The problem:** Minor divergence, mostly resolved.

| | langchain4j | langchain4j-cdi | quarkus-langchain4j |
|--|-------------|-----------------|---------------------|
| Declaration | `tools` class array | `tools` + `toolNames` + `toolProviderName` | `tools` + `toolProviderSupplier` |
| Resolution | Direct instantiation | CDI → no-arg constructor fallback | CDI only (build-time validated) |

**Root cause:** Tool resolution follows the same component resolution divergence. Once the resolution SPI converges (item 2), tool handling aligns automatically.

**Convergence proposal:** Covered by item 2 (resolution SPI). No separate proposal needed.

---

## Part 2: Convergence Roadmap

Each item is an upstream proposal, framed as framework-agnostic. The "Enables" column shows what both CDI integrations gain.

| # | Upstream Proposal | Status | Enables for CDI integrations | Priority | Scale | Complexity |
|---|-------------------|--------|------------------------------|----------|-------|------------|
| 1 | Add optional attributes to `@Agent` annotations (guardrails, listener, scope hint) | New | Both integrations use upstream annotations instead of forking | High | M | Med |
| 2 | Generalise `SupplierParameterResolver` into a full component resolution SPI | Seed merged (#5394) | All component types resolved via DI — not just supplier params | High | M | Med |
| 3 | Add guardrail class-array attributes to `@Agent` and AI service annotations | New | Consistent guardrail declaration across all frameworks | High | S | Low |
| 4 | Add `agentListener` class attribute to `@Agent` | New | Standard listener attachment point — frameworks resolve via DI | Med | S | Low |
| 5 | Config override convention for annotation attributes | New (evaluate) | Frameworks know which attributes are config-overridable | Low | S | Med |
| 6 | `@ParallelExecutor` DI params (#5378) | Filed | CDI-injected params on parallel executors | Med | S | Low |
| 7 | Widen `AgentConfigurator` to workflow builders (#5399) | Filed | Config overlay on loops, supervisors, A2A — not just agents | Med | S | Low |
| 8 | `A2AService.setA2AService()` setter (#5400) | Filed | CDI lifecycle integration for A2A services | Med | S | Low |

### Item detail

**Item 1 — Annotation extension points:** The central proposal. Add optional attributes to upstream's `@Agent`, `@SequenceAgentService`, `@LoopAgentService`, etc. that CDI integrations can read:

```java
@Agent(
    name = "myAgent",
    // Existing attributes unchanged
    
    // New optional CDI-friendly attributes (framework-agnostic)
    inputGuardrails = {RateLimitGuardrail.class, ContentFilterGuardrail.class},
    outputGuardrails = {PiiRedactionGuardrail.class},
    agentListener = MetricsAgentListener.class
)
```

These are `Class<?>` arrays — no CDI imports. Upstream ignores them in its default runtime. Framework integrations (CDI, Spring, etc.) read them and resolve via their DI container. This eliminates the reason langchain4j-cdi forked the annotations.

**Framing for Mario:** "These attributes let any framework integration declaratively attach guardrails and listeners to agents without the framework needing its own annotation set. It's the same pattern as `tools = {MyTool.class}` — which already works this way."

**Item 2 — Full resolution SPI:** `SupplierParameterResolver` (#5394, merged) proved the pattern: upstream defines an SPI, frameworks register resolvers. Generalise this to cover all component types that AI services and agents need — chat models, memory providers, content retrievers, moderation models, tool providers. A single `ComponentResolver` SPI that frameworks implement once.

```java
// In upstream — framework-agnostic SPI
public interface ComponentResolver {
    <T> T resolve(Class<T> type, String name);
    boolean canResolve(Class<?> type);
}
```

Upstream's default: `ServiceLoader` + no-arg constructor. CDI integration: `CDI.current().select(type)`. Quarkus integration: build-time synthetic bean injection. Spring: `ApplicationContext.getBean(type)`.

**Framing for Mario:** "This is the generalisation of `SupplierParameterResolver` we discussed in #5377. Instead of one SPI per supplier type, one SPI covers all component resolution. Makes langchain4j genuinely framework-pluggable."

**Item 3 — Guardrail attributes:** Subset of item 1, broken out because it can ship independently. Add `inputGuardrails()` and `outputGuardrails()` as `Class<?>[]` to `@Agent` (and to `@RegisterAiService` if upstream ever adds one). The guardrail interfaces already exist in upstream — this just adds the declaration point.

**Item 4 — AgentListener attribute:** Another subset of item 1. Add `Class<? extends AgentListener> agentListener()` to `@Agent`. Currently langchain4j-cdi uses a string name; quarkus-langchain4j uses CDI auto-discovery. A class attribute on the annotation standardises attachment while letting frameworks resolve differently.

**Item 5 — Config override convention:** Evaluate only. The question is whether upstream should provide metadata about which annotation attributes are config-overridable (e.g., `maxIterations` on `@LoopAgentService`). quarkus-langchain4j already does this via its C6 config namespace; langchain4j-cdi does it via expression resolution. A shared convention would let both integrations use the same config property names. But this may be over-engineering — each framework's config namespace is inherently its own.

---

## Part 3: Gravity Effect

What happens when langchain4j + quarkus-langchain4j converge:

**For langchain4j-cdi:**
- Upstream annotations now have the attributes they forked to get (guardrails, listener, scope hint)
- The resolution SPI means they can use upstream annotations with CDI bean lookup — no custom annotations needed
- Continuing to maintain 11 forked annotations becomes tech debt, not differentiation
- Users who start with langchain4j-cdi and move to Quarkus get the same annotation API — lower friction

**For the ecosystem:**
- One set of user-facing annotations across all Java frameworks
- Documentation, tutorials, and examples are portable
- Community contributions (new topologies, new guardrails) land in upstream and work everywhere
- Third-party integrations (Spring, Micronaut) can use the same SPIs

**For upstream:**
- langchain4j becomes genuinely framework-pluggable, not just "works without a framework"
- CDI, Spring, and Micronaut integrations share a common SPI surface
- Reduces maintenance burden — framework-specific quirks handled by framework integrations, not upstream workarounds

---

## Existing Upstream Contributions (foundation)

| Issue | Status | Convergence role |
|-------|--------|------------------|
| #5394 (SupplierParameterResolver) | **Merged** | Seed for item 2 (resolution SPI) |
| #5376 (DefaultExecutorProvider) | Open | Parallel execution pluggability |
| #5377 (Generalise ChatSupplierParameterResolver) | Closed (covered by #5394) | — |
| #5378 (@ParallelExecutor DI params) | Open | Item 6 |
| #5399 (Widen AgentConfigurator) | Open | Item 7 |
| #5400 (A2AService setter) | Open | Item 8 |

---

## References

- langchain4j-cdi source: https://github.com/langchain4j/langchain4j-cdi
- Prior fit-gap: `specs/2026-06-08-langchain4j-cdi-fitgap.md`
- #2572 issue: https://github.com/quarkiverse/quarkus-langchain4j/issues/2572
- Agentic PR chain: #2534 → #2555 → #2544 → #2550
- Upstream contributions: #5394 (merged), #5376, #5378, #5399, #5400
