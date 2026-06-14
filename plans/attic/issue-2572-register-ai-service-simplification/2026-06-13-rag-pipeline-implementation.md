# @RagPipeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `@RagPipeline` composition annotation for declarative RAG pipeline configuration, replacing `@RegisterAiService.retrievalAugmentor`.

**Architecture:** New `RagPipelineProcessor` scans `@RagPipeline` annotations, validates, and produces `RagPipelineBuildItem` (companion mode) or `SyntheticBeanBuildItem` (standalone mode). `AiServicesProcessor` consumes the build items. Shared `RagPipelineSupport` utility builds `DefaultRetrievalAugmentor` at runtime. Two commits: (1) additive — new annotation + processor + tests; (2) removal — delete `retrievalAugmentor` attribute, migrate all callers.

**Tech Stack:** Quarkus extension (deployment/runtime split), Jandex annotation scanning, `SyntheticBeanBuildItem`, langchain4j `DefaultRetrievalAugmentor.builder()`, SmallRye `ManagedExecutor`.

**Spec:** `specs/2026-06-13-rag-pipeline-design.md` (rev 3)

---

## File Map

### New files — runtime (`core/runtime/src/main/java/io/quarkiverse/langchain4j/`)

| File | Responsibility |
|------|----------------|
| `RagPipeline.java` | Annotation definition |
| `runtime/rag/RagPipelineCreateInfo.java` | Serializable record carrying component entries from build time to runtime |
| `runtime/rag/RagPipelineSupport.java` | Static utility — `buildAugmentor()` shared by companion and standalone modes |
| `runtime/rag/RagPipelineRecorder.java` | Recorder for standalone synthetic beans |

### New files — deployment (`core/deployment/src/main/java/io/quarkiverse/langchain4j/deployment/`)

| File | Responsibility |
|------|----------------|
| `RagPipelineBuildItem.java` | Carries pipeline config from `RagPipelineProcessor` to `AiServicesProcessor` |
| `RagPipelineProcessor.java` | `@BuildStep` — scan, validate, produce build items and standalone beans |

### Modified files

| File | Change |
|------|--------|
| `LangChain4jDotNames.java` | Add `RAG_PIPELINE` constant |
| `RegisterAiService.java` | Remove `retrievalAugmentor` attribute (Task 9) |
| `DeclarativeAiServiceBuildItem.java` | Remove retrieval augmentor fields (Task 9) |
| `DeclarativeAiServiceCreateInfo.java` | Remove `retrievalAugmentor` `ComponentEntry`, add `ragPipelineCreateInfo` field |
| `AiServicesProcessor.java` | Consume `RagPipelineBuildItem`, remove old retrieval augmentor code, add injection point helper |
| `AiServicesRecorder.java` | Replace retrieval augmentor switch with `RagPipelineSupport.buildAugmentor()` call, remove `RETRIEVAL_AUGMENTOR_TYPE_LITERAL` |

### Migration targets (Task 9–12)

| File | Current pattern | New pattern |
|------|----------------|-------------|
| `integration-tests/rag/src/main/java/org/acme/example/AiServiceWithAutoDiscoveredRetrievalAugmentor.java` | Bare `@RegisterAiService` + `@Produces RA` | `@RagPipeline(retrievers = {InMemoryRetriever.class})` |
| `integration-tests/rag/src/main/java/org/acme/example/AiServiceWithSpecifiedRetrievalAugmentor.java` | `retrievalAugmentor = NaiveRagAugmentor.class` (Supplier) | `@RagPipeline(retrievers = {NaiveRetriever.class})` |
| `integration-tests/rag/src/main/java/org/acme/example/AiServiceWithQueryRouterAndContentInjector.java` | `retrievalAugmentor = QueryRouterAugmentor.class` (Supplier) | `@RagPipeline(router = DogCatRouter.class, injector = PrependingInjector.class)` |
| `integration-tests/rag/src/main/java/org/acme/example/AiServiceWithQueryTransformer.java` | `retrievalAugmentor = QueryCompressionAugmentor.class` (Supplier) | `@RagPipeline(retrievers = {LowercaseRetriever.class}, transformer = LowercaseTransformer.class)` |
| `integration-tests/rag/src/main/java/org/acme/example/AiServiceWithReranking.java` | `retrievalAugmentor = AugmentorWithReranking.class` (Supplier) | `@RagPipeline(augmentor = RerankingAugmentor.class)` — pre-built mode (uses custom `ReRankingContentAggregator` with scoring model) |
| `integration-tests/rag/src/main/java/org/acme/example/AiServiceWithNoRetrievalAugmentor.java` | Old sentinel | Remove attribute — no `@RagPipeline` |
| `core/deployment/src/test/java/.../GuardrailWithAugmentationTest.java` | `retrievalAugmentor = MyRetrievalAugmentor.class` | `@RagPipeline(augmentor = MyRetrievalAugmentor.class)` |
| `core/deployment/src/test/java/.../ResponseAugmenterWithAugmentationResultTest.java` | `retrievalAugmentor = MyRetrievalAugmentor.class` | `@RagPipeline(augmentor = MyRetrievalAugmentor.class)` |
| `model-providers/openai/openai-vanilla/deployment/src/test/java/.../DeclarativeAiServicesTest.java` | `retrievalAugmentor = DummyRetriever.Supplier.class` | `@RagPipeline(augmentor = DummyRetriever.class)` — convert `DummyRetriever` to implement `RetrievalAugmentor` directly |
| `samples/chatbot/src/main/java/.../Bot.java` | `retrievalAugmentor = AugmentorExample.class` | `@RagPipeline(augmentor = AugmentorExample.class)` |
| `samples/sql-chatbot/src/main/java/.../MovieMuse.java` | `retrievalAugmentor = MovieMuseRetrievalAugmentor.class` | `@RagPipeline(augmentor = MovieMuseRetrievalAugmentor.class)` |
| `samples/secure-sql-chatbot/src/main/java/.../MovieMuse.java` | `retrievalAugmentor = MovieMuseRetrievalAugmentor.class` | `@RagPipeline(augmentor = MovieMuseRetrievalAugmentor.class)` |
| `samples/secure-fraud-detection/src/main/java/.../FraudDetectionAi.java` | `retrievalAugmentor = FraudDetectionRetrievalAugmentor.class` | `@RagPipeline(augmentor = FraudDetectionRetrievalAugmentor.class)` |

