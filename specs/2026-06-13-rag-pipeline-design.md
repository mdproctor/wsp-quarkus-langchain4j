# @RagPipeline Composition Annotation — Design Spec

**Date:** 2026-06-13
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
| `router` | `Class<?>` | `void.class` | `QueryRouter` bean — overrides multi-retriever default |
| `transformer` | `Class<?>` | `void.class` | `QueryTransformer` bean |
| `aggregator` | `Class<?>` | `void.class` | `ContentAggregator` bean — upstream `DefaultContentAggregator` when not set |
| `injector` | `Class<?>` | `void.class` | `ContentInjector` bean — upstream `DefaultContentInjector` when not set |

All `Class<?>` attributes follow the tri-state resolution from Foundation PR:
- `void.class` → SKIP
- Interface type (e.g. `QueryTransformer.class`) → AUTO_DISCOVER
- Concrete class → EXPLICIT

The `retrievers` array does not use tri-state — empty means no retrievers. Each element is EXPLICIT.

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
- `RagPipelineRecorder.java` — recorder
- `RagPipelineCreateInfo.java` — serializable record

### Build item flow

```
RagPipelineProcessor
  @BuildStep scanRagPipelines()
    - Scan Jandex for @RagPipeline
    - Resolve attributes via resolveComponent() (same tri-state)
    - Validate (Section 5)
    - Companion: produce RagPipelineBuildItem(aiServiceClassName, createInfo)
    - Standalone: produce SyntheticBeanBuildItem(RetrievalAugmentor + interface type)
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
    ComponentEntry injector,
    boolean standalone
) {}
```

Reuses `ComponentEntry` and `ComponentResolutionMode` from Foundation PR.

---

## 4. Recorder

`RagPipelineRecorder` in `core/runtime`:

**Standalone mode** — produces a `Function` for `SyntheticBeanBuildItem.createWith()`.

**Companion mode** — `buildAugmentor()` called from `AiServicesRecorder` during AI service creation, sharing the same `SyntheticCreationalContext`.

### Wiring logic

Pre-built mode (`augmentor` set): resolve the augmentor bean directly, return it.

Decomposed mode:
1. Resolve `retrievers` → wrap in `DefaultQueryRouter`
2. If explicit `router` set → overrides the retriever-based router
3. Wire `transformer`, `aggregator`, `injector` if set — upstream defaults otherwise
4. Inject `ManagedExecutor` for context-propagated parallel retrieval
5. Return `DefaultRetrievalAugmentor.builder().build()`

### AiServicesRecorder integration

```java
if (info.ragPipelineCreateInfo() != null) {
    RetrievalAugmentor augmentor = ragPipelineRecorder
        .buildAugmentor(creationalContext, info.ragPipelineCreateInfo());
    quarkusAiServices.retrievalAugmentor(augmentor);
}
```

`RagPipelineRecorder` instance passed as a parameter to `createDeclarativeAiService()`.

---

## 5. Validation

All at build time. Failures are `DeploymentException`.

| Rule | Condition | Message |
|------|-----------|---------|
| Mode conflict | `augmentor` set + any other non-default attribute | "Pre-built augmentor mode cannot be combined with decomposed pipeline attributes" |
| Empty decomposed | No `augmentor`, no `retrievers`, no `router` | "At least one retriever or a router must be specified" |
| Router + retrievers | Both set | Valid — explicit router overrides. Build-time warning: "Explicit router overrides retriever-based routing — retrievers will be ignored" |
| Interface only | `@RagPipeline` on a concrete class | "@RagPipeline must be applied to an interface" |
| Bean resolution | Concrete class not indexable | Warning (existing pattern from `validateClassExistsAndRegister`) |

Future PR hooks (not implemented here):
- `@TenantIsolation` + `@RagPipeline(augmentor = ...)` → error (deferred to PR 6)

---

## 6. @RegisterAiService Changes

### Removed

```java
// Deleted from @RegisterAiService:
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

### AiServicesProcessor

1. Remove `retrievalAugmentorResolution` block
2. Add `List<RagPipelineBuildItem>` parameter to `handleDeclarativeServices()`
3. Look up matching `RagPipelineBuildItem` by service class name
4. Pass `RagPipelineCreateInfo` (or null) to `DeclarativeAiServiceCreateInfo`

### AiServicesRecorder

Replace `retrievalAugmentorEntry` switch block with `ragPipelineCreateInfo` null-check + delegation to `RagPipelineRecorder.buildAugmentor()`.

### Injection points

When companion `@RagPipeline` is present, processor adds injection points for all referenced component classes + `ManagedExecutor` to the AI service's `SyntheticBeanBuildItem`.

---

## 7. Test Migration

Existing tests in `integration-tests/rag/` pre-date this work. They use the old `retrievalAugmentor` attribute and `Supplier<RetrievalAugmentor>` pattern. This PR migrates them.

| Test | Current | New |
|------|---------|-----|
| `AiServiceWithAutoDiscoveredRetrievalAugmentor` | Bare `@RegisterAiService` + `@Produces RA` | `@RagPipeline(augmentor = ...)` pointing to producer bean |
| `AiServiceWithSpecifiedRetrievalAugmentor` | `retrievalAugmentor = NaiveRagAugmentor.class` (Supplier) | Decomposed `@RagPipeline(retrievers = {...})` or direct RA bean |
| `AiServiceWithQueryRouterAndContentInjector` | Supplier wrapping router + injector | `@RagPipeline(router = MyRouter.class, injector = MyInjector.class)` |
| `AiServiceWithQueryTransformer` | Supplier with custom transformer | `@RagPipeline(retrievers = {...}, transformer = MyTransformer.class)` |
| `AiServiceWithReranking` | Supplier with reranking | Pre-built augmentor bean or decomposed |
| `AiServiceWithNoRetrievalAugmentor` | Old sentinel | Remove attribute — no `@RagPipeline` = no RAG |

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
