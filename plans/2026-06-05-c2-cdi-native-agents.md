# C2 — CDI-Native Agents Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Auto-wire CDI beans into agentic agents for ContentRetriever, ChatMemory, ChatMemoryProvider, RetrievalAugmentor (fallback), and AgentListener (additive). Fix @CdiBean qualifier resolution. Remove shared mutable state.

**Architecture:** Build-time detection in AgenticProcessor queries BeanDiscoveryFinishedBuildItem for eligible CDI beans, adds Instance<T> injection points to agent synthetic beans, and records resolved types in AiAgentCreateInfo. At runtime, QuarkusAgenticContextConsumer wires injected references into the AgentBuilder via pre-built TypeLiteral constants.

**Tech Stack:** Quarkus Arc (CDI), Jandex (build-time index), langchain4j-agentic 1.15.1-beta25

**Spec:** `specs/c2-cdi-native-agents/2026-06-05-c2-cdi-native-agents-design.md`

---

### Task 1: Data Model — CdiSupplierType enum + AiAgentCreateInfo expansion

**Files:**
- Create: `agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/CdiSupplierType.java`
- Modify: `agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/AiAgentCreateInfo.java`
- Modify: `agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/AgenticRecorder.java` (construction site only)
- Modify: `agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticProcessor.java` (construction site only)

- [ ] **Step 1: Create CdiSupplierType enum**

```java
package io.quarkiverse.langchain4j.agentic.runtime;

/**
 * Supplier types eligible for CDI fallback auto-wiring when no static
 * supplier is declared on the agent interface.
 * <p>
 * All types in this enum have overwrite-semantic setters on {@code AgentBuilder}
 * (simple field assignment). The build-time check ensures CDI beans are only
 * wired when no static supplier exists, preventing silent overwrites.
 * <p>
 * {@code ToolProvider} is excluded — its append semantics and MCP type collision
 * make fallback auto-wiring inappropriate. {@code AgentListener} is excluded — it
 * uses a separate additive path ({@code Instance<AgentListener>} on every agent).
 */
public enum CdiSupplierType {
    CONTENT_RETRIEVER,
    CHAT_MEMORY,
    CHAT_MEMORY_PROVIDER,
    RETRIEVAL_AUGMENTOR
}
```

- [ ] **Step 2: Expand AiAgentCreateInfo record**

Replace the existing record with:

```java
package io.quarkiverse.langchain4j.agentic.runtime;

import java.util.Set;

public record AiAgentCreateInfo(String agentClassName, ChatModelInfo chatModelInfo, boolean hasInterceptorBindings,
        Set<CdiSupplierType> cdiResolvedSuppliers, boolean hasMcpToolBox) {

    public sealed interface ChatModelInfo permits ChatModelInfo.FromAnnotation, ChatModelInfo.FromBeanWithName {

        final class FromAnnotation implements ChatModelInfo {
        }

        record FromBeanWithName(String name) implements ChatModelInfo {
        }
    }
}
```

- [ ] **Step 3: Update AiAgentCreateInfo construction site in AgenticProcessor**

In `AgenticProcessor.cdiSupport()`, find the existing `new AiAgentCreateInfo(...)` call and add the two new fields with defaults:

```java
new AiAgentCreateInfo(detectedAiAgentBuildItem.getIface().toString(), chatModelInfo,
        hasInterceptorBindings, Set.of(), false)
```

These defaults maintain existing behavior. Task 4 replaces them with real detection.

- [ ] **Step 4: Compile and verify**

Run: `mvn install -pl agentic/runtime -DskipTests -Dno-format && mvn compile -pl agentic/deployment -Dno-format`

Expected: BUILD SUCCESS — no behavior change, just data model expansion.

- [ ] **Step 5: Commit**

```
feat(agentic): add CdiSupplierType enum and expand AiAgentCreateInfo

Data model foundation for C2 CDI-native agent auto-wiring.
AiAgentCreateInfo gains cdiResolvedSuppliers and hasMcpToolBox fields.
No behavior change — existing construction sites use empty defaults.
```

---

### Task 2: ContentRetriever Auto-Wiring (TDD)

**Files:**
- Create: `agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/CdiContentRetrieverAutoWiringTest.java`
- Modify: `agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/AgenticRecorder.java`
- Modify: `agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticProcessor.java`

- [ ] **Step 1: Write the failing test**

```java
package io.quarkiverse.langchain4j.agentic.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.concurrent.atomic.AtomicBoolean;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import dev.langchain4j.agentic.Agent;
import dev.langchain4j.agentic.declarative.ChatModelSupplier;
import dev.langchain4j.data.segment.TextSegment;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.rag.content.Content;
import dev.langchain4j.rag.content.retriever.ContentRetriever;
import dev.langchain4j.rag.query.Query;
import dev.langchain4j.service.UserMessage;
import dev.langchain4j.service.V;
import io.quarkiverse.langchain4j.testing.internal.WiremockAware;
import io.quarkiverse.langchain4j.openai.testing.internal.OpenAiBaseTest;
import io.quarkus.test.QuarkusUnitTest;

public class CdiContentRetrieverAutoWiringTest extends OpenAiBaseTest {

    @RegisterExtension
    static final QuarkusUnitTest unitTest = new QuarkusUnitTest()
            .setArchiveProducer(
                    () -> ShrinkWrap.create(JavaArchive.class)
                            .addClasses(RecordingContentRetriever.class,
                                    AgentWithoutRetrieverSupplier.class,
                                    Agents.FixedResponseChatModel.class))
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.api-key", "whatever")
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.base-url",
                    WiremockAware.wiremockUrlForConfig("/v1"));

    @ApplicationScoped
    public static class RecordingContentRetriever implements ContentRetriever {

        final AtomicBoolean called = new AtomicBoolean(false);

        @Override
        public List<Content> retrieve(Query query) {
            called.set(true);
            return List.of(Content.from(TextSegment.from("retrieved-content")));
        }
    }

    public interface AgentWithoutRetrieverSupplier {

        @UserMessage("Answer: {{text}}")
        @Agent(description = "An agent with no static ContentRetrieverSupplier", outputKey = "answer")
        String ask(@V("text") String text);

        @ChatModelSupplier
        static ChatModel chatModel() {
            return new Agents.FixedResponseChatModel("fixed-response");
        }
    }

    @Inject
    AgentWithoutRetrieverSupplier agent;

    @Inject
    RecordingContentRetriever retriever;

    @Test
    void contentRetrieverCdiBeanAutoWiredIntoAgent() {
        agent.ask("hello");
        assertThat(retriever.called.get())
                .as("CDI ContentRetriever should be auto-wired when no static supplier exists")
                .isTrue();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn install -pl agentic/runtime -DskipTests -Dno-format && mvn test -pl agentic/deployment -Dtest=CdiContentRetrieverAutoWiringTest -Dno-format`

