# @HybridSearch + @Corpus Design Spec

**Date:** 2026-06-15
**Issue:** [#2575](https://github.com/quarkiverse/quarkus-langchain4j/issues/2575)
**Branch:** `issue-2575-hybrid-search`
**Stacked on:** `issue-2574-rag-pipeline` (PR #2597 — @RagPipeline)

---

## Context

`@RagPipeline` (PR #2597) produces a `RetrievalAugmentor` — the full query-side pipeline
(transformer → retriever(s) → aggregator → injector). What it doesn't model is
_how a single retrieval step works internally_: running multiple search legs in parallel,
fusing their results, and re-ranking the merged list before returning to the pipeline.

`@HybridSearch` fills that gap. It produces a `ContentRetriever` — a composable retrieval
unit that `@RagPipeline` can use as one of its `retrievers`, or that can be wired directly
into an AI service as a shorthand.

`@Corpus` is a CDI qualifier that ships alongside `@HybridSearch`. It disambiguates multiple
beans of the same type (e.g. two `ContentRetriever` beans for different document corpora).
The `@HybridSearch` processor propagates it to generated beans automatically.

---

## Annotations

### @HybridSearch

```java
@Retention(RUNTIME)
@Target(TYPE)
public @interface HybridSearch {
    /** ContentRetriever beans — the search legs. At least one required. */
    Class<?>[] retrievers();

    /** ScoringModel bean for post-fusion re-ranking. Optional (void.class = skip). */
    Class<?> reranker() default void.class;

    /**
     * Reciprocal Rank Fusion k parameter. Only used when retrievers.length > 1.
     * Higher k reduces the impact of high-rank positions. Standard default: 60.
     */
    int fusionK() default 60;
}
```

Annotates an interface. The processor generates a synthetic `ContentRetriever` CDI bean
typed as both `ContentRetriever` and the annotated interface. The interface need not
extend `ContentRetriever` — the processor adds it.

### @Corpus

```java
@Qualifier
@Retention(RUNTIME)
@Target({TYPE, METHOD, FIELD, PARAMETER})
public @interface Corpus {
    String value();
}
```

Pure CDI qualifier — no annotation processor needed. When `@Corpus` appears on a
`@HybridSearch` interface, the processor propagates the qualifier to the generated
synthetic bean so it can be injected and disambiguated by corpus name.

---

## Processing Modes

### Standalone — plain interface

```java
@HybridSearch(retrievers = {DenseRetriever.class, SparseRetriever.class},
              reranker   = CohereScorer.class)
public interface ProductSearch {}
// → ContentRetriever + ProductSearch synthetic bean, @ApplicationScoped
```

Standalone when the annotated interface does NOT carry `@RegisterAiService`.
`HybridSearchProcessor` produces a `StandaloneHybridSearchBuildItem`. At
`RUNTIME_INIT`, a `SyntheticBeanBuildItem` is created via `HybridSearchRecorder`.

### Companion — alongside @RegisterAiService

```java
@RegisterAiService
@HybridSearch(retrievers = {DenseRetriever.class, SparseRetriever.class},
              reranker   = CohereScorer.class)
public interface ProductAI { ... }
// → ContentRetriever bean + minimal RetrievalAugmentor wired to the AI service
```

Sugar for the common case: "just add hybrid search to my AI service." The processor
creates the `ContentRetriever` bean and produces a `HybridSearchBuildItem` carrying
the `HybridSearchCreateInfo`. `AiServicesProcessor` consumes `HybridSearchBuildItem`
and wraps the retriever in a minimal `DefaultRetrievalAugmentor` wired to the service —
identical to how it consumes `RagPipelineBuildItem`.

**Constraint:** `@HybridSearch` + `@RagPipeline` on the same `@RegisterAiService`
interface → `DeploymentException`. Both own retrieval; they cannot coexist on one service.
Use a split-interface form when composing `@HybridSearch` into a larger `@RagPipeline`:

```java
@HybridSearch(retrievers = {DenseRetriever.class, SparseRetriever.class})
public interface ProductSearch {}  // ContentRetriever

@RegisterAiService
@RagPipeline(retrievers = {ProductSearch.class}, transformer = HydeTransformer.class)
public interface ProductAI { ... }
```

---

## Validation Rules

| Condition | Result |
|-----------|--------|
| `retrievers` empty | `DeploymentException` |
| `retrievers.length == 1` AND `reranker == void.class` | `DeploymentException` — nothing hybrid; use `@RagPipeline(retrievers=...)` |
| `@HybridSearch` + `@RagPipeline` on same `@RegisterAiService` | `DeploymentException` |
| Annotated target is not an interface | `DeploymentException` |
| Any retriever class not a CDI bean assignable to `ContentRetriever` | `DeploymentException` |
| `reranker` class not a CDI bean assignable to `ScoringModel` | `DeploymentException` |

Single-retriever + reranker is valid — it's "reranked single retrieval," a legitimate use
case not expressible by `@RagPipeline` alone.

---

## Runtime: HybridSearchContentRetriever

New class in `core/runtime`. Implements `ContentRetriever`. Created by
`HybridSearchRecorder` via `SyntheticCreationalContext`.

```
retrieve(Query):
  if single retriever:
      results ← retriever.retrieve(query)
  else:
      results ← run all retrievers in parallel (ManagedExecutor)
               → RRF-fuse(results, fusionK)
  if reranker present:
      scores  ← reranker.scoreAll(segments, query.text())
      results ← re-sort by score descending
  return results
```

**RRF formula:** `score(doc) = Σ 1 / (k + rank(doc, list_i))` across all result lists.
Documents appearing in multiple lists accumulate higher scores. Standard k=60.

**Parallel execution:** `ManagedExecutor` is injected (same as `@RagPipeline` multi-retriever
mode). Propagates CDI request context, OTel spans, and security context across retrieval
threads. Only injected when `retrievers.length > 1`; single-retriever mode has no executor.

**Reranker integration:** `ScoringModel.scoreAll(List<TextSegment>, String query)` scores
all fused results in one call. Results are re-sorted descending by score. The reranker
operates on whatever the fusion produced — no separate `maxResults` trimming at this layer
(the individual retrievers control result counts; the pipeline above controls aggregation).

---

## Build-time Components

| Artifact | Purpose |
|----------|---------|
| `HybridSearch.java` (runtime) | Annotation |
| `Corpus.java` (runtime) | CDI qualifier annotation |
| `HybridSearchCreateInfo.java` (runtime) | Record: retriever class names, reranker class name, fusionK |
| `HybridSearchContentRetriever.java` (runtime) | Runtime ContentRetriever implementation |
| `HybridSearchRecorder.java` (runtime) | `@Recorder` — creates `HybridSearchContentRetriever` via `SyntheticCreationalContext` |
| `HybridSearchBuildItem.java` (deployment) | Companion mode: carries service class name + `HybridSearchCreateInfo` |
| `StandaloneHybridSearchBuildItem.java` (deployment) | Standalone mode: carries interface `DotName` + `HybridSearchCreateInfo` |
| `HybridSearchProcessor.java` (deployment) | Scans `@HybridSearch`, validates, produces build items, creates standalone synthetic beans |
| `LangChain4jDotNames.java` (deployment) | Add `HYBRID_SEARCH`, `CORPUS` `DotName` constants |
| `AiServicesProcessor.java` (deployment) | Consume `HybridSearchBuildItem` for companion wiring (parallel to `RagPipelineBuildItem`) |

**Injection points on the synthetic bean** mirror what `HybridSearchRecorder` resolves via
`ctx.getInjectedReference()`:
- One injection point per retriever class (by exact type)
- One injection point for `ScoringModel` (by exact type) if reranker is set
- One injection point for `ManagedExecutor` if `retrievers.length > 1`

**`@Corpus` propagation:** If the annotated interface carries `@Corpus("value")`, the
processor adds the qualifier to the `SyntheticBeanBuildItem` configurator via
`addQualifier(AnnotationInstance)`. No special handling required beyond detecting the
annotation and forwarding it.

---

## Tests

| Test class | What it covers |
|------------|---------------|
| `HybridSearchStandaloneTest` | Two retrievers, RRF fusion, standalone ContentRetriever injection |
| `HybridSearchSingleRetrieverRerankerTest` | Single retriever + reranker — valid path |
| `HybridSearchCompanionTest` | @HybridSearch on @RegisterAiService — AI service wired correctly |
| `HybridSearchCorpusQualifierTest` | @Corpus propagated to generated bean, injected by qualifier |
| `HybridSearchConflictValidationTest` | @HybridSearch + @RagPipeline on same service → DeploymentException |
| `HybridSearchEmptyRetrieversValidationTest` | Empty retrievers[] → DeploymentException |
| `HybridSearchSingleNoRerankerValidationTest` | Single retriever, no reranker → DeploymentException |

---

## Decision Log (PR Notes)

The following records the alternatives considered and why each decision was made. Included
in the PR description for reviewer context.

---

**Decision 1 — `@HybridSearch` produces `ContentRetriever`; companion mode is sugar**

*Chosen:* `@HybridSearch` always produces a `ContentRetriever` synthetic bean. When placed
on a `@RegisterAiService` interface (companion), the processor additionally wires the
retriever into the AI service via a minimal `DefaultRetrievalAugmentor`. The produced
type is always `ContentRetriever` regardless of mode.

*Alternative A — always produces `RetrievalAugmentor`:* Rejected. Duplicates `@RagPipeline`'s
responsibility with no clear selection criterion. If both produce `RetrievalAugmentor`, the
user has no principled reason to choose one over the other.

*Alternative B — produces different types by mode:* Rejected. Bi-type output is surprising.
Callers cannot reason about what `@HybridSearch` returns without reading the context.
Single produced type, context-sensitive wiring.

---

**Decision 2 — `retrievers[]` of `ContentRetriever` beans, not named `dense`/`sparse` slots**

*Chosen:* Array of `ContentRetriever` beans. Quarkus fuses their outputs. No named slots.

*Alternative — named `dense`/`sparse` slots with `EmbeddingModel` + `EmbeddingStore`
components:* Rejected for two independent reasons.

First: `EmbeddingModel` is a type lie. No `SparseEmbeddingModel` interface exists in
LangChain4j 1.16.2. The interface returns `float[]` (dense) regardless of what the model
produces. A processor cannot detect a dense model passed in a `sparseModel` slot. The
runtime cannot detect it. The user gets two dense searches fused — a silent correctness
failure.

Second: the shared-store assumption breaks for the primary sparse retrieval implementations.
Qdrant stores dense and sparse vectors in separate named vector fields. Calling
`EmbeddingStore.search()` twice with different query embeddings does not compose into native
hybrid search — it performs two ANN searches on whatever the store's default vector field is.

*Alternative — named `dense`/`sparse` slots with `ContentRetriever` beans (no shared store):*
Closer, but rejected. "Dense" and "sparse" are implementation assumptions encoded as API.
Two dense retrievers over different corpora is valid hybrid search and cannot be expressed
with named slots without lying. Array of retrievers with user-chosen semantics is more honest.

*Future path:* If LangChain4j ships `SparseEmbeddingModel`, a component-shorthand mode
(provide model + store, processor assembles the retriever) can be added on top of the
retriever-based design without breaking it.

---

**Decision 3 — `fusionK = 60` int attribute, RRF baked in, no `RetrievalFusionStrategy` SPI**

*Chosen:* RRF is the fusion strategy. k is configurable per annotation as an int attribute.

*Alternative — `RetrievalFusionStrategy` SPI with `RrfFusionStrategy` as `@DefaultBean`:*
Rejected. RRF covers 95%+ of use cases. The only meaningful tunable is k. With a SPI,
per-annotation k configuration requires multiple qualified beans — worse DX than a single
int attribute. If non-RRF fusion is genuinely needed, the user implements a `ContentRetriever`
that does it internally; that is what the abstraction exists for.

---

**Decision 4 — `reranker` slot accepts `ScoringModel` bean**

*Chosen:* `ScoringModel` — `scoreAll(List<TextSegment>, String query) → List<Double>`.
This is exactly the reranker operation: score existing retrieved content against a query.

*Alternative — `ContentRetriever` for reranking:* Rejected. Wrong type contract. A reranker
operates on already-retrieved content; it does not issue new retrieval calls. Using
`ContentRetriever` would be a type lie structurally identical to the `EmbeddingModel`-for-sparse
problem in Decision 2.

---

**Decision 5 — Companion mode exists; `@HybridSearch` + `@RagPipeline` on same service is an error**

*Chosen:* `@HybridSearch` on `@RegisterAiService` (companion) auto-wires into the service.
`@HybridSearch` + `@RagPipeline` on the same service → `DeploymentException`.

*Alternative — always split-interface, no companion mode:* Rejected. Asymmetry with
`@RagPipeline`'s companion mode creates a documentation and support burden. The DX cost
of always requiring a separate interface is real even for simple cases. Companion mode
is pure sugar — it does not change the type produced.

---

**Decision 6 — `@Corpus` lands in this PR**

*Chosen:* CDI qualifier defined here, processor propagation wired into `HybridSearchProcessor`.

*Alternative — defer to `@DocumentIngestion` PR (#2576):* Rejected. `@Corpus` is needed
by `@HybridSearch` first. Deferring means this PR ships incomplete processor support
(no qualifier propagation), and #2576 retrofits it. No structural reason to split.
