# @HybridSearch + @Corpus Design Spec

**Date:** 2026-06-15 (revised post-review)
**Issue:** [#2575](https://github.com/quarkiverse/quarkus-langchain4j/issues/2575)
**Branch:** `issue-2575-hybrid-search`
**Stacked on:** `issue-2574-rag-pipeline` (PR #2597 — @RagPipeline)

---

## Context

`@RagPipeline` (PR #2597) produces a `RetrievalAugmentor` — the full query-side pipeline
(transformer → retriever(s) → aggregator → injector). What it doesn't model is
_how a single retrieval step works internally_: running multiple search legs in parallel,
fusing their results, and optionally re-ranking the merged list before returning to
the pipeline.

`@HybridSearch` fills that gap. It produces a `ContentRetriever` — a composable retrieval
unit that `@RagPipeline` can use as one of its `retrievers`, or that can be wired directly
into an AI service as a shorthand.

`@Corpus` is a CDI qualifier that ships alongside `@HybridSearch`. It disambiguates multiple
`ContentRetriever` beans injected by the bare `ContentRetriever` type (e.g. two hybrid
search beans for different corpora, distinguished at injection sites). The `@HybridSearch`
processor propagates it to generated beans automatically.

---

## Annotations

### @HybridSearch

```java
@Retention(RUNTIME)
@Target(TYPE)
public @interface HybridSearch {
    /**
     * ContentRetriever CDI beans — the search legs. At least one required.
     * Two or more legs trigger RRF fusion.
     */
    Class<?>[] retrievers();

    /**
     * ScoringModel CDI bean for post-fusion re-ranking. Optional (void.class = skip).
     * When set, scoreAll() is called on fused candidates before returning results.
     */
    Class<?> reranker() default void.class;

    /**
     * Reciprocal Rank Fusion k parameter. Only consulted when retrievers.length > 1.
     * Higher k reduces the weight of top-rank positions. Standard default: 60.
     */
    int fusionK() default 60;

    /**
     * Maximum number of fused candidates passed to the reranker.
     * Only consulted when a reranker is set. -1 means pass all fused results.
     * Set this when deploying expensive cross-encoder rerankers (Cohere, Jina, etc.)
     * to bound reranker cost. Individual retrievers control their own result counts.
     */
    int maxCandidatesForReranking() default -1;
}
```

Annotates an interface. The processor generates a synthetic `ContentRetriever` CDI bean
typed as both `ContentRetriever` and the annotated interface. The interface need not
extend `ContentRetriever` — the processor adds it as a bean type. This matches how
`@RagPipeline` does not require `extends RetrievalAugmentor`.

### @Corpus

```java
@Qualifier
@Retention(RUNTIME)
@Target({TYPE, METHOD, FIELD, PARAMETER})
public @interface Corpus {
    String value();
}
```

Pure CDI qualifier — no annotation processor logic. Primary use case: disambiguating
injection by `ContentRetriever` type when multiple `@HybridSearch` beans exist:

```java
@Corpus("products") @HybridSearch(retrievers = {ProductDense.class, ProductSparse.class})
public interface ProductSearch {}

@Corpus("manuals") @HybridSearch(retrievers = {ManualsDense.class, ManualsSparse.class})
public interface ManualsSearch {}

// Resolved by ContentRetriever type + qualifier:
@Inject @Corpus("products") ContentRetriever productRetriever;
```

When `@Corpus` appears on a `@HybridSearch` interface, the processor propagates the
qualifier to the generated synthetic bean.

---

## Processing Modes

### Standalone — plain interface

```java
@HybridSearch(retrievers = {DenseRetriever.class, SparseRetriever.class},
              reranker   = CohereScorer.class)
public interface ProductSearch {}
// → synthetic ContentRetriever bean typed as (ContentRetriever + ProductSearch)
//   scope: @ApplicationScoped
```

Standalone when the annotated interface does NOT carry `@RegisterAiService`.
`HybridSearchProcessor` produces a `StandaloneHybridSearchBuildItem`. At
`RUNTIME_INIT`, a `SyntheticBeanBuildItem` is created via `HybridSearchRecorder`.

The synthetic bean:
- Types: `ContentRetriever` + the annotated interface
- Qualifiers: any `@Corpus` annotation found on the annotated interface, propagated verbatim
- Scope: `@ApplicationScoped`
- Injection points: one per retriever class (by exact type), one for `ScoringModel`
  (by exact type) if reranker is set, one for `ManagedExecutor` if `retrievers.length > 1`

### Companion — alongside @RegisterAiService

```java
@RegisterAiService
@HybridSearch(retrievers = {DenseRetriever.class, SparseRetriever.class},
              reranker   = CohereScorer.class)
public interface ProductAI { ... }
```

Sugar for the common case: "add hybrid search to my AI service." Companion is detected
when the annotated interface also carries `@RegisterAiService`.

**No separate ContentRetriever bean is created in companion mode.** The annotated interface
IS the AI service interface (`ProductAI`). Creating a ContentRetriever bean also typed as
`ProductAI` would conflict with the AI service bean — CDI throws
`AmbiguousResolutionException`. Inline construction avoids this.

The implementation follows the same pattern as `@RagPipeline` companion:

- `HybridSearchProcessor` produces a `HybridSearchBuildItem` carrying the
  `HybridSearchCreateInfo` + the AI service class name.
- `AiServicesProcessor.handleDeclarativeServices` consumes `HybridSearchBuildItem` and
  calls `HybridSearchProcessor.addHybridSearchInjectionPoints(configurator, createInfo)`
  — adding injection points for the raw retrievers, optional reranker, and optional
  `ManagedExecutor` directly onto the AI service's own `SyntheticBeanBuildItem`
  configurator. This is package-private static, parallel to `addRagInjectionPoints`.
- The recorder builds `HybridSearchContentRetriever` inline during AI service creation,
  wraps it in a `DefaultRetrievalAugmentor`, and wires it to the service.

If external reachability of the `ContentRetriever` matters, use the split-interface form:

```java
@HybridSearch(retrievers = {DenseRetriever.class, SparseRetriever.class})
public interface ProductSearch {}  // ContentRetriever — reachable by other code

@RegisterAiService
@RagPipeline(retrievers = {ProductSearch.class}, transformer = HydeTransformer.class)
public interface ProductAI { ... }
```

**Constraint:** `@HybridSearch` + `@RagPipeline` on the same `@RegisterAiService`
interface → `DeploymentException`. Both own retrieval on the same service and cannot
coexist. The split-interface form above is the resolution.

---

## Validation Rules

| Condition | Result |
|-----------|--------|
| `retrievers` empty | `DeploymentException` |
| `retrievers.length == 1` AND `reranker == void.class` | `DeploymentException` — inject the retriever directly, or add a reranker to justify `@HybridSearch` |
| `@HybridSearch` + `@RagPipeline` on same `@RegisterAiService` | `DeploymentException` |
| Annotated target is not an interface | `DeploymentException` |
| Any retriever class not a CDI bean assignable to `ContentRetriever` | `DeploymentException` |
| `reranker` class not a CDI bean assignable to `ScoringModel` | `DeploymentException` |

Single-retriever + reranker (`retrievers.length == 1`, `reranker` set) is valid — "reranked
single retrieval" is a legitimate use case. `fusionK` and `maxCandidatesForReranking` are
ignored in single-retriever mode.

Note: using `@HybridSearch(retrievers = {X})` with no reranker solely to propagate a
`@Corpus` qualifier onto an existing bean is not supported. Use a CDI producer instead:
```java
@Produces @Corpus("products") @ApplicationScoped
ContentRetriever productSearch(ThirdPartyRetriever r) { return r; }
```

---

## Runtime: HybridSearchContentRetriever

New class in `core/runtime`. Implements `ContentRetriever`. Created by
`HybridSearchRecorder` via `SyntheticCreationalContext`.

### retrieve() algorithm

```
retrieve(Query query):

  // 1. Execute retrieval legs
  if retrievers.size() == 1:
      results ← retrievers[0].retrieve(query)       // List<Content>
  else:
      futures ← submit each retriever.retrieve(query) to ManagedExecutor
      resultLists ← await all futures              // List<List<Content>>
      results ← rrfFuse(resultLists, fusionK)      // List<Content>

  // 2. Optional: trim candidates before reranking
  if reranker != null AND maxCandidatesForReranking > 0:
      results ← results.subList(0, min(results.size(), maxCandidatesForReranking))

  // 3. Optional reranking
  if reranker != null:
      segments ← results.stream().map(Content::textSegment).toList()  // List<TextSegment>
      scores   ← reranker.scoreAll(segments, query.text()).content()   // List<Double>
      // zip scores with original Content objects (parallel lists, same order)
      // sort descending by score — pairs must stay aligned through sort
      results  ← sortedByScoreDescending(zip(results, scores))

  return results

rrfFuse(resultLists, k):
  // Identity key: TextSegment.text() — same text from different retrievers = same document
  // For duplicates: keep Content from the highest-ranked occurrence
  //   (earliest position across lists; ties broken by list index, lower index wins)
  scoreMap ← {}   // Map<String, Double>  (text → accumulated RRF score)
  contentMap ← {} // Map<String, Content> (text → best Content for this text)
  for each list L at index i in resultLists:
    for each Content c at rank r (1-based) in L:
      text ← c.textSegment().text()
      scoreMap[text] += 1.0 / (k + r)
      if text not in contentMap or (i, r) < contentMap[text].bestPosition:
        contentMap[text] = c
  return contentMap.values() sorted descending by scoreMap[text]
```

**Error handling:** if any retriever throws during parallel execution, the exception
propagates immediately (fail-fast). Partial results from remaining retrievers are
discarded. Consistent with `DefaultRetrievalAugmentor`'s parallel retrieval behaviour.

**ManagedExecutor:** injected only when `retrievers.length > 1`. Propagates CDI request
context, OTel spans, and security context across retrieval threads. Not exposed as an
annotation attribute — platform manages execution context.

---

## Build-time Components

| Artifact | Location | Purpose |
|----------|----------|---------|
| `HybridSearch.java` | runtime | Annotation |
| `Corpus.java` | runtime | CDI qualifier annotation |
| `HybridSearchCreateInfo.java` | runtime | Record: retriever class names, reranker class name (nullable), fusionK, maxCandidatesForReranking |
| `HybridSearchContentRetriever.java` | runtime | Runtime `ContentRetriever` implementation |
| `HybridSearchRecorder.java` | runtime | `@Recorder` — creates `HybridSearchContentRetriever` via `SyntheticCreationalContext`; also provides `buildContentRetriever(ctx, info)` for inline use in companion mode |
| `HybridSearchBuildItem.java` | deployment | **Companion mode only**: carries AI service class name + `HybridSearchCreateInfo`; consumed by `AiServicesProcessor` |
| `StandaloneHybridSearchBuildItem.java` | deployment | **Standalone mode only**: carries annotated interface `DotName` + `HybridSearchCreateInfo`; triggers synthetic bean creation in `RUNTIME_INIT` |
| `HybridSearchProcessor.java` | deployment | Scans `@HybridSearch`, validates, produces build items; exposes static `addHybridSearchInjectionPoints()` (package-private, parallel to `addRagInjectionPoints`) |
| `LangChain4jDotNames.java` | deployment | Add `HYBRID_SEARCH`, `CORPUS` `DotName` constants |
| `AiServicesProcessor.java` | deployment | Consume `HybridSearchBuildItem` for companion wiring, parallel to `RagPipelineBuildItem` handling |

**Injection points** on the standalone synthetic bean (created by processor, mirrored
exactly in `HybridSearchRecorder` via `ctx.getInjectedReference()`):
- One per retriever class — by exact class type
- One for `ScoringModel` by exact class type — only when reranker is set
- One for `ManagedExecutor` — only when `retrievers.length > 1`

---

## Tests

| Test class | What it covers |
|------------|---------------|
| `HybridSearchStandaloneTest` | Two retrievers, RRF fusion, standalone `ContentRetriever` injection by annotated interface type |
| `HybridSearchSingleRetrieverRerankerTest` | Single retriever + reranker — valid; `fusionK` ignored |
| `HybridSearchCompanionTest` | `@HybridSearch` on `@RegisterAiService` — AI service wired correctly, no separate `ContentRetriever` bean |
| `HybridSearchCorpusQualifierTest` | `@Corpus` propagated to generated bean; two `@HybridSearch` beans disambiguated by `@Corpus` at `ContentRetriever` injection site |
| `HybridSearchContextPropagationTest` | CDI request context survives `ManagedExecutor` parallel retrieval (both legs see same request context) |
| `HybridSearchMaxCandidatesTest` | `maxCandidatesForReranking` trims fused results before `scoreAll()`; excess candidates not scored |
| `HybridSearchRrfDeduplicationTest` | Same document text returned by two retrievers: appears once in output, RRF score is sum of both-list contributions, `Content` from higher-ranked occurrence is kept |
| `HybridSearchConflictValidationTest` | `@HybridSearch` + `@RagPipeline` on same service → `DeploymentException` |
| `HybridSearchEmptyRetrieversValidationTest` | Empty `retrievers[]` → `DeploymentException` |
| `HybridSearchSingleNoRerankerValidationTest` | Single retriever, no reranker → `DeploymentException` with message directing user to inject directly or add reranker |

---

## Decision Log (PR Notes)

The following records the alternatives considered and why each decision was made.
Included in the PR description for reviewer context.

---

**Decision 1 — `@HybridSearch` produces `ContentRetriever`; companion mode uses inline construction**

*Chosen:* `@HybridSearch` always produces a `ContentRetriever`. In standalone mode, a
synthetic `ContentRetriever` bean is registered. In companion mode (on `@RegisterAiService`),
the retriever is built inline during AI service construction — no separate bean — following
the same pattern as `@RagPipeline` companion.

*Alternative A — always produces `RetrievalAugmentor`:* Rejected. Duplicates `@RagPipeline`'s
responsibility. No principled selection criterion between the two.

*Alternative B — produces `ContentRetriever` bean in both modes, typed as annotated interface:*
Rejected. In companion mode, the annotated interface IS the AI service interface. A second
CDI bean typed as that interface conflicts with the AI service bean —
`AmbiguousResolutionException`. A generated-qualifier workaround adds complexity for no
structural gain. Inline construction is simpler and consistent with how `@RagPipeline`
companion works.

*Why companion mode exists:* symmetry with `@RagPipeline` companion; avoids requiring an
extra interface definition for the simple "add hybrid search to my service" case. When
external reachability of the `ContentRetriever` matters, the split-interface form is
the correct choice.

---

**Decision 2 — `retrievers[]` of `ContentRetriever` beans, not named `dense`/`sparse` slots**

*Chosen:* Array of `ContentRetriever` beans. Quarkus fuses outputs. No named slots.

*Alternative — named `dense`/`sparse` slots with `EmbeddingModel` + `EmbeddingStore`
components:* Rejected for two independent reasons.

First: `EmbeddingModel` is a type lie. No `SparseEmbeddingModel` interface exists in
LangChain4j 1.16.2. The interface returns `float[]` (dense) regardless. A processor
cannot detect a dense model passed in a `sparseModel` slot. The user silently gets two
dense searches fused — correctness failure with no signal.

Second: the shared-store assumption breaks for the primary sparse retrieval implementations.
Qdrant stores dense and sparse vectors in separate named vector fields; calling
`EmbeddingStore.search()` twice with different query embeddings does not compose into
native hybrid search.

*Alternative — named `dense`/`sparse` slots with `ContentRetriever` beans:* Closer, but
rejected. "Dense" and "sparse" are implementation assumptions encoded as API. Two dense
retrievers over different corpora is valid hybrid search; named slots cannot express it
without lying about what goes in which slot.

*Future path:* If LangChain4j ships `SparseEmbeddingModel`, a component-shorthand mode
(model + store → processor assembles the retriever) can be added on top without breaking
the retriever-based design.

---

**Decision 3 — `fusionK = 60` int attribute, RRF baked in, no `RetrievalFusionStrategy` SPI**

*Chosen:* RRF is the fusion strategy. k is configurable per annotation as an int attribute.

*Alternative — `RetrievalFusionStrategy` SPI with `RrfFusionStrategy` as `@DefaultBean`:*
Rejected. RRF covers the overwhelming majority of use cases. The only meaningful tunable
is k. Per-annotation k configuration via a SPI requires multiple qualified beans — worse
DX than a single int. If non-RRF fusion is genuinely needed, implement a `ContentRetriever`
that handles it internally; that is the purpose of the abstraction.

---

**Decision 4 — `reranker` slot accepts `ScoringModel` bean**

*Chosen:* `ScoringModel` — `scoreAll(List<TextSegment>, String query) → Response<List<Double>>`.

*Alternative — `ContentRetriever` for reranking:* Rejected. Wrong type contract. A reranker
scores existing content; it does not issue new retrieval calls. Using `ContentRetriever`
would be structurally identical to the `EmbeddingModel`-for-sparse type lie in Decision 2.

---

**Decision 5 — `maxCandidatesForReranking = -1` (no default trimming)**

*Chosen:* Attribute present; default `-1` means pass all fused results to the reranker.
Production operators deploying expensive cross-encoders set this explicitly.

*Alternative — default to 20 (silent trim):* Rejected. A silent trim-to-20 makes documents
ranked 21+ disappear from reranker input regardless of their true relevance. Hard to debug.
The operator who knows their reranker's cost profile is the right person to set this limit.

---

**Decision 6 — Companion mode: `@HybridSearch` + `@RagPipeline` on same service is an error**

*Chosen:* `DeploymentException`. Both own retrieval and cannot coexist on one service.
Split-interface form resolves the composition case.

*Alternative — @HybridSearch auto-wires as a retriever inside @RagPipeline on the same service:*
Rejected. Implicit dependency between two annotations on the same class creates invisible
coupling and an ordering requirement on processors. Explicit composition via split-interface
is unambiguous.

---

**Decision 7 — `@Corpus` lands in this PR**

*Chosen:* CDI qualifier defined here, processor propagation in `HybridSearchProcessor`.

*Alternative — defer to `@DocumentIngestion` PR (#2576):* Rejected. `@Corpus` is needed
by `@HybridSearch` first. Deferring means this PR ships without qualifier propagation
and #2576 retrofits it with no benefit to either PR.
