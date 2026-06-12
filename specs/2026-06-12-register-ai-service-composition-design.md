# @RegisterAiService Simplification + Composition Annotations — Design Spec

**Date:** 2026-06-12
**Issue:** [#2572](https://github.com/quarkiverse/quarkus-langchain4j/issues/2572)
**Covers:** #2572, #2574, #2575, #2576, #2577, #2578, neural-text#16
**Approach:** Composable annotation layers (Approach B)
**Breaking changes:** Yes — supplier-class attributes removed, auto-discovery for optional components removed

---

## Overview

Replace `@RegisterAiService`'s supplier-class attribute pattern with direct bean-class references, and add composable annotations for RAG, ingestion, search, memory, tools, tenancy, metadata, caching, and collection lifecycle. One spec, multiple PRs.

**Design principles:**
- Explicit over auto-discover — optional components are absent unless declared
- `void.class` = disabled, interface type = auto-discover CDI bean, concrete class = inject specific bean
- Each concern is an independent annotation that composes onto the AI service interface
- Conflict detection at build time — overlapping configuration is a `DeploymentException`
- All component references are `Class<?>` resolved as CDI beans at build time
- Reactive bridging via Quarkus platform dispatch (return type detection), not annotation flags

---

## 1. Foundation — @RegisterAiService Cleanup

### Current state

10 supplier-class attributes (`Class<? extends Supplier<T>>`) with 17 sentinel marker inner classes. Each marker's `get()` throws `UnsupportedOperationException`. `DeclarativeAiServiceCreateInfo` record has a 30-parameter constructor with string class names. Processor and recorder have large if-else chains comparing class name strings against sentinel names.

### New design

All supplier-class attributes become `Class<?>` with direct bean-class references.

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
| `systemMessageProviderSupplier` | `systemMessageProvider` | `Class<?>` | `void.class` |
| `maxSequentialToolInvocations` | *removed* | — | Already `@Deprecated(forRemoval = true)` |

**Unchanged attributes:** `tools`, `modelName`, `maxToolCallingRoundTrips`, `maxToolCallsPerResponse`, `allowContinuousForcedToolCalling`, `shouldThrowExceptionOnEventError`, `toolHallucinationStrategy`

**Resolution semantics:**

| Value | Meaning | Processor action |
|-------|---------|-----------------|
| `void.class` | Disabled / not configured | Skip — do not inject |
| Interface type (e.g. `ChatMemoryProvider.class`) | Auto-discover | `Instance<T>.isResolvable()` check |
| Concrete class | Use this specific bean | Inject bean by type |

**Deleted:** All 17 sentinel marker inner classes (`BeanChatLanguageModelSupplier`, `BeanStreamingChatLanguageModelSupplier`, `BeanChatMemoryProviderSupplier`, `NoChatMemoryProviderSupplier`, `NoRetriever`, `NoToolProviderSupplier`, `BeanIfExistsRetrievalAugmentorSupplier`, `NoRetrievalAugmentorSupplier`, `BeanIfExistsModerationModelSupplier`, `BeanIfExistsImageModelSupplier`, `BeanIfExistsToolProviderSupplier`, `BeanIfExistsToolSearchStrategySupplier`, `NoToolSearchStrategySupplier`, `BeanIfExistsToolHallucinationStrategy`, `NoSystemMessageProviderSupplier`, `BeanIfExistsSystemMessageProviderSupplier`, `DefaultChatMemoryFlushStrategySupplier`).

**Breaking change justification:** The supplier pattern was an internal workaround for the absence of CDI-friendly resolution on upstream annotations. It leaked implementation complexity into the user API — 17 marker classes whose only purpose is to signal "use CDI" or "skip" to the processor. Direct bean-class references are simpler, type-safe at build time, and self-documenting. Every call site that references a sentinel marker must change — the migration is mechanical (replace `XxxSupplier.class` with the actual bean class or `void.class`) and the breakage is the point: it forces every caller to be explicit about what components their AI service uses.

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

---

## 2. @RagPipeline

Composable query-side RAG pipeline. Maps to `DefaultRetrievalAugmentor.builder()`.

```java
@Retention(RUNTIME)
@Target(TYPE)
public @interface RagPipeline {
    Class<?>[] retrievers() default {};
    Class<?> router() default void.class;
    Class<?>[] transformers() default {};
    Class<?> aggregator() default void.class;
    Class<?> injector() default void.class;
}
```

| Attribute | Maps to | Notes |
|-----------|---------|-------|
| `retrievers` | `contentRetriever` / `queryRouter` | 1 retriever → `contentRetriever(bean)`. N retrievers → auto-creates `DefaultQueryRouter`. |
| `router` | `queryRouter` | Explicit router overrides multi-retriever default. |
| `transformers` | `queryTransformer` | Applied in order: `query → transformers[0] → transformers[1] → ... → retriever(s)`. Processor composes a chaining transformer. |
| `aggregator` | `contentAggregator` | Defaults to upstream's RRF. |
| `injector` | `contentInjector` | Defaults to upstream's `DefaultContentInjector`. |

**Validation:** At least one retriever OR a router must be specified. All classes resolved as CDI beans.

**Conflicts with @RegisterAiService:** `@RagPipeline` replaces `retrievalAugmentor` on `@RegisterAiService`. Both present → `DeploymentException`.

---

## 3. @HybridSearch

Retrieval internals — how a single `ContentRetriever` executes. Annotates a `ContentRetriever` interface, not the AI service.

```java
@Retention(RUNTIME)
@Target(TYPE)
public @interface HybridSearch {
    Class<?> denseModel() default void.class;
    Class<?> sparseModel() default void.class;
    Class<?> store() default void.class;
    Class<?> reranker() default void.class;
    Class<?> fusion() default void.class;
    int maxResults() default 3;
    double minScore() default 0.0;
    Class<?> filter() default void.class;
}
```

**Usage:**

```java
@HybridSearch(denseModel = OnnxModel.class, store = ProductStore.class, reranker = CrossEncoder.class)
public interface ProductRetriever extends ContentRetriever {}
```

**Processor behaviour:**
- Dense only → single `EmbeddingStoreContentRetriever`
- Dense + sparse → two retrievers, fused (RRF by default or explicit fusion bean)
- Reranker → post-retrieval reranking step wrapping the retriever(s)
- `denseModel` and `store` default to `void.class` — auto-discover CDI beans of `EmbeddingModel` and `EmbeddingStore`. These are functionally required (can't do embedding search without them), so auto-discovery is appropriate here — same principle as `ChatMemoryProvider.class` default on `@RegisterAiService`. If no bean is resolvable → `DeploymentException`.
- The processor generates the `ContentRetriever` implementation bean
- `filter` → static `Filter` bean. Dynamic filters use a `Function<Query, Filter>` CDI bean referenced separately

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
    Class<?> embeddingModel() default void.class;
    Class<?> store() default void.class;
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

**Processor behaviour:**
- Generates a CDI bean wrapping `EmbeddingStoreIngestor.builder()` with declared components
- `parser` is outside upstream's ingestor — composed as a pre-step: parse → transform → split → segment-transform → embed → store
- `embeddingModel` and `store` default to `void.class` — auto-discover CDI beans. Functionally required; if not resolvable → `DeploymentException`

---

## 5. @MemoryConfig

Groups memory-related attributes.

```java
@Retention(RUNTIME)
@Target(TYPE)
public @interface MemoryConfig {
    Class<?> provider() default ChatMemoryProvider.class;
    Class<?> flushStrategy() default void.class;
}
```

When present, takes ownership of memory configuration. `@RegisterAiService.chatMemoryProvider` set alongside `@MemoryConfig` → `DeploymentException`.

Opt-out: `@MemoryConfig(provider = void.class)` disables chat memory.

---

## 6. @ToolConfig

Groups tool-related attributes.

```java
@Retention(RUNTIME)
@Target(TYPE)
public @interface ToolConfig {
    Class<?>[] tools() default {};
    Class<?> provider() default void.class;
    Class<?> searchStrategy() default void.class;
    Class<?> hallucinationStrategy() default void.class;
}
```

When present, takes ownership of tool configuration. Overlapping attributes on `@RegisterAiService` → `DeploymentException`.

---

## 7. @Corpus

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

## 8. @TenantIsolation

Cross-cutting multi-tenancy — weaves into `@RagPipeline` (query filtering) and `@DocumentIngestion` (metadata tagging).

```java
@Retention(RUNTIME)
@Target(TYPE)
public @interface TenantIsolation {
    Class<?> strategy();
    Class<?> tenantResolver() default void.class;
}
```

**SPI (quarkus-langchain4j runtime):**

```java
public interface TenantIsolationStrategy {
    Filter tenantFilter(String tenantId, Query query);
    String collectionName(String tenantId, String baseCollectionName);
    Metadata tenantMetadata(String tenantId, Metadata existing);
}
```

**Built-in strategies:**
- `FilterBasedTenantStrategy` — `tenant_id` filter on queries, `tenant_id` metadata on ingested documents. Single collection.
- `PrefixBasedTenantStrategy` — tenant-prefixed collection names. Separate collections per tenant.

**Tenant resolver:** If `tenantResolver` is `void.class`, default implementation looks for a `@RequestScoped` CDI bean of type `TenantContext` or falls back to `@MemoryId`.

**Validation:** `@TenantIsolation` requires `@RagPipeline` or `@DocumentIngestion` on the same class.

---

## 9. @MetadataExtractor

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

## 10. @EmbeddingCache

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

## 11. @VectorStoreCollection

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

## 12. Reactive Bridging

No new annotation. Quarkus platform dispatch handles this natively. The processor detects `Uni<T>` / `Multi<T>` return types on AI service methods and auto-dispatches blocking RAG/ingestion components to worker threads. Same pattern as RESTEasy Reactive.

---

## 13. Composition Model

### On an AI service interface (@RegisterAiService):

```java
@RegisterAiService           // required — the root
@RagPipeline(...)            // optional — replaces retrievalAugmentor attribute
@MemoryConfig(...)           // optional — replaces chatMemoryProvider attribute
@ToolConfig(...)             // optional — replaces tools/toolProvider attributes
@TenantIsolation(...)        // optional — cross-cuts RagPipeline and DocumentIngestion
```

### On a ContentRetriever interface:

```java
@HybridSearch(...)           // defines retrieval internals
@Corpus("name")             // qualifier for multi-corpus
@VectorStoreCollection(...)  // collection lifecycle
@EmbeddingCache(...)         // embedding cache decorator
```

### On an ingestor interface:

```java
@DocumentIngestion(...)      // defines ingestion pipeline
@Corpus("name")             // qualifier
@MetadataExtractor(...)      // metadata extraction step
@EmbeddingCache(...)         // embedding cache decorator
@VectorStoreCollection(...)  // collection lifecycle
@TenantIsolation(...)        // tenant metadata injection
```

### Conflict rules (build-time DeploymentException):

- `@RegisterAiService.retrievalAugmentor` + `@RagPipeline` → error
- `@RegisterAiService.chatMemoryProvider` + `@MemoryConfig` → error
- `@RegisterAiService.tools`/`toolProvider` + `@ToolConfig` → error
- `@EmbeddingCache` without `@HybridSearch` or `@DocumentIngestion` → error
- `@MetadataExtractor` without `@DocumentIngestion` → error
- `@VectorStoreCollection` without `@HybridSearch` or `@DocumentIngestion` → error
- `@TenantIsolation` without `@RagPipeline` or `@DocumentIngestion` → error

---

## 14. PR Split

One spec, multiple draft PRs. Undraft sequentially as parents merge.

| PR | Content | Depends on | Issue |
|---|---------|-----------|-------|
| 1 | **Foundation** — direct bean-class attributes, delete 17 markers, update processor + recorder | — | #2578 |
| 2 | **@MemoryConfig + @ToolConfig** — grouping annotations | PR 1 | #2577 |
| 3 | **@RagPipeline** — composable RAG pipeline | PR 1 | #2574 |
| 4 | **@HybridSearch + @Corpus** — retriever composition, CDI qualifier | PR 1 | #2575 |
| 5 | **@DocumentIngestion + @MetadataExtractor** — ingestion pipeline | PR 1 | #2576 |
| 6 | **@EmbeddingCache** — caching decorator | PR 3 or 4 | — |
| 7 | **@VectorStoreCollection + CollectionManager SPI** — collection lifecycle | PR 4 | — |
| 8 | **@TenantIsolation + TenantIsolationStrategy SPI** — cross-cutting tenancy | PR 3 + 5 | — |

PRs 2–5 depend only on PR 1 and can be reviewed in parallel. PRs 6–8 have deeper dependencies.

---

## 15. New SPIs

Four new SPIs defined in quarkus-langchain4j runtime:

| SPI | Purpose | PR |
|-----|---------|-----|
| `TenantIsolationStrategy` | Tenant filtering, collection naming, metadata tagging | 8 |
| `DocumentMetadataExtractor` | Structured metadata extraction from documents | 5 |
| `EmbeddingCacheStore` | Content-hash embedding cache | 6 |
| `CollectionManager` | Vector store collection lifecycle | 7 |

All follow the same pattern: interface in runtime module, CDI bean resolution at build time. A CDI protocol entry should be created documenting this pattern.

---

## 16. Coherence Review

**PLATFORM.md:**
- Does this belong upstream? No — Quarkus CDI composition annotations. Quarkus first.
- Is this CDI-native? Yes — all components resolved as CDI beans.
- Chapter plan conflict? No — core module work, outside agentic chapter plan.
- Upstream coupling? Neutral — composes upstream's existing interfaces declaratively.

**Protocols:** No violations. No Maven coordinate changes, no Flyway migrations, no SPI blocking/reactive parity concerns.

---

## References

- Issue: [#2572](https://github.com/quarkiverse/quarkus-langchain4j/issues/2572)
- Child issues: #2574, #2575, #2576, #2577, #2578
- Neural-text tracking: casehubio/neural-text#16
- Convergence roadmap: `roadmap.md`
- Prior fit-gap: `specs/2026-06-08-langchain4j-cdi-fitgap.md`