---

## Task 1: Create runtime types

**Files:**
- Create: `core/runtime/src/main/java/io/quarkiverse/langchain4j/RagPipeline.java`
- Create: `core/runtime/src/main/java/io/quarkiverse/langchain4j/runtime/rag/RagPipelineCreateInfo.java`
- Create: `core/runtime/src/main/java/io/quarkiverse/langchain4j/runtime/rag/RagPipelineSupport.java`
- Create: `core/runtime/src/main/java/io/quarkiverse/langchain4j/runtime/rag/RagPipelineRecorder.java`

- [ ] **Step 1: Create `@RagPipeline` annotation**

```java
package io.quarkiverse.langchain4j;

import static java.lang.annotation.RetentionPolicy.RUNTIME;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.Target;

@Retention(RUNTIME)
@Target(ElementType.TYPE)
public @interface RagPipeline {

    Class<?> augmentor() default void.class;

    Class<?>[] retrievers() default {};

    Class<?> router() default void.class;

    Class<?> transformer() default void.class;

    Class<?> aggregator() default void.class;

    Class<?> injector() default void.class;
}
```

- [ ] **Step 2: Create `RagPipelineCreateInfo` record**

```java
package io.quarkiverse.langchain4j.runtime.rag;

import java.util.List;

import io.quarkiverse.langchain4j.runtime.aiservice.DeclarativeAiServiceCreateInfo.ComponentEntry;

public record RagPipelineCreateInfo(
        ComponentEntry augmentor,
        List<String> retrieverClassNames,
        ComponentEntry router,
        ComponentEntry transformer,
        ComponentEntry aggregator,
        ComponentEntry injector) {
}
```

- [ ] **Step 3: Create `RagPipelineSupport` utility**

```java
package io.quarkiverse.langchain4j.runtime.rag;

import java.util.List;

import org.eclipse.microprofile.context.ManagedExecutor;

import dev.langchain4j.rag.DefaultRetrievalAugmentor;
import dev.langchain4j.rag.RetrievalAugmentor;
import dev.langchain4j.rag.content.aggregator.ContentAggregator;
import dev.langchain4j.rag.content.injector.ContentInjector;
import dev.langchain4j.rag.content.retriever.ContentRetriever;
import dev.langchain4j.rag.query.router.DefaultQueryRouter;
import dev.langchain4j.rag.query.router.QueryRouter;
import dev.langchain4j.rag.query.transformer.QueryTransformer;
import io.quarkiverse.langchain4j.runtime.aiservice.ComponentResolutionMode;
import io.quarkus.arc.SyntheticCreationalContext;

public final class RagPipelineSupport {

    private RagPipelineSupport() {
    }

    public static RetrievalAugmentor buildAugmentor(
            SyntheticCreationalContext<?> ctx, RagPipelineCreateInfo info) {

        if (info.augmentor().mode() == ComponentResolutionMode.EXPLICIT) {
            return (RetrievalAugmentor) ctx.getInjectedReference(loadClass(info.augmentor().className()));
        }

        DefaultRetrievalAugmentor.DefaultRetrievalAugmentorBuilder builder = DefaultRetrievalAugmentor.builder();

        if (!info.retrieverClassNames().isEmpty()) {
            List<ContentRetriever> retrievers = info.retrieverClassNames().stream()
                    .map(name -> (ContentRetriever) ctx.getInjectedReference(loadClass(name)))
                    .toList();
            builder.queryRouter(new DefaultQueryRouter(retrievers));
        }

        if (info.router().mode() == ComponentResolutionMode.EXPLICIT) {
            builder.queryRouter((QueryRouter) ctx.getInjectedReference(loadClass(info.router().className())));
        }

        if (info.transformer().mode() == ComponentResolutionMode.EXPLICIT) {
            builder.queryTransformer(
                    (QueryTransformer) ctx.getInjectedReference(loadClass(info.transformer().className())));
        }

        if (info.aggregator().mode() == ComponentResolutionMode.EXPLICIT) {
            builder.contentAggregator(
                    (ContentAggregator) ctx.getInjectedReference(loadClass(info.aggregator().className())));
        }

        if (info.injector().mode() == ComponentResolutionMode.EXPLICIT) {
            builder.contentInjector(
                    (ContentInjector) ctx.getInjectedReference(loadClass(info.injector().className())));
        }

        builder.executor(ctx.getInjectedReference(ManagedExecutor.class));

        return builder.build();
    }

    private static Class<?> loadClass(String className) {
        try {
            return Thread.currentThread().getContextClassLoader().loadClass(className);
        } catch (ClassNotFoundException e) {
            throw new RuntimeException("Failed to load class: " + className, e);
        }
    }
}
```

