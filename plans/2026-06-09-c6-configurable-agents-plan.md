# C6 — Configurable Agents Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add per-agent runtime config overrides for `maxIterations` and `a2aServerUrl`, config expression resolution in A2A annotations, and Vert.x HTTP transport for A2A agents.

**Architecture:** Wrap upstream's `WorkflowAgentsBuilder` SPI and `A2AService` to intercept and override annotation values from Quarkus config. Implement `A2AHttpClientProvider` with Vert.x WebClient. All wrappers are temporary — removed when upstream adds workflow-level `AgentConfigurator`. Two independent PRs: config overrides (PR1) and Vert.x transport (PR2).

**Tech Stack:** Quarkus `@ConfigMapping` with `@WithParentName`/`@WithDefaults`, SmallRye Config, Jandex, langchain4j-agentic SPI (`WorkflowAgentsBuilder`, `A2AService`), Vert.x WebClient, A2A Java SDK (`A2AHttpClientProvider`)

**Spec:** `specs/2026-06-09-c6-configurable-agents-design.md`

---

## File Map

### PR1 — Config Overrides

| Action | File | Responsibility |
|--------|------|---------------|
| Modify | `agentic/runtime/src/main/java/.../runtime/AgenticRuntimeConfig.java` | Restructure: two-@WithParentName + @WithDefaults, AgentConfig, DevUiConfig sibling |
| Create | `agentic/runtime/src/main/java/.../runtime/config/ConfigAwareWorkflowAgentsBuilder.java` | Wraps WorkflowAgentsBuilder, returns ConfigAwareLoopBuilder for loop agents |
| Create | `agentic/runtime/src/main/java/.../runtime/config/ConfigAwareLoopBuilder.java` | Fluent decorator over LoopAgentService — 16 delegate+return-this, 1 build with override |
| Create | `agentic/runtime/src/main/java/.../runtime/config/ConfigAwareA2AService.java` | Wraps A2AService, resolves URL from config/expression before delegating |
| Modify | `agentic/runtime/src/main/java/.../runtime/AgenticRecorder.java` | Add registration methods for wrappers |
| Modify | `agentic/deployment/src/main/java/.../deployment/AgenticProcessor.java` | Add agent name extraction @BuildStep + validations |
| Modify | `agentic/deployment/src/test/.../deployment/AgenticRuntimeConfigTest.java` | Update for new config structure |
| Create | `agentic/deployment/src/test/.../deployment/AgentConfigOverrideTest.java` | Config override integration test |
| Create | `agentic/deployment/src/test/.../deployment/AgentNameValidationTest.java` | Build-time validation tests |
| Create | `agentic/deployment/src/test/.../deployment/A2AConfigExpressionTest.java` | A2A URL expression resolution test |

### PR2 — Vert.x A2A Transport

| Action | File | Responsibility |
|--------|------|---------------|
| Create | `agentic/runtime/src/main/java/.../runtime/a2a/VertxA2AHttpClientProvider.java` | ServiceLoader provider, priority 100, lazy Vertx resolution |
| Create | `agentic/runtime/src/main/java/.../runtime/a2a/VertxA2AHttpClient.java` | A2AHttpClient impl using Vert.x WebClient |
| Create | `agentic/runtime/src/main/resources/META-INF/services/org.a2aproject.sdk.client.http.A2AHttpClientProvider` | ServiceLoader registration |
| Modify | `agentic/runtime/pom.xml` | Add optional a2a-java-sdk-http-client dependency |
| Modify | `agentic/deployment/src/main/java/.../deployment/AgenticProcessor.java` | ServiceProviderBuildItem for native image |
| Create | `agentic/deployment/src/test/.../deployment/VertxA2AHttpClientTest.java` | Vert.x HTTP client tests |

**Note:** `.../` abbreviates `io/quarkiverse/langchain4j/agentic/` throughout.

---

## Pre-implementation: File Upstream Issues

Before any code, file two issues on `langchain4j/langchain4j`:

- [ ] **File Issue 1: Workflow-level AgentConfigurator** — widen `AgentConfigurator` to fire for all builder types (`LoopAgentService`, `SupervisorAgentService`, `A2AClientBuilder`), not just `AgentBuilder`. Framework-agnostic — Spring/Micronaut have the same need.

- [ ] **File Issue 2: Public `A2AService.setA2AService()` setter** — matches existing `AgenticServices.setWorkflowAgentsBuilder()` pattern. Removes the need for reflection.

Record the issue numbers — they're referenced in temporary-workaround comments throughout PR1.

---

## PR1 — Config Overrides

### Task 1: Restructure AgenticRuntimeConfig

**Files:**
- Modify: `agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/AgenticRuntimeConfig.java`
- Modify: `agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticRuntimeConfigTest.java`

- [ ] **Step 1: Write the test for the new config structure**

Replace the existing test to verify the restructured config with `@WithParentName`/`@WithDefaults` and the new `AgentConfig` interface:

```java
package io.quarkiverse.langchain4j.agentic.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import jakarta.inject.Inject;

import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import io.quarkiverse.langchain4j.agentic.runtime.AgenticRuntimeConfig;
import io.quarkus.test.QuarkusUnitTest;

public class AgenticRuntimeConfigTest {

    @RegisterExtension
    static final QuarkusUnitTest unitTest = new QuarkusUnitTest()
            .setArchiveProducer(() -> ShrinkWrap.create(JavaArchive.class))
            .overrideRuntimeConfigKey("quarkus.langchain4j.agent.dev-ui.eager-init", "false")
            .overrideRuntimeConfigKey("quarkus.langchain4j.agent.max-iterations", "15")
            .overrideRuntimeConfigKey("quarkus.langchain4j.agent.\"story-loop\".max-iterations", "25")
            .overrideRuntimeConfigKey("quarkus.langchain4j.agent.\"remote-writer\".a2a-server-url",
                    "https://prod.example.com")
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.api-key", "test-key")
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.base-url", "http://localhost");

    @Inject
    AgenticRuntimeConfig config;

    @Test
    void eagerInitConfigPropertyIsReadable() {
        assertThat(config.devUi().eagerInit()).isFalse();
    }

    @Test
    void defaultMaxIterationsIsReadable() {
        assertThat(config.defaultConfig().maxIterations()).hasValue(15);
    }

    @Test
    void namedAgentMaxIterationsOverridesDefault() {
        var storyLoop = config.namedConfig().get("story-loop");
        assertThat(storyLoop).isNotNull();
        assertThat(storyLoop.maxIterations()).hasValue(25);
    }

    @Test
    void namedAgentA2AServerUrlIsReadable() {
        var remoteWriter = config.namedConfig().get("remote-writer");
        assertThat(remoteWriter).isNotNull();
        assertThat(remoteWriter.a2aServerUrl()).hasValue("https://prod.example.com");
    }

    @Test
    void unnamedAgentInheritsDefaults() {
        var storyLoop = config.namedConfig().get("story-loop");
        assertThat(storyLoop.a2aServerUrl()).isEmpty();
    }

    @Test
    void maxAgentsInvocationsDefaultsToEmpty() {
        assertThat(config.defaultConfig().maxAgentsInvocations()).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/agentic/deployment/pom.xml test -Dtest=AgenticRuntimeConfigTest -pl .`

