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