- [ ] **Step 4: Create `RagPipelineRecorder`**

```java
package io.quarkiverse.langchain4j.runtime.rag;

import java.util.function.Function;

import dev.langchain4j.rag.RetrievalAugmentor;
import io.quarkus.arc.SyntheticCreationalContext;
import io.quarkus.runtime.annotations.Recorder;

@Recorder
public class RagPipelineRecorder {

    public Function<SyntheticCreationalContext<RetrievalAugmentor>, RetrievalAugmentor> createStandaloneRagPipeline(
            RagPipelineCreateInfo info) {
        return ctx -> RagPipelineSupport.buildAugmentor(ctx, info);
    }
}
```

- [ ] **Step 5: Commit**

```
git add core/runtime/src/main/java/io/quarkiverse/langchain4j/RagPipeline.java \
       core/runtime/src/main/java/io/quarkiverse/langchain4j/runtime/rag/
git commit -m "feat(#2574): add @RagPipeline runtime types — annotation, record, support, recorder"
```

---

## Task 2: Create deployment types

**Files:**
- Create: `core/deployment/src/main/java/io/quarkiverse/langchain4j/deployment/RagPipelineBuildItem.java`
- Modify: `core/deployment/src/main/java/io/quarkiverse/langchain4j/deployment/LangChain4jDotNames.java`

- [ ] **Step 1: Create `RagPipelineBuildItem`**

```java
package io.quarkiverse.langchain4j.deployment;

import io.quarkiverse.langchain4j.runtime.rag.RagPipelineCreateInfo;
import io.quarkus.builder.item.MultiBuildItem;

public final class RagPipelineBuildItem extends MultiBuildItem {

    private final String aiServiceClassName;
    private final RagPipelineCreateInfo createInfo;

    public RagPipelineBuildItem(String aiServiceClassName, RagPipelineCreateInfo createInfo) {
        this.aiServiceClassName = aiServiceClassName;
        this.createInfo = createInfo;
    }

    public String getAiServiceClassName() {
        return aiServiceClassName;
    }

    public RagPipelineCreateInfo getCreateInfo() {
        return createInfo;
    }
}
```

- [ ] **Step 2: Add `RAG_PIPELINE` to `LangChain4jDotNames`**

Add after the `RETRIEVAL_AUGMENTOR` constant (line 106):

```java
public static final DotName RAG_PIPELINE = DotName.createSimple(RagPipeline.class);
```

Add the import:
```java
import io.quarkiverse.langchain4j.RagPipeline;
```

- [ ] **Step 3: Commit**

```
git add core/deployment/src/main/java/io/quarkiverse/langchain4j/deployment/RagPipelineBuildItem.java \
       core/deployment/src/main/java/io/quarkiverse/langchain4j/deployment/LangChain4jDotNames.java
git commit -m "feat(#2574): add RagPipelineBuildItem and RAG_PIPELINE DotName"
```

---

## Task 3: Create `RagPipelineProcessor`

**Files:**
- Create: `core/deployment/src/main/java/io/quarkiverse/langchain4j/deployment/RagPipelineProcessor.java`

- [ ] **Step 1: Create the processor**

