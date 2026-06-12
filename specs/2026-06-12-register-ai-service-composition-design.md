# @RegisterAiService Simplification + Composition Annotations — Design Spec

**Date:** 2026-06-12 (revised 2026-06-13)
**Issue:** [#2572](https://github.com/quarkiverse/quarkus-langchain4j/issues/2572)
**Covers:** #2572, #2574, #2575, #2576, #2578, neural-text#16
**Approach:** Composable annotation layers (Approach B)
**Breaking changes:** Yes — supplier-class attributes removed, auto-discovery for optional components removed, attributes migrated to composition annotations

---

## Overview

Replace `@RegisterAiService`'s supplier-class attribute pattern with direct bean-class references, and add composable annotations for RAG, ingestion, search, tenancy, metadata, caching, and collection lifecycle. One spec, multiple PRs.

**Design principles:**
- Explicit over auto-discover — optional components are absent unless declared
- `void.class` = disabled, interface type = auto-discover CDI bean, concrete class = inject specific bean
- Each composition annotation owns its concern exclusively — no dual-path coexistence with @RegisterAiService attributes
- When a composition annotation lands, the @RegisterAiService attributes it replaces are removed — not deprecated, not coexisted with conflict guards
- All component references are `Class<?>` resolved as CDI beans at build time
- Reactive bridging via Quarkus platform dispatch (return type detection), not annotation flags

---

## 1. Foundation — @RegisterAiService Cleanup

### Current state

10 supplier-class attributes (`Class<? extends Supplier<T>>`) with 17 sentinel marker inner classes. Each marker's `get()` throws `UnsupportedOperationException`. `DeclarativeAiServiceCreateInfo` record has a 30-parameter constructor with string class names. Processor and recorder have large if-else chains comparing class name strings against sentinel names.

### New design

All supplier-class attributes become `Class<?>` with direct bean-class references. All 17 sentinel marker inner classes are deleted.

**Attribute changes:**

| Old attribute | New attribute | Type | Default |
|---------------|---------------|------|---------|
| `chatLanguageModelSupplier` | *removed* | — | Use `modelName` only |
| `streamingChatLanguageModelSupplier` | *removed* | — | Auto-detected from return types |
| `chatMemoryProviderSupplier` | `chatMemoryProvider` | `Class<?>` | `ChatMemoryProvider.class` (auto-discover) |
| `chatMemoryFlushStrategySupplier` | `chatMemoryFlushStrategy` | `Class<?>` | `void.class` |
| `retrievalAugmentor` (supplier) | `retrievalAugmentor` | `Class<?>` | `void.class` |
| `moderationModelSupplier` | `moderationModel` | `Class<?>` | `void.class` |
| `toolProviderSupplier` | `toolProvider` | `Class<?>` | `void.class` |
| `toolSearchStrategySupplier` | `toolSearchStrategy` | `Class<?>` | `void.class` |
| `toolHallucinationStrategy` | `toolHallucinationStrategy` | `Class<?>` | `void.class` |
| `systemMessageProviderSupplier` | `systemMessageProvider` | `Class<?>` | `void.class` |
| `maxSequentialToolInvocations` | *removed* | — | Already `@Deprecated(forRemoval = true)` |

**Attributes remaining on @RegisterAiService after all composition annotations land:**
`modelName`, `chatMemoryProvider`, `chatMemoryFlushStrategy`, `tools`, `toolProvider`, `toolSearchStrategy`, `toolHallucinationStrategy`, `moderationModel`, `systemMessageProvider`, `maxToolCallingRoundTrips`, `maxToolCallsPerResponse`, `allowContinuousForcedToolCalling`, `shouldThrowExceptionOnEventError`

**Attributes removed from @RegisterAiService by later PRs:**
`retrievalAugmentor` — removed when `@RagPipeline` lands (PR 2). RAG is only configurable via `@RagPipeline` after that PR.

**Resolution semantics:**

| Value | Meaning | Processor action |
|-------|---------|-----------------|
| `void.class` | Disabled / not configured | Skip — do not inject |
| Interface type (e.g. `ChatMemoryProvider.class`) | Auto-discover | `Instance<T>.isResolvable()` check |
| Concrete class | Use this specific bean | Inject bean by type |

**Deleted:** All 17 sentinel marker inner classes (`BeanChatLanguageModelSupplier`, `BeanStreamingChatLanguageModelSupplier`, `BeanChatMemoryProviderSupplier`, `NoChatMemoryProviderSupplier`, `NoRetriever`, `NoToolProviderSupplier`, `BeanIfExistsRetrievalAugmentorSupplier`, `NoRetrievalAugmentorSupplier`, `BeanIfExistsModerationModelSupplier`, `BeanIfExistsImageModelSupplier`, `BeanIfExistsToolProviderSupplier`, `BeanIfExistsToolSearchStrategySupplier`, `NoToolSearchStrategySupplier`, `BeanIfExistsToolHallucinationStrategy`, `NoSystemMessageProviderSupplier`, `BeanIfExistsSystemMessageProviderSupplier`, `DefaultChatMemoryFlushStrategySupplier`).

### Migration table

| Current usage | New equivalent | Notes |
|---------------|----------------|-------|
| `chatLanguageModelSupplier = BeanChatLanguageModelSupplier.class` (default) | Remove attribute — `modelName` handles model selection | |
| `chatLanguageModelSupplier = MyCustomSupplier.class` | Convert `MyCustomSupplier` to a CDI bean producing `ChatModel`; use `modelName` | Test suppliers must become CDI beans |
| `streamingChatLanguageModelSupplier = FakeStreamedChatModelSupplier.class` | Convert to CDI `@Produces StreamingChatModel` bean | Test migration required |
| `chatMemoryProviderSupplier = BeanChatMemoryProviderSupplier.class` (default) | `chatMemoryProvider = ChatMemoryProvider.class` (default — remove attribute) | |
| `chatMemoryProviderSupplier = NoChatMemoryProviderSupplier.class` | `chatMemoryProvider = void.class` | |
| `chatMemoryProviderSupplier = MySupplier.class` | `chatMemoryProvider = MyProvider.class` (the bean itself, not a supplier wrapping it) | |
| `retrievalAugmentor = BeanIfExistsRetrievalAugmentorSupplier.class` (default) | Remove attribute — default is now `void.class` (disabled) | **Breaking:** services that relied on auto-discovery must add `retrievalAugmentor = MyAugmentor.class` or `@RagPipeline` |
| `retrievalAugmentor = NoRetrievalAugmentorSupplier.class` | Remove attribute — `void.class` is the default | |
| `retrievalAugmentor = MySupplier.class` | `retrievalAugmentor = MyAugmentor.class` | |
| `moderationModelSupplier = BeanIfExistsModerationModelSupplier.class` (default) | Remove attribute — `void.class` is the default | **Breaking:** services with `@Moderate` must add `moderationModel = MyModerator.class` |
| `toolProviderSupplier = BeanIfExistsToolProviderSupplier.class` (default) | Remove attribute — `void.class` is the default | **Breaking:** services relying on auto-discovered `ToolProvider` must add `toolProvider = MyProvider.class` |
| `toolProviderSupplier = NoToolProviderSupplier.class` | Remove attribute — `void.class` is the default | |
| `toolSearchStrategySupplier = BeanIfExistsToolSearchStrategySupplier.class` (default) | Remove attribute — `void.class` is the default | |
| `toolHallucinationStrategy = BeanIfExistsToolHallucinationStrategy.class` (default) | Remove attribute — `void.class` is the default | |

**Test migration:** 30+ tests use non-CDI `Supplier<ChatModel>` implementations. In the new design, all models must be CDI beans. Each test supplier becomes either an `@ApplicationScoped` inner static class or a `@Produces` method. This is mechanical but touches many test files.

### Breaking change justification

The supplier pattern was an internal workaround for the absence of CDI-friendly resolution on upstream annotations. It leaked implementation complexity into the user API — 17 marker classes whose only purpose is to signal "use CDI" or "skip" to the processor. Direct bean-class references are simpler, type-safe at build time, and self-documenting.

**Auto-discovery removal justification:** Optional components (retrieval augmentor, tool provider, moderation model, tool search strategy) previously used `BeanIfExists` auto-discovery — any CDI bean of the matching type was silently wired into every AI service. This made AI services non-self-describing: you couldn't look at the annotation and know what the service had. A global `RetrievalAugmentor` bean meant every service got RAG, requiring explicit opt-out markers on services that shouldn't have it. Explicit declaration makes each AI service self-contained and predictable.

### Processor changes

- `getSupplierDotName()` replaced with `getComponentDotName()` — returns the class reference directly, checks for `void.class` (skip) vs. interface type (auto-discover) vs. concrete (inject)
- `validateSupplierAndRegister()` simplified — no supplier unwrapping, just validate the class exists and is a CDI bean
- `handleDeclarativeServices()` — string comparisons against sentinel class names replaced with enum-based resolution mode (`SKIP`, `AUTO_DISCOVER`, `EXPLICIT`)
- `DeclarativeAiServiceCreateInfo` — cleaner record with resolution mode per component instead of nullable string class names

### Recorder changes

- `createDeclarativeAiService()` — no more if-else chains comparing class name strings. Resolution mode drives the wiring:
  - `SKIP` → don't wire
  - `AUTO_DISCOVER` → `Instance<T>`, use if resolvable
  - `EXPLICIT` → `creationalContext.getInjectedReference(loadClass(className))`

### Cross-cutting: agent-implied AI services

Agent interfaces are processed as AI services via `AnnotationsImpliesAiServiceBuildItem` → `AiServicesProcessor.determinedImpliedRegisterAiService()` → `DeclarativeAiServiceBuildItem`. The resolution mode enum (`SKIP`/`AUTO_DISCOVER`/`EXPLICIT`) in `DeclarativeAiServiceCreateInfo` must work for these agent-implied services too. The implied `@RegisterAiService` uses the same defaults (void.class for optional, interface type for required) — agents don't get special treatment.

---

## 2. @RagPipeline

Composable query-side RAG pipeline. Maps to `DefaultRetrievalAugmentor.builder()`. **When this PR lands, `retrievalAugmentor` is removed from `@RegisterAiService`.**

```java
@Retention(RUNTIME)
@Target(TYPE)
public @interface RagPipeline {
    Class<?>[] retrievers() default {};
    Class<?> router() default void.class;
    Class<?> transformer() default void.class;
    Class<?> aggregator() default void.class;
    Class<?> injector() default void.class;
}
```

| Attribute | Type | Default | Maps to |
|-----------|------|---------|---------|
| `retrievers` | `Class<?>[]` | `{}` | `contentRetriever` (single) / `queryRouter` (multiple via auto-created `DefaultQueryRouter`) |
| `router` | `Class<?>` | `void.class` | `queryRouter` — explicit router overrides multi-retriever default |
| `transformer` | `Class<?>` | `void.class` | `queryTransformer` — single `QueryTransformer` CDI bean |
| `aggregator` | `Class<?>` | `void.class` | `contentAggregator` — defaults to `DefaultContentAggregator` (concatenation) |
| `injector` | `Class<?>` | `void.class` | `contentInjector` — defaults to `DefaultContentInjector` |

**Why single `transformer`, not `transformers[]`:** Upstream's `QueryTransformer.transform()` returns `Collection<Query>` — it's fan-out, not pipeline. Chaining fan-out transformers requires explicit semantics (flatMap? first-only? cartesian product?) that can't be defaulted safely. A single `Class<?>` reference keeps the annotation honest. Users who need multi-transformer composition write a CDI bean that orchestrates the chain internally — explicit, testable, no hidden fan-out semantics.

**Executor:** The processor always injects a `ManagedExecutor` for context propagation when multiple retrievers are routed in parallel. Not exposed as an annotation attribute — the platform manages execution context.

**Validation:** At least one retriever OR a router must be specified. All classes resolved as CDI beans.

---

## 3. @HybridSearch

Retrieval internals — how a single `ContentRetriever` executes. Annotates a `ContentRetriever` interface, not the AI service.

```java
@Retention(RUNTIME)
@Target(TYPE)
public @interface HybridSearch {
    Class<?> denseModel() default EmbeddingModel.class;
    Class<?> sparseModel() default void.class;
    Class<?> store() default EmbeddingStore.class;
    Class<?> reranker() default void.class;
    Class<?> fusion() default void.class;
    int maxResults() default 3;
    double minScore() default 0.0;
    Class<?> filter() default void.class;
}
```

**Usage:**

```java
@HybridSearch(denseModel = OnnxModel.class, store = ProductStore.class, reranker = CohereScorer.class)
public interface ProductRetriever extends ContentRetriever {}
```

| Attribute | Type | Default | Expected bean type |
|-----------|------|---------|-------------------|
| `denseModel` | `Class<?>` | `EmbeddingModel.class` (auto-discover) | `EmbeddingModel` — functionally required |
| `sparseModel` | `Class<?>` | `void.class` (disabled) | `EmbeddingModel` — custom sparse embedding bean |
| `store` | `Class<?>` | `EmbeddingStore.class` (auto-discover) | `EmbeddingStore` — functionally required |
| `reranker` | `Class<?>` | `void.class` (disabled) | `ScoringModel` (upstream `dev.langchain4j.model.scoring.ScoringModel`) |
| `fusion` | `Class<?>` | `void.class` (disabled) | `RetrievalFusionStrategy` (new SPI — see Section 10) |
| `filter` | `Class<?>` | `void.class` (disabled) | `Filter` (upstream) — static filter bean. Dynamic filters use `Function<Query, Filter>` CDI bean |

**Defaults use interface types for required attributes:** `denseModel` defaults to `EmbeddingModel.class` and `store` defaults to `EmbeddingStore.class` — consistent with the tri-state contract (`interface type = auto-discover`). These are functionally required; if no bean is resolvable → `DeploymentException`.

**Processor behaviour:**
- Dense only → single `EmbeddingStoreContentRetriever`
- Dense + sparse → two retrievers, fused (built-in `RrfFusionStrategy` by default, or explicit fusion bean)
- Reranker → post-retrieval scoring/reranking step wrapping the retriever(s), using upstream's `ScoringModel`
- The processor generates the `ContentRetriever` implementation bean

---

## 4. @DocumentIngestion

Write-side ingestion pipeline. Maps to `EmbeddingStoreIngestor.builder()` plus pre-parse step.

```java
@Retention(RUNTIME)
@Target(TYPE)
public @interface DocumentIngestion {
    Class<?> parser() default void.class;
    Class<?> splitter() default void.class;
    Class<?> documentTransformer() default void.class;
    Class<?> segmentTransformer() default void.class;
    Class<?> embeddingModel() default EmbeddingModel.class;
    Class<?> store() default EmbeddingStore.class;
}
```

**Usage:**

```java
@DocumentIngestion(
    parser = TikaDocumentParser.class,
    splitter = ParagraphSplitter.class,
    store = ProductStore.class
)
public interface ProductIngestor {}
```

**Defaults use interface types for required attributes:** `embeddingModel` defaults to `EmbeddingModel.class` and `store` defaults to `EmbeddingStore.class` — auto-discover. If not resolvable → `DeploymentException`.

**Processor behaviour:**
- Generates a CDI bean wrapping `EmbeddingStoreIngestor.builder()` with declared components
- `parser` is outside upstream's ingestor — composed as a pre-step: parse → transform → split → segment-transform → embed → store

---

## 5. @Corpus

CDI qualifier for multi-corpus applications.

```java
@Qualifier
@Retention(RUNTIME)
@Target({TYPE, METHOD, FIELD, PARAMETER})
public @interface Corpus {
    String value();
}
```

Pure CDI — not processed by the annotation processor. Disambiguates multiple beans of the same type (e.g., two `EmbeddingStore` beans).

When `@HybridSearch` or `@DocumentIngestion` interfaces also carry `@Corpus`, the processor propagates the qualifier to the generated bean.

**Usage:**

```java
@Corpus("products")
@HybridSearch(denseModel = OnnxModel.class, store = ProductStore.class)
public interface ProductRetriever extends ContentRetriever {}

@Produces @Corpus("products") @ApplicationScoped
EmbeddingStore<TextSegment> productStore() { return qdrantStore("products"); }
```

---

## 6. @TenantIsolation

Cross-cutting multi-tenancy — weaves into `@RagPipeline` (query filtering) and `@DocumentIngestion` (metadata tagging).

```java
@Retention(RUNTIME)
@Target(TYPE)
public @interface TenantIsolation {
    Class<?> strategy();
    Class<?> tenantResolver();
}
```

**Both attributes are required.** No defaults, no fallbacks.

**SPI (quarkus-langchain4j runtime):**

```java
public interface TenantIsolationStrategy {
    Filter tenantFilter(String tenantId, Query query);
    String collectionName(String tenantId, String baseCollectionName);
    Metadata tenantMetadata(String tenantId, Metadata existing);
}

public interface TenantResolver {
    String resolve();
}
```

**Built-in strategies:**
- `FilterBasedTenantStrategy` — `tenant_id` filter on queries, `tenant_id` metadata on ingested documents. Single collection.
- `PrefixBasedTenantStrategy` — tenant-prefixed collection names. Separate collections per tenant.

**No fallback to @MemoryId.** Tenant identity and memory identity are architecturally distinct. A memory ID is per-conversation-session. A tenant ID is per-organization or per-user-account. Conflating them produces subtle bugs in any system where they differ (e.g., admin reviewing another tenant's conversations). `tenantResolver` must be explicitly set → no implicit guessing.

**Validation:** `@TenantIsolation` requires `@RagPipeline` or `@DocumentIngestion` on the same class. Both `strategy` and `tenantResolver` must be resolvable CDI beans.

---

## 7. @MetadataExtractor

Composable metadata extraction during ingestion.

```java
@Retention(RUNTIME)
@Target(TYPE)
public @interface MetadataExtractor {
    Class<?>[] extractors();
}
```

**SPI (quarkus-langchain4j runtime):**

```java
public interface DocumentMetadataExtractor {
    Metadata extract(Document document);
}
```

Each extractor returns metadata for the document. Results merged in order (later extractors overwrite conflicting keys). Composed into `@DocumentIngestion` as a `DocumentTransformer` step — runs after parsing, before splitting.

**Validation:** Requires `@DocumentIngestion` on the same class.

---

## 8. @EmbeddingCache

Content-hash caching decorator for `EmbeddingModel`.

```java
@Retention(RUNTIME)
@Target(TYPE)
public @interface EmbeddingCache {
    Class<?> store();
}
```

**SPI (quarkus-langchain4j runtime):**

```java
public interface EmbeddingCacheStore {
    Embedding get(String contentHash);
    void put(String contentHash, Embedding embedding);
}
```

**Built-in:** `InMemoryEmbeddingCacheStore` (`ConcurrentHashMap`-backed, JVM lifetime).

The processor wraps the `EmbeddingModel` with a caching decorator. SHA-256 content hash, transparent to the rest of the pipeline.

**Composable:** Can go on `@HybridSearch` interfaces or `@DocumentIngestion` interfaces — anywhere an `EmbeddingModel` is used.

**Validation:** Requires `@HybridSearch` or `@DocumentIngestion` on the same class.

---

## 9. @VectorStoreCollection

Declarative collection lifecycle — create-if-absent, validate schema on startup.

```java
@Retention(RUNTIME)
@Target(TYPE)
public @interface VectorStoreCollection {
    String name();
    int dimensions() default 0;
    DistanceMetric distance() default DistanceMetric.COSINE;
    NamedVector[] vectors() default {};
    boolean createIfAbsent() default true;
    boolean validateOnStartup() default true;
}

public @interface NamedVector {
    String name();
    int dimensions() default 0;
    DistanceMetric distance() default DistanceMetric.COSINE;
}

public enum DistanceMetric {
    COSINE, DOT_PRODUCT, EUCLIDEAN
}
```

**SPI (quarkus-langchain4j runtime):**

```java
public interface CollectionManager {
    void createCollection(String name, CollectionSchema schema);
    boolean collectionExists(String name);
    CollectionSchema describeCollection(String name);
}
```

Each vector store extension (quarkus-langchain4j-qdrant, etc.) provides a `CollectionManager` bean.

**Processor behaviour:**
- Generates startup observer that calls `CollectionManager` with the declared schema
- `dimensions = 0` → infer from `EmbeddingModel.dimension()` at startup
- `validateOnStartup = true` → check existing collection matches; `DeploymentException` on mismatch
- Interaction with `@TenantIsolation` — `PrefixBasedTenantStrategy` derives collection names from `@VectorStoreCollection.name` + tenant prefix

**Validation:** Requires `@HybridSearch` or `@DocumentIngestion` on the same class. `CollectionManager` bean must be resolvable.

---

## 10. New SPIs

Six new SPIs defined in quarkus-langchain4j runtime:

| SPI | Purpose | PR |
|-----|---------|-----|
| `TenantIsolationStrategy` | Tenant filtering, collection naming, metadata tagging | 6 |
| `TenantResolver` | Resolves the current tenant ID from request context | 6 |
| `DocumentMetadataExtractor` | Structured metadata extraction from documents | 4 |
| `EmbeddingCacheStore` | Content-hash embedding cache | 5 |
| `CollectionManager` | Vector store collection lifecycle | 6 |
| `RetrievalFusionStrategy` | Fusion of dense + sparse retrieval results | 3 |

**`RetrievalFusionStrategy`:**

```java
public interface RetrievalFusionStrategy {
    List<Content> fuse(List<Content> denseResults, List<Content> sparseResults);
}
```

Built-in: `RrfFusionStrategy` (Reciprocal Rank Fusion). Used by `@HybridSearch` when both `denseModel` and `sparseModel` are set.

All SPIs follow the same pattern: interface in runtime module, CDI bean resolution at build time.

---

## 11. Processor Architecture

### Current state

`AiServicesProcessor` (2945 lines) generates one kind of synthetic bean: AI service proxies backed by `QuarkusAiServiceContext`. The agentic module's `AgenticProcessor` produces `AnnotationsImpliesAiServiceBuildItem` to route agent interfaces through this same processor.

### New processors

The composition annotations introduce fundamentally new code-generation pipelines. These are NOT extensions of `AiServicesProcessor` — they are separate `@BuildStep` processors with their own recorder methods.

| Processor | Annotations | Generates | Recorder |
|-----------|-------------|-----------|----------|
| `AiServicesProcessor` (existing) | `@RegisterAiService` | `QuarkusAiServiceContext` synthetic bean (AI service proxy) | `AiServicesRecorder` |
| `RagPipelineProcessor` (new) | `@RagPipeline` | `RetrievalAugmentor` synthetic bean via `DefaultRetrievalAugmentor.builder()` | `RagPipelineRecorder` |
| `HybridSearchProcessor` (new) | `@HybridSearch` | `ContentRetriever` synthetic bean via `EmbeddingStoreContentRetriever.builder()` + fusion/reranking | `HybridSearchRecorder` |
| `DocumentIngestionProcessor` (new) | `@DocumentIngestion`, `@MetadataExtractor` | Ingestor synthetic bean via `EmbeddingStoreIngestor.builder()` | `DocumentIngestionRecorder` |
| `VectorStoreProcessor` (new) | `@VectorStoreCollection` | Startup observer for collection lifecycle | `VectorStoreRecorder` |
| `TenantIsolationProcessor` (new) | `@TenantIsolation` | Filter/metadata decoration on retriever and ingestor beans | `TenantIsolationRecorder` |

### Build item flow

New processors produce build items that `AiServicesProcessor` consumes:

```
HybridSearchProcessor
  → produces: ContentRetrieverBuildItem (generated ContentRetriever beans)

RagPipelineProcessor
  → consumes: ContentRetrieverBuildItem (resolves retriever references)
  → produces: RetrievalAugmentorBuildItem (generated RetrievalAugmentor bean)

AiServicesProcessor
  → consumes: RetrievalAugmentorBuildItem (wires into QuarkusAiServiceContext)
```

Each new processor:
- Has its own `@BuildStep` methods for validation, synthetic bean creation, and unremovable bean marking
- Produces `ReflectiveClassBuildItem` for native image support
- Uses `SyntheticBeanBuildItem` to create generated beans with proper injection points
- Operates independently — no modifications to `AiServicesProcessor` beyond consuming the new build items

### Recorder responsibilities

Each new recorder creates the component using upstream's builder API:

```java
// RagPipelineRecorder
public Function<SyntheticCreationalContext<RetrievalAugmentor>, RetrievalAugmentor>
    createRagPipeline(RagPipelineCreateInfo info) { ... }

// HybridSearchRecorder
public Function<SyntheticCreationalContext<ContentRetriever>, ContentRetriever>
    createHybridSearchRetriever(HybridSearchCreateInfo info) { ... }

// DocumentIngestionRecorder
public Function<SyntheticCreationalContext<EmbeddingStoreIngestor>, EmbeddingStoreIngestor>
    createIngestor(DocumentIngestionCreateInfo info) { ... }
```

---

## 12. Reactive Bridging

No new annotation. Quarkus platform dispatch handles this natively. The processor detects `Uni<T>` / `Multi<T>` return types on AI service methods and auto-dispatches blocking RAG/ingestion components to worker threads. Same pattern as RESTEasy Reactive.

---

## 13. Composition Model

### On an AI service interface (@RegisterAiService):

```java
@RegisterAiService(modelName = "gpt-4o")   // core service declaration
@RagPipeline(                               // RAG pipeline (replaces retrievalAugmentor attribute)
    retrievers = {ProductRetriever.class},
    transformer = HydeTransformer.class
)
@TenantIsolation(                           // cross-cutting tenancy
    strategy = FilterBasedTenantStrategy.class,
    tenantResolver = RequestScopedTenantResolver.class
)
public interface ProductAssistant { ... }
```

### On a ContentRetriever interface:

```java
@Corpus("products")                         // qualifier for multi-corpus
@HybridSearch(                              // retrieval internals
    denseModel = OnnxModel.class,
    store = ProductStore.class,
    reranker = CohereScorer.class
)
@VectorStoreCollection(name = "products", dimensions = 384)
@EmbeddingCache(store = RedisCacheStore.class)
public interface ProductRetriever extends ContentRetriever {}
```

### On an ingestor interface:

```java
@Corpus("products")
@DocumentIngestion(
    parser = TikaDocumentParser.class,
    splitter = ParagraphSplitter.class,
    store = ProductStore.class
)
@MetadataExtractor(extractors = {DateExtractor.class, CategoryExtractor.class})
@EmbeddingCache(store = RedisCacheStore.class)
@VectorStoreCollection(name = "products", dimensions = 384)
@TenantIsolation(
    strategy = FilterBasedTenantStrategy.class,
    tenantResolver = RequestScopedTenantResolver.class
)
public interface ProductIngestor {}
```

### Validation rules (build-time DeploymentException):

- `@EmbeddingCache` without `@HybridSearch` or `@DocumentIngestion` → error
- `@MetadataExtractor` without `@DocumentIngestion` → error
- `@VectorStoreCollection` without `@HybridSearch` or `@DocumentIngestion` → error
- `@TenantIsolation` without `@RagPipeline` or `@DocumentIngestion` → error
- `@RagPipeline` with neither retrievers nor router → error
- `@HybridSearch` or `@DocumentIngestion` with required attributes (EmbeddingModel, EmbeddingStore) unresolvable → error

---

## 14. PR Split

One spec, multiple draft PRs. Undraft sequentially as parents merge. Each composition PR removes the @RegisterAiService attributes it replaces.

| PR | Content | Removes from @RegisterAiService | Depends on | Issue |
|---|---------|--------------------------------|-----------|-------|
| 1 | **Foundation** — direct bean-class attributes, delete 17 markers, update processor + recorder | `chatLanguageModelSupplier`, `streamingChatLanguageModelSupplier`, `maxSequentialToolInvocations` | — | #2578 |
| 2 | **@RagPipeline** — composable RAG pipeline + `RagPipelineProcessor` | `retrievalAugmentor` | PR 1 | #2574 |
| 3 | **@HybridSearch + @Corpus** — retriever composition + `HybridSearchProcessor` | — | PR 1 | #2575 |
| 4 | **@DocumentIngestion + @MetadataExtractor** — ingestion pipeline + `DocumentIngestionProcessor` | — | PR 1 | #2576 |
| 5 | **@EmbeddingCache** — caching decorator | — | PR 2 or 3 | File before PR |
| 6 | **@VectorStoreCollection + @TenantIsolation** — collection lifecycle + tenancy + `VectorStoreProcessor` + `TenantIsolationProcessor` | — | PR 2 + 4 | File before PR |

PRs 2–4 depend only on PR 1 and can be reviewed in parallel. PRs 5–6 have deeper dependencies. Issues for PRs 5–6 to be filed before those PRs are created.

### Dev UI impact (deferred)

The current Dev UI shows AI services. `@RagPipeline` configuration, `@HybridSearch` retrievers, and `@DocumentIngestion` pipelines should also appear — this is an L7 concern, not part of this spec. To be designed when the composition annotations are implemented.

### Native image

Each new processor registers `ReflectiveClassBuildItem` entries for generated synthetic bean types, ensuring GraalVM native image compatibility. This follows the existing pattern in `AiServicesProcessor.nativeSupport()`.

---

## 15. Coherence Review

**PLATFORM.md:**
- Does this belong upstream? No — Quarkus CDI composition annotations. Quarkus first.
- Is this CDI-native? Yes — all components resolved as CDI beans.
- Chapter plan conflict? No — core module work, outside agentic chapter plan.
- Upstream coupling? Neutral — composes upstream's existing interfaces declaratively.

**Cross-cutting impact:** The resolution mode enum (`SKIP`/`AUTO_DISCOVER`/`EXPLICIT`) in `DeclarativeAiServiceCreateInfo` affects agent-implied AI services via `AnnotationsImpliesAiServiceBuildItem`. Agent interfaces processed through `AiServicesProcessor` will use the same resolution semantics. This is correct behaviour — agents don't get special treatment for component resolution.

**Protocols:** No violations. No Maven coordinate changes, no Flyway migrations, no SPI blocking/reactive parity concerns.

---

## References

- Issue: [#2572](https://github.com/quarkiverse/quarkus-langchain4j/issues/2572)
- Child issues: #2574, #2575, #2576, #2578
- Neural-text tracking: casehubio/neural-text#16
- Convergence roadmap: `roadmap.md`
- Prior fit-gap: `specs/2026-06-08-langchain4j-cdi-fitgap.md`
