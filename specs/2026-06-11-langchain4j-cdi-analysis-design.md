# langchain4j-cdi Comparative Analysis — Design Spec

**Date:** 2026-06-11
**Type:** Analysis document + adoption roadmap
**Output:** `/Users/mdproctor/claude/public/quarkus-langchain4j/roadmap.md`
**Builds on:** `specs/2026-06-08-langchain4j-cdi-fitgap.md` (positioning — complementary, not competing)

---

## Goal

Produce a two-part analysis comparing langchain4j-cdi and quarkus-langchain4j across their full surface area, then distill actionable adoption items into a prioritised roadmap. The analysis treats the two projects as peer CDI integrations with different design choices, evaluated on technical merit for Quarkus users.

## Scope

- Full CDI wiring surface: component resolution, configuration, tools, guardrails, RAG, memory, scope
- Full agentic surface: agent topology annotations, composition model, observability
- Covers langchain4j-cdi as of June 2026 (modules: core, portable-ext, build-compatible-ext, config, a2a, mcp)
- Evaluates against quarkus-langchain4j current state plus planned work (#2572, agentic PR chain C4–C8)

## Non-goals

- Not a competitive positioning document — the fit-gap already established they're complementary
- Not a migration guide — no one is switching between them
- Not prescriptive on timeline — the roadmap prioritises but doesn't schedule

---

## Part 1: Strategic Map

Eight dimensions, each with:
- **How langchain4j-cdi does it** — concrete patterns and code examples
- **How quarkus-langchain4j does it** — current state and planned work
- **Verdict** — who got it right, and what can be learned

### Dimension 1: Component Resolution

**langchain4j-cdi:** Name-based bean references. `chatModelName = "my-model"` resolves via `Instance.select(NamedLiteral.of(name))` at runtime. Default `"#default"` selects the unnamed default bean. No supplier classes, no sentinel markers.

**quarkus-langchain4j:** Supplier-class attributes with 15+ sentinel marker classes (`BeanChatLanguageModelSupplier`, `BeanIfExistsRetrievalAugmentorSupplier`, `NoChatMemoryProviderSupplier`). Each marker's `get()` throws `UnsupportedOperationException` — they're flags, not real suppliers. Build-time processor detects marker vs. custom supplier and generates appropriate injection.

**#2572 plan:** Add direct bean-class attributes (`Class<? extends T>`) alongside existing supplier attributes. Deprecate supplier versions. The bean class is resolved as a CDI bean at build time — no supplier wrapper, no sentinel markers.

**Verdict:** All three approaches converge on "just reference the bean." langchain4j-cdi uses strings (flexible but loses type safety). #2572 uses class references (type-safe, build-time validated). The current supplier pattern is the worst of all worlds — complex API surface for no user benefit. #2572's direction is right.

### Dimension 2: Configuration

**langchain4j-cdi:** Pluggable `LLMConfig` SPI discovered via `ServiceLoader`. Default implementation reads MicroProfile Config under `dev.langchain4j.cdi.plugin.<bean-name>.config.<property>`. Can instantiate beans entirely from config properties (`class=com.example.MyModel`). Expression resolution SPI chains `${mp.config}` and `#{jakarta.el}` resolvers — all string attributes in annotations pass through the resolver chain before CDI lookup.

**quarkus-langchain4j:** Direct Quarkus Config integration under `quarkus.langchain4j.*`. Model providers (OpenAI, Ollama, etc.) each have typed config classes generated at build time. Agentic module adds `quarkus.langchain4j.agent.<name>.*` namespace (C6 PR). No expression resolution in annotation attributes — config values are resolved in recorders, not annotations.

**Verdict:** Expression resolution is the interesting idea here. Being able to write `@RegisterSimpleAgent(chatModelName = "${ai.model.name}")` and have it resolved from config is genuinely useful for multi-environment deployments. However, Quarkus already has `@ConfigProperty` injection and programmatic config access — the same thing is achievable without an SPI. The property-based bean creation (`class=com.example.MyModel` in config) is clever for generic CDI but unnecessary in Quarkus where extensions create typed config classes. **Action: evaluate expression resolution as a lightweight addition, but don't adopt the LLMConfig SPI.**

### Dimension 3: Agent Topology

**langchain4j-cdi:** 11 dedicated annotations (`@RegisterSimpleAgent`, `@RegisterSequenceAgent`, `@RegisterLoopAgent`, `@RegisterParallelAgent`, `@RegisterSupervisorAgent`, `@RegisterPlannerAgent`, `@RegisterA2AAgent`, `@RegisterMcpClientAgent`, `@RegisterHumanInTheLoopAgent`, `@RegisterConditionalAgent`, `@RegisterParallelMapperAgent`). Each defines one agent node. All share common attributes (name, description, outputKey, scope, optional, summarizedContext) plus topology-specific ones.

**quarkus-langchain4j:** Uses upstream `langchain4j-agentic` annotations directly (`@Agent`, `@Agents`, `@SequenceAgentService`, `@LoopAgentService`, `@SupervisorAgentService`, `@ParallelAgent`, `@A2AClient`). The agentic module adds Quarkus-native build-time validation, CDI wiring, and config overlay — but the user-facing annotations are upstream's.

**Verdict:** langchain4j-cdi reinvents what langchain4j-agentic provides. This creates a maintenance burden and divergence risk — when upstream adds a new topology, langchain4j-cdi needs a new annotation. quarkus-langchain4j's approach of wrapping upstream is architecturally sounder. However, langchain4j-cdi's annotations are more CDI-native (scope, named-bean resolution built into the annotation) whereas upstream's annotations are framework-agnostic. **Skip — wrapping upstream is the right strategy. The extra CDI-native attributes are better handled via our config overlay (C6) than by forking the annotations.**

### Dimension 4: Agent Composition

**langchain4j-cdi:** String-based `subAgentNames = {"stepA", "stepB"}`. Runtime resolution via `Instance.select(NamedLiteral.of(name))`. Unresolvable names are skipped with a WARNING. No compile-time or build-time validation that referenced agents exist.

**quarkus-langchain4j:** Type-safe agent references. Build-time processor validates that referenced agents exist, checks scope compatibility, verifies no circular dependencies. Missing agents are `DeploymentException` — fail fast at startup, not a silent runtime skip.

**Verdict:** Build-time validation is strictly better. String-based composition is convenient to write but fragile to maintain — a typo in `subAgentNames` produces a silent runtime failure. quarkus-langchain4j's approach is the right one. **Skip — our approach is better.**

### Dimension 5: Tool Handling

**langchain4j-cdi:** Three resolution strategies composable on a single service: `tools = {BookingService.class}` (class array), `toolNames = {"bookingTool"}` (named beans), `toolProviderName = "dynamicProvider"` (ToolProvider bean). Resolution: CDI bean lookup first, no-arg constructor fallback second.

**quarkus-langchain4j:** `tools = {BookingService.class}` on `@RegisterAiService`. Build-time processor discovers tool methods via Jandex, validates signatures, generates optimised invokers. `toolProviderSupplier` attribute for dynamic tools. No named-bean tool resolution. No no-arg constructor fallback — tools must be CDI beans.

**Verdict:** The no-arg constructor fallback is pragmatic for simple tools that don't need injection, but goes against Quarkus CDI philosophy (everything is a bean, validated at build time). Named-bean tool resolution (`toolNames`) adds flexibility for dynamic tool sets but is achievable via `ToolProvider` in quarkus-langchain4j. **Skip — the current approach is Quarkus-appropriate. ToolProvider covers the dynamic case.**

### Dimension 6: Guardrails

**langchain4j-cdi:** Built into `@RegisterAIService` and `@RegisterSimpleAgent` — `inputGuardrails`/`outputGuardrails` (class arrays) and `inputGuardrailNames`/`outputGuardrailNames` (string arrays). Resolved as ordered list. CDI bean lookup with no-arg constructor fallback. Classes take precedence if both specified. Only on `@RegisterAIService` and `@RegisterSimpleAgent` — other topologies lack guardrail attributes.

**quarkus-langchain4j:** `@RegisterAiService` has no guardrail attributes. The agentic module (C5 PR #2555) adds `@AgentInputGuardrails`/`@AgentOutputGuardrails` on all agent interface types — not just simple agents. Build-time validated. However, guardrails on `@RegisterAiService` (non-agentic AI services) are absent.

**Verdict:** langchain4j-cdi has guardrails on `@RegisterAIService` today — quarkus-langchain4j doesn't. This is a real gap for non-agentic AI services. Users who want input validation on a plain `@RegisterAiService` chat service have no declarative option. The agentic module's approach (`@AgentInputGuardrails`) is the right pattern — extend it to `@RegisterAiService`. **Adopt — add guardrail attributes to `@RegisterAiService`, mirroring the agentic module's approach.**

### Dimension 7: RAG & Retrieval

**langchain4j-cdi:** `contentRetrieverName` and `retrievalAugmentorName` on `@RegisterAIService` — string-based named-bean lookup. RetrievalAugmentor takes precedence over ContentRetriever when both specified. Configuration via properties.

**quarkus-langchain4j:** `retrievalAugmentor` supplier attribute on `@RegisterAiService`. `BeanIfExistsRetrievalAugmentorSupplier` as default — auto-discovers a `RetrievalAugmentor` bean if one exists. No direct `contentRetriever` attribute. #2572 proposes `@RagPipeline` composition annotation and direct bean-class attributes.

**Verdict:** Similar capabilities, different resolution mechanisms. #2572's `@RagPipeline` goes significantly beyond either current approach — it composes the full pipeline (router → transformer → retriever → injector → aggregator) declaratively. langchain4j-cdi's approach is simpler but less powerful. **#2572 is the right direction — no additional ideas to adopt from langchain4j-cdi here.**

### Dimension 8: Scope & Lifecycle

**langchain4j-cdi:** Every annotation has `scope()` defaulting to `RequestScoped` (for `@RegisterAIService`) or `ApplicationScoped` (for agent annotations). Users can override per service/agent.

**quarkus-langchain4j:** `@RegisterAiService` services are generated as `@ApplicationScoped` synthetic beans. Scope is not configurable via the annotation. The agentic module's CDI wiring doesn't expose scope control.

**Verdict:** Configurable scope per service is a reasonable feature. `RequestScoped` makes sense for chat services with per-request memory; `ApplicationScoped` makes sense for stateless utility services. However, in Quarkus, scope is typically controlled by the CDI annotation on the bean, not by a framework attribute. Adding a `scope` attribute to `@RegisterAiService` would be non-standard Quarkus CDI. **Evaluate — useful in principle, but the Quarkus-native approach would be to let users annotate their interface with `@RequestScoped` directly. Check if the current processor respects scope annotations on the interface.**

---

## Part 2: Adoption Roadmap

| # | Pattern | Source | Action | Fits with | Priority | Scale | Complexity | Rationale |
|---|---------|--------|--------|-----------|----------|-------|------------|-----------|
| 1 | Direct bean-class attributes on `@RegisterAiService` | Converging | **Adopt** | #2572 step 1 | High | S | Low | Eliminates supplier markers. #2572 already proposes this — validates the direction. |
| 2 | Guardrails on `@RegisterAiService` | langchain4j-cdi | **Adopt** | Standalone or #2572 | High | M | Low | Real gap — non-agentic AI services have no declarative guardrails. Mirror agentic module's `@AgentInputGuardrails` pattern. |
| 3 | Expression resolution in annotation attributes | langchain4j-cdi | **Evaluate** | Standalone | Med | M | Med | `chatModelName = "${ai.model}"` is ergonomic for multi-env. But Quarkus Config + `@ConfigProperty` may already cover this. Need to check if there are cases that config injection can't reach. |
| 4 | Configurable scope per AI service | langchain4j-cdi | **Evaluate** | Standalone | Low | S | Low | Useful but may conflict with Quarkus CDI conventions. Check if processor already respects `@RequestScoped` on the interface. If yes, skip. |
| 5 | `@RagPipeline` composition annotation | #2572 (original) | **Adopt** | #2572 step 2 | High | M | Med | Goes beyond langchain4j-cdi's simple name-ref. Full pipeline composition is a differentiator. |
| 6 | No-arg constructor fallback for tools | langchain4j-cdi | **Skip** | — | — | — | — | Violates Quarkus CDI philosophy. Tools should be beans. |
| 7 | Named-bean tool resolution (`toolNames`) | langchain4j-cdi | **Skip** | — | — | — | — | `ToolProvider` already covers dynamic tool sets. |
| 8 | Property-based bean creation | langchain4j-cdi | **Skip** | — | — | — | — | Quarkus extensions generate typed config classes. Property-based instantiation is for generic CDI. |
| 9 | Separate agent topology annotations | langchain4j-cdi | **Skip** | — | — | — | — | Wrapping upstream langchain4j-agentic is architecturally sounder. |
| 10 | String-based agent composition | langchain4j-cdi | **Skip** | — | — | — | — | Build-time type-safe validation is strictly better. |

### Adoption item detail

**Item 1 — Direct bean-class attributes:** Add `contentRetriever`, `chatMemoryProvider`, `moderationModel`, `toolProvider` as `Class<? extends T>` attributes to `@RegisterAiService`. The build-time processor resolves the class as a CDI bean — same validation as today, simpler API. Deprecate supplier equivalents. This is #2572 step 1 with validation from langchain4j-cdi's successful use of the same pattern (theirs is name-based but the principle is identical: reference the bean directly).

**Item 2 — Guardrails on @RegisterAiService:** Add `inputGuardrails` and `outputGuardrails` attributes to `@RegisterAiService`. Use class-array references (not string names) for type safety. Build-time validated: guardrail classes must be CDI beans implementing `InputGuardrail`/`OutputGuardrail`. Ordered list execution matches langchain4j-cdi's model. This fills a real gap — the agentic module has guardrails on agents, but plain AI services don't.

**Item 3 — Expression resolution:** Investigate whether there are annotation attributes in `@RegisterAiService` or agent annotations where config-driven values would be useful but `@ConfigProperty` injection can't reach. Likely candidate: `modelName` on `@RegisterAiService` — currently a string literal, could benefit from `${profile.model}` resolution. If the use case is narrow, a targeted solution (e.g., config overlay in the processor) is better than a general expression resolution SPI.

**Item 4 — Configurable scope:** Check `AiServicesProcessor` for whether `@RequestScoped` or `@SessionScoped` on the AI service interface is respected. If yes, document it. If no, consider supporting it — minor processor change to read the scope annotation from the interface class.

---

## References

- langchain4j-cdi source: https://github.com/langchain4j/langchain4j-cdi
- Prior fit-gap: `specs/2026-06-08-langchain4j-cdi-fitgap.md`
- #2572 issue: https://github.com/quarkiverse/quarkus-langchain4j/issues/2572
- #2572 breakdown: HANDOFF.md §#2572 Breakdown
- Agentic PR chain: #2534 → #2555 → #2544 → #2550