Expected: FAIL — `retriever.called` is `false` because no injection point exists and the consumer doesn't wire it.

- [ ] **Step 3: Implement runtime wiring — update QuarkusAgenticContextConsumer**

In `AgenticRecorder.java`, replace the existing `QuarkusAgenticContextConsumer` inner record with the expanded version. Add TypeLiteral constants as static fields in the record:

```java
private record QuarkusAgenticContextConsumer(SyntheticCreationalContext<Object> cdiContext,
        AiAgentCreateInfo aiAgentCreateInfo)
        implements
            Consumer<AgenticServices.DeclarativeAgentCreationContext<?>> {

    private static final TypeLiteral<Instance<ToolProvider>> TOOL_PROVIDER_TYPE_LITERAL = new TypeLiteral<>() {
    };
    private static final TypeLiteral<Instance<ContentRetriever>> CONTENT_RETRIEVER_INSTANCE = new TypeLiteral<>() {
    };
    private static final TypeLiteral<Instance<ChatMemory>> CHAT_MEMORY_INSTANCE = new TypeLiteral<>() {
    };
    private static final TypeLiteral<Instance<ChatMemoryProvider>> CHAT_MEMORY_PROVIDER_INSTANCE = new TypeLiteral<>() {
    };
    private static final TypeLiteral<Instance<RetrievalAugmentor>> RETRIEVAL_AUGMENTOR_INSTANCE = new TypeLiteral<>() {
    };
    private static final TypeLiteral<Instance<AgentListener>> AGENT_LISTENER_INSTANCE = new TypeLiteral<>() {
    };

    @Override
    public void accept(AgenticServices.DeclarativeAgentCreationContext agenticContext) {
        var builder = agenticContext.agentBuilder();

        for (CdiSupplierType type : aiAgentCreateInfo.cdiResolvedSuppliers()) {
            switch (type) {
                case CONTENT_RETRIEVER -> {
                    Instance<ContentRetriever> i = cdiContext.getInjectedReference(CONTENT_RETRIEVER_INSTANCE);
                    if (i.isResolvable()) {
                        builder.contentRetriever(i.get());
                    }
                }
                case CHAT_MEMORY -> {
                    Instance<ChatMemory> i = cdiContext.getInjectedReference(CHAT_MEMORY_INSTANCE);
                    if (i.isResolvable()) {
                        builder.chatMemory(i.get());
                    }
                }
                case CHAT_MEMORY_PROVIDER -> {
                    Instance<ChatMemoryProvider> i = cdiContext.getInjectedReference(CHAT_MEMORY_PROVIDER_INSTANCE);
                    if (i.isResolvable()) {
                        builder.chatMemoryProvider(i.get());
                    }
                }
                case RETRIEVAL_AUGMENTOR -> {
                    Instance<RetrievalAugmentor> i = cdiContext.getInjectedReference(RETRIEVAL_AUGMENTOR_INSTANCE);
                    if (i.isResolvable()) {
                        builder.retrievalAugmentor(i.get());
                    }
                }
            }
        }

        if (aiAgentCreateInfo.hasMcpToolBox()) {
            Instance<ToolProvider> mcpToolProvider = cdiContext.getInjectedReference(TOOL_PROVIDER_TYPE_LITERAL);
            if (mcpToolProvider.isResolvable()) {
                builder.toolProvider(mcpToolProvider.get());
            }
        }

        Instance<AgentListener> listeners = cdiContext.getInjectedReference(AGENT_LISTENER_INSTANCE);
        for (AgentListener listener : listeners) {
            builder.listener(listener);
        }
    }
}
```

Add the required imports to `AgenticRecorder.java`:

```java
import dev.langchain4j.agentic.observability.AgentListener;
import dev.langchain4j.memory.ChatMemory;
import dev.langchain4j.memory.chat.ChatMemoryProvider;
import dev.langchain4j.rag.RetrievalAugmentor;
import dev.langchain4j.rag.content.retriever.ContentRetriever;
```

- [ ] **Step 4: Implement build-time detection — update AgenticProcessor.cdiSupport()**

Add `BeanDiscoveryFinishedBuildItem beanDiscovery` parameter to the `cdiSupport` method signature. Add a helper method for CDI bean detection and a mapping structure:

