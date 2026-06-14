# @RagPipeline Composition Annotation — Design Spec

**Date:** 2026-06-13 (revised after review)
**Issue:** [#2574](https://github.com/quarkiverse/quarkus-langchain4j/issues/2574)
**Parent:** [#2572](https://github.com/quarkiverse/quarkus-langchain4j/issues/2572)
**Depends on:** PR #2591 (Foundation — #2578)
**Approach:** Separate processor (Approach A)

---

## Overview

Add `@RagPipeline` as a composable annotation for query-side RAG pipeline configuration. It replaces `@RegisterAiService`'s `retrievalAugmentor` attribute with a decomposed, declarative alternative that maps to `DefaultRetrievalAugmentor.builder()`.

Two modes: companion (on AI service interface) and standalone (on a separate interface, reusable across services).

When this PR lands, the `retrievalAugmentor` attribute is removed from `@RegisterAiService`.

---

## 1. Annotation

```java
@Retention(RUNTIME)
@Target(TYPE)
public @interface RagPipeline {
    Class<?> augmentor() default void.class;
    Class<?>[] retrievers() default {};
    Class<?> router() default void.class;
    Class<?> transformer() default void.class;
    Class<?> aggregator() default void.class;
    Class<?> injector() default void.class;
}
```

Package: `io.quarkiverse.langchain4j` (alongside `RegisterAiService`).

| Attribute | Type | Default | Maps to |
|-----------|------|---------|---------|
| `augmentor` | `Class<?>` | `void.class` | Pre-built `RetrievalAugmentor` bean — bypasses decomposed mode |
| `retrievers` | `Class<?>[]` | `{}` | `ContentRetriever` beans → `DefaultQueryRouter` |
| `router` | `Class<?>` | `void.class` | `QueryRouter` bean |
| `transformer` | `Class<?>` | `void.class` | `QueryTransformer` bean |
| `aggregator` | `Class<?>` | `void.class` | `ContentAggregator` bean — upstream `DefaultContentAggregator` when not set |
| `injector` | `Class<?>` | `void.class` | `ContentInjector` bean — upstream `DefaultContentInjector` when not set |

### Resolution model — two-state, not tri-state

All `Class<?>` attributes use **two-state resolution** (SKIP / EXPLICIT):
- `void.class` → SKIP (not configured, use upstream default or don't wire)
- Any concrete class → EXPLICIT (inject that specific CDI bean)

AUTO_DISCOVER is not supported on any `@RagPipeline` attribute. The parent spec's principle is "explicit over auto-discover — optional components are absent unless declared." The Foundation PR specifically removed `RetrievalAugmentor` auto-discovery. Re-introducing it through `@RagPipeline` would contradict that decision. If you know you need a component, you know its type.

In the processor, all `resolveComponent()` calls pass `null` as `interfaceType` — the same pattern used by `chatMemoryFlushStrategy` in the Foundation PR:

```java
ComponentResolution augmentorResolution = resolveComponent(annotationValue, null);
```

The `retrievers` array does not use `resolveComponent` at all — empty means no retrievers, each element is resolved as EXPLICIT.

Single `transformer` (not array): `QueryTransformer.transform()` returns `Collection<Query>` — fan-out semantics can't be defaulted for chaining. Users compose multi-transformer logic inside a single CDI bean.

---

## 2. Usage Modes

### Companion mode

`@RagPipeline` on the same interface as `@RegisterAiService`:

```java
@RegisterAiService(modelName = "gpt-4o")
@RagPipeline(
    retrievers = {ProductRetriever.class},
    transformer = HydeTransformer.class
)
public interface ProductAssistant {
    String chat(String message);
}
```

Processor detects both annotations on the same `ClassInfo`. Generates a `RetrievalAugmentor` and wires it directly into the AI service's `QuarkusAiServiceContext` — no intermediate CDI bean.

### Standalone mode

`@RagPipeline` on an interface that is NOT an AI service:

```java
@RagPipeline(
    retrievers = {ProductRetriever.class},
    transformer = HydeTransformer.class
)
public interface ProductRag {}
```

Generates a synthetic `RetrievalAugmentor` CDI bean with both `RetrievalAugmentor` and `ProductRag` as bean types. Referenceable from any AI service:

```java
@RegisterAiService(modelName = "gpt-4o")
@RagPipeline(augmentor = ProductRag.class)
public interface ProductAssistant { ... }

@RegisterAiService(modelName = "gpt-4o-mini")
@RagPipeline(augmentor = ProductRag.class)
public interface CheapAssistant { ... }
```

`augmentor = ProductRag.class` resolves `ProductRag` as a CDI bean. Works because the synthetic bean has `ProductRag` as a type and IS a `RetrievalAugmentor`.

### Pre-built augmentor mode

For hand-rolled beans that directly implement `RetrievalAugmentor`:

```java
@RagPipeline(augmentor = MyCustomAugmentor.class)
public interface ProductAssistant { ... }
```

No `@RagPipeline` needed on the augmentor class — just a regular CDI bean.

### Detection logic

1. Scan for all `@RagPipeline`-annotated classes
2. Check if the class also has `@RegisterAiService` or is agent-implied via `AnnotationsImpliesAiServiceBuildItem`
3. Yes → companion mode: produce `RagPipelineBuildItem` keyed to AI service class name
4. No → standalone mode: generate synthetic `RetrievalAugmentor` CDI bean with interface as additional type

---

## 3. Processor Architecture

### New files — deployment

- `RagPipelineProcessor.java` — `@BuildStep` methods
- `RagPipelineBuildItem.java` — carries config to `AiServicesProcessor`

### New files — runtime

- `RagPipeline.java` — annotation
- `RagPipelineRecorder.java` — recorder (standalone mode only)
- `RagPipelineSupport.java` — static utility with `buildAugmentor()` logic (shared by both modes)
- `RagPipelineCreateInfo.java` — serializable record

### Build item flow

```
RagPipelineProcessor
  @BuildStep scanRagPipelines()
    - Scan Jandex for @RagPipeline
    - Resolve attributes via resolveComponent(value, null) (two-state: SKIP / EXPLICIT)
    - Validate (Section 5)
    - Companion: produce RagPipelineBuildItem(aiServiceClassName, createInfo)
    - Standalone: produce SyntheticBeanBuildItem(RetrievalAugmentor + interface type)
      using RagPipelineRecorder.createStandaloneRagPipeline(createInfo)
    - Both: produce ReflectiveClassBuildItem, UnremovableBeanBuildItem

AiServicesProcessor
  @BuildStep handleDeclarativeServices()
    - Consumes List<RagPipelineBuildItem>
    - For each AI service, checks for matching RagPipelineBuildItem
    - Found → sets ragPipelineCreateInfo on DeclarativeAiServiceCreateInfo
    - Not found → ragPipelineCreateInfo = null (no RAG)
```

### RagPipelineBuildItem

```java
public final class RagPipelineBuildItem extends MultiBuildItem {
    private final String aiServiceClassName;
    private final RagPipelineCreateInfo createInfo;
}
```

### RagPipelineCreateInfo

```java
public record RagPipelineCreateInfo(
    ComponentEntry augmentor,
    List<String> retrieverClassNames,
    ComponentEntry router,
    ComponentEntry transformer,
    ComponentEntry aggregator,
    ComponentEntry injector
) {}
```

Reuses `ComponentEntry` and `ComponentResolutionMode` from Foundation PR. No `standalone` field — the mode is structural (different call sites), not data-driven.

---

## 4. Recorder and Runtime Wiring

### Shared utility — `RagPipelineSupport`

Static utility class in `core/runtime`. Contains the core augmentor building logic used by both modes:

```java
public final class RagPipelineSupport {

    public static RetrievalAugmentor buildAugmentor(
            SyntheticCreationalContext<?> ctx, RagPipelineCreateInfo info) {

        // Pre-built mode
        if (info.augmentor().mode() == ComponentResolutionMode.EXPLICIT) {
            return (RetrievalAugmentor) ctx.getInjectedReference(
                loadClass(info.augmentor().className()));
        }

        // Decomposed mode
        var builder = DefaultRetrievalAugmentor.builder();

        // Retrievers → DefaultQueryRouter
        if (!info.retrieverClassNames().isEmpty()) {
            List<ContentRetriever> retrievers = info.retrieverClassNames().stream()
                .map(name -> (ContentRetriever) ctx.getInjectedReference(loadClass(name)))
                .toList();
            builder.queryRouter(new DefaultQueryRouter(retrievers));
        }

        // Explicit router (mutually exclusive with retrievers — enforced by validation)
        if (info.router().mode() == ComponentResolutionMode.EXPLICIT) {
            builder.queryRouter((QueryRouter) ctx.getInjectedReference(
                loadClass(info.router().className())));
        }

        // Optional components
        if (info.transformer().mode() == ComponentResolutionMode.EXPLICIT) {
            builder.queryTransformer((QueryTransformer) ctx.getInjectedReference(
                loadClass(info.transformer().className())));
        }
        if (info.aggregator().mode() == ComponentResolutionMode.EXPLICIT) {
            builder.contentAggregator((ContentAggregator) ctx.getInjectedReference(
                loadClass(info.aggregator().className())));
        }
        if (info.injector().mode() == ComponentResolutionMode.EXPLICIT) {
            builder.contentInjector((ContentInjector) ctx.getInjectedReference(
                loadClass(info.injector().className())));
        }

        // ManagedExecutor for context-propagated parallel retrieval
        builder.executor(ctx.getInjectedReference(ManagedExecutor.class));

        return builder.build();
    }
}
```

### Standalone mode — `RagPipelineRecorder`

Recorder for standalone synthetic beans only. Called from `RagPipelineProcessor`:

```java
@Recorder
public class RagPipelineRecorder {

    public Function<SyntheticCreationalContext<RetrievalAugmentor>, RetrievalAugmentor>
            createStandaloneRagPipeline(RagPipelineCreateInfo info) {
        return ctx -> RagPipelineSupport.buildAugmentor(ctx, info);
    }
}
```

Build step usage:

```java
@BuildStep @Record(RUNTIME_INIT)
void createStandaloneRagPipelines(RagPipelineRecorder recorder, ...) {
    for (StandaloneRagPipeline pipeline : standalonePipelines) {
        var configurator = SyntheticBeanBuildItem
            .configure(RetrievalAugmentor.class)
            .addType(pipeline.interfaceDotName())
            .scope(ApplicationScoped.class)
            .unremovable()
            .setRuntimeInit()
            .createWith(recorder.createStandaloneRagPipeline(pipeline.createInfo()));

        addRagInjectionPoints(configurator, pipeline.createInfo());

        beanProducer.produce(configurator.done());
    }
}
```

### Companion mode — `AiServicesRecorder`

No cross-recorder parameter. `RagPipelineCreateInfo` is embedded in `DeclarativeAiServiceCreateInfo` (serialized at build time). Inside the lambda returned by `createDeclarativeAiService()`, calls `RagPipelineSupport.buildAugmentor()` directly:

```java
public Function<SyntheticCreationalContext<QuarkusAiServiceContext>, QuarkusAiServiceContext>
        createDeclarativeAiService(DeclarativeAiServiceCreateInfo info) {
    return ctx -> {
        // ... existing wiring ...

        // RAG pipeline (replaces old retrievalAugmentorEntry switch)
        if (info.ragPipelineCreateInfo() != null) {
            RetrievalAugmentor augmentor = RagPipelineSupport
                .buildAugmentor(ctx, info.ragPipelineCreateInfo());
            quarkusAiServices.retrievalAugmentor(augmentor);
        }

        // ... rest unchanged ...
    };
}
```

The `handleDeclarativeServices()` build step signature does NOT change — no `RagPipelineRecorder` parameter needed. `AiServicesRecorder.createDeclarativeAiService()` remains a single-parameter method.

### Shared injection point helper

Both standalone and companion modes need identical injection points matching every `ctx.getInjectedReference()` call in `RagPipelineSupport.buildAugmentor()`. Without declared injection points, CDI won't resolve beans in the creational context and every `getInjectedReference` throws. (Confirmed by existing `EasyRagProcessor` pattern — it declares `.addInjectionPoint(ClassType.create(EmbeddingStore.class))` and `.addInjectionPoint(ClassType.create(EmbeddingModel.class))` for exactly this reason.)

Shared helper used by both `RagPipelineProcessor` (standalone) and `AiServicesProcessor` (companion):

```java
static void addRagInjectionPoints(
        SyntheticBeanBuildItem.ExtendedBeanConfigurator configurator,
        RagPipelineCreateInfo info) {
    if (info.augmentor().mode() == ComponentResolutionMode.EXPLICIT) {
        configurator.addInjectionPoint(
            ClassType.create(DotName.createSimple(info.augmentor().className())));
        return; // pre-built mode — augmentor manages its own executor
    }
    for (String retriever : info.retrieverClassNames()) {
        configurator.addInjectionPoint(
            ClassType.create(DotName.createSimple(retriever)));
    }
    if (info.router().mode() == ComponentResolutionMode.EXPLICIT) {
        configurator.addInjectionPoint(
            ClassType.create(DotName.createSimple(info.router().className())));
    }
    if (info.transformer().mode() == ComponentResolutionMode.EXPLICIT) {
        configurator.addInjectionPoint(
            ClassType.create(DotName.createSimple(info.transformer().className())));
    }
    if (info.aggregator().mode() == ComponentResolutionMode.EXPLICIT) {
        configurator.addInjectionPoint(
            ClassType.create(DotName.createSimple(info.aggregator().className())));
    }
    if (info.injector().mode() == ComponentResolutionMode.EXPLICIT) {
        configurator.addInjectionPoint(
            ClassType.create(DotName.createSimple(info.injector().className())));
    }
    configurator.addInjectionPoint(ClassType.create(ManagedExecutor.class));
}
```

Pre-built mode adds only the augmentor injection point — no `ManagedExecutor` needed since the pre-built augmentor manages its own executor. Decomposed mode adds each component + `ManagedExecutor`.

### Wiring logic summary

Pre-built mode (`augmentor` set): resolve the augmentor bean directly, return it.

Decomposed mode:
1. Resolve `retrievers` → wrap in `DefaultQueryRouter` (validation ensures `router` and `retrievers` are mutually exclusive)
2. If `router` set → use it directly as `QueryRouter`
3. Wire `transformer`, `aggregator`, `injector` if set — upstream defaults otherwise
4. Inject `ManagedExecutor` for context-propagated parallel retrieval
5. Return `DefaultRetrievalAugmentor.builder().build()`

---

## 5. Validation

All at build time. Failures are `DeploymentException`.

| Rule | Condition | Error message |
|------|-----------|---------------|
| Mode conflict | `augmentor` set + any other non-default attribute | "Pre-built augmentor mode cannot be combined with decomposed pipeline attributes" |
| Empty decomposed | No `augmentor`, no `retrievers`, no `router` | "At least one retriever or a router must be specified" |
| Router + retrievers conflict | Both `router` and `retrievers` set | "Cannot specify both router and retrievers — router defines its own retrieval strategy" |
| Interface only | `@RagPipeline` on a concrete class | "@RagPipeline must be applied to an interface" |
| Bean resolution | Concrete class not indexable | Warning (existing pattern from `validateClassExistsAndRegister`) |

Router + retrievers is an error, not a warning. There is no valid runtime reason to declare both — the router decides its own retrieval strategy. An error forces the user to be explicit about which path they're using. When `router` is set, retriever resolution and injection points are skipped entirely — `retrieverClassNames` is empty in the create info.

Future PR hooks (not implemented here):
- `@TenantIsolation` + `@RagPipeline(augmentor = ...)` → error (deferred to PR 6)

---

## 6. @RegisterAiService Changes

### Removed from `@RegisterAiService`

```java
// Deleted:
Class<?> retrievalAugmentor() default void.class;
```

### DeclarativeAiServiceCreateInfo

```java
public record DeclarativeAiServiceCreateInfo(
    String serviceClassName,
    Map<String, AnnotationLiteral<?>> toolsClassInfo,
    ComponentEntry chatMemoryProvider,
    ComponentEntry chatMemoryFlushStrategy,
    // REMOVED: ComponentEntry retrievalAugmentor
    RagPipelineCreateInfo ragPipelineCreateInfo,  // NEW — nullable
    ComponentEntry moderationModel,
    // ... rest unchanged
) {}
```

### DeclarativeAiServiceBuildItem

Remove retrieval augmentor fields:

| Removed | Location |
|---------|----------|
| `DotName retrievalAugmentorClassDotName` | Field |
| `ComponentResolutionMode retrievalAugmentorResolutionMode` | Field |
| Both corresponding constructor parameters | Constructor |
| `getRetrievalAugmentorClassDotName()` | Accessor |
| `getRetrievalAugmentorResolutionMode()` | Accessor |

### AiServicesProcessor — `findDeclarativeServices()`

Remove at lines 575-576:
```java
// DELETE:
retrievalAugmentorResolution.className(),
retrievalAugmentorResolution.mode(),
```

Remove at lines 465-468:
```java
// DELETE:
ComponentResolution retrievalAugmentorResolution = resolveComponent(
    instance.valueWithDefault(index, "retrievalAugmentor"), LangChain4jDotNames.RETRIEVAL_AUGMENTOR);
// and its validateClassExistsAndRegister block
```

### AiServicesProcessor — `handleDeclarativeServices()`

1. Add `List<RagPipelineBuildItem>` parameter
2. Remove at lines 951-952:
   ```java
   // DELETE:
   ComponentEntry retrievalAugmentorEntry = toComponentEntry(bi.getRetrievalAugmentorClassDotName(),
       bi.getRetrievalAugmentorResolutionMode());
   ```
3. Look up matching `RagPipelineBuildItem` by service class name; pass `RagPipelineCreateInfo` (or null) to `DeclarativeAiServiceCreateInfo`
4. Remove retrieval augmentor injection point block at lines 1171-1185:
   ```java
   // DELETE entire switch block:
   switch (bi.getRetrievalAugmentorResolutionMode()) { ... }
   ```
5. When `RagPipelineBuildItem` is present: call `addRagInjectionPoints(configurator, ragPipelineCreateInfo)` — shared helper from Section 4

### AiServicesRecorder

Replace `retrievalAugmentorEntry` switch block with:
```java
if (info.ragPipelineCreateInfo() != null) {
    RetrievalAugmentor augmentor = RagPipelineSupport
        .buildAugmentor(creationalContext, info.ragPipelineCreateInfo());
    quarkusAiServices.retrievalAugmentor(augmentor);
}
```

No cross-recorder parameter. `RagPipelineSupport` is a static utility — callable from any runtime context.

---

## 7. Test Migration

Existing tests in `integration-tests/rag/` pre-date this work. They use the old `retrievalAugmentor` attribute and `Supplier<RetrievalAugmentor>` pattern. This PR migrates them.

| Test | Current | New |
|------|---------|-----|
| `AiServiceWithAutoDiscoveredRetrievalAugmentor` | Bare `@RegisterAiService` + `@Produces RetrievalAugmentor` | Decomposed: extract retriever as `@ApplicationScoped` CDI bean implementing `ContentRetriever`, use `@RagPipeline(retrievers = {InMemoryRetriever.class})`. Delete the producer. See migration example below. |
| `AiServiceWithSpecifiedRetrievalAugmentor` | `retrievalAugmentor = NaiveRagAugmentor.class` (Supplier) | Decomposed: extract retriever bean, use `@RagPipeline(retrievers = {NaiveRetriever.class})`. Delete the Supplier class. |
| `AiServiceWithQueryRouterAndContentInjector` | Supplier wrapping router + injector | `@RagPipeline(router = DogCatRouter.class, injector = PrependingInjector.class)` — extract inline router and injector as `@ApplicationScoped` CDI beans |
| `AiServiceWithQueryTransformer` | Supplier with custom transformer | `@RagPipeline(retrievers = {...}, transformer = LowercaseTransformer.class)` |
| `AiServiceWithReranking` | Supplier with reranking augmentor | Pre-built: convert to `@ApplicationScoped` bean implementing `RetrievalAugmentor`, use `@RagPipeline(augmentor = RerankingAugmentor.class)` |
| `AiServiceWithNoRetrievalAugmentor` | Old sentinel / void.class | Remove attribute — no `@RagPipeline` = no RAG |

### Migration example — `AiServiceWithAutoDiscoveredRetrievalAugmentor`

Before:
```java
@RegisterAiService
public interface AiServiceWithAutoDiscoveredRetrievalAugmentor {
    String chat(String message);

    @ApplicationScoped
    class AugmentorProducer {
        @Inject InMemoryEmbeddingStore<TextSegment> store;
        @Inject EmbeddingModel embeddingModel;

        @Produces
        public RetrievalAugmentor get() {
            return DefaultRetrievalAugmentor.builder()
                .contentRetriever(EmbeddingStoreContentRetriever.builder()
                    .embeddingModel(embeddingModel).embeddingStore(store).maxResults(1).build())
                .build();
        }
    }
}
```

After:
```java
@RegisterAiService
@RagPipeline(retrievers = {AiServiceWithDecomposedRag.InMemoryRetriever.class})
public interface AiServiceWithDecomposedRag {
    String chat(String message);

    @ApplicationScoped
    class InMemoryRetriever implements ContentRetriever {
        @Inject InMemoryEmbeddingStore<TextSegment> store;
        @Inject EmbeddingModel embeddingModel;
        private EmbeddingStoreContentRetriever delegate;

        @PostConstruct
        void init() {
            delegate = EmbeddingStoreContentRetriever.builder()
                .embeddingModel(embeddingModel).embeddingStore(store).maxResults(1).build();
        }

        @Override
        public List<Content> retrieve(Query query) {
            return delegate.retrieve(query);
        }
    }
}
```

### New tests

| Test | Validates |
|------|-----------|
| Companion — single retriever | `@RagPipeline(retrievers = {R.class})` on AI service |
| Companion — multiple retrievers | `@RagPipeline(retrievers = {R1.class, R2.class})` → auto `DefaultQueryRouter` |
| Companion — explicit router | `@RagPipeline(router = MyRouter.class)` |
| Companion — full pipeline | All decomposed attributes set |
| Standalone | `@RagPipeline` on separate interface, `augmentor =` reference |
| Pre-built augmentor | `@RagPipeline(augmentor = MyAugmentor.class)` with plain CDI bean |
| Validation — mode conflict | `augmentor` + `retrievers` → `DeploymentException` |
| Validation — router + retrievers | `router` + `retrievers` → `DeploymentException` |
| Validation — empty decomposed | No augmentor/retrievers/router → `DeploymentException` |
| No @RagPipeline | AI service without RAG — baseline regression |

---

## 8. Coherence Review

**PLATFORM.md:**
- Does this belong upstream? No — Quarkus CDI composition annotation.
- Is this CDI-native? Yes — all components resolved as CDI beans.
- Chapter plan? Core module work, outside agentic chapter plan.
- Upstream coupling? Uses upstream's `DefaultRetrievalAugmentor.builder()` — tight but intentional.

**Protocols:** No violations. No Maven coordinate changes, no Flyway migrations, no SPI concerns.

**Cross-cutting:** Agent-implied AI services (via `AnnotationsImpliesAiServiceBuildItem`) can also carry `@RagPipeline` — companion mode works the same way. Detection checks for the implied annotation as well as explicit `@RegisterAiService`.

---

## Revision History

**2026-06-13 rev 1** — Initial spec.

**2026-06-13 rev 2** — Post-review revisions:
1. All `@RagPipeline` attributes use two-state resolution (SKIP / EXPLICIT), not tri-state. No AUTO_DISCOVER. `resolveComponent()` called with `null` interfaceType for all attributes.
2. `DeclarativeAiServiceBuildItem` field removals specified: `retrievalAugmentorClassDotName`, `retrievalAugmentorResolutionMode`, constructor params, accessors, injection point block. Line references in `AiServicesProcessor` listed.
3. `router` + `retrievers` changed from warning to `DeploymentException`. Mutually exclusive — router defines its own retrieval strategy. When router is set, retriever injection points skipped, `retrieverClassNames` empty.
4. `standalone` field removed from `RagPipelineCreateInfo`. Mode is structural, not data-driven.
5. Test migration for `AiServiceWithAutoDiscoveredRetrievalAugmentor` specified concretely: decomposed mode with extracted `ContentRetriever` CDI bean. Full before/after example added.
6. Recorder integration clarified: `RagPipelineSupport` static utility for shared building logic. `RagPipelineRecorder` used only for standalone mode. `AiServicesRecorder` calls `RagPipelineSupport.buildAugmentor()` directly — no cross-recorder parameter, `handleDeclarativeServices()` build step signature unchanged.

**2026-06-13 rev 3** — Second review revisions:
7. Standalone build step now declares injection points via `addRagInjectionPoints()` helper. Without them, `ctx.getInjectedReference()` calls in `RagPipelineSupport.buildAugmentor()` would fail at runtime. Helper shared between standalone (`RagPipelineProcessor`) and companion (`AiServicesProcessor`) modes. Pre-built mode adds only augmentor injection point (no `ManagedExecutor`). Follows `EasyRagProcessor` pattern.
8. Standalone synthetic bean scoped to `@ApplicationScoped`. `RetrievalAugmentor` is a stateless pipeline — build once, reuse. `@Dependent` default would create a new instance per injection point.
9. Test migration example fixed: `EmbeddingStoreContentRetriever` built once via `@PostConstruct`, not rebuilt per `retrieve()` call.