Expected: FAIL — `defaultConfig()`, `namedConfig()`, `AgentConfig` don't exist yet.

- [ ] **Step 3: Implement the restructured config**

Replace `AgenticRuntimeConfig.java`:

```java
package io.quarkiverse.langchain4j.agentic.runtime;

import java.util.Map;
import java.util.Optional;

import io.quarkus.runtime.annotations.ConfigDocMapKey;
import io.quarkus.runtime.annotations.ConfigPhase;
import io.quarkus.runtime.annotations.ConfigRoot;
import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;
import io.smallrye.config.WithDefaults;
import io.smallrye.config.WithParentName;

@ConfigRoot(phase = ConfigPhase.RUN_TIME)
@ConfigMapping(prefix = "quarkus.langchain4j.agent")
public interface AgenticRuntimeConfig {

    @WithParentName
    AgentConfig defaultConfig();

    @WithParentName
    @WithDefaults
    @ConfigDocMapKey("agent-name")
    Map<String, AgentConfig> namedConfig();

    DevUiConfig devUi();

    interface AgentConfig {

        Optional<Integer> maxIterations();

        /**
         * Declared but not wired in C6 — requires upstream supervisor builder SPI
         * or workflow-level AgentConfigurator.
         */
        Optional<Integer> maxAgentsInvocations();

        Optional<String> a2aServerUrl();
    }

    interface DevUiConfig {

        @WithDefault("true")
        boolean eagerInit();
    }
}
```

- [ ] **Step 4: Update recorder to use new config shape**

In `AgenticRecorder.java`, the `conditionallyEagerInitRootAgents` method references `runtimeConfig.getValue().devUi().eagerInit()` — this is unchanged because `devUi()` stays in the same position. No recorder changes needed for this task.

- [ ] **Step 5: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/agentic/deployment/pom.xml test -Dtest=AgenticRuntimeConfigTest -pl .`

Expected: PASS — all 6 tests green.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/quarkus-langchain4j add agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/AgenticRuntimeConfig.java agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticRuntimeConfigTest.java
git -C /Users/mdproctor/claude/quarkus-langchain4j commit -m "feat(agentic): restructure AgenticRuntimeConfig — per-agent config with @WithDefaults"
```

---

### Task 2: Build-time Agent Name Extraction and Validation

**Files:**
- Modify: `agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticProcessor.java`
- Create: `agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/AgentNameValidationTest.java`

- [ ] **Step 1: Write test for duplicate agent name validation**

```java
package io.quarkiverse.langchain4j.agentic.deployment;

import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import dev.langchain4j.agentic.declarative.ChatModelSupplier;
import dev.langchain4j.agentic.declarative.LoopAgent;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import dev.langchain4j.model.chat.response.ChatResponse;
import dev.langchain4j.data.message.AiMessage;
import dev.langchain4j.service.V;
import io.quarkus.test.QuarkusUnitTest;

import static org.assertj.core.api.Assertions.assertThat;
import static org.junit.jupiter.api.Assertions.fail;

public class AgentNameValidationTest {

    @RegisterExtension
    static final QuarkusUnitTest duplicateNameTest = new QuarkusUnitTest()
            .setArchiveProducer(() -> ShrinkWrap.create(JavaArchive.class)
                    .addClasses(AgentA.class, AgentB.class, DummyModel.class, DummySubAgent.class))
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.api-key", "test")
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.base-url", "http://localhost")
            .assertException(t -> assertThat(t).hasMessageContaining("Duplicate agent config key"));

    @Test
    void shouldFailBuild() {
        fail("Should not reach here — build should have failed");
    }

    public interface DummySubAgent {
        @dev.langchain4j.agentic.Agent(description = "dummy")
        String run(@V("input") String input);

        @ChatModelSupplier
        static ChatModel model() {
            return new DummyModel();
        }
    }

    public interface AgentA {
        @LoopAgent(name = "my-loop", description = "loop A", outputKey = "out",
                subAgents = { DummySubAgent.class })
        String process(@V("input") String input);

        @ChatModelSupplier
        static ChatModel model() {
            return new DummyModel();
        }
    }

    public interface AgentB {
        @LoopAgent(name = "my-loop", description = "loop B", outputKey = "out",
                subAgents = { DummySubAgent.class })
        String process(@V("input") String input);

        @ChatModelSupplier
        static ChatModel model() {
            return new DummyModel();
        }
    }

    public static class DummyModel implements ChatModel {
        @Override
        public ChatResponse doChat(ChatRequest request) {
            return ChatResponse.builder().aiMessage(new AiMessage("ok")).build();
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails (for the right reason)**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/agentic/deployment/pom.xml test -Dtest=AgentNameValidationTest -pl .`

Expected: FAIL — the build does NOT fail because the validation doesn't exist yet. The test expects a build failure with "Duplicate agent config key" but the build succeeds.

- [ ] **Step 3: Implement agent name extraction @BuildStep**

Add to `AgenticProcessor.java`:

```java
import io.quarkiverse.langchain4j.agentic.deployment.AgentConfigKeyBuildItem;

// New build item class (inner or separate file)
// Add as a new file: agentic/deployment/src/main/java/.../deployment/AgentConfigKeyBuildItem.java

// In AgenticProcessor, add this @BuildStep:

@BuildStep
void extractAgentConfigKeys(
        List<DetectedAiAgentBuildItem> agents,
        CombinedIndexBuildItem indexBuildItem,
        BuildProducer<AgentConfigKeyBuildItem> configKeyProducer) {

    Map<String, List<String>> keyToClasses = new HashMap<>();
    Set<String> reservedNames = Set.of("dev-ui");

    for (DetectedAiAgentBuildItem agent : agents) {
        ClassInfo iface = agent.getIface();
        for (MethodInfo method : agent.getAgenticMethods()) {
            String configKey = resolveConfigKey(method);

            if (reservedNames.contains(configKey)) {
                throw new IllegalConfigurationException(
                        "Agent config key '" + configKey + "' on " + iface.name() + "#"
                                + method.name() + " is reserved for internal use.");
            }

            keyToClasses.computeIfAbsent(configKey, k -> new ArrayList<>())
                    .add(iface.name().toString() + "#" + method.name());
        }

        // For the config key lookup at runtime, we map className → configKey.
        // Each interface has exactly one root agentic method (upstream takes the first match).
        if (!agent.getAgenticMethods().isEmpty()) {
            String configKey = resolveConfigKey(agent.getAgenticMethods().get(0));
            configKeyProducer.produce(
                    new AgentConfigKeyBuildItem(iface.name().toString(), configKey));
        }
    }

    // Validate uniqueness
    for (var entry : keyToClasses.entrySet()) {
        if (entry.getValue().size() > 1) {
            throw new IllegalConfigurationException(
                    "Duplicate agent config key '" + entry.getKey() + "' found on: "
                            + String.join(", ", entry.getValue())
                            + ". Agent config keys must be unique across all interfaces.");
        }
    }
}

private static String resolveConfigKey(MethodInfo method) {
    // Check all agentic annotation types for a 'name' attribute
    for (AnnotationInstance ann : method.annotations()) {
        AnnotationValue nameValue = ann.value("name");
        if (nameValue != null && !nameValue.asString().isEmpty()) {
            return nameValue.asString();
        }
    }
    // Fall back to method name, kebab-cased
    return kebabCase(method.name());
}

private static String kebabCase(String camelCase) {
    StringBuilder sb = new StringBuilder();
    for (int i = 0; i < camelCase.length(); i++) {
        char c = camelCase.charAt(i);
        if (Character.isUpperCase(c)) {
            if (i > 0) {
                sb.append('-');
            }
            sb.append(Character.toLowerCase(c));
        } else {
            sb.append(c);
        }
    }
    return sb.toString();
}
```

Create `AgentConfigKeyBuildItem.java`:

```java
package io.quarkiverse.langchain4j.agentic.deployment;

import io.quarkus.builder.item.MultiBuildItem;

public final class AgentConfigKeyBuildItem extends MultiBuildItem {

    private final String agentClassName;
    private final String configKey;

    public AgentConfigKeyBuildItem(String agentClassName, String configKey) {
        this.agentClassName = agentClassName;
        this.configKey = configKey;
    }

    public String getAgentClassName() {
        return agentClassName;
    }

    public String getConfigKey() {
        return configKey;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/agentic/deployment/pom.xml test -Dtest=AgentNameValidationTest -pl .`

Expected: PASS — the build fails with "Duplicate agent config key 'my-loop'" and the test assertion catches it.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/quarkus-langchain4j add agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticProcessor.java agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgentConfigKeyBuildItem.java agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/AgentNameValidationTest.java
git -C /Users/mdproctor/claude/quarkus-langchain4j commit -m "feat(agentic): build-time agent name extraction and duplicate key validation"
```

---

### Task 3: ConfigAwareLoopBuilder — Fluent Decorator

**Files:**
- Create: `agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/config/ConfigAwareLoopBuilder.java`

- [ ] **Step 1: Create the decorator**

This is a pure delegate — every fluent method delegates and returns `this`. The `build()` method applies config override before delegating. No separate test for this class — it's tested via the integration test in Task 4.

```java
package io.quarkiverse.langchain4j.agentic.runtime.config;

import java.util.Collection;
import java.util.Optional;
import java.util.function.BiPredicate;
import java.util.function.Consumer;
import java.util.function.Function;
import java.util.function.Predicate;

import dev.langchain4j.agentic.AgenticScope;
import dev.langchain4j.agentic.ErrorContext;
import dev.langchain4j.agentic.ErrorRecoveryResult;
import dev.langchain4j.agentic.TypedKey;
import dev.langchain4j.agentic.observability.AgentListener;
import dev.langchain4j.agentic.workflow.LoopAgentService;

/**
 * TEMPORARY WORKAROUND — will be removed when upstream provides
 * workflow-level AgentConfigurator (see langchain4j/langchain4j#NNNN).
 *
 * Fluent decorator over {@link LoopAgentService}. Every fluent method
 * delegates to the real builder AND returns {@code this} to prevent
 * chain escape. On {@code build()}, applies the config-resolved
 * maxIterations before delegating.
 */
final class ConfigAwareLoopBuilder<T> implements LoopAgentService<T> {

    private final LoopAgentService<T> delegate;
    private final Optional<Integer> configMaxIterations;

    ConfigAwareLoopBuilder(LoopAgentService<T> delegate, Optional<Integer> configMaxIterations) {
        this.delegate = delegate;
        this.configMaxIterations = configMaxIterations;
    }

    @Override
    public T build() {
        configMaxIterations.ifPresent(delegate::maxIterations);
        return delegate.build();
    }

    // --- LoopAgentService methods ---

    @Override
    public LoopAgentService<T> maxIterations(int maxIterations) {
        delegate.maxIterations(maxIterations);
        return this;
    }

    @Override
    public LoopAgentService<T> exitCondition(Predicate<AgenticScope> exitCondition) {
        delegate.exitCondition(exitCondition);
        return this;
    }