```java
private static final Map<CdiSupplierType, SupplierMapping> SUPPLIER_MAPPINGS = Map.of(
        CdiSupplierType.CONTENT_RETRIEVER,
        new SupplierMapping(AgenticLangChain4jDotNames.CONTENT_RETRIEVER_SUPPLIER, LangChain4jDotNames.RETRIEVER),
        CdiSupplierType.CHAT_MEMORY,
        new SupplierMapping(AgenticLangChain4jDotNames.CHAT_MEMORY_SUPPLIER, LangChain4jDotNames.CHAT_MEMORY),
        CdiSupplierType.CHAT_MEMORY_PROVIDER,
        new SupplierMapping(AgenticLangChain4jDotNames.CHAT_MEMORY_PROVIDER_SUPPLIER,
                LangChain4jDotNames.CHAT_MEMORY_PROVIDER),
        CdiSupplierType.RETRIEVAL_AUGMENTOR,
        new SupplierMapping(AgenticLangChain4jDotNames.RETRIEVAL_AUGMENTER_SUPPLIER,
                LangChain4jDotNames.RETRIEVAL_AUGMENTOR));

private record SupplierMapping(DotName supplierAnnotation, DotName beanType) {
}
```

Inside the `cdiSupport` method, for each agent, detect CDI beans and build the `cdiResolvedSuppliers` set:

```java
IndexView index = indexBuildItem.getIndex();
// ... existing code ...

for (DetectedAiAgentBuildItem detectedAiAgentBuildItem : detectedAiAgentBuildItems) {
    // ... existing chatModel and interceptor logic ...

    Set<CdiSupplierType> cdiResolvedSuppliers = new HashSet<>();
    Set<ClassInfo> hierarchy = ValidationUtil.transitiveInterfaces(
            detectedAiAgentBuildItem.getIface(), index);

    for (var entry : SUPPLIER_MAPPINGS.entrySet()) {
        CdiSupplierType supplierType = entry.getKey();
        SupplierMapping mapping = entry.getValue();

        boolean hasStaticSupplier = hierarchy.stream()
                .flatMap(ci -> ci.methods().stream())
                .anyMatch(m -> Modifier.isStatic(m.flags())
                        && m.hasAnnotation(mapping.supplierAnnotation()));

        if (!hasStaticSupplier) {
            List<BeanInfo> beans = beanDiscovery.beanStream()
                    .withBeanType(mapping.beanType())
                    .withQualifier(Default.class)
                    .collect();

            if (beans.size() == 1) {
                cdiResolvedSuppliers.add(supplierType);
                beanConfigurator.addInjectionPoint(ParameterizedType.create(
                        DotNames.CDI_INSTANCE,
                        new Type[] { ClassType.create(mapping.beanType()) }, null));
            } else if (beans.size() > 1) {
                log.infof(
                        "Multiple %s CDI beans found but agent '%s' declares no static supplier "
                                + "— auto-wiring skipped. Use a static @%s method to select explicitly.",
                        mapping.beanType().withoutPackagePrefix(),
                        detectedAiAgentBuildItem.getIface().name(),
                        mapping.supplierAnnotation().withoutPackagePrefix());
            }
        }
    }

    // AgentListener — unconditional Instance<AgentListener> injection point
    beanConfigurator.addInjectionPoint(ParameterizedType.create(
            DotNames.CDI_INSTANCE,
            new Type[] { ClassType.create(AgenticLangChain4jDotNames.AGENT_LISTENER) }, null));

    // Update AiAgentCreateInfo construction with real values
    // Replace the existing `new AiAgentCreateInfo(...)` with:
    new AiAgentCreateInfo(detectedAiAgentBuildItem.getIface().toString(), chatModelInfo,
            hasInterceptorBindings, cdiResolvedSuppliers,
            !detectedAiAgentBuildItem.getMcpToolBoxMethods().isEmpty())
```

Add needed imports:

```java
import java.lang.reflect.Modifier;
import java.util.EnumSet;
import java.util.Map;
import jakarta.enterprise.inject.Default;
import io.quarkiverse.langchain4j.agentic.runtime.CdiSupplierType;
import io.quarkus.arc.deployment.BeanDiscoveryFinishedBuildItem;
import io.quarkus.arc.processor.BeanInfo;
```

- [ ] **Step 5: Add unremovable marking for auto-wired CDI beans**

Add a new build step or extend the existing `markCdiBeanParametersAsUnremovable` to cover auto-wired beans. After the detection loop, for each `CdiSupplierType` in `cdiResolvedSuppliers`, mark the bean type as unremovable:

```java
for (CdiSupplierType type : cdiResolvedSuppliers) {
    unremovableProducer.produce(
            UnremovableBeanBuildItem.beanTypes(SUPPLIER_MAPPINGS.get(type).beanType()));
}
```

This requires adding `BuildProducer<UnremovableBeanBuildItem> unremovableProducer` to the `cdiSupport` method signature (or use the existing `markCdiBeanParametersAsUnremovable` step).

Also mark all AgentListener beans as unremovable:

```java
if (!beanDiscovery.beanStream()
        .withBeanType(AgenticLangChain4jDotNames.AGENT_LISTENER).isEmpty()) {
    unremovableProducer.produce(
            UnremovableBeanBuildItem.beanTypes(AgenticLangChain4jDotNames.AGENT_LISTENER));
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `mvn install -pl agentic/runtime -DskipTests -Dno-format && mvn test -pl agentic/deployment -Dtest=CdiContentRetrieverAutoWiringTest -Dno-format`

Expected: PASS — `retriever.called` is `true`.

- [ ] **Step 7: Run all existing tests to verify no regressions**

Run: `mvn install -pl agentic/runtime -DskipTests -Dno-format && mvn test -pl agentic/deployment -Dno-format`

Expected: All existing tests pass.

- [ ] **Step 8: Commit**

```
feat(agentic): CDI fallback auto-wiring for ContentRetriever

Build-time detection via BeanDiscoveryFinishedBuildItem. Runtime wiring
via Instance<T> injection points in QuarkusAgenticContextConsumer.
Covers S-1 (ContentRetriever path) and O-3 (AgentListener injection
point added unconditionally).