```java
package io.quarkiverse.langchain4j.deployment;

import java.util.ArrayList;
import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;

import org.eclipse.microprofile.context.ManagedExecutor;
import org.jboss.jandex.AnnotationInstance;
import org.jboss.jandex.AnnotationValue;
import org.jboss.jandex.ClassInfo;
import org.jboss.jandex.DotName;
import org.jboss.jandex.IndexView;
import org.jboss.jandex.Type;
import org.jboss.logging.Logger;

import dev.langchain4j.rag.RetrievalAugmentor;
import io.quarkiverse.langchain4j.runtime.aiservice.ComponentResolutionMode;
import io.quarkiverse.langchain4j.runtime.aiservice.DeclarativeAiServiceCreateInfo.ComponentEntry;
import io.quarkiverse.langchain4j.runtime.rag.RagPipelineCreateInfo;
import io.quarkiverse.langchain4j.runtime.rag.RagPipelineRecorder;
import io.quarkus.arc.deployment.SyntheticBeanBuildItem;
import io.quarkus.arc.deployment.UnremovableBeanBuildItem;
import io.quarkus.deployment.annotations.BuildProducer;
import io.quarkus.deployment.annotations.BuildStep;
import io.quarkus.deployment.annotations.ExecutionTime;
import io.quarkus.deployment.annotations.Record;
import io.quarkus.deployment.builditem.CombinedIndexBuildItem;
import io.quarkus.deployment.builditem.nativeimage.ReflectiveClassBuildItem;

public class RagPipelineProcessor {

    private static final Logger log = Logger.getLogger(RagPipelineProcessor.class);
    private static final DotName VOID_CLASS = DotName.createSimple(void.class);

    @BuildStep
    @Record(ExecutionTime.RUNTIME_INIT)
    void scanRagPipelines(
            CombinedIndexBuildItem combinedIndex,
            List<DeclarativeAiServiceBuildItem> aiServices,
            List<AnnotationsImpliesAiServiceBuildItem> impliedAiServices,
            RagPipelineRecorder recorder,
            BuildProducer<RagPipelineBuildItem> ragPipelineProducer,
            BuildProducer<SyntheticBeanBuildItem> syntheticBeanProducer,
            BuildProducer<ReflectiveClassBuildItem> reflectiveClassProducer,
            BuildProducer<UnremovableBeanBuildItem> unremovableBeanProducer) {

        IndexView index = combinedIndex.getIndex();
        var annotations = index.getAnnotations(LangChain4jDotNames.RAG_PIPELINE);
        if (annotations.isEmpty()) {
            return;
        }

        // Collect AI service class names for companion mode detection
        var aiServiceClassNames = new java.util.HashSet<DotName>();
        for (DeclarativeAiServiceBuildItem bi : aiServices) {
            aiServiceClassNames.add(bi.getServiceClassInfo().name());
        }
        for (AnnotationsImpliesAiServiceBuildItem bi : impliedAiServices) {
            aiServiceClassNames.add(bi.getAnnotationName());
        }

        for (AnnotationInstance annotation : annotations) {
            if (annotation.target().kind() != org.jboss.jandex.AnnotationTarget.Kind.CLASS) {
                continue;
            }
            ClassInfo classInfo = annotation.target().asClass();

            // Validation: must be an interface
            if (!classInfo.isInterface()) {
                throw new IllegalStateException(
                        "@RagPipeline must be applied to an interface, but was found on: " + classInfo.name());
            }

            RagPipelineCreateInfo createInfo = buildCreateInfo(annotation, index,
                    reflectiveClassProducer, unremovableBeanProducer);

            boolean isAiService = aiServiceClassNames.contains(classInfo.name())
                    || classInfo.hasAnnotation(LangChain4jDotNames.REGISTER_AI_SERVICES);

            if (isAiService) {
                // Companion mode
                ragPipelineProducer.produce(
                        new RagPipelineBuildItem(classInfo.name().toString(), createInfo));
            } else {
                // Standalone mode — generate synthetic RetrievalAugmentor bean
                SyntheticBeanBuildItem.ExtendedBeanConfigurator configurator = SyntheticBeanBuildItem
                        .configure(RetrievalAugmentor.class)
                        .addType(Type.create(classInfo.name(), Type.Kind.CLASS))
                        .scope(ApplicationScoped.class)
                        .unremovable()
                        .setRuntimeInit()
                        .createWith(recorder.createStandaloneRagPipeline(createInfo));

                addRagInjectionPoints(configurator, createInfo);

                syntheticBeanProducer.produce(configurator.done());
            }
        }
    }

    private RagPipelineCreateInfo buildCreateInfo(
            AnnotationInstance annotation,
            IndexView index,
            BuildProducer<ReflectiveClassBuildItem> reflectiveClassProducer,
            BuildProducer<UnremovableBeanBuildItem> unremovableBeanProducer) {

        // Resolve each attribute — two-state: SKIP / EXPLICIT (null interfaceType = no auto-discover)
        ComponentEntry augmentorEntry = resolveComponent(annotation.value("augmentor"));
        ComponentEntry routerEntry = resolveComponent(annotation.value("router"));
        ComponentEntry transformerEntry = resolveComponent(annotation.value("transformer"));
        ComponentEntry aggregatorEntry = resolveComponent(annotation.value("aggregator"));
        ComponentEntry injectorEntry = resolveComponent(annotation.value("injector"));

        // Resolve retrievers array
        List<String> retrieverClassNames = new ArrayList<>();
        AnnotationValue retrieversValue = annotation.value("retrievers");
        if (retrieversValue != null) {
            for (Type type : retrieversValue.asClassArray()) {
                String className = type.name().toString();
                retrieverClassNames.add(className);
                registerForReflectionAndUnremovable(type.name(), reflectiveClassProducer, unremovableBeanProducer);
            }
        }

        // Validation
        boolean hasAugmentor = augmentorEntry.mode() == ComponentResolutionMode.EXPLICIT;
        boolean hasRetrievers = !retrieverClassNames.isEmpty();
        boolean hasRouter = routerEntry.mode() == ComponentResolutionMode.EXPLICIT;
        boolean hasTransformer = transformerEntry.mode() == ComponentResolutionMode.EXPLICIT;
        boolean hasAggregator = aggregatorEntry.mode() == ComponentResolutionMode.EXPLICIT;
        boolean hasInjector = injectorEntry.mode() == ComponentResolutionMode.EXPLICIT;

        if (hasAugmentor && (hasRetrievers || hasRouter || hasTransformer || hasAggregator || hasInjector)) {
            throw new IllegalStateException(
                    "Pre-built augmentor mode cannot be combined with decomposed pipeline attributes on "
                            + annotation.target().asClass().name());
        }

        if (!hasAugmentor && !hasRetrievers && !hasRouter) {
            throw new IllegalStateException(
                    "At least one retriever or a router must be specified on "
                            + annotation.target().asClass().name());
        }

        if (hasRouter && hasRetrievers) {
            throw new IllegalStateException(
                    "Cannot specify both router and retrievers on "
                            + annotation.target().asClass().name()
                            + " — router defines its own retrieval strategy");
        }

        // Register EXPLICIT components for reflection and unremovable
        if (hasAugmentor) {
            registerForReflectionAndUnremovable(
                    DotName.createSimple(augmentorEntry.className()),
                    reflectiveClassProducer, unremovableBeanProducer);
        }
        if (hasRouter) {
            registerForReflectionAndUnremovable(
                    DotName.createSimple(routerEntry.className()),
                    reflectiveClassProducer, unremovableBeanProducer);
        }
        if (hasTransformer) {
            registerForReflectionAndUnremovable(
                    DotName.createSimple(transformerEntry.className()),
                    reflectiveClassProducer, unremovableBeanProducer);
        }
        if (hasAggregator) {
            registerForReflectionAndUnremovable(
                    DotName.createSimple(aggregatorEntry.className()),
                    reflectiveClassProducer, unremovableBeanProducer);
        }
        if (hasInjector) {
            registerForReflectionAndUnremovable(
                    DotName.createSimple(injectorEntry.className()),
                    reflectiveClassProducer, unremovableBeanProducer);
        }

        return new RagPipelineCreateInfo(
                augmentorEntry, retrieverClassNames, routerEntry,
                transformerEntry, aggregatorEntry, injectorEntry);
    }

    private ComponentEntry resolveComponent(AnnotationValue annotationValue) {
        if (annotationValue == null) {
            return ComponentEntry.SKIP;
        }
        DotName dotName = annotationValue.asClass().name();
        if (VOID_CLASS.equals(dotName)) {
            return ComponentEntry.SKIP;
        }
        return new ComponentEntry(dotName.toString(), ComponentResolutionMode.EXPLICIT);
    }

    private void registerForReflectionAndUnremovable(
            DotName classDotName,
            BuildProducer<ReflectiveClassBuildItem> reflectiveClassProducer,
            BuildProducer<UnremovableBeanBuildItem> unremovableBeanProducer) {
        reflectiveClassProducer.produce(
                ReflectiveClassBuildItem.builder(classDotName.toString()).constructors(true).build());
        unremovableBeanProducer.produce(
                UnremovableBeanBuildItem.beanTypes(classDotName));
    }

    static void addRagInjectionPoints(
            SyntheticBeanBuildItem.ExtendedBeanConfigurator configurator,
            RagPipelineCreateInfo info) {
        if (info.augmentor().mode() == ComponentResolutionMode.EXPLICIT) {
            configurator.addInjectionPoint(
                    org.jboss.jandex.ClassType.create(DotName.createSimple(info.augmentor().className())));
            return;
        }
        for (String retriever : info.retrieverClassNames()) {
            configurator.addInjectionPoint(
                    org.jboss.jandex.ClassType.create(DotName.createSimple(retriever)));
        }
        if (info.router().mode() == ComponentResolutionMode.EXPLICIT) {
            configurator.addInjectionPoint(
                    org.jboss.jandex.ClassType.create(DotName.createSimple(info.router().className())));
        }
        if (info.transformer().mode() == ComponentResolutionMode.EXPLICIT) {
            configurator.addInjectionPoint(
                    org.jboss.jandex.ClassType.create(DotName.createSimple(info.transformer().className())));
        }
        if (info.aggregator().mode() == ComponentResolutionMode.EXPLICIT) {
            configurator.addInjectionPoint(
                    org.jboss.jandex.ClassType.create(DotName.createSimple(info.aggregator().className())));
        }
        if (info.injector().mode() == ComponentResolutionMode.EXPLICIT) {
            configurator.addInjectionPoint(
                    org.jboss.jandex.ClassType.create(DotName.createSimple(info.injector().className())));
        }
        configurator.addInjectionPoint(
                org.jboss.jandex.ClassType.create(DotName.createSimple(ManagedExecutor.class.getName())));
    }
}
```