    @Override
    public LoopAgentService<T> exitCondition(BiPredicate<AgenticScope, Integer> exitCondition) {
        delegate.exitCondition(exitCondition);
        return this;
    }

    @Override
    public LoopAgentService<T> exitCondition(String exitConditionDescription,
            Predicate<AgenticScope> exitCondition) {
        delegate.exitCondition(exitConditionDescription, exitCondition);
        return this;
    }

    @Override
    public LoopAgentService<T> exitCondition(String exitConditionDescription,
            BiPredicate<AgenticScope, Integer> exitCondition) {
        delegate.exitCondition(exitConditionDescription, exitCondition);
        return this;
    }

    @Override
    public LoopAgentService<T> testExitAtLoopEnd(boolean checkExitConditionAtLoopEnd) {
        delegate.testExitAtLoopEnd(checkExitConditionAtLoopEnd);
        return this;
    }

    // --- AgenticService methods ---

    @Override
    public LoopAgentService<T> subAgents(Object... agents) {
        delegate.subAgents(agents);
        return this;
    }

    @Override
    public LoopAgentService<T> subAgents(Collection<?> agents) {
        delegate.subAgents(agents);
        return this;
    }

    @Override
    public LoopAgentService<T> beforeCall(Consumer<AgenticScope> beforeCall) {
        delegate.beforeCall(beforeCall);
        return this;
    }

    @Override
    public LoopAgentService<T> name(String name) {
        delegate.name(name);
        return this;
    }

    @Override
    public LoopAgentService<T> description(String description) {
        delegate.description(description);
        return this;
    }

    @Override
    public LoopAgentService<T> outputKey(String outputKey) {
        delegate.outputKey(outputKey);
        return this;
    }

    @Override
    public LoopAgentService<T> outputKey(Class<? extends TypedKey<?>> outputKey) {
        delegate.outputKey(outputKey);
        return this;
    }

    @Override
    public LoopAgentService<T> output(Function<AgenticScope, Object> output) {
        delegate.output(output);
        return this;
    }

    @Override
    public LoopAgentService<T> errorHandler(Function<ErrorContext, ErrorRecoveryResult> errorHandler) {
        delegate.errorHandler(errorHandler);
        return this;
    }

    @Override
    public LoopAgentService<T> listener(AgentListener listener) {
        delegate.listener(listener);
        return this;
    }
}
```

- [ ] **Step 2: Verify it compiles**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/agentic/runtime/pom.xml compile -pl .`

Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/quarkus-langchain4j add agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/config/ConfigAwareLoopBuilder.java
git -C /Users/mdproctor/claude/quarkus-langchain4j commit -m "feat(agentic): ConfigAwareLoopBuilder — fluent decorator for loop config override"
```

---

### Task 4: ConfigAwareWorkflowAgentsBuilder + Recorder Registration

**Files:**
- Create: `agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/config/ConfigAwareWorkflowAgentsBuilder.java`
- Modify: `agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/AgenticRecorder.java`
- Modify: `agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticProcessor.java`
- Create: `agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/AgentConfigOverrideTest.java`

- [ ] **Step 1: Write the integration test**

```java
package io.quarkiverse.langchain4j.agentic.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.concurrent.atomic.AtomicInteger;

import jakarta.inject.Inject;

import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import dev.langchain4j.agentic.Agent;
import dev.langchain4j.agentic.AgenticScope;
import dev.langchain4j.agentic.declarative.ChatModelSupplier;
import dev.langchain4j.agentic.declarative.ExitCondition;
import dev.langchain4j.agentic.declarative.LoopAgent;
import dev.langchain4j.data.message.AiMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import dev.langchain4j.model.chat.response.ChatResponse;
import dev.langchain4j.service.V;
import io.quarkus.test.QuarkusUnitTest;

public class AgentConfigOverrideTest {

    static final AtomicInteger iterationCount = new AtomicInteger(0);

    @RegisterExtension
    static final QuarkusUnitTest unitTest = new QuarkusUnitTest()
            .setArchiveProducer(() -> ShrinkWrap.create(JavaArchive.class)
                    .addClasses(CountingLoopAgent.class, IncrementAgent.class,
                            CountingModel.class, AgentConfigOverrideTest.class))
            .overrideRuntimeConfigKey("quarkus.langchain4j.agent.\"counting-loop\".max-iterations", "3")
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.api-key", "test-key")
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.base-url", "http://localhost");

    @Inject
    CountingLoopAgent agent;

    @Test
    void configOverridesAnnotationMaxIterations() {
        iterationCount.set(0);
        try {
            agent.loop("start");
        } catch (Exception ignored) {
            // Agent may throw due to no real LLM — we only care about iteration count
        }
        // Annotation says maxIterations=10, config says 3
        assertThat(iterationCount.get()).isLessThanOrEqualTo(3);
    }

    public interface IncrementAgent {
        @Agent(description = "Increments counter", outputKey = "result")
        String increment(@V("input") String input);

        @ChatModelSupplier
        static ChatModel model() {
            return new CountingModel();
        }
    }

    public interface CountingLoopAgent {
        @LoopAgent(name = "counting-loop", description = "Counts iterations",
                outputKey = "result", maxIterations = 10,
                subAgents = { IncrementAgent.class })
        String loop(@V("input") String input);

        @ExitCondition
        static boolean shouldExit(AgenticScope scope) {
            return false; // never exit — rely on maxIterations
        }

        @ChatModelSupplier
        static ChatModel model() {
            return new CountingModel();
        }
    }