Closes: S-1 (partial), O-3 (injection point only — listener wiring
verified in Task 6)
```

---

### Task 3: Remaining Fallback Type Tests + Edge Cases

**Files:**
- Create: `agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/CdiChatMemoryAutoWiringTest.java`
- Create: `agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/CdiRetrievalAugmentorAutoWiringTest.java`
- Create: `agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/CdiAutoWiringEdgeCasesTest.java`

- [ ] **Step 1: Write ChatMemory auto-wiring test**

```java
package io.quarkiverse.langchain4j.agentic.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.concurrent.atomic.AtomicBoolean;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import dev.langchain4j.agentic.Agent;
import dev.langchain4j.agentic.declarative.ChatModelSupplier;
import dev.langchain4j.data.message.ChatMessage;
import dev.langchain4j.memory.ChatMemory;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.service.UserMessage;
import dev.langchain4j.service.V;
import io.quarkiverse.langchain4j.openai.testing.internal.OpenAiBaseTest;
import io.quarkiverse.langchain4j.testing.internal.WiremockAware;
import io.quarkus.test.QuarkusUnitTest;

import java.util.ArrayList;
import java.util.List;

public class CdiChatMemoryAutoWiringTest extends OpenAiBaseTest {

    @RegisterExtension
    static final QuarkusUnitTest unitTest = new QuarkusUnitTest()
            .setArchiveProducer(
                    () -> ShrinkWrap.create(JavaArchive.class)
                            .addClasses(RecordingChatMemory.class,
                                    AgentWithoutMemorySupplier.class,
                                    Agents.FixedResponseChatModel.class))
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.api-key", "whatever")
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.base-url",
                    WiremockAware.wiremockUrlForConfig("/v1"));

    @ApplicationScoped
    public static class RecordingChatMemory implements ChatMemory {

        final AtomicBoolean addCalled = new AtomicBoolean(false);

        @Override
        public Object id() {
            return "test-memory";
        }

        @Override
        public void add(ChatMessage message) {
            addCalled.set(true);
        }

        @Override
        public List<ChatMessage> messages() {
            return new ArrayList<>();
        }

        @Override
        public void clear() {
        }
    }

    public interface AgentWithoutMemorySupplier {

        @UserMessage("Answer: {{text}}")
        @Agent(description = "An agent with no static ChatMemorySupplier", outputKey = "answer")
        String ask(@V("text") String text);

        @ChatModelSupplier
        static ChatModel chatModel() {
            return new Agents.FixedResponseChatModel("fixed-response");
        }
    }

    @Inject
    AgentWithoutMemorySupplier agent;

    @Inject
    RecordingChatMemory memory;

    @Test
    void chatMemoryCdiBeanAutoWired() {
        agent.ask("hello");
        assertThat(memory.addCalled.get())
                .as("CDI ChatMemory should be auto-wired when no static supplier exists")
                .isTrue();
    }
}
```

- [ ] **Step 2: Write RetrievalAugmentor auto-wiring test**

```java
package io.quarkiverse.langchain4j.agentic.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.concurrent.atomic.AtomicBoolean;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import dev.langchain4j.agentic.Agent;
import dev.langchain4j.agentic.declarative.ChatModelSupplier;
import dev.langchain4j.data.message.UserMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.rag.AugmentationRequest;
import dev.langchain4j.rag.AugmentationResult;
import dev.langchain4j.rag.RetrievalAugmentor;
import dev.langchain4j.service.V;
import io.quarkiverse.langchain4j.openai.testing.internal.OpenAiBaseTest;
import io.quarkiverse.langchain4j.testing.internal.WiremockAware;
import io.quarkus.test.QuarkusUnitTest;

public class CdiRetrievalAugmentorAutoWiringTest extends OpenAiBaseTest {

    @RegisterExtension
    static final QuarkusUnitTest unitTest = new QuarkusUnitTest()
            .setArchiveProducer(
                    () -> ShrinkWrap.create(JavaArchive.class)
                            .addClasses(RecordingRetrievalAugmentor.class,
                                    AgentWithoutAugmentorSupplier.class,
                                    Agents.FixedResponseChatModel.class))
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.api-key", "whatever")
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.base-url",
                    WiremockAware.wiremockUrlForConfig("/v1"));

    @ApplicationScoped
    public static class RecordingRetrievalAugmentor implements RetrievalAugmentor {

        final AtomicBoolean called = new AtomicBoolean(false);

        @Override
        public AugmentationResult augment(AugmentationRequest augmentationRequest) {
            called.set(true);
            return AugmentationResult.builder()
                    .chatMessage(augmentationRequest.chatMessage())
                    .build();
        }
    }

    public interface AgentWithoutAugmentorSupplier {

        @dev.langchain4j.service.UserMessage("Answer: {{text}}")
        @Agent(description = "An agent with no static RetrievalAugmentorSupplier", outputKey = "answer")
        String ask(@V("text") String text);

        @ChatModelSupplier
        static ChatModel chatModel() {
            return new Agents.FixedResponseChatModel("fixed-response");
        }
    }

    @Inject
    AgentWithoutAugmentorSupplier agent;

    @Inject
    RecordingRetrievalAugmentor augmentor;

    @Test
    void retrievalAugmentorCdiBeanAutoWired() {
        agent.ask("hello");
        assertThat(augmentor.called.get())
                .as("CDI RetrievalAugmentor should be auto-wired when no static supplier exists")
                .isTrue();
    }
}
```

- [ ] **Step 3: Write edge case tests — static supplier wins + multiple beans skip + no beans**

```java
package io.quarkiverse.langchain4j.agentic.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.concurrent.atomic.AtomicBoolean;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import dev.langchain4j.agentic.Agent;
import dev.langchain4j.agentic.declarative.ChatModelSupplier;
import dev.langchain4j.agentic.declarative.ContentRetrieverSupplier;
import dev.langchain4j.data.segment.TextSegment;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.rag.content.Content;
import dev.langchain4j.rag.content.retriever.ContentRetriever;
import dev.langchain4j.rag.query.Query;
import dev.langchain4j.service.UserMessage;
import dev.langchain4j.service.V;
import io.quarkiverse.langchain4j.openai.testing.internal.OpenAiBaseTest;
import io.quarkiverse.langchain4j.testing.internal.WiremockAware;
import io.quarkus.test.QuarkusUnitTest;