- [ ] **Step 2: Commit**

```
git add core/deployment/src/main/java/io/quarkiverse/langchain4j/deployment/RagPipelineProcessor.java
git commit -m "feat(#2574): add RagPipelineProcessor — scan, validate, companion + standalone"
```

---

## Task 4: Integrate with `AiServicesProcessor` and `AiServicesRecorder`

**Files:**
- Modify: `core/runtime/src/main/java/io/quarkiverse/langchain4j/runtime/aiservice/DeclarativeAiServiceCreateInfo.java`
- Modify: `core/deployment/src/main/java/io/quarkiverse/langchain4j/deployment/AiServicesProcessor.java`
- Modify: `core/runtime/src/main/java/io/quarkiverse/langchain4j/runtime/AiServicesRecorder.java`

- [ ] **Step 1: Add `ragPipelineCreateInfo` to `DeclarativeAiServiceCreateInfo`**

Add the field after `retrievalAugmentor` (line 15). Both fields temporarily coexist during this additive phase:

```java
ComponentEntry retrievalAugmentor,
RagPipelineCreateInfo ragPipelineCreateInfo,
```

Add the import:
```java
import io.quarkiverse.langchain4j.runtime.rag.RagPipelineCreateInfo;
```

Update all call sites that construct `DeclarativeAiServiceCreateInfo` — pass `null` for the new parameter. There is one call site in `AiServicesProcessor.handleDeclarativeServices()` around line 1062.

- [ ] **Step 2: Update `AiServicesProcessor.handleDeclarativeServices()` to consume `RagPipelineBuildItem`**

Add the parameter to the `@BuildStep` method:

```java
List<RagPipelineBuildItem> ragPipelines,
```

Build a lookup map at the start of the method:

```java
Map<String, RagPipelineCreateInfo> ragPipelineMap = new java.util.HashMap<>();
for (RagPipelineBuildItem rp : ragPipelines) {
    ragPipelineMap.put(rp.getAiServiceClassName(), rp.getCreateInfo());
}
```

In the per-AI-service loop, look up the matching pipeline and pass it to `DeclarativeAiServiceCreateInfo`:

```java
String serviceClassName = bi.getServiceClassInfo().name().toString();
RagPipelineCreateInfo ragPipelineCreateInfo = ragPipelineMap.get(serviceClassName);
```

Pass `ragPipelineCreateInfo` (may be null) to the `DeclarativeAiServiceCreateInfo` constructor.

When `ragPipelineCreateInfo != null`, call `RagPipelineProcessor.addRagInjectionPoints(configurator, ragPipelineCreateInfo)` to add injection points to the AI service's synthetic bean.

- [ ] **Step 3: Update `AiServicesRecorder` to handle `ragPipelineCreateInfo`**

After the existing retrieval augmentor switch block (lines 305–321), add:

```java
// RAG pipeline (companion mode)
if (info.ragPipelineCreateInfo() != null) {
    RetrievalAugmentor augmentor = RagPipelineSupport
            .buildAugmentor(creationalContext, info.ragPipelineCreateInfo());
    quarkusAiServices.retrievalAugmentor(augmentor);
}
```

Add the import:
```java
import io.quarkiverse.langchain4j.runtime.rag.RagPipelineSupport;
```

During this additive phase, both code paths coexist — the old `retrievalAugmentorEntry` switch and the new `ragPipelineCreateInfo` check. They're mutually exclusive per AI service (an AI service either has the old attribute or `@RagPipeline`, never both).

- [ ] **Step 4: Verify build compiles**

Run:
```bash
/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/core/pom.xml compile -T 1C -q
```

Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```
git add core/runtime/src/main/java/io/quarkiverse/langchain4j/runtime/aiservice/DeclarativeAiServiceCreateInfo.java \
       core/deployment/src/main/java/io/quarkiverse/langchain4j/deployment/AiServicesProcessor.java \
       core/runtime/src/main/java/io/quarkiverse/langchain4j/runtime/AiServicesRecorder.java
git commit -m "feat(#2574): integrate RagPipelineBuildItem into AiServicesProcessor and recorder"
```

---

## Task 5: Write and pass companion mode tests

**Files:**
- Create: `core/deployment/src/test/java/io/quarkiverse/langchain4j/test/rag/RagPipelineCompanionTest.java`

This test uses `QuarkusUnitTest` (the `@RegisterExtension` pattern used by existing core deployment tests like `GuardrailWithAugmentationTest`).

- [ ] **Step 1: Write the test**

The test verifies:
- Single retriever companion mode
- Multiple retrievers companion mode (auto `DefaultQueryRouter`)
- Explicit router companion mode
- Full pipeline (router + transformer + aggregator + injector)
- Pre-built augmentor companion mode

Each inner AI service interface uses `@RegisterAiService` + `@RagPipeline`, with inner `@ApplicationScoped` CDI bean implementations for the components. The test uses a mock `ChatModel` (same pattern as `GuardrailWithAugmentationTest`).

Write the test class with test methods for each scenario. Each method calls the AI service and asserts the RAG pipeline was executed (content was injected into the user message).

- [ ] **Step 2: Run the test**

```bash
/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/core/deployment/pom.xml test -Dtest=RagPipelineCompanionTest -T 1C
```

Expected: All tests PASS.

- [ ] **Step 3: Commit**

```
git add core/deployment/src/test/java/io/quarkiverse/langchain4j/test/rag/RagPipelineCompanionTest.java
git commit -m "test(#2574): companion mode tests — single/multi retriever, router, full pipeline, pre-built"
```

---

## Task 6: Write and pass standalone mode test

**Files:**
- Create: `core/deployment/src/test/java/io/quarkiverse/langchain4j/test/rag/RagPipelineStandaloneTest.java`

- [ ] **Step 1: Write the test**

The test verifies:
- `@RagPipeline` on a separate interface (not `@RegisterAiService`) generates a `RetrievalAugmentor` CDI bean
- The bean is `@ApplicationScoped`
- An AI service references it via `@RagPipeline(augmentor = StandaloneRag.class)`
- Two AI services sharing the same standalone pipeline

Define a standalone interface:
```java
@RagPipeline(retrievers = {TestRetriever.class})
public interface SharedRag {}
```

And two AI services referencing it:
```java
@RegisterAiService
@RagPipeline(augmentor = SharedRag.class)
public interface Assistant1 { String chat(String msg); }

@RegisterAiService
@RagPipeline(augmentor = SharedRag.class)
public interface Assistant2 { String chat(String msg); }
```

Assert both services retrieve content from the same pipeline.

- [ ] **Step 2: Run the test**

```bash
/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/core/deployment/pom.xml test -Dtest=RagPipelineStandaloneTest -T 1C
```

Expected: PASS.

- [ ] **Step 3: Commit**

```
git add core/deployment/src/test/java/io/quarkiverse/langchain4j/test/rag/RagPipelineStandaloneTest.java
git commit -m "test(#2574): standalone mode test — shared pipeline across two AI services"
```

