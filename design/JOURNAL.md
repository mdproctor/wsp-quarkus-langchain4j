# Design Journal — issue-2572-register-ai-service-simplification

## 2026-06-12 — Convergence analysis + spec design

Started with langchain4j-cdi comparative analysis. Reframed from "what to adopt" to cross-project convergence roadmap — langchain4j + quarkus-langchain4j align first, creating gravity for langchain4j-cdi. Three-phase strategy: finish Quarkus work → upstream proposals → engage langchain4j-cdi.

Produced `roadmap.md` (convergence roadmap) and `specs/2026-06-12-register-ai-service-composition-design.md` (full annotation design). Key design decisions:

- Composable annotation layers (Approach B) — each concern gets its own annotation
- Tri-state resolution: `void.class` = disabled, interface type = auto-discover, concrete class = explicit
- Dropped @MemoryConfig/@ToolConfig (don't earn their existence — no compositional value)
- Single `transformer` on @RagPipeline (not array — upstream QueryTransformer is fan-out, not pipeline)
- `@RagPipeline(augmentor = ...)` for pre-built augmentors (EasyRetrievalAugmentor path)
- No @MemoryId fallback in @TenantIsolation — tenant and memory identity are distinct
- Six new SPIs: TenantIsolationStrategy, TenantResolver, DocumentMetadataExtractor, EmbeddingCacheStore, CollectionManager, RetrievalFusionStrategy
- Six new processors (not bolted onto AiServicesProcessor's 2945 lines)

Spec went through 3 review rounds. All findings resolved.

## 2026-06-13 — Foundation implementation (PR 1, Tasks 1-7)

Wrote implementation plan for PR 1 (Foundation). Executed Tasks 1-7: framework changes complete. ComponentResolutionMode enum, @RegisterAiService rewritten (all 17 markers deleted), LangChain4jDotNames cleaned, DeclarativeAiServiceBuildItem/CreateInfo updated with ComponentEntry records, AiServicesProcessor uses resolveComponent(), AiServicesRecorder uses mode switches.

Task 8 (migration of ~100 test/sample files) deferred to next session — mechanical but large.

## 2026-06-14 — @RagPipeline design + implementation (PR 2)

Designed and implemented `@RagPipeline` — composable annotation for query-side RAG pipeline. Spec went through 3 review cycles (9 revisions total). Key design refinements from review:

- Two-state resolution (SKIP/EXPLICIT) for all @RagPipeline attributes — no AUTO_DISCOVER. Consistent with Foundation PR's "explicit over auto-discover" principle. `resolveComponent(value, null)` pattern.
- Router + retrievers is a DeploymentException, not a warning — mutually exclusive.
- `RagPipelineSupport` static utility shared between companion and standalone modes — avoids cross-recorder parameter pattern. AiServicesRecorder calls it directly.
- Injection points must mirror every `ctx.getInjectedReference()` call — `addRagInjectionPoints()` helper shared between RagPipelineProcessor (standalone) and AiServicesProcessor (companion).
- Standalone beans scoped `@ApplicationScoped` (not `@Dependent` default).

Implementation: 6 new files (annotation, record, support, recorder, build item, processor), 18 modified files (removal + migration). 732 tests green. PR #2597 created on blessed repo, targeting main, stacked on #2591 (Foundation).

Removed `retrievalAugmentor` from `@RegisterAiService`. Migrated 13 callers (6 RAG tests, 2 core tests, 1 provider test, 4 samples).

## 2026-06-14 — Code review pass on all three PRs; chain rebased and squashed

Ran parallel AI-assisted code review on PR #2591 (Foundation), PR #2534 (Agentic C1), and PR #2597 (@RagPipeline). Several findings needed careful verification before applying:

Genuine bugs fixed:
- systemMessageProvider EXPLICIT used raw reflection instead of CDI — fixed to use `creationalContext.getInjectedReference()`
- Dead `import javax.tools.Tool` (Java Compiler API, not langchain4j @Tool) removed from AiServicesProcessor
- `AGENT_INSTANCE_FACTORY` in AgenticRecorder used TCCL-only classloader — extended with three-classloader fallback matching `loadClassSafe()`
- `validateMcpToolBox` used `&&` instead of `||` — agents with multiple @McpToolBox + single agentic method silently passed
- AgentListener @Dependent check fired unconditionally — now guarded on agents being present

False positives (pushed back with reasoning):
- "@Moderate is silently broken" — intentional design documented in Foundation spec as breaking change
- "retrievalAugmentor dead code in #2591" — attribute still existed in that branch (removed in #2574)
- "Advertised @Retry/@Transactional checks don't exist" — reviewer misread the Javadoc; checks are implemented

All three PRs squashed to single clean commits. PR #2597 rebased onto updated Foundation via `git rebase --onto`. Chain verified: 734 tests green.