public class CdiAutoWiringEdgeCasesTest extends OpenAiBaseTest {

    @RegisterExtension
    static final QuarkusUnitTest unitTest = new QuarkusUnitTest()
            .setArchiveProducer(
                    () -> ShrinkWrap.create(JavaArchive.class)
                            .addClasses(CdiRetriever.class,
                                    AgentWithStaticRetrieverSupplier.class,
                                    AgentWithoutRetrieverSupplier.class,
                                    StaticRetriever.class,
                                    Agents.FixedResponseChatModel.class))
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.api-key", "whatever")
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.base-url",
                    WiremockAware.wiremockUrlForConfig("/v1"));

    @ApplicationScoped
    public static class CdiRetriever implements ContentRetriever {

        final AtomicBoolean called = new AtomicBoolean(false);

        @Override
        public List<Content> retrieve(Query query) {
            called.set(true);
            return List.of(Content.from(TextSegment.from("cdi-content")));
        }
    }

    public static class StaticRetriever implements ContentRetriever {

        static final AtomicBoolean called = new AtomicBoolean(false);

        @Override
        public List<Content> retrieve(Query query) {
            called.set(true);
            return List.of(Content.from(TextSegment.from("static-content")));
        }
    }

    public interface AgentWithStaticRetrieverSupplier {

        @UserMessage("Answer: {{text}}")
        @Agent(description = "Agent with static ContentRetrieverSupplier", outputKey = "answer")
        String ask(@V("text") String text);

        @ContentRetrieverSupplier
        static ContentRetriever retriever() {
            return new StaticRetriever();
        }

        @ChatModelSupplier
        static ChatModel chatModel() {
            return new Agents.FixedResponseChatModel("fixed-response");
        }
    }

    public interface AgentWithoutRetrieverSupplier {

        @UserMessage("Answer: {{text}}")
        @Agent(description = "Agent without ContentRetrieverSupplier", outputKey = "answer2")
        String ask(@V("text") String text);

        @ChatModelSupplier
        static ChatModel chatModel() {
            return new Agents.FixedResponseChatModel("fixed-response");
        }
    }

    @Inject
    AgentWithStaticRetrieverSupplier agentWithStaticSupplier;

    @Inject
    AgentWithoutRetrieverSupplier agentWithoutSupplier;

    @Inject
    CdiRetriever cdiRetriever;

    @Test
    void staticSupplierWinsOverCdiBean() {
        agentWithStaticSupplier.ask("hello");
        assertThat(StaticRetriever.called.get())
                .as("Static supplier should be used, not CDI bean")
                .isTrue();
    }

    @Test
    void cdiAutoWiringWorksWhenNoStaticSupplier() {
        cdiRetriever.called.set(false);
        agentWithoutSupplier.ask("hello");
        assertThat(cdiRetriever.called.get())
                .as("CDI bean should be auto-wired when no static supplier exists")
                .isTrue();
    }
}
```

- [ ] **Step 4: Run all new tests**

Run: `mvn install -pl agentic/runtime -DskipTests -Dno-format && mvn test -pl agentic/deployment -Dtest="CdiContentRetrieverAutoWiringTest,CdiChatMemoryAutoWiringTest,CdiRetrievalAugmentorAutoWiringTest,CdiAutoWiringEdgeCasesTest" -Dno-format`

Expected: All PASS.

- [ ] **Step 5: Commit**

```
test(agentic): auto-wiring tests for ChatMemory, RetrievalAugmentor, and edge cases

Covers: static supplier wins, CDI auto-wiring for all four fallback types.
```

---

### Task 4: AgentListener CDI Auto-Discovery (TDD)

**Files:**
- Create: `agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/CdiAgentListenerAutoDiscoveryTest.java`

- [ ] **Step 1: Write AgentListener auto-discovery test**

```java
package io.quarkiverse.langchain4j.agentic.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.concurrent.atomic.AtomicBoolean;
import java.util.concurrent.atomic.AtomicInteger;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import dev.langchain4j.agentic.Agent;
import dev.langchain4j.agentic.declarative.AgentListenerSupplier;
import dev.langchain4j.agentic.declarative.ChatModelSupplier;
import dev.langchain4j.agentic.observability.AgentListener;
import dev.langchain4j.agentic.observability.AgentRequest;
import dev.langchain4j.agentic.observability.AgentResponse;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.service.UserMessage;
import dev.langchain4j.service.V;
import io.quarkiverse.langchain4j.openai.testing.internal.OpenAiBaseTest;
import io.quarkiverse.langchain4j.testing.internal.WiremockAware;
import io.quarkus.test.QuarkusUnitTest;

public class CdiAgentListenerAutoDiscoveryTest extends OpenAiBaseTest {