    public static class CountingModel implements ChatModel {
        @Override
        public ChatResponse doChat(ChatRequest request) {
            iterationCount.incrementAndGet();
            return ChatResponse.builder().aiMessage(new AiMessage("iteration")).build();
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/agentic/deployment/pom.xml test -Dtest=AgentConfigOverrideTest -pl .`

Expected: FAIL — config override is not wired yet, so the annotation value (10) is used instead of config value (3).

- [ ] **Step 3: Create ConfigAwareWorkflowAgentsBuilder**

```java
package io.quarkiverse.langchain4j.agentic.runtime.config;

import java.util.Collection;
import java.util.Map;
import java.util.Optional;

import dev.langchain4j.agentic.UntypedAgent;
import dev.langchain4j.agentic.workflow.ConditionalAgentService;
import dev.langchain4j.agentic.workflow.LoopAgentService;
import dev.langchain4j.agentic.workflow.ParallelAgentService;
import dev.langchain4j.agentic.workflow.ParallelMapperService;
import dev.langchain4j.agentic.workflow.SequentialAgentService;
import dev.langchain4j.agentic.workflow.WorkflowAgentsBuilder;
import io.quarkiverse.langchain4j.agentic.runtime.AgenticRuntimeConfig;

/**
 * TEMPORARY WORKAROUND — will be removed when upstream provides
 * workflow-level AgentConfigurator (see langchain4j/langchain4j#NNNN).
 */
public final class ConfigAwareWorkflowAgentsBuilder implements WorkflowAgentsBuilder {

    private final WorkflowAgentsBuilder delegate;
    private final AgenticRuntimeConfig config;
    private final Map<String, String> classNameToConfigKey;

    public ConfigAwareWorkflowAgentsBuilder(WorkflowAgentsBuilder delegate,
            AgenticRuntimeConfig config,
            Map<String, String> classNameToConfigKey) {
        this.delegate = delegate;
        this.config = config;
        this.classNameToConfigKey = classNameToConfigKey;
    }

    @Override
    public <T> LoopAgentService<T> loopBuilder(Class<T> agentServiceClass) {
        LoopAgentService<T> real = delegate.loopBuilder(agentServiceClass);
        Optional<Integer> configMaxIterations = resolveMaxIterations(agentServiceClass);
        return new ConfigAwareLoopBuilder<>(real, configMaxIterations);
    }

    @Override
    public LoopAgentService<UntypedAgent> loopBuilder() {
        return delegate.loopBuilder();
    }

    @Override
    public <T> SequentialAgentService<T> sequenceBuilder(Class<T> agentServiceClass) {
        return delegate.sequenceBuilder(agentServiceClass);
    }

    @Override
    public SequentialAgentService<UntypedAgent> sequenceBuilder() {
        return delegate.sequenceBuilder();
    }

    @Override
    public <T> ParallelAgentService<T> parallelBuilder(Class<T> agentServiceClass) {
        return delegate.parallelBuilder(agentServiceClass);
    }

    @Override
    public ParallelAgentService<UntypedAgent> parallelBuilder() {
        return delegate.parallelBuilder();
    }

    @Override
    public <T> ConditionalAgentService<T> conditionalBuilder(Class<T> agentServiceClass) {
        return delegate.conditionalBuilder(agentServiceClass);
    }

    @Override
    public ConditionalAgentService<UntypedAgent> conditionalBuilder() {
        return delegate.conditionalBuilder();
    }

    @Override
    public <T> ParallelMapperService<T> parallelMapperBuilder(Class<T> agentServiceClass) {
        return delegate.parallelMapperBuilder(agentServiceClass);
    }

    @Override
    public ParallelMapperService<UntypedAgent> parallelMapperBuilder() {
        return delegate.parallelMapperBuilder();
    }

    private <T> Optional<Integer> resolveMaxIterations(Class<T> agentServiceClass) {
        String configKey = classNameToConfigKey.get(agentServiceClass.getName());
        if (configKey == null) {
            return Optional.empty();
        }
        var agentConfig = config.namedConfig().get(configKey);
        if (agentConfig != null && agentConfig.maxIterations().isPresent()) {
            return agentConfig.maxIterations();
        }
        return Optional.empty();
    }
}
```

- [ ] **Step 4: Add recorder registration method**

Add to `AgenticRecorder.java`:

```java
@RuntimeInit
public void registerConfigAwareWorkflowAgentsBuilder(Map<String, String> classNameToConfigKey) {
    WorkflowAgentsBuilder current = getCurrentWorkflowAgentsBuilder();
    AgenticServices.setWorkflowAgentsBuilder(
            new ConfigAwareWorkflowAgentsBuilder(current, runtimeConfig.getValue(), classNameToConfigKey));
}

private static WorkflowAgentsBuilder getCurrentWorkflowAgentsBuilder() {
    // The current builder is accessible via the private enum —
    // we call setWorkflowAgentsBuilder which replaces it.
    // To get the current one, we use AgenticServices internal state.
    // Since there's no getter, we create a default instance.
    return dev.langchain4j.agentic.workflow.impl.WorkflowAgentsBuilderImpl.INSTANCE;
}
```

Add imports:
```java
import dev.langchain4j.agentic.AgenticServices;
import dev.langchain4j.agentic.workflow.WorkflowAgentsBuilder;
import io.quarkiverse.langchain4j.agentic.runtime.config.ConfigAwareWorkflowAgentsBuilder;
import java.util.Map;
```

- [ ] **Step 5: Add deployment @BuildStep to wire registration**

Add to `AgenticProcessor.java`:

```java
@BuildStep
@Record(ExecutionTime.RUNTIME_INIT)
void registerConfigAwareWorkflowAgentsBuilder(
        AgenticRecorder recorder,
        List<AgentConfigKeyBuildItem> configKeys) {
    Map<String, String> classNameToConfigKey = new HashMap<>();
    for (AgentConfigKeyBuildItem item : configKeys) {
        classNameToConfigKey.put(item.getAgentClassName(), item.getConfigKey());
    }
    recorder.registerConfigAwareWorkflowAgentsBuilder(classNameToConfigKey);
}
```

Add import: `import io.quarkus.deployment.annotations.Record;` and `import io.quarkus.deployment.annotations.ExecutionTime;`

- [ ] **Step 6: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/agentic/deployment/pom.xml test -Dtest=AgentConfigOverrideTest -pl .`

Expected: PASS — config value 3 overrides annotation value 10.

- [ ] **Step 7: Run full test suite to check for regressions**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/agentic/deployment/pom.xml test -pl .`

Expected: All existing tests pass.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/quarkus-langchain4j add agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/config/ConfigAwareWorkflowAgentsBuilder.java agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/AgenticRecorder.java agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticProcessor.java agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/AgentConfigOverrideTest.java
git -C /Users/mdproctor/claude/quarkus-langchain4j commit -m "feat(agentic): config-aware workflow builder — maxIterations override from config"
```

---

### Task 5: ConfigAwareA2AService — URL Override + Expression Resolution

**Files:**
- Create: `agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/config/ConfigAwareA2AService.java`
- Modify: `agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/AgenticRecorder.java`
- Modify: `agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticProcessor.java`
- Create: `agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/A2AConfigExpressionTest.java`

- [ ] **Step 1: Write the config expression test**

```java
package io.quarkiverse.langchain4j.agentic.deployment;

import static org.assertj.core.api.Assertions.assertThat;
import static org.junit.jupiter.api.Assertions.assertThrows;

import jakarta.enterprise.inject.CreationException;
import jakarta.inject.Inject;

import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import dev.langchain4j.agentic.declarative.A2AClientAgent;
import dev.langchain4j.agentic.declarative.ChatModelSupplier;
import dev.langchain4j.agentic.declarative.SequenceAgent;
import dev.langchain4j.data.message.AiMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import dev.langchain4j.model.chat.response.ChatResponse;
import dev.langchain4j.service.V;
import io.quarkus.test.QuarkusUnitTest;

public class A2AConfigExpressionTest {

    @RegisterExtension
    static final QuarkusUnitTest unitTest = new QuarkusUnitTest()
            .setArchiveProducer(() -> ShrinkWrap.create(JavaArchive.class)
                    .addClasses(ExpressionA2AAgent.class, OrchestratorAgent.class,
                            DummyChatModel.class))
            .overrideRuntimeConfigKey("remote.agent.url", "http://resolved-from-config:9999")
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.api-key", "test")
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.base-url", "http://localhost");

    @Inject
    OrchestratorAgent orchestrator;

    @Test
    void configExpressionInA2AUrlIsResolved() {
        // The A2A agent uses ${remote.agent.url} which should resolve to
        // http://resolved-from-config:9999. The agent creation will fail
        // trying to fetch the agent card from that URL, but the error message
        // should contain the RESOLVED URL, not the expression.
        CreationException ex = assertThrows(CreationException.class,
                () -> orchestrator.run("test"));
        assertThat(ex.getMessage()).contains("resolved-from-config:9999");
        assertThat(ex.getMessage()).doesNotContain("${remote.agent.url}");
    }

    public interface ExpressionA2AAgent {
        @A2AClientAgent(a2aServerUrl = "${remote.agent.url}",
                description = "Remote agent", outputKey = "result")
        String call(@V("input") String input);

        @ChatModelSupplier
        static ChatModel model() {
            return new DummyChatModel();
        }
    }

    public interface OrchestratorAgent {
        @SequenceAgent(outputKey = "result",
                subAgents = { ExpressionA2AAgent.class })
        String run(@V("input") String input);
    }

    public static class DummyChatModel implements ChatModel {
        @Override
        public ChatResponse doChat(ChatRequest request) {
            return ChatResponse.builder().aiMessage(new AiMessage("ok")).build();
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/agentic/deployment/pom.xml test -Dtest=A2AConfigExpressionTest -pl .`

Expected: FAIL — the `${remote.agent.url}` is passed as-is (unresolved) because no expression resolution exists yet.

- [ ] **Step 3: Create ConfigAwareA2AService**

```java
package io.quarkiverse.langchain4j.agentic.runtime.config;

import java.lang.reflect.Method;
import java.util.Map;
import java.util.Optional;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

import org.eclipse.microprofile.config.ConfigProvider;
import org.jboss.logging.Logger;

import dev.langchain4j.agentic.internal.A2AClientBuilder;
import dev.langchain4j.agentic.internal.A2AService;
import dev.langchain4j.agentic.internal.AgentExecutor;
import dev.langchain4j.agentic.internal.InternalAgent;
import io.quarkiverse.langchain4j.agentic.runtime.AgenticRuntimeConfig;

/**
 * TEMPORARY WORKAROUND — will be removed when upstream provides
 * workflow-level AgentConfigurator (see langchain4j/langchain4j#NNNN).
 * Reflection on A2AService.Provider.a2aService will be removed when
 * upstream adds A2AService.setA2AService() (see langchain4j/langchain4j#MMMM).
 */
public final class ConfigAwareA2AService implements A2AService {

    private static final Logger log = Logger.getLogger(ConfigAwareA2AService.class);
    private static final Pattern CONFIG_EXPRESSION = Pattern.compile("^\\$\\{(.+)}$");

    private final A2AService delegate;
    private final AgenticRuntimeConfig config;
    private final Map<String, String> classNameToConfigKey;

    public ConfigAwareA2AService(A2AService delegate, AgenticRuntimeConfig config,
            Map<String, String> classNameToConfigKey) {
        this.delegate = delegate;
        this.config = config;
        this.classNameToConfigKey = classNameToConfigKey;
    }

    @Override
    public <T> A2AClientBuilder<T> a2aBuilder(String a2aServerUrl, Class<T> agentServiceClass) {
        String resolvedUrl = resolveUrl(a2aServerUrl, agentServiceClass);
        return delegate.a2aBuilder(resolvedUrl, agentServiceClass);
    }

    @Override
    public Optional<AgentExecutor> methodToAgentExecutor(InternalAgent agent, Method method) {
        return delegate.methodToAgentExecutor(agent, method);
    }

    private <T> String resolveUrl(String annotationUrl, Class<T> agentServiceClass) {
        // Priority 1: per-agent named config
        String configKey = classNameToConfigKey.get(agentServiceClass.getName());
        if (configKey != null) {
            var agentConfig = config.namedConfig().get(configKey);
            if (agentConfig != null && agentConfig.a2aServerUrl().isPresent()) {
                log.debugf("A2A URL for %s resolved from config key '%s': %s",
                        agentServiceClass.getSimpleName(), configKey,
                        agentConfig.a2aServerUrl().get());
                return agentConfig.a2aServerUrl().get();
            }
        }

        // Priority 2: config expression in annotation value
        Matcher matcher = CONFIG_EXPRESSION.matcher(annotationUrl);
        if (matcher.matches()) {
            String configPropertyName = matcher.group(1);
            try {
                String resolved = ConfigProvider.getConfig()
                        .getValue(configPropertyName, String.class);
                log.debugf("A2A URL for %s resolved from expression '${%s}': %s",
                        agentServiceClass.getSimpleName(), configPropertyName, resolved);
                return resolved;
            } catch (java.util.NoSuchElementException e) {
                throw new IllegalStateException(
                        "Config expression '${" + configPropertyName
                                + "}' in @A2AClientAgent.a2aServerUrl on "
                                + agentServiceClass.getName()
                                + " could not be resolved. Define '"
                                + configPropertyName + "' in application.properties.",
                        e);
            }
        }

        // Priority 3: raw annotation value
        return annotationUrl;
    }
}
```

- [ ] **Step 4: Add recorder registration for A2AService**

Add to `AgenticRecorder.java`:

```java
@RuntimeInit
public void registerConfigAwareA2AService(Map<String, String> classNameToConfigKey) {
    try {
        A2AService current = A2AService.get();
        ConfigAwareA2AService wrapper = new ConfigAwareA2AService(
                current, runtimeConfig.getValue(), classNameToConfigKey);
        java.lang.reflect.Field field = A2AService.Provider.class.getDeclaredField("a2aService");
        field.setAccessible(true);
        field.set(null, wrapper);
    } catch (NoSuchFieldException | IllegalAccessException e) {
        log.warn("Failed to register ConfigAwareA2AService — A2A URL config overrides will not work", e);
    }
}
```

Add imports:
```java
import dev.langchain4j.agentic.internal.A2AService;
import io.quarkiverse.langchain4j.agentic.runtime.config.ConfigAwareA2AService;
```

- [ ] **Step 5: Add deployment @BuildStep for A2AService registration**

Add to `AgenticProcessor.java`, extending the existing `registerConfigAwareWorkflowAgentsBuilder` step or as a new step:

```java
@BuildStep
@Record(ExecutionTime.RUNTIME_INIT)
void registerConfigAwareA2AService(
        AgenticRecorder recorder,
        List<AgentConfigKeyBuildItem> configKeys) {
    Map<String, String> classNameToConfigKey = new HashMap<>();
    for (AgentConfigKeyBuildItem item : configKeys) {
        classNameToConfigKey.put(item.getAgentClassName(), item.getConfigKey());
    }
    recorder.registerConfigAwareA2AService(classNameToConfigKey);
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/agentic/deployment/pom.xml test -Dtest=A2AConfigExpressionTest -pl .`

Expected: PASS — `${remote.agent.url}` resolves to `http://resolved-from-config:9999`.

- [ ] **Step 7: Run full test suite**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/agentic/deployment/pom.xml test -pl .`

Expected: All tests pass.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/quarkus-langchain4j add agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/config/ConfigAwareA2AService.java agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/AgenticRecorder.java agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticProcessor.java agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/A2AConfigExpressionTest.java
git -C /Users/mdproctor/claude/quarkus-langchain4j commit -m "feat(agentic): config-aware A2A service — URL override and expression resolution"
```

---

## PR2 — Vert.x A2A Transport (Separate PR)

### Task 6: VertxA2AHttpClientProvider and VertxA2AHttpClient

**Files:**
- Modify: `agentic/runtime/pom.xml` — add optional `a2a-java-sdk-http-client` dependency
- Create: `agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/a2a/VertxA2AHttpClientProvider.java`
- Create: `agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/a2a/VertxA2AHttpClient.java`
- Create: `agentic/runtime/src/main/resources/META-INF/services/org.a2aproject.sdk.client.http.A2AHttpClientProvider`
- Modify: `agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticProcessor.java` — add ServiceProviderBuildItem
- Create: `agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/VertxA2AHttpClientTest.java`

- [ ] **Step 1: Add dependency to runtime pom.xml**

Add to `agentic/runtime/pom.xml` `<dependencies>`:

```xml
<dependency>
    <groupId>org.a2aproject.sdk</groupId>
    <artifactId>a2a-java-sdk-http-client</artifactId>
    <optional>true</optional>
</dependency>
```

Check if the version is managed in the parent BOM. If not, add version `1.0.0.CR1` (matching the version already on the classpath transitively via langchain4j-agentic-a2a).

- [ ] **Step 2: Create VertxA2AHttpClientProvider**

```java
package io.quarkiverse.langchain4j.agentic.runtime.a2a;

import org.a2aproject.sdk.client.http.A2AHttpClient;
import org.a2aproject.sdk.client.http.A2AHttpClientProvider;

import io.quarkus.arc.Arc;
import io.vertx.core.Vertx;

public class VertxA2AHttpClientProvider implements A2AHttpClientProvider {

    @Override
    public A2AHttpClient create() {
        Vertx vertx = Arc.container().instance(Vertx.class).get();
        return new VertxA2AHttpClient(vertx);
    }

    @Override
    public int priority() {
        return 100;
    }
}
```

- [ ] **Step 3: Create VertxA2AHttpClient**

```java
package io.quarkiverse.langchain4j.agentic.runtime.a2a;

import java.util.Map;
import java.util.concurrent.CompletableFuture;
import java.util.function.Consumer;

import org.a2aproject.sdk.client.http.A2AHttpClient;
import org.a2aproject.sdk.client.http.A2AHttpResponse;
import org.a2aproject.sdk.client.http.SSEEvent;

import io.vertx.core.Vertx;
import io.vertx.core.buffer.Buffer;
import io.vertx.core.http.HttpMethod;
import io.vertx.ext.web.client.HttpRequest;
import io.vertx.ext.web.client.HttpResponse;
import io.vertx.ext.web.client.WebClient;

public class VertxA2AHttpClient implements A2AHttpClient {

    private final WebClient webClient;

    VertxA2AHttpClient(Vertx vertx) {
        this.webClient = WebClient.create(vertx);
    }

    @Override
    public GetBuilder createGet() {
        return new VertxGetBuilder();
    }

    @Override
    public PostBuilder createPost() {
        return new VertxPostBuilder();
    }

    @Override
    public DeleteBuilder createDelete() {
        return new VertxDeleteBuilder();
    }

    private A2AHttpResponse toResponse(HttpResponse<Buffer> resp) {
        String body = resp.body() != null ? resp.body().toString() : "";
        return new A2AHttpResponse(resp.statusCode(), body,
                resp.headers().entries().stream()
                        .collect(java.util.stream.Collectors.toMap(
                                Map.Entry::getKey, Map.Entry::getValue,
                                (a, b) -> b)));
    }

    private class VertxGetBuilder implements GetBuilder {
        private String url;
        private final Map<String, String> headers = new java.util.LinkedHashMap<>();

        @Override
        public GetBuilder url(String url) {
            this.url = url;
            return this;
        }

        @Override
        public GetBuilder addHeader(String name, String value) {
            headers.put(name, value);
            return this;
        }

        @Override
        public A2AHttpResponse get() {
            HttpRequest<Buffer> request = webClient.getAbs(url);
            headers.forEach(request::putHeader);
            HttpResponse<Buffer> resp = request.sendAndAwait();
            return toResponse(resp);
        }
    }

    private class VertxPostBuilder implements PostBuilder {
        private String url;
        private String body;
        private final Map<String, String> headers = new java.util.LinkedHashMap<>();

        @Override
        public PostBuilder url(String url) {
            this.url = url;
            return this;
        }

        @Override
        public PostBuilder body(String body) {
            this.body = body;
            return this;
        }

        @Override
        public PostBuilder addHeader(String name, String value) {
            headers.put(name, value);
            return this;
        }

        @Override
        public A2AHttpResponse post() {
            HttpRequest<Buffer> request = webClient.postAbs(url);
            headers.forEach(request::putHeader);
            HttpResponse<Buffer> resp = request
                    .sendBufferAndAwait(Buffer.buffer(body != null ? body : ""));
            return toResponse(resp);
        }

        @Override
        public CompletableFuture<Void> postAsyncSSE(
                Consumer<SSEEvent> eventConsumer,
                Consumer<Throwable> errorConsumer,
                Runnable completeConsumer) {
            CompletableFuture<Void> future = new CompletableFuture<>();
            HttpRequest<Buffer> request = webClient.postAbs(url);
            headers.forEach(request::putHeader);
            request.putHeader(ACCEPT, EVENT_STREAM);

            // Use Vert.x raw HTTP request for SSE streaming
            webClient.post(url)
                    .putHeader(CONTENT_TYPE, APPLICATION_JSON)
                    .putHeader(ACCEPT, EVENT_STREAM);

            // SSE implementation requires raw Vert.x HTTP client for streaming.
            // This is a placeholder — the actual implementation should use
            // vertx.createHttpClient() for proper SSE event parsing.
            // TODO: Implement proper SSE streaming with Vert.x HttpClient
            // For now, fall back to synchronous post and simulate events.
            try {
                A2AHttpResponse response = post();
                eventConsumer.accept(new SSEEvent(null, response.body()));
                completeConsumer.run();
                future.complete(null);
            } catch (Exception e) {
                errorConsumer.accept(e);
                future.completeExceptionally(e);
            }
            return future;
        }
    }

    private class VertxDeleteBuilder implements DeleteBuilder {
        private String url;
        private final Map<String, String> headers = new java.util.LinkedHashMap<>();

        @Override
        public DeleteBuilder url(String url) {
            this.url = url;
            return this;
        }

        @Override
        public DeleteBuilder addHeader(String name, String value) {
            headers.put(name, value);
            return this;
        }

        @Override
        public A2AHttpResponse delete() {
            HttpRequest<Buffer> request = webClient.deleteAbs(url);
            headers.forEach(request::putHeader);
            HttpResponse<Buffer> resp = request.sendAndAwait();
            return toResponse(resp);
        }
    }
}
```

**Implementation note — SSE streaming:** The `postAsyncSSE()` above is a synchronous fallback. The production implementation must use Vert.x's `HttpClient` (not `WebClient`) for proper SSE streaming: open a raw HTTP connection, parse `text/event-stream` lines (`event:`, `data:`, empty-line delimiters), and dispatch `SSEEvent` instances to the consumer. Consult `JdkA2AHttpClient.postAsyncSSE()` in `a2a-java-sdk-http-client` for the exact SSE parsing contract. This requires a separate implementation step within this task — the synchronous fallback is for compilation and basic test scaffolding only.

- [ ] **Step 4: Create ServiceLoader registration file**

Create `agentic/runtime/src/main/resources/META-INF/services/org.a2aproject.sdk.client.http.A2AHttpClientProvider`:

```
io.quarkiverse.langchain4j.agentic.runtime.a2a.VertxA2AHttpClientProvider
```

- [ ] **Step 5: Add ServiceProviderBuildItem for native image**

Add to `AgenticProcessor.java`:

```java
@BuildStep
void registerA2AHttpClientProviderForNativeImage(
        BuildProducer<io.quarkus.deployment.builditem.nativeimage.ServiceProviderBuildItem> serviceProvider) {
    serviceProvider.produce(new io.quarkus.deployment.builditem.nativeimage.ServiceProviderBuildItem(
            "org.a2aproject.sdk.client.http.A2AHttpClientProvider",
            "io.quarkiverse.langchain4j.agentic.runtime.a2a.VertxA2AHttpClientProvider"));
}
```

- [ ] **Step 6: Verify compilation**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/agentic/runtime/pom.xml compile -pl .`

Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/quarkus-langchain4j add agentic/runtime/pom.xml agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/a2a/VertxA2AHttpClientProvider.java agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/a2a/VertxA2AHttpClient.java agentic/runtime/src/main/resources/META-INF/services/org.a2aproject.sdk.client.http.A2AHttpClientProvider agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticProcessor.java
git -C /Users/mdproctor/claude/quarkus-langchain4j commit -m "feat(agentic): Vert.x A2A HTTP client provider — replaces JDK HttpClient"
```