---

## Task 7: Write and pass validation tests

**Files:**
- Create: `core/deployment/src/test/java/io/quarkiverse/langchain4j/test/rag/RagPipelineValidationTest.java`

- [ ] **Step 1: Write the test**

Three validation scenarios, each a separate `QuarkusUnitTest` with `@ShouldThrow(Exception.class)` or assertion on deployment failure:

1. **Mode conflict:** `@RagPipeline(augmentor = X.class, retrievers = {Y.class})` → expects `IllegalStateException` containing "Pre-built augmentor mode cannot be combined"
2. **Empty decomposed:** `@RagPipeline()` with no attributes → expects `IllegalStateException` containing "At least one retriever or a router must be specified"
3. **Router + retrievers conflict:** `@RagPipeline(router = R.class, retrievers = {Y.class})` → expects `IllegalStateException` containing "Cannot specify both router and retrievers"

Use `assertThatThrownBy` on `QuarkusUnitTest` or the `@ShouldThrow` annotation.

- [ ] **Step 2: Run the tests**

```bash
/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/core/deployment/pom.xml test -Dtest=RagPipelineValidationTest -T 1C
```

Expected: PASS (all three validations fire correctly).

- [ ] **Step 3: Commit**

```
git add core/deployment/src/test/java/io/quarkiverse/langchain4j/test/rag/RagPipelineValidationTest.java
git commit -m "test(#2574): validation tests — mode conflict, empty decomposed, router+retrievers"
```

---

## Task 8: Run full core test suite

- [ ] **Step 1: Run core tests**

```bash
/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/core/pom.xml test -T 1C
```

Expected: All existing tests still pass (additive changes only so far). New RAG pipeline tests pass.

- [ ] **Step 2: Fix any failures**

If any existing test fails, investigate. The additive phase should not break anything — both old and new code paths coexist.

---

## Task 9: Remove `retrievalAugmentor` and migrate all callers

This is the breaking change commit. All removals and migrations happen atomically.

**Files:**
- Modify: `core/runtime/src/main/java/io/quarkiverse/langchain4j/RegisterAiService.java`
- Modify: `core/deployment/src/main/java/io/quarkiverse/langchain4j/deployment/DeclarativeAiServiceBuildItem.java`
- Modify: `core/runtime/src/main/java/io/quarkiverse/langchain4j/runtime/aiservice/DeclarativeAiServiceCreateInfo.java`
- Modify: `core/deployment/src/main/java/io/quarkiverse/langchain4j/deployment/AiServicesProcessor.java`
- Modify: `core/runtime/src/main/java/io/quarkiverse/langchain4j/runtime/AiServicesRecorder.java`
- Modify: All migration target files (see File Map above)

- [ ] **Step 1: Remove `retrievalAugmentor` from `@RegisterAiService`**

Delete the `retrievalAugmentor()` method (line 131–138) and its Javadoc from `RegisterAiService.java`. Remove the `RetrievalAugmentor` and `ContentRetriever` imports if no longer used.

- [ ] **Step 2: Remove retrieval augmentor fields from `DeclarativeAiServiceBuildItem`**

Delete:
- Field `retrievalAugmentorClassDotName` (line 25)
- Field `retrievalAugmentorResolutionMode` (line 26)
- Both constructor parameters
- `getRetrievalAugmentorClassDotName()` accessor
- `getRetrievalAugmentorResolutionMode()` accessor

Update all constructor call sites (in `AiServicesProcessor.findDeclarativeServices()` around line 567–577).

- [ ] **Step 3: Remove `retrievalAugmentor` from `DeclarativeAiServiceCreateInfo`**

Delete the `ComponentEntry retrievalAugmentor` field (line 15). The `ragPipelineCreateInfo` field added in Task 4 remains.

Update the constructor call site in `AiServicesProcessor.handleDeclarativeServices()` (around line 1062).

- [ ] **Step 4: Remove old code from `AiServicesProcessor`**

In `findDeclarativeServices()`:
- Delete `ComponentResolution retrievalAugmentorResolution = resolveComponent(...)` (line 465–466)
- Delete the `validateClassExistsAndRegister` block for retrievalAugmentor (lines 467–469)
- Delete the `retrievalAugmentorResolution.className()` and `retrievalAugmentorResolution.mode()` constructor args (lines 575–576)

In `handleDeclarativeServices()`:
- Delete `ComponentEntry retrievalAugmentorEntry = toComponentEntry(...)` (line 951–952)
- Delete the retrieval augmentor injection point switch block (lines 1171–1185)
- Delete `boolean needsRetrievalAugmentorBean = false;` (line 916) and the `if (needsRetrievalAugmentorBean)` block (lines 1261–1263)
- Delete the `retrievalAugmentorEntry` argument from the `DeclarativeAiServiceCreateInfo` constructor call

- [ ] **Step 5: Remove old code from `AiServicesRecorder`**

Delete the `RETRIEVAL_AUGMENTOR_TYPE_LITERAL` field (line 49).
Delete the retrieval augmentor switch block (lines 304–321).
Remove unused imports (`Instance`, `RetrievalAugmentor` if no other references).

- [ ] **Step 6: Migrate RAG integration tests**

For each test in `integration-tests/rag/src/main/java/org/acme/example/`:

**`AiServiceWithAutoDiscoveredRetrievalAugmentor`** → Rename to `AiServiceWithDecomposedRag`. Replace bare `@RegisterAiService` with `@RegisterAiService` + `@RagPipeline(retrievers = {...})`. Delete the `AugmentorProducer` class. Create an `@ApplicationScoped` CDI bean implementing `ContentRetriever` with `@PostConstruct` initialization.

**`AiServiceWithSpecifiedRetrievalAugmentor`** → Rename to `AiServiceWithSingleRetriever`. Replace `@RegisterAiService(retrievalAugmentor = ...)` with `@RagPipeline(retrievers = {NaiveRetriever.class})`. Convert `NaiveRagAugmentor extends Supplier<RA>` to `NaiveRetriever implements ContentRetriever` with `@PostConstruct`.

**`AiServiceWithQueryRouterAndContentInjector`** → Replace `@RegisterAiService(retrievalAugmentor = ...)` with `@RagPipeline(router = DogCatRouter.class, injector = PrependingInjector.class)`. Extract the inline `QueryRouter` and `ContentInjector` as `@ApplicationScoped` CDI beans.

**`AiServiceWithQueryTransformer`** → Replace with `@RagPipeline(retrievers = {LowercaseRetriever.class}, transformer = LowercaseTransformer.class)`. Extract inline `QueryTransformer` and `ContentRetriever` as CDI beans.

**`AiServiceWithReranking`** → Replace with `@RagPipeline(augmentor = RerankingAugmentor.class)`. Convert `AugmentorWithReranking` from `Supplier<RA>` to directly implement `RetrievalAugmentor`.

**`AiServiceWithNoRetrievalAugmentor`** → Remove the `retrievalAugmentor` attribute. Should already work since `void.class` was the default.

Update test classes in `integration-tests/rag/src/test/java/` if class names changed.

- [ ] **Step 7: Migrate core tests**

**`GuardrailWithAugmentationTest`** (line 116): Replace `@RegisterAiService(retrievalAugmentor = MyRetrievalAugmentor.class)` with `@RegisterAiService` + `@RagPipeline(augmentor = MyRetrievalAugmentor.class)`. The `MyRetrievalAugmentor` class already implements `RetrievalAugmentor` — no change needed.

**`ResponseAugmenterWithAugmentationResultTest`** (line 52): Same pattern — add `@RagPipeline(augmentor = MyRetrievalAugmentor.class)`.

- [ ] **Step 8: Migrate provider test**

**`DeclarativeAiServicesTest`** (line 148): Replace `@RegisterAiService(retrievalAugmentor = DummyRetriever.Supplier.class)` with `@RagPipeline(augmentor = DummyRetriever.class)`. Convert `DummyRetriever` to implement `RetrievalAugmentor` directly (remove the `Supplier` inner class, move the `augment()` logic into the class itself).

- [ ] **Step 9: Migrate samples**

For each sample, replace `@RegisterAiService(retrievalAugmentor = X.class)` with `@RegisterAiService` + `@RagPipeline(augmentor = X.class)`:
- `samples/chatbot/src/main/java/.../Bot.java`
- `samples/sql-chatbot/src/main/java/.../MovieMuse.java`
- `samples/secure-sql-chatbot/src/main/java/.../MovieMuse.java`
- `samples/secure-fraud-detection/src/main/java/.../FraudDetectionAi.java`

These augmentor classes already implement `RetrievalAugmentor` — the migration is mechanical (add `@RagPipeline(augmentor = ...)`, remove the attribute from `@RegisterAiService`).

Update `samples/chatbot-web-search/src/main/java/.../WebSearchRetrievalAugmentor.java` comment to reference `@RagPipeline` instead of `retrievalAugmentor =`.

- [ ] **Step 10: Commit**

```
git add -A
git commit -m "feat(#2574)!: remove retrievalAugmentor from @RegisterAiService, migrate all callers to @RagPipeline"
```

---

## Task 10: Run full test suite

- [ ] **Step 1: Format code**

```bash
/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/pom.xml process-sources -T 1C
```

- [ ] **Step 2: Run core tests**

```bash
/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/core/pom.xml test -T 1C
```

Expected: All tests pass (660+ core tests).

- [ ] **Step 3: Run RAG integration tests**

```bash
/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/integration-tests/rag/pom.xml test
```

Expected: All migrated RAG tests pass.

- [ ] **Step 4: Fix any failures and recommit**

If tests fail, fix and commit:
```
git commit -m "fix(#2574): address test failures after retrievalAugmentor removal"
```

---

## Task 11: Squash into two clean commits

- [ ] **Step 1: Interactive rebase to two commits**

Squash Tasks 1–8 into one additive commit. Squash Task 9–10 into one breaking commit:

Commit 1: `feat(#2574): add @RagPipeline composition annotation for declarative RAG pipeline`
Commit 2: `feat(#2574)!: remove retrievalAugmentor from @RegisterAiService, migrate all callers`

- [ ] **Step 2: Verify tests pass after squash**

```bash
/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/core/pom.xml test -T 1C
```

---

## Task 12: Format and push

- [ ] **Step 1: Run formatter**

```bash
/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/pom.xml process-sources -T 1C
```

- [ ] **Step 2: Verify formatter didn't break anything**

```bash
/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/core/pom.xml test -T 1C
```

- [ ] **Step 3: Push to fork**

```bash
git push fork issue-2572-register-ai-service-simplification
```