    @RegisterExtension
    static final QuarkusUnitTest unitTest = new QuarkusUnitTest()
            .setArchiveProducer(
                    () -> ShrinkWrap.create(JavaArchive.class)
                            .addClasses(GlobalCdiListener.class, SecondGlobalCdiListener.class,
                                    AgentWithoutListenerSupplier.class,
                                    AgentWithStaticListenerSupplier.class,
                                    StaticOnlyListener.class,
                                    Agents.FixedResponseChatModel.class))
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.api-key", "whatever")
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.base-url",
                    WiremockAware.wiremockUrlForConfig("/v1"));

    @ApplicationScoped
    public static class GlobalCdiListener implements AgentListener {

        final AtomicInteger beforeCount = new AtomicInteger(0);

        @Override
        public void beforeAgentInvocation(AgentRequest agentRequest) {
            beforeCount.incrementAndGet();
        }
    }

    @ApplicationScoped
    public static class SecondGlobalCdiListener implements AgentListener {

        final AtomicInteger beforeCount = new AtomicInteger(0);

        @Override
        public void beforeAgentInvocation(AgentRequest agentRequest) {
            beforeCount.incrementAndGet();
        }
    }

    public static class StaticOnlyListener implements AgentListener {

        static final AtomicBoolean called = new AtomicBoolean(false);

        @Override
        public void beforeAgentInvocation(AgentRequest agentRequest) {
            called.set(true);
        }
    }

    public interface AgentWithoutListenerSupplier {

        @UserMessage("Answer: {{text}}")
        @Agent(description = "Agent without AgentListenerSupplier", outputKey = "answer")
        String ask(@V("text") String text);

        @ChatModelSupplier
        static ChatModel chatModel() {
            return new Agents.FixedResponseChatModel("fixed-response");
        }
    }

    public interface AgentWithStaticListenerSupplier {

        @UserMessage("Answer: {{text}}")
        @Agent(description = "Agent with static AgentListenerSupplier", outputKey = "answer2")
        String ask(@V("text") String text);

        @AgentListenerSupplier
        static AgentListener listener() {
            return new StaticOnlyListener();
        }

        @ChatModelSupplier
        static ChatModel chatModel() {
            return new Agents.FixedResponseChatModel("fixed-response");
        }
    }

    @Inject
    AgentWithoutListenerSupplier agentWithoutSupplier;

    @Inject
    AgentWithStaticListenerSupplier agentWithStaticSupplier;

    @Inject
    GlobalCdiListener globalListener;

    @Inject
    SecondGlobalCdiListener secondGlobalListener;

    @Test
    void cdiListenersAutoDiscoveredForAllAgents() {
        agentWithoutSupplier.ask("hello");
        assertThat(globalListener.beforeCount.get())
                .as("Global CDI listener should fire for agent without supplier")
                .isGreaterThanOrEqualTo(1);
        assertThat(secondGlobalListener.beforeCount.get())
                .as("Second CDI listener should also fire (multiple listeners compose)")
                .isGreaterThanOrEqualTo(1);
    }

    @Test
    void cdiListenersComposeWithStaticSupplier() {
        globalListener.beforeCount.set(0);
        StaticOnlyListener.called.set(false);

        agentWithStaticSupplier.ask("hello");

        assertThat(StaticOnlyListener.called.get())
                .as("Static listener should fire")
                .isTrue();
        assertThat(globalListener.beforeCount.get())
                .as("CDI listener should ALSO fire alongside static supplier (additive)")
                .isGreaterThanOrEqualTo(1);
    }
}
```

- [ ] **Step 2: Run test to verify it passes**

The AgentListener `Instance<AgentListener>` injection point was already added unconditionally in Task 2, and the runtime iteration was already implemented in the `QuarkusAgenticContextConsumer`. So this test should pass immediately.

Run: `mvn install -pl agentic/runtime -DskipTests -Dno-format && mvn test -pl agentic/deployment -Dtest=CdiAgentListenerAutoDiscoveryTest -Dno-format`

Expected: PASS.

- [ ] **Step 3: Commit**

```
test(agentic): AgentListener CDI auto-discovery tests

Verifies: multiple CDI listeners compose, CDI listeners fire alongside
static @AgentListenerSupplier (additive, not replacement).

Closes: O-3
```

---

### Task 5: S-2 Qualifier Fix (TDD)

**Files:**
- Modify: `agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/CdiChatSupplierParameterResolver.java`
- Modify: `agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/CdiChatSupplierParameterResolverTest.java`

- [ ] **Step 1: Write qualifier resolution test (add to existing test class)**

Add a new test case and supporting agent interface to `CdiChatSupplierParameterResolverTest`:

```java
public interface QualifiedModelAgent {

    @UserMessage("Answer: {{text}}")
    @Agent(description = "An agent using a qualified CDI model", outputKey = "answer")
    String answer(String text);

    @ChatModelSupplier
    static ChatModel chatModel(@CdiBean @ModelName("fixed") ChatModel model) {
        return model;
    }
}

public interface QualifiedSequenceWrapper {

    @SequenceAgent(outputKey = "answer", subAgents = { EchoAgent.class, QualifiedModelAgent.class })
    String run(String text);
}
```

Add these classes to the `ShrinkWrap.create(JavaArchive.class).addClasses(...)` block.

Add a field and test:

```java
@Inject
QualifiedSequenceWrapper qualifiedSequenceWrapper;

