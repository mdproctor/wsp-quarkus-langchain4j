# Foundation: Direct Bean-Class Attributes Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace all supplier-class attributes on `@RegisterAiService` with `Class<?>` direct bean-class references, delete all 17 sentinel marker inner classes, and migrate all usages.

**Architecture:** The tri-state resolution model (`void.class` = disabled, interface type = auto-discover, concrete class = explicit bean) replaces the sentinel marker pattern. A `ComponentResolutionMode` enum drives processor and recorder logic, eliminating string-based class name comparisons against sentinel names. All test `Supplier<ChatModel>` implementations become `@ApplicationScoped` CDI beans.

**Tech Stack:** Java 17, Quarkus CDI (Arc), Jandex, `@BuildStep` processors, `@Recorder` runtime init

**Spec:** `specs/2026-06-12-register-ai-service-composition-design.md` §1
**Issue:** [#2578](https://github.com/quarkiverse/quarkus-langchain4j/issues/2578)

---

## File Map

### New files
- `core/runtime/src/main/java/io/quarkiverse/langchain4j/runtime/aiservice/ComponentResolutionMode.java` — resolution mode enum
- `core/deployment/src/test/java/io/quarkiverse/langchain4j/test/DirectBeanClassAttributeTest.java` — TDD anchor test

### Modified files (framework)
- `core/runtime/src/main/java/io/quarkiverse/langchain4j/RegisterAiService.java` — annotation rewrite + marker deletion
- `core/deployment/src/main/java/io/quarkiverse/langchain4j/deployment/LangChain4jDotNames.java` — remove sentinel DotNames
- `core/deployment/src/main/java/io/quarkiverse/langchain4j/deployment/DeclarativeAiServiceBuildItem.java` — add resolution modes
- `core/runtime/src/main/java/io/quarkiverse/langchain4j/runtime/aiservice/DeclarativeAiServiceCreateInfo.java` — new record with resolution modes
- `core/deployment/src/main/java/io/quarkiverse/langchain4j/deployment/AiServicesProcessor.java` — new resolution logic
- `core/runtime/src/main/java/io/quarkiverse/langchain4j/runtime/AiServicesRecorder.java` — new wiring logic

### Modified files (migration — ~100 files)
All files referencing `chatLanguageModelSupplier`, `streamingChatLanguageModelSupplier`, `chatMemoryProviderSupplier`, `moderationModelSupplier`, `toolProviderSupplier`, `toolSearchStrategySupplier`, `chatMemoryFlushStrategySupplier`, `systemMessageProviderSupplier`, or any sentinel marker class. See Task 6 for batch migration patterns.

---

### Task 1: Create ComponentResolutionMode enum

**Files:**
- Create: `core/runtime/src/main/java/io/quarkiverse/langchain4j/runtime/aiservice/ComponentResolutionMode.java`

- [ ] **Step 1: Write the enum**

```java
package io.quarkiverse.langchain4j.runtime.aiservice;

/**
 * How a component reference on {@code @RegisterAiService} should be resolved.
 * Determined at build time from the annotation attribute value.
 */
public enum ComponentResolutionMode {
    /** {@code void.class} — component is not configured / disabled. */
    SKIP,
    /** Interface type (e.g. {@code ChatMemoryProvider.class}) — auto-discover CDI bean via {@code Instance<T>}. */
    AUTO_DISCOVER,
    /** Concrete class — inject specific CDI bean by type. */
    EXPLICIT
}
```

- [ ] **Step 2: Commit**

```
git add core/runtime/src/main/java/io/quarkiverse/langchain4j/runtime/aiservice/ComponentResolutionMode.java
git commit -m "feat(#2578): add ComponentResolutionMode enum for tri-state resolution"
```

---

### Task 2: Write TDD anchor test for new attribute pattern

**Files:**
- Create: `core/deployment/src/test/java/io/quarkiverse/langchain4j/test/DirectBeanClassAttributeTest.java`

This test exercises the desired end-state: `@RegisterAiService` with `Class<?>` attributes, no suppliers, no sentinel markers.

- [ ] **Step 1: Write the failing test**

```java
package io.quarkiverse.langchain4j.test;

import static org.assertj.core.api.Assertions.assertThat;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import dev.langchain4j.memory.chat.ChatMemoryProvider;
import dev.langchain4j.memory.chat.MessageWindowChatMemory;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import dev.langchain4j.model.chat.response.ChatResponse;
import dev.langchain4j.model.chat.response.ChatResponseMetadata;
import dev.langchain4j.service.UserMessage;
import io.quarkiverse.langchain4j.RegisterAiService;
import io.quarkus.test.QuarkusUnitTest;

/**
 * Validates the new Class<?> direct bean-class attribute pattern on @RegisterAiService.
 * Tests tri-state resolution: void.class (skip), interface type (auto-discover), concrete class (explicit).
 */
public class DirectBeanClassAttributeTest {

    @RegisterExtension
    static final QuarkusUnitTest unitTest = new QuarkusUnitTest()
            .setArchiveProducer(() -> ShrinkWrap.create(JavaArchive.class)
                    .addClasses(
                            ServiceWithExplicitMemory.class,
                            ServiceWithNoMemory.class,
                            ServiceWithAutoDiscoverMemory.class,
                            FixedChatModel.class,
                            ExplicitMemoryProvider.class));

    @Inject
    ServiceWithExplicitMemory explicitMemoryService;

    @Inject
    ServiceWithNoMemory noMemoryService;

    @Inject
    ServiceWithAutoDiscoverMemory autoDiscoverService;

    @Test
    void explicitMemoryProvider_usesSpecifiedBean() {
        String result = explicitMemoryService.chat("hello");
        assertThat(result).isNotNull();
    }

    @Test
    void noMemory_worksWithoutMemoryProvider() {
        String result = noMemoryService.chat("hello");
        assertThat(result).isNotNull();
    }

    @Test
    void autoDiscoverMemory_findsDefaultBean() {
        String result = autoDiscoverService.chat("hello");
        assertThat(result).isNotNull();
    }

    // Explicit memory provider — concrete class reference
    @RegisterAiService(chatMemoryProvider = ExplicitMemoryProvider.class)
    public interface ServiceWithExplicitMemory {
        @UserMessage("{msg}")
        String chat(String msg);
    }

    // No memory — void.class disables memory
    @RegisterAiService(chatMemoryProvider = void.class)
    public interface ServiceWithNoMemory {
        @UserMessage("{msg}")
        String chat(String msg);
    }

    // Auto-discover — interface type as default
    @RegisterAiService
    public interface ServiceWithAutoDiscoverMemory {
        @UserMessage("{msg}")
        String chat(String msg);
    }

    @ApplicationScoped
    public static class FixedChatModel implements ChatModel {
        @Override
        public ChatResponse chat(ChatRequest chatRequest) {
            return ChatResponse.builder()
                    .aiMessage(dev.langchain4j.data.message.AiMessage.from("fixed-response"))
                    .metadata(ChatResponseMetadata.builder().build())
                    .build();
        }
    }

    @ApplicationScoped
    public static class ExplicitMemoryProvider implements ChatMemoryProvider {
        @Override
        public dev.langchain4j.memory.ChatMemory get(Object memoryId) {
            return MessageWindowChatMemory.withMaxMessages(10);
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -f core/deployment/pom.xml test -pl . -Dtest=DirectBeanClassAttributeTest -Dsurefire.failIfNoSpecifiedTests=false`

Expected: FAIL — compilation error because `chatMemoryProvider` attribute doesn't exist on `@RegisterAiService` yet.

- [ ] **Step 3: Commit the failing test**

```
git add core/deployment/src/test/java/io/quarkiverse/langchain4j/test/DirectBeanClassAttributeTest.java
git commit -m "test(#2578): TDD anchor — direct bean-class attribute test (expected to fail)"
```

---

### Task 3: Rewrite @RegisterAiService annotation

**Files:**
- Modify: `core/runtime/src/main/java/io/quarkiverse/langchain4j/RegisterAiService.java`

This is the big-bang change. Delete all 17 sentinel marker inner classes, replace supplier-class attributes with `Class<?>` attributes.

- [ ] **Step 1: Rewrite the annotation**

The new `@RegisterAiService` annotation keeps:
- `modelName()` — unchanged
- `maxToolCallingRoundTrips()` — unchanged
- `maxToolCallsPerResponse()` — unchanged
- `tools()` — unchanged
- `allowContinuousForcedToolCalling()` — unchanged
- `shouldThrowExceptionOnEventError()` — unchanged

Changes:
- `chatLanguageModelSupplier` → **removed** (modelName + CDI handles model selection)
- `streamingChatLanguageModelSupplier` → **removed** (auto-detected from return types)
- `chatMemoryProviderSupplier` → `chatMemoryProvider` `Class<?>` default `ChatMemoryProvider.class`
- `chatMemoryFlushStrategySupplier` → `chatMemoryFlushStrategy` `Class<?>` default `void.class`
- `retrievalAugmentor` (supplier) → `retrievalAugmentor` `Class<?>` default `void.class`
- `moderationModelSupplier` → `moderationModel` `Class<?>` default `void.class`
- `toolProviderSupplier` → `toolProvider` `Class<?>` default `void.class`
- `toolSearchStrategySupplier` → `toolSearchStrategy` `Class<?>` default `void.class`
- `toolHallucinationStrategy` → `toolHallucinationStrategy` `Class<?>` default `void.class`
- `systemMessageProviderSupplier` → `systemMessageProvider` `Class<?>` default `void.class`
- `maxSequentialToolInvocations` → **removed** (was already deprecated)

Delete ALL 17 inner classes: `BeanChatLanguageModelSupplier`, `BeanStreamingChatLanguageModelSupplier`, `BeanChatMemoryProviderSupplier`, `NoChatMemoryProviderSupplier`, `NoRetriever`, `NoToolProviderSupplier`, `BeanIfExistsRetrievalAugmentorSupplier`, `NoRetrievalAugmentorSupplier`, `BeanIfExistsModerationModelSupplier`, `BeanIfExistsImageModelSupplier`, `BeanIfExistsToolProviderSupplier`, `BeanIfExistsToolSearchStrategySupplier`, `NoToolSearchStrategySupplier`, `BeanIfExistsToolHallucinationStrategy`, `NoSystemMessageProviderSupplier`, `BeanIfExistsSystemMessageProviderSupplier`, `DefaultChatMemoryFlushStrategySupplier`.

Remove imports that are only used by deleted sentinel classes: `Supplier`, `Function`, `ToolExecutionRequest`, `ToolExecutionResultMessage`, `ChatMemory`, `ChatMemoryProvider` (used by inner class), `MessageWindowChatMemory`, `ChatModel`, `StreamingChatModel`, `ImageModel`, `ModerationModel`, `RetrievalAugmentor`, `ContentRetriever`, `Content`, `Query`, `ToolProvider`, `ToolSearchStrategy`, `ChatMemoryStore`, `InMemoryChatMemoryStore`, `ChatMemoryFlushStrategy`, `SystemMessageProvider`.

Keep imports needed by annotation attributes: `ChatMemoryProvider` (used as default value for `chatMemoryProvider()`), `ToolChoice` (referenced in javadoc).

The resulting file should be ~100 lines (annotation attributes + javadoc), down from ~455 lines.

- [ ] **Step 2: Verify the file compiles in isolation**

This will NOT compile project-wide yet — everything that references old attributes or sentinel classes will break. That's expected. Verify the file itself has no syntax errors by checking IntelliJ diagnostics on this file only.

- [ ] **Step 3: Commit**

```
git add core/runtime/src/main/java/io/quarkiverse/langchain4j/RegisterAiService.java
git commit -m "feat(#2578): rewrite @RegisterAiService — Class<?> attributes, delete 17 sentinel markers"
```

---

### Task 4: Update LangChain4jDotNames

**Files:**
- Modify: `core/deployment/src/main/java/io/quarkiverse/langchain4j/deployment/LangChain4jDotNames.java`

- [ ] **Step 1: Remove sentinel DotName constants**

Remove these constants:
- `BEAN_CHAT_MODEL_SUPPLIER`
- `BEAN_STREAMING_CHAT_MODEL_SUPPLIER`
- `BEAN_CHAT_MEMORY_PROVIDER_SUPPLIER`
- `NO_CHAT_MEMORY_PROVIDER_SUPPLIER`
- `NO_RETRIEVER`
- `BEAN_IF_EXISTS_RETRIEVAL_AUGMENTOR_SUPPLIER`
- `NO_RETRIEVAL_AUGMENTOR_SUPPLIER`
- `BEAN_IF_EXISTS_MODERATION_MODEL_SUPPLIER`
- `BEAN_IF_EXISTS_IMAGE_MODEL_SUPPLIER`
- `BEAN_IF_EXISTS_TOOL_PROVIDER_SUPPLIER`
- `NO_TOOL_PROVIDER_SUPPLIER`
- `BEAN_IF_EXISTS_TOOL_SEARCH_STRATEGY_SUPPLIER`
- `NO_TOOL_SEARCH_STRATEGY_SUPPLIER`
- `NO_SYSTEM_MESSAGE_PROVIDER_SUPPLIER`
- `BEAN_IF_EXISTS_SYSTEM_MESSAGE_PROVIDER_SUPPLIER`
- `DEFAULT_CHAT_MEMORY_FLUSH_STRATEGY_SUPPLIER`

Add a new constant:
```java
static final DotName VOID_CLASS = DotName.createSimple("void");
```

Keep existing constants that are NOT sentinel references: `CHAT_MEMORY_PROVIDER`, `RETRIEVER`, `RETRIEVAL_AUGMENTOR`, `TOOL_PROVIDER`, `TOOL_SEARCH_STRATEGY`, `MODERATION_MODEL`, `IMAGE_MODEL`, etc. These are interface type constants used for auto-discover resolution.

- [ ] **Step 2: Commit**

```
git add core/deployment/src/main/java/io/quarkiverse/langchain4j/deployment/LangChain4jDotNames.java
git commit -m "feat(#2578): remove sentinel DotName constants from LangChain4jDotNames"
```

---

### Task 5: Update DeclarativeAiServiceBuildItem and DeclarativeAiServiceCreateInfo

**Files:**
- Modify: `core/deployment/src/main/java/io/quarkiverse/langchain4j/deployment/DeclarativeAiServiceBuildItem.java`
- Modify: `core/runtime/src/main/java/io/quarkiverse/langchain4j/runtime/aiservice/DeclarativeAiServiceCreateInfo.java`

- [ ] **Step 1: Update DeclarativeAiServiceBuildItem**

Replace supplier-specific fields with component fields that carry a `ComponentResolutionMode`:

Key changes:
- Remove `chatLanguageModelSupplierClassDotName` and `streamingChatLanguageModelSupplierClassDotName` — model selection is solely via `modelName` + CDI.
- Rename `chatMemoryProviderSupplierClassDotName` → `chatMemoryProviderClassDotName` (no longer a supplier)
- Add `ComponentResolutionMode chatMemoryProviderResolutionMode`
- Similarly for all other supplier fields: rename to remove "Supplier", add resolution mode field
- Remove `customRetrievalAugmentorSupplierClassIsABean` — the resolution mode replaces this flag
- Remove `chatMemoryFlushStrategySupplierClassDotName` → `chatMemoryFlushStrategyClassDotName` + mode

Update the constructor and all getters to match.

- [ ] **Step 2: Update DeclarativeAiServiceCreateInfo record**

Replace nullable string class names with component entries that pair a class name with a resolution mode:

```java
public record DeclarativeAiServiceCreateInfo(
        String serviceClassName,
        Map<String, AnnotationLiteral<?>> toolsClassInfo,
        ComponentEntry chatMemoryProvider,
        ComponentEntry chatMemoryFlushStrategy,
        ComponentEntry retrievalAugmentor,
        ComponentEntry moderationModel,
        ComponentEntry imageModel,
        ComponentEntry toolProvider,
        ComponentEntry toolSearchStrategy,
        ComponentEntry toolHallucinationStrategy,
        ComponentEntry systemMessageProvider,
        String chatMemorySeederClassName,
        String thinkingHandlerClassName,
        String chatModelName,
        String moderationModelName,
        String imageModelName,
        boolean needsStreamingChatModel,
        boolean needsModerationModel,
        boolean needsImageModel,
        String toolArgumentsErrorHandlerClassName,
        String toolExecutionErrorHandlerClassName,
        InputGuardrailsLiteral inputGuardrails,
        OutputGuardrailsLiteral outputGuardrails,
        Integer maxToolCallingRoundTrips,
        Integer maxToolCallsPerResponse,
        boolean allowContinuousForcedToolCalling,
        boolean shouldThrowExceptionOnEventError,
        String defaultMemoryIdProviderClassName) {

    public record ComponentEntry(String className, ComponentResolutionMode mode) {
        public static final ComponentEntry SKIP = new ComponentEntry(null, ComponentResolutionMode.SKIP);
    }
}
```

- [ ] **Step 3: Commit**

```
git add core/deployment/src/main/java/io/quarkiverse/langchain4j/deployment/DeclarativeAiServiceBuildItem.java
git add core/runtime/src/main/java/io/quarkiverse/langchain4j/runtime/aiservice/DeclarativeAiServiceCreateInfo.java
git commit -m "feat(#2578): update build item and create info for tri-state resolution"
```

---

### Task 6: Update AiServicesProcessor

**Files:**
- Modify: `core/deployment/src/main/java/io/quarkiverse/langchain4j/deployment/AiServicesProcessor.java`

This is the largest change — ~300 lines of processor logic.

- [ ] **Step 1: Replace getSupplierDotName with getComponentResolution**

Replace the existing `getSupplierDotName()` method (~line 901) with:

```java
private record ComponentResolution(DotName className, ComponentResolutionMode mode) {
    static final ComponentResolution SKIP = new ComponentResolution(null, ComponentResolutionMode.SKIP);
}

private ComponentResolution resolveComponent(
        AnnotationValue annotationValue,
        DotName interfaceType) {
    if (annotationValue == null) {
        return ComponentResolution.SKIP;
    }
    DotName dotName = annotationValue.asClass().name();
    if (DotName.createSimple("void").equals(dotName)) {
        return ComponentResolution.SKIP;
    }
    if (dotName.equals(interfaceType)) {
        return new ComponentResolution(dotName, ComponentResolutionMode.AUTO_DISCOVER);
    }
    return new ComponentResolution(dotName, ComponentResolutionMode.EXPLICIT);
}
```

- [ ] **Step 2: Update findDeclarativeServices**

Replace all calls to `getSupplierDotName()` with `resolveComponent()`. Update the construction of `DeclarativeAiServiceBuildItem` to pass resolution modes.

Key changes in `findDeclarativeServices()` (~line 402):
- Remove `chatLanguageModelSupplier` and `streamingChatLanguageModelSupplier` processing — model selection is solely via `chatModelName` + CDI
- Replace `chatMemoryProviderSupplier` processing with `chatMemoryProvider` using `resolveComponent(instance.value("chatMemoryProvider"), LangChain4jDotNames.CHAT_MEMORY_PROVIDER)`
- Similarly for all other supplier attributes → component attributes
- Remove `validateSupplierAndRegister()` and replace with `validateComponentAndRegister()` that just checks the class exists in the index and marks it unremovable

- [ ] **Step 3: Update handleDeclarativeServices**

Replace the sentinel class name string comparisons with resolution mode checks.

For each component, the pattern changes from:
```java
// OLD: string comparison against sentinel name
if (LangChain4jDotNames.BEAN_CHAT_MEMORY_PROVIDER_SUPPLIER.toString().equals(chatMemoryProviderSupplierClassName)) {
    configurator.addInjectionPoint(ClassType.create(LangChain4jDotNames.CHAT_MEMORY_PROVIDER));
    needsChatMemoryProviderBean = true;
}
```

To:
```java
// NEW: resolution mode
ComponentEntry chatMemoryProvider = info.chatMemoryProvider();
switch (chatMemoryProvider.mode()) {
    case SKIP -> {} // no injection point
    case AUTO_DISCOVER -> {
        configurator.addInjectionPoint(ParameterizedType.create(DotNames.CDI_INSTANCE,
                new Type[] { ClassType.create(LangChain4jDotNames.CHAT_MEMORY_PROVIDER) }, null));
        needsChatMemoryProviderBean = true;
    }
    case EXPLICIT -> {
        configurator.addInjectionPoint(ClassType.create(chatMemoryProvider.className()));
        unremovableProducer.produce(UnremovableBeanBuildItem.beanClassNames(chatMemoryProvider.className()));
    }
}
```

Apply this pattern to: chatMemoryProvider, chatMemoryFlushStrategy, retrievalAugmentor, moderationModel, imageModel, toolProvider, toolSearchStrategy, toolHallucinationStrategy, systemMessageProvider.

- [ ] **Step 4: Commit**

```
git add core/deployment/src/main/java/io/quarkiverse/langchain4j/deployment/AiServicesProcessor.java
git commit -m "feat(#2578): update AiServicesProcessor for tri-state component resolution"
```

---

### Task 7: Update AiServicesRecorder

**Files:**
- Modify: `core/runtime/src/main/java/io/quarkiverse/langchain4j/runtime/AiServicesRecorder.java`

- [ ] **Step 1: Replace sentinel string comparisons with resolution mode switches**

The recorder's `createDeclarativeAiService()` method currently has massive if-else chains like:

```java
// OLD
if (RegisterAiService.BeanChatMemoryProviderSupplier.class.getName()
        .equals(info.chatMemoryProviderSupplierClassName())) {
    quarkusAiServices.chatMemoryProvider(creationalContext.getInjectedReference(ChatMemoryProvider.class));
} else {
    Supplier<? extends ChatMemoryProvider> supplier = createSupplier(info.chatMemoryProviderSupplierClassName());
    quarkusAiServices.chatMemoryProvider(supplier.get());
}
```

Replace with:

```java
// NEW
ComponentEntry chatMemoryProvider = info.chatMemoryProvider();
switch (chatMemoryProvider.mode()) {
    case SKIP -> {} // no memory
    case AUTO_DISCOVER -> {
        Instance<ChatMemoryProvider> instance = creationalContext.getInjectedReference(CHAT_MEMORY_PROVIDER_TYPE_LITERAL);
        if (instance.isResolvable()) {
            quarkusAiServices.chatMemoryProvider(instance.get());
        }
    }
    case EXPLICIT -> {
        ChatMemoryProvider provider = (ChatMemoryProvider) creationalContext.getInjectedReference(
                loadClass(chatMemoryProvider.className()));
        quarkusAiServices.chatMemoryProvider(provider);
    }
}
```

Apply this pattern to all components. Remove the `createSupplier()` helper method — no longer needed.

- [ ] **Step 2: Remove model supplier handling**

Remove the block that checks `info.languageModelSupplierClassName()` and `info.streamingChatLanguageModelSupplierClassName()` — model selection is now always via `modelName` + CDI injection.

- [ ] **Step 3: Commit**

```
git add core/runtime/src/main/java/io/quarkiverse/langchain4j/runtime/AiServicesRecorder.java
git commit -m "feat(#2578): update AiServicesRecorder — resolution mode switches replace sentinel comparisons"
```

---

### Task 8: Migrate test files (batch)

**Files:** ~100 test files across `core/deployment/src/test/`, `chat-scopes/`, `mcp/`, `model-providers/`, `agentic/`

This is mechanical migration. Apply these patterns systematically:

- [ ] **Step 1: Understand migration patterns**

| Old pattern | New pattern |
|-------------|-------------|
| `chatLanguageModelSupplier = MySupplier.class` | Remove attribute. Convert `MySupplier` to `@ApplicationScoped` CDI bean implementing `ChatModel`. |
| `streamingChatLanguageModelSupplier = MySupplier.class` | Remove attribute. Convert to `@ApplicationScoped` CDI bean implementing `StreamingChatModel`. |
| `chatMemoryProviderSupplier = NoChatMemoryProviderSupplier.class` | `chatMemoryProvider = void.class` |
| `chatMemoryProviderSupplier = BeanChatMemoryProviderSupplier.class` | Remove attribute (default is auto-discover) |
| `chatMemoryProviderSupplier = MySupplier.class` | `chatMemoryProvider = MyProvider.class` (convert supplier to direct bean) |
| `toolProviderSupplier = NoToolProviderSupplier.class` | Remove attribute (default is `void.class`) |
| `toolProviderSupplier = BeanIfExistsToolProviderSupplier.class` | Remove attribute (default is `void.class`) |
| `toolProviderSupplier = MySupplier.class` | `toolProvider = MyProvider.class` |
| `retrievalAugmentor = BeanIfExistsRetrievalAugmentorSupplier.class` | Remove attribute (default is `void.class`) |
| `retrievalAugmentor = NoRetrievalAugmentorSupplier.class` | Remove attribute (default is `void.class`) |
| `retrievalAugmentor = MySupplier.class` | `retrievalAugmentor = MyAugmentor.class` |
| `toolSearchStrategySupplier = NoToolSearchStrategySupplier.class` | Remove attribute (default is `void.class`) |
| `chatMemoryFlushStrategySupplier = MySupplier.class` | `chatMemoryFlushStrategy = MyStrategy.class` |
| `systemMessageProviderSupplier = MyProvider.class` | `systemMessageProvider = MyProvider.class` |
| `maxSequentialToolInvocations = N` | Remove attribute (use `maxToolCallingRoundTrips` instead) |

For test supplier conversions, the typical pattern is:

```java
// OLD: Supplier<ChatModel> — NOT a CDI bean
public static class MyChatModelSupplier implements Supplier<ChatModel> {
    @Override
    public ChatModel get() {
        return new MyChatModel();
    }
}

// NEW: Direct CDI bean
@ApplicationScoped
public static class MyChatModel implements ChatModel {
    @Override
    public ChatResponse chat(ChatRequest request) {
        // same logic as before, but directly on the ChatModel
    }
}
```

If the supplier wraps a model constructor with parameters, use `@Produces`:
```java
@Produces @ApplicationScoped
ChatModel chatModel() {
    return new SomeChatModel(config);
}
```

- [ ] **Step 2: Migrate core/deployment tests**

Migrate all files in `core/deployment/src/test/java/` that reference old attributes. This is the largest batch (~60 files). Work module by module:

1. `test/` root package — `BlockingChatLanguageModelSupplierTest`, `StreamingChatLanguageModelSupplierTest`, `StreamingAndBlockingChatLanguageModelSupplierTest`, `NoNoArgCtorSupplierTest`, `ChatMemoryStoreHitCountTest`, `ChatMemoryFlushStrategyTest`, `ProgrammaticServiceConfigurationTest`, `NamedAiServicesAreResolvableByNameTest`, `AiServiceMethodParametersAnnotationsTest`, `ChatRequestParametersTest`, `DeclarativeSystemMessageProviderTest`
2. `test/guardrails/` — all guardrail tests
3. `test/tools/` — all tool tests
4. `test/toolresolution/` — all tool resolution tests
5. `test/toolsearch/` — tool search tests
6. `test/response/` — response augmenter tests
7. `deployment/` — `ShouldThrowExceptionOnEventErrorTest`

For each file:
- Replace supplier attribute references with new attribute names
- Convert `Supplier<T>` inner classes to `@ApplicationScoped` CDI beans implementing `T` directly
- Remove unused `Supplier` imports
- Add `@ApplicationScoped` import if needed

- [ ] **Step 3: Migrate chat-scopes tests**

Files in `chat-scopes/core/deployment/src/test/`:
- `ChatScopeMemoryTest`, `ChatMemoryTest`, `ToolErrorTest`, `ChatScopeThinkingForwarderTest`, `WipeTest`

Same migration patterns as Step 2.

- [ ] **Step 4: Migrate MCP tests**

Files in `mcp/deployment/src/test/`:
- `MultipleMcpClientsTest`

- [ ] **Step 5: Migrate model-provider tests**

Files in `model-providers/*/deployment/src/test/`:
- `openai/openai-vanilla`: `DeclarativeAiServicesTest`, `MultipleChatModelsDeclarativeServiceTest`
- `watsonx`: `AiChatServiceTest`

- [ ] **Step 6: Migrate agentic tests**

Files in `agentic/deployment/src/test/`:
- `AgentMeterRegistryTest`

- [ ] **Step 7: Migrate samples**

Files in `samples/`:
- `analyze-pdf-document/PdfDocumentAnalyzer.java`
- `image-to-plantuml/ImageToPlantUMLService.java`
- `weather-agent/CityExtractorAgent.java`
- `cli-translator/TranslatorAiService.java`

- [ ] **Step 8: Migrate integration tests**

Files in `integration-tests/`:
- `openai/DescribeImageResource.java`
- `multiple-providers/MultipleScoringModelsTest.java` (if applicable)

- [ ] **Step 9: Update Javadoc references**

Files referencing sentinel markers in javadoc/comments:
- `core/runtime/src/main/java/io/quarkiverse/langchain4j/AudioUrl.java`
- `core/runtime/src/main/java/io/quarkiverse/langchain4j/ImageUrl.java`
- `core/runtime/src/main/java/io/quarkiverse/langchain4j/VideoUrl.java`
- `core/runtime/src/main/java/io/quarkiverse/langchain4j/PdfUrl.java`

Update example code in javadoc from `chatMemoryProviderSupplier = RegisterAiService.NoChatMemoryProviderSupplier.class` to `chatMemoryProvider = void.class`.

- [ ] **Step 10: Commit all migrations**

```
git add -A
git commit -m "feat(#2578): migrate all @RegisterAiService usages to direct bean-class attributes"
```

---

### Task 9: Verify and run tests

- [ ] **Step 1: Run the TDD anchor test**

Run: `mvn -f core/deployment/pom.xml test -pl . -Dtest=DirectBeanClassAttributeTest`

Expected: PASS — all three resolution modes (explicit, void.class skip, auto-discover) work.

- [ ] **Step 2: Run full agentic module tests**

Run: `mvn -f agentic/pom.xml test -T 1C`

Expected: PASS — agent-implied AI services work with new resolution semantics.

- [ ] **Step 3: Run core module tests**

Run: `mvn -f core/pom.xml test -T 1C`

Expected: PASS — all migrated tests compile and pass.

- [ ] **Step 4: Run formatter**

Run: `mvn -f pom.xml process-sources -T 1C`

Fix any formatting issues. Commit if changes.

- [ ] **Step 5: Run full project build**

Run: `mvn -f pom.xml clean install -DskipTests -T 1C`

Expected: BUILD SUCCESS — everything compiles.

- [ ] **Step 6: Run full test suite**

Run: `mvn -f pom.xml test -T 1C`

Expected: PASS — complete green build.

- [ ] **Step 7: Final commit (if any fixes needed)**

```
git add -A
git commit -m "fix(#2578): test fixes from full build verification"
```

---

### Task 10: Squash and push

- [ ] **Step 1: Interactive squash into single commit**

Squash all commits from Tasks 1–9 into a single commit with message:

```
feat(#2578): replace supplier-class attributes with direct bean-class references

Replace all supplier-class attributes on @RegisterAiService with Class<?>
direct bean-class references. Delete all 17 sentinel marker inner classes.

Tri-state resolution model:
- void.class = disabled / not configured
- Interface type (e.g. ChatMemoryProvider.class) = auto-discover CDI bean
- Concrete class = inject specific CDI bean by type

Breaking changes:
- chatLanguageModelSupplier removed — use modelName + CDI
- streamingChatLanguageModelSupplier removed — auto-detected from return types
- All supplier attributes renamed (remove "Supplier" suffix)
- Auto-discovery removed for optional components (retrievalAugmentor,
  moderationModel, toolProvider, toolSearchStrategy) — explicit declaration only
- maxSequentialToolInvocations removed (was deprecated)
- Test Supplier<ChatModel> implementations converted to @ApplicationScoped CDI beans

Resolves: #2578
```

- [ ] **Step 2: Push to fork**

```
git push fork issue-2572-register-ai-service-simplification
```

- [ ] **Step 3: Create draft PR**

```
gh pr create --repo quarkiverse/quarkus-langchain4j --draft \
  --title "feat(#2578): replace supplier-class attributes with direct bean-class references" \
  --body "..."
```

---

## Notes for the implementing engineer

### The `createSupplier()` method in AiServicesRecorder

This method handles the "supplier is a CDI bean or plain class" logic. In the new design, ALL components are CDI beans — no no-arg constructor fallback. Remove `createSupplier()` entirely.

### The easy-rag extension

`EasyRetrievalAugmentor` directly implements `RetrievalAugmentor`. In PR 1, it continues to work via `retrievalAugmentor = EasyRetrievalAugmentor.class` (explicit resolution). In PR 2, this moves to `@RagPipeline(augmentor = EasyRetrievalAugmentor.class)`. The easy-rag extension's processor that wires this bean may need updating to NOT use the old sentinel marker approach.

### The `AnnotationsImpliesAiServiceBuildItem` path

Agent interfaces are processed as AI services via this build item. The resolution mode changes apply to these implied services too. The implied `@RegisterAiService` uses defaults (void.class for optional, interface type for required). Verify that the agentic module's tests pass after migration.

### Image model handling

The annotation doesn't have an explicit `imageModel` attribute — image model injection is auto-detected from method return types (like streaming chat model). The `BeanIfExistsImageModelSupplier` sentinel is deleted, but the processor still auto-detects image model needs from return types. Update the processor to use `AUTO_DISCOVER` resolution mode for image model when detected.