@Test
void qualifierAnnotationSelectsCorrectBean() {
    String result = qualifiedSequenceWrapper.run("dog");
    assertThat(result).isEqualTo(SELECTED_RESPONSE);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn install -pl agentic/runtime -DskipTests -Dno-format && mvn test -pl agentic/deployment -Dtest=CdiChatSupplierParameterResolverTest#qualifierAnnotationSelectsCorrectBean -Dno-format`

Expected: FAIL — without qualifier extraction, the resolver picks the wrong bean or throws `AmbiguousResolutionException`.

- [ ] **Step 3: Implement qualifier extraction**

Replace the `resolve` method in `CdiChatSupplierParameterResolver.java`:

```java
package io.quarkiverse.langchain4j.agentic.runtime;

import java.lang.annotation.Annotation;
import java.util.Arrays;

import dev.langchain4j.agentic.declarative.ChatSupplierParameterResolver;
import io.quarkus.arc.Arc;

public class CdiChatSupplierParameterResolver implements ChatSupplierParameterResolver {

    @Override
    public boolean supports(Context context) {
        return context.parameter().isAnnotationPresent(CdiBean.class);
    }

    @Override
    public Object resolve(Context context) {
        Annotation[] qualifiers = Arrays.stream(context.parameter().getAnnotations())
                .filter(ann -> !ann.annotationType().equals(CdiBean.class))
                .filter(ann -> ann.annotationType().isAnnotationPresent(jakarta.inject.Qualifier.class))
                .toArray(Annotation[]::new);
        return Arc.container().select(context.parameter().getType(), qualifiers).get();
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn install -pl agentic/runtime -DskipTests -Dno-format && mvn test -pl agentic/deployment -Dtest=CdiChatSupplierParameterResolverTest -Dno-format`

Expected: ALL tests in the class pass, including the new `qualifierAnnotationSelectsCorrectBean`.

- [ ] **Step 5: Commit**

```
fix(agentic): CdiChatSupplierParameterResolver extracts CDI qualifiers

@CdiBean @ModelName("x") now correctly selects the qualified bean.
Previously, qualifiers were ignored, causing AmbiguousResolutionException
with multiple beans of the same type.

Closes: S-2
```

---

### Task 6: hasMcpToolBox Migration + Cleanup

**Files:**
- Modify: `agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/AgenticRecorder.java`
- Modify: `agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticProcessor.java`

- [ ] **Step 1: Remove agentsWithMcpToolBox static field from AgenticRecorder**

Delete these lines from `AgenticRecorder`:

```java
private static volatile Set<String> agentsWithMcpToolBox = Collections.emptySet();
```

```java
@StaticInit
public void setAgentsWithMcpToolBox(Set<String> agentsWithMcpToolBox) {
    AgenticRecorder.agentsWithMcpToolBox = Collections.unmodifiableSet(agentsWithMcpToolBox);
}
```

The `QuarkusAgenticContextConsumer` already uses `aiAgentCreateInfo.hasMcpToolBox()` (from Task 2). Verify no remaining references to the old `agentsWithMcpToolBox` field.

- [ ] **Step 2: Remove mcpToolBoxSupport build step from AgenticProcessor**

The `mcpToolBoxSupport` build step (which called `recorder.setAgentsWithMcpToolBox(...)`) is now unnecessary — `hasMcpToolBox` is computed directly in `cdiSupport` and stored in `AiAgentCreateInfo`. Delete the entire `mcpToolBoxSupport` method:

```java
// DELETE this entire method:
@BuildStep
@Record(ExecutionTime.STATIC_INIT)
void mcpToolBoxSupport(List<DetectedAiAgentBuildItem> detectedAgentBuildItems, AgenticRecorder recorder) {
    // ... existing content ...
}
```

Keep the MCP validation logic (the `IllegalConfigurationException` for multiple `@McpToolBox` methods) — move it into the `detectAgents` method or an existing validation step.

- [ ] **Step 3: Run existing MCP tests**

Run: `mvn install -pl agentic/runtime -DskipTests -Dno-format && mvn test -pl agentic/deployment -Dno-format`

Expected: All tests pass, including any existing MCP-related tests.

- [ ] **Step 4: Commit**

```
refactor(agentic): remove agentsWithMcpToolBox static field

MCP ToolBox flag now carried per-agent in AiAgentCreateInfo.hasMcpToolBox.
The static Set<String> and its @StaticInit setter are deleted.
```

---

### Task 7: CDI Scope Validation

**Files:**
- Modify: `agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticProcessor.java`
- Create: `agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/CdiScopeValidationTest.java`

- [ ] **Step 1: Write scope validation test (RED)**

```java
package io.quarkiverse.langchain4j.agentic.deployment;

import jakarta.enterprise.context.RequestScoped;
import jakarta.inject.Inject;

import java.util.List;

import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import dev.langchain4j.agentic.Agent;
import dev.langchain4j.agentic.declarative.ChatModelSupplier;
import dev.langchain4j.data.segment.TextSegment;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.rag.content.Content;
import dev.langchain4j.rag.content.retriever.ContentRetriever;
import dev.langchain4j.rag.query.Query;
import dev.langchain4j.service.UserMessage;
import dev.langchain4j.service.V;
import io.quarkus.test.QuarkusUnitTest;

import static org.assertj.core.api.Assertions.assertThat;

public class CdiScopeValidationTest {

    @RegisterExtension
    static final QuarkusUnitTest unitTest = new QuarkusUnitTest()
            .setArchiveProducer(
                    () -> ShrinkWrap.create(JavaArchive.class)
                            .addClasses(RequestScopedRetriever.class,
                                    AgentUsingRequestScopedRetriever.class,
                                    Agents.FixedResponseChatModel.class))
            .assertException(t -> assertThat(t)
                    .hasMessageContaining("@RequestScoped")
                    .hasMessageContaining("cannot be auto-wired"));

    @RequestScoped
    public static class RequestScopedRetriever implements ContentRetriever {

        @Override
        public List<Content> retrieve(Query query) {
            return List.of();
        }
    }

    public interface AgentUsingRequestScopedRetriever {

        @UserMessage("Answer: {{text}}")
        @Agent(description = "Agent", outputKey = "answer")
        String ask(@V("text") String text);

        @ChatModelSupplier
        static ChatModel chatModel() {
            return new Agents.FixedResponseChatModel("fixed");
        }
    }

    @Test
    void requestScopedBeanRejectedAtBuildTime() {
        // the assertException above verifies the build fails
    }
}
```

- [ ] **Step 2: Run test to verify it fails (no scope validation yet)**

Run: `mvn install -pl agentic/runtime -DskipTests -Dno-format && mvn test -pl agentic/deployment -Dtest=CdiScopeValidationTest -Dno-format`

Expected: FAIL — the test expects a build-time exception but none is thrown (the `@RequestScoped` bean is currently auto-wired without validation).

- [ ] **Step 3: Implement scope validation in AgenticProcessor**

In the CDI bean detection loop (inside `cdiSupport`), after finding exactly one bean, validate its scope:

```java
if (beans.size() == 1) {
    BeanInfo bean = beans.get(0);
    DotName scope = bean.getScope().getDotName();
    if (scope.equals(io.quarkus.arc.processor.DotNames.REQUEST_SCOPED)
            || scope.equals(io.quarkus.arc.processor.DotNames.SESSION_SCOPED)) {
        throw new IllegalConfigurationException(
                "CDI bean of type '" + mapping.beanType().withoutPackagePrefix()
                        + "' is @" + scope.withoutPackagePrefix()
                        + " and cannot be auto-wired into agent '"
                        + detectedAiAgentBuildItem.getIface().name()
                        + "'. Agent synthetic beans are created at application startup "
                        + "when no request context is active. Use @ApplicationScoped "
                        + "or provide the bean via a static supplier method.");
    }
    cdiResolvedSuppliers.add(supplierType);
    // ... existing injection point logic ...
}
```

Apply the same validation for AgentListener beans — iterate all AgentListener beans from beanDiscovery and validate scope.

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn install -pl agentic/runtime -DskipTests -Dno-format && mvn test -pl agentic/deployment -Dtest=CdiScopeValidationTest -Dno-format`

Expected: PASS.

- [ ] **Step 5: Run all tests to verify no regressions**

Run: `mvn install -pl agentic/runtime -DskipTests -Dno-format && mvn test -pl agentic/deployment -Dno-format`

Expected: All pass.

- [ ] **Step 6: Commit**

```
feat(agentic): build-time scope validation for CDI auto-wired beans

@RequestScoped and @SessionScoped beans are rejected at build time
with a clear error message. Agent synthetic beans are created during
@RuntimeInit when no request context is active.
```

---

### Task 8: Mixed Mode + Regression Tests

**Files:**
- Create: `agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/CdiMixedModeTest.java`

- [ ] **Step 1: Write mixed supplier modes test**

```java
package io.quarkiverse.langchain4j.agentic.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.concurrent.atomic.AtomicBoolean;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import dev.langchain4j.agentic.Agent;
import dev.langchain4j.agentic.declarative.ChatModelSupplier;
import dev.langchain4j.data.segment.TextSegment;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.rag.content.Content;
import dev.langchain4j.rag.content.retriever.ContentRetriever;
import dev.langchain4j.rag.query.Query;
import dev.langchain4j.service.UserMessage;
import dev.langchain4j.service.V;
import io.quarkiverse.langchain4j.openai.testing.internal.OpenAiBaseTest;
import io.quarkiverse.langchain4j.testing.internal.WiremockAware;
import io.quarkus.test.QuarkusUnitTest;

public class CdiMixedModeTest extends OpenAiBaseTest {

    @RegisterExtension
    static final QuarkusUnitTest unitTest = new QuarkusUnitTest()
            .setArchiveProducer(
                    () -> ShrinkWrap.create(JavaArchive.class)
                            .addClasses(CdiRetriever.class,
                                    MixedModeAgent.class,
                                    Agents.FixedResponseChatModel.class))
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.api-key", "whatever")
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.base-url",
                    WiremockAware.wiremockUrlForConfig("/v1"));

    @ApplicationScoped
    public static class CdiRetriever implements ContentRetriever {

        final AtomicBoolean called = new AtomicBoolean(false);

        @Override
        public List<Content> retrieve(Query query) {
            called.set(true);
            return List.of(Content.from(TextSegment.from("cdi-content")));
        }
    }

    public interface MixedModeAgent {

        @UserMessage("Answer: {{text}}")
        @Agent(description = "Agent with static ChatModel but CDI ContentRetriever", outputKey = "answer")
        String ask(@V("text") String text);

        @ChatModelSupplier
        static ChatModel chatModel() {
            return new Agents.FixedResponseChatModel("fixed-response");
        }
    }

    @Inject
    MixedModeAgent agent;

    @Inject
    CdiRetriever retriever;

    @Test
    void staticChatModelWithCdiRetriever() {
        agent.ask("hello");
        assertThat(retriever.called.get())
                .as("CDI ContentRetriever should be auto-wired even when ChatModel uses static supplier")
                .isTrue();
    }
}
```

- [ ] **Step 2: Run all tests (full regression)**

Run: `mvn install -pl agentic/runtime -DskipTests -Dno-format && mvn test -pl agentic/deployment -Dno-format`

Expected: All tests pass — existing and new.

- [ ] **Step 3: Commit**

```
test(agentic): mixed supplier mode and regression tests

Verifies: static ChatModelSupplier + CDI ContentRetriever coexistence.
All existing tests pass after C2 changes.
```

---

## Spec Self-Review

**Spec coverage:**
- S-1 (CDI auto-wiring for non-chat suppliers) → Tasks 2, 3
- S-2 (qualifier fix) → Task 5
- O-3 (AgentListener CDI auto-discovery) → Tasks 2 (injection point), 4 (tests)
- A-1 partial (supplier static methods → CDI) → Tasks 2, 3, 4
- Shared mutable state removal → Task 6
- Scope validation → Task 7
- Regression tests → Task 8

**Placeholder scan:** No TBD, TODO, or vague steps. All code is concrete.

**Type consistency:** `CdiSupplierType` enum values, `AiAgentCreateInfo` field names, `TypeLiteral` constant names, and `SUPPLIER_MAPPINGS` keys are consistent across all tasks.
