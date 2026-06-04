# C1 Quick Wins & Safety Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement all Chapter 1 remaining items (D-1, C-7, F-7, S-3, F-3, F-4, F-5, F-1, L-1, C-6) for the quarkus-langchain4j agentic module.

**Architecture:** Build-time validation enhancements in `AgenticProcessor` + `ValidationUtil`, one-liner runtime fixes in `AgenticRecorder`, a new `AgenticRuntimeConfig` config skeleton, and test additions covering FaultTolerance integration. All changes are in the `agentic/` module — no cross-module API changes.

**Tech Stack:** Java 21+, Quarkus build extension model (Jandex, `@BuildStep`, `@Recorder`), JUnit 5 + AssertJ, `@QuarkusUnitTest`

---

## File Map

| Action | File | Purpose |
|--------|------|---------|
| Modify | `agentic/deployment/.../devui/AgenticDevUIProcessor.java` | D-1: add `onlyIf = IsDevelopment.class` |
| Modify | `agentic/runtime/.../AgenticRecorder.java` | C-7: fix TCCL in `loadClassSafe` + `eagerlyInitRootAgents` |
| Modify | `agentic/deployment/.../ValidationUtil.java` | Add `transitiveInterfaces`, `hasAnnotation` |
| Modify | `agentic/deployment/.../AgenticProcessor.java` | S-3, F-7, F-3, F-4, F-5 fixes |
| Modify | `agentic/deployment/.../FaultToleranceTest.java` | F-1: add @Retry and @CircuitBreaker test cases |
| Create | `agentic/runtime/.../AgenticRuntimeConfig.java` | L-1: config skeleton |
| Modify | `agentic/deployment/.../devui/AgenticDevUIProcessor.java` | L-1: wire `eagerInit` |
| Create | `agentic/deployment/.../validation/FallbackMethodOnAgentTest.java` | F-3 test |
| Create | `agentic/deployment/.../FaultToleranceRetryWarningTest.java` | F-4 test |
| Create | `agentic/deployment/.../TransactionalRetryWarningTest.java` | F-5 test |
| Create | `agentic/deployment/.../F7InterceptorInheritanceTest.java` | F-7 test |
| Create | `agentic/deployment/.../S3CdiBeanInheritedSupplierTest.java` | S-3 test |

**Maven commands used throughout:**
```bash
# Compile check
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl agentic/runtime,agentic/deployment -DskipTests -Dno-format -q

# Run a single test
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agentic/deployment -Dtest=<TestClassName> -Dno-format

# Run all agentic tests
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agentic/deployment,agentic/runtime -Dno-format
```

---

## Task 1: D-1 — `jsonRpcProvider` dev-mode gate

**Files:**
- Modify: `agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/devui/AgenticDevUIProcessor.java:91`

- [ ] **Step 1: Verify the existing smoke test passes before any changes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agentic/deployment -Dtest=AgentBeanSmokeTest -Dno-format
```
Expected: BUILD SUCCESS — this is the regression baseline.

- [ ] **Step 2: Apply the fix**

In `AgenticDevUIProcessor.java`, change line 91 from:
```java
@BuildStep
void jsonRpcProvider(BuildProducer<JsonRPCProvidersBuildItem> producers) {
```
to:
```java
@BuildStep(onlyIf = IsDevelopment.class)
void jsonRpcProvider(BuildProducer<JsonRPCProvidersBuildItem> producers) {
```

- [ ] **Step 3: Verify smoke test still passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agentic/deployment -Dtest=AgentBeanSmokeTest -Dno-format
```
Expected: BUILD SUCCESS (test profile is not dev mode — fix is exercised).

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/quarkus-langchain4j add agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/devui/AgenticDevUIProcessor.java
git -C /Users/mdproctor/claude/quarkus-langchain4j commit -m "fix(agentic): gate jsonRpcProvider on dev mode

AgenticJsonRpcService was registered in all profiles including production.
Three other build steps in the same class already had onlyIf=IsDevelopment.class.

Fixes quarkiverse/quarkus-langchain4j#2527 (D-1)"
```

---

## Task 2: C-7 — `loadClassSafe` and `eagerlyInitRootAgents` classloader

**Files:**
- Modify: `agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/AgenticRecorder.java`

- [ ] **Step 1: Apply the fix in `loadClassSafe` (line ~188)**

Find the `loadClassSafe` private method at the bottom of `AgenticRecorder.java`. Change from:
```java
private static Class<?> loadClassSafe(AiAgentCreateInfo info) {
    try {
        return Class.forName(info.agentClassName(), true, Thread.currentThread().getContextClassLoader());
    } catch (ClassNotFoundException e) {
        log.error("Unable to load agent class '" + info.agentClassName() + "'", e);
        throw new RuntimeException(e);
    }
}
```
to:
```java
private static Class<?> loadClassSafe(AiAgentCreateInfo info) {
    try {
        // Do not use Thread.currentThread().getContextClassLoader() — TCCL is not
        // guaranteed to be the deployment classloader on Vert.x I/O threads or virtual
        // threads spawned by Executors.newVirtualThreadPerTaskExecutor(). The recorder's
        // own classloader is always the correct deployment classloader.
        return Class.forName(info.agentClassName(), true, AgenticRecorder.class.getClassLoader());
    } catch (ClassNotFoundException e) {
        log.error("Unable to load agent class '" + info.agentClassName() + "'", e);
        throw new RuntimeException(e);
    }
}
```

- [ ] **Step 2: Apply the fix in `eagerlyInitRootAgents` (line ~72)**

Change:
```java
Class<?> clazz = Class.forName(className, true, Thread.currentThread().getContextClassLoader());
```
to:
```java
// TCCL not reliable in dev mode startup on virtual threads; use recorder classloader.
Class<?> clazz = Class.forName(className, true, AgenticRecorder.class.getClassLoader());
```

- [ ] **Step 3: Compile check**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl agentic/runtime -DskipTests -Dno-format -q
```
Expected: no output (clean compile).

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/quarkus-langchain4j add agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/AgenticRecorder.java
git -C /Users/mdproctor/claude/quarkus-langchain4j commit -m "fix(agentic): use recorder classloader instead of TCCL in loadClassSafe

TCCL is not the deployment classloader on Vert.x I/O threads or virtual
threads. AgenticRecorder.class.getClassLoader() is always correct.

Fixes quarkiverse/quarkus-langchain4j#2527 (C-7)"
```

---

## Task 3: ValidationUtil helpers

**Files:**
- Modify: `agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/ValidationUtil.java`

These helpers are used by Tasks 4 (S-3) and 5 (F-7). Write them first.

- [ ] **Step 1: Add imports and helpers to `ValidationUtil.java`**

Add the following imports at the top:
```java
import java.util.ArrayDeque;
import java.util.Collection;
import java.util.Deque;
import java.util.LinkedHashSet;
import java.util.Set;
import org.jboss.jandex.AnnotationInstance;
import org.jboss.jandex.ClassInfo;
import org.jboss.jandex.DotName;
import org.jboss.jandex.IndexView;
```

Add these two static methods to the bottom of `ValidationUtil`:

```java
/**
 * Returns the given interface and all its transitive parent interfaces, in BFS order.
 * The result includes {@code start} itself as the first element.
 */
static Set<ClassInfo> transitiveInterfaces(ClassInfo start, IndexView index) {
    Set<ClassInfo> result = new LinkedHashSet<>();
    Deque<ClassInfo> queue = new ArrayDeque<>();
    queue.add(start);
    while (!queue.isEmpty()) {
        ClassInfo current = queue.poll();
        if (!result.add(current)) {
            continue;
        }
        for (DotName parentName : current.interfaceNames()) {
            ClassInfo parent = index.getClassByName(parentName);
            if (parent != null) {
                queue.add(parent);
            }
        }
    }
    return result;
}

/**
 * Returns true if any annotation in {@code annotations} has a name in {@code names}.
 */
static boolean hasAnnotation(Collection<AnnotationInstance> annotations, Set<DotName> names) {
    for (AnnotationInstance ann : annotations) {
        if (names.contains(ann.name())) {
            return true;
        }
    }
    return false;
}
```

- [ ] **Step 2: Compile check**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl agentic/deployment -DskipTests -Dno-format -q
```
Expected: clean compile.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/quarkus-langchain4j add agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/ValidationUtil.java
git -C /Users/mdproctor/claude/quarkus-langchain4j commit -m "feat(agentic): add transitiveInterfaces and hasAnnotation to ValidationUtil

Shared utilities for interface hierarchy traversal, used by S-3 (CDI
bean unremovable guard) and F-7 (interceptor binding inheritance).

Refs quarkiverse/quarkus-langchain4j#2527"
```

---

## Task 4: S-3 — `markCdiBeanParametersAsUnremovable` covers all CDI-capable suppliers

**Files:**
- Modify: `agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticProcessor.java`
- Create: `agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/S3CdiBeanInheritedSupplierTest.java`

- [ ] **Step 1: Write the failing test**

Create `S3CdiBeanInheritedSupplierTest.java`. This tests that a `@CdiBean` parameter on a `@ChatModelSupplier` method declared on a **parent** interface is marked unremovable. Without the fix, the supplier is on the parent interface and not found by the old `getChatModelSupplier()`-only scan.

```java
package io.quarkiverse.langchain4j.agentic.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import jakarta.inject.Inject;
import jakarta.inject.Singleton;

import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import dev.langchain4j.agentic.Agent;
import dev.langchain4j.agentic.declarative.ChatModelSupplier;
import dev.langchain4j.agentic.declarative.SequenceAgent;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.service.UserMessage;
import io.quarkiverse.langchain4j.agentic.runtime.CdiBean;
import io.quarkiverse.langchain4j.openai.testing.internal.OpenAiBaseTest;
import io.quarkiverse.langchain4j.testing.internal.WiremockAware;
import io.quarkus.test.QuarkusUnitTest;

/**
 * Verifies S-3: @CdiBean parameters on supplier methods declared on parent interfaces
 * are marked as unremovable, preventing UnsatisfiedResolutionException at runtime.
 */
public class S3CdiBeanInheritedSupplierTest extends OpenAiBaseTest {

    @RegisterExtension
    static final QuarkusUnitTest unitTest = new QuarkusUnitTest()
            .setArchiveProducer(() -> ShrinkWrap.create(JavaArchive.class)
                    .addClasses(ModelSelector.class, BaseAgent.class, ConcreteAgent.class,
                                Agents.FixedResponseChatModel.class))
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.api-key", "test-key")
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.base-url",
                    WiremockAware.wiremockUrlForConfig("/v1"));

    /** A CDI bean that selects a model — declared as @CdiBean parameter on a parent interface supplier. */
    @Singleton
    public static class ModelSelector {
        public ChatModel select() {
            return new Agents.FixedResponseChatModel("selected");
        }
    }

    /** Base interface declares the @ChatModelSupplier with a @CdiBean parameter. */
    public interface BaseAgent {
        @ChatModelSupplier
        static ChatModel model(@CdiBean ModelSelector selector) {
            return selector.select();
        }
    }

    /** Concrete agent extends base — the @ChatModelSupplier is inherited. */
    public interface ConcreteAgent extends BaseAgent {
        @UserMessage("{{input}}")
        @Agent(description = "Agent using inherited CDI supplier")
        String answer(String input);
    }

    @Inject
    ConcreteAgent agent;

    @Test
    void agentBootsWithoutUnsatisfiedResolution() {
        // If ModelSelector were removable, Arc would fail to resolve it and this inject would fail at boot.
        assertThat(agent).isNotNull();
    }
}
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agentic/deployment -Dtest=S3CdiBeanInheritedSupplierTest -Dno-format 2>&1 | tail -20
```
Expected: FAIL — `UnsatisfiedResolutionException` or test bootstrap failure because `ModelSelector` is removed by Arc.

- [ ] **Step 3: Add the `ALL_CDI_CAPABLE_SUPPLIER_ANNOTATIONS` constant to `AgenticProcessor`**

Add this constant near the top of the `AgenticProcessor` class (after the existing logger declaration):
```java
private static final List<DotName> ALL_CDI_CAPABLE_SUPPLIER_ANNOTATIONS = List.of(
        AgenticLangChain4jDotNames.CHAT_MODEL_SUPPLIER,
        AgenticLangChain4jDotNames.CHAT_MEMORY_SUPPLIER,
        AgenticLangChain4jDotNames.CHAT_MEMORY_PROVIDER_SUPPLIER,
        AgenticLangChain4jDotNames.CONTENT_RETRIEVER_SUPPLIER,
        AgenticLangChain4jDotNames.RETRIEVAL_AUGMENTER_SUPPLIER,
        AgenticLangChain4jDotNames.TOOL_SUPPLIER,
        AgenticLangChain4jDotNames.TOOL_PROVIDER_SUPPLIER,
        AgenticLangChain4jDotNames.AGENT_LISTENER_SUPPLIER
        // PARALLEL_EXECUTOR excluded: executor config annotation, validated to have no parameters
);
```

- [ ] **Step 4: Rewrite `markCdiBeanParametersAsUnremovable`**

Replace the existing method (lines ~438-452) with:
```java
@BuildStep
void markCdiBeanParametersAsUnremovable(
        List<DetectedAiAgentBuildItem> detectedAiAgentBuildItems,
        CombinedIndexBuildItem indexBuildItem,
        BuildProducer<UnremovableBeanBuildItem> unremovableProducer) {
    IndexView index = indexBuildItem.getIndex();
    for (DetectedAiAgentBuildItem item : detectedAiAgentBuildItems) {
        for (ClassInfo classInfo : ValidationUtil.transitiveInterfaces(item.getIface(), index)) {
            for (MethodInfo method : classInfo.methods()) {
                boolean isSupplierMethod = ALL_CDI_CAPABLE_SUPPLIER_ANNOTATIONS.stream()
                        .anyMatch(method::hasAnnotation);
                if (!isSupplierMethod) {
                    continue;
                }
                for (MethodParameterInfo param : method.parameters()) {
                    if (param.hasAnnotation(AgenticLangChain4jDotNames.CDI_BEAN)) {
                        unremovableProducer.produce(UnremovableBeanBuildItem.beanTypes(param.type().name()));
                    }
                }
            }
        }
    }
}
```

Ensure `MethodInfo` and `MethodParameterInfo` are already imported (they are).
Add `CombinedIndexBuildItem` import if not present: `import io.quarkus.deployment.builditem.CombinedIndexBuildItem;` (already imported for `validateAgenticParameterTypes`).

- [ ] **Step 5: Run the test to verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agentic/deployment -Dtest=S3CdiBeanInheritedSupplierTest -Dno-format 2>&1 | tail -10
```
Expected: BUILD SUCCESS.

- [ ] **Step 6: Run existing CDI supplier test to verify no regression**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agentic/deployment -Dtest=CdiChatSupplierParameterResolverTest -Dno-format 2>&1 | tail -5
```
Expected: BUILD SUCCESS.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/quarkus-langchain4j add \
  agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticProcessor.java \
  agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/S3CdiBeanInheritedSupplierTest.java
git -C /Users/mdproctor/claude/quarkus-langchain4j commit -m "fix(agentic): mark @CdiBean params unremovable for all supplier types

Previously only @ChatModelSupplier was scanned. Now scans all CDI-capable
supplier annotations across the full interface hierarchy. Prerequisite for
S-1 (full CDI auto-wire for non-chat suppliers).

Fixes quarkiverse/quarkus-langchain4j#2527 (S-3)"
```

---

## Task 5: F-7 — Inherited interceptor binding detection

**Files:**
- Modify: `agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticProcessor.java`
- Create: `agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/F7InterceptorInheritanceTest.java`

- [ ] **Step 1: Write the failing test**

Create `F7InterceptorInheritanceTest.java`. The agent interface extends a base with `@Timeout` at the class level. If `hasAnyInterceptorBindings` misses it, `injectInterceptionProxy()` is not called, and `@Timeout` does not fire — the agent call takes longer than the timeout without throwing.

```java
package io.quarkiverse.langchain4j.agentic.deployment;

import static org.assertj.core.api.Assertions.assertThat;
import static org.junit.jupiter.api.Assertions.assertThrows;

import java.time.temporal.ChronoUnit;

import jakarta.inject.Inject;

import org.eclipse.microprofile.faulttolerance.Timeout;
import org.eclipse.microprofile.faulttolerance.exceptions.TimeoutException;
import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import dev.langchain4j.agentic.Agent;
import dev.langchain4j.agentic.agent.AgentInvocationException;
import dev.langchain4j.agentic.declarative.ChatModelSupplier;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.service.UserMessage;
import io.quarkiverse.langchain4j.testing.internal.WiremockAware;
import io.quarkus.test.QuarkusUnitTest;

/**
 * Verifies F-7: @Timeout on a parent interface class declaration is detected by
 * hasAnyInterceptorBindings and the interception proxy is wired correctly.
 */
public class F7InterceptorInheritanceTest {

    @RegisterExtension
    static final QuarkusUnitTest unitTest = new QuarkusUnitTest()
            .setArchiveProducer(() -> ShrinkWrap.create(JavaArchive.class)
                    .addClasses(TimedBase.class, SlowAgent.class, SlowChatModel.class))
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.api-key", "test-key")
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.base-url",
                    WiremockAware.wiremockUrlForConfig("/v1"));

    /** Timeout declared at class level on the BASE interface. */
    @Timeout(value = 50, unit = ChronoUnit.MILLIS)
    public interface TimedBase {
    }

    /** Agent does not redeclare @Timeout — it must be inherited from TimedBase. */
    public interface SlowAgent extends TimedBase {
        @UserMessage("{{q}}")
        @Agent(description = "Intentionally slow agent")
        String answer(String q);

        @ChatModelSupplier
        static ChatModel model() {
            return new SlowChatModel();
        }
    }

    /** Chat model that sleeps longer than the 50ms timeout. */
    public static class SlowChatModel implements ChatModel {
        @Override
        public dev.langchain4j.model.output.Response<dev.langchain4j.data.message.AiMessage> generate(
                java.util.List<dev.langchain4j.data.message.ChatMessage> messages) {
            try {
                Thread.sleep(200);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return dev.langchain4j.model.output.Response.from(
                    dev.langchain4j.data.message.AiMessage.from("too slow"));
        }
    }

    @Inject
    SlowAgent slowAgent;

    @Test
    void timeoutIsAppliedFromParentInterface() {
        AgentInvocationException ex = assertThrows(AgentInvocationException.class,
                () -> slowAgent.answer("hello"));
        assertThat(ex).hasRootCauseInstanceOf(TimeoutException.class);
    }
}
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agentic/deployment -Dtest=F7InterceptorInheritanceTest -Dno-format 2>&1 | tail -15
```
Expected: FAIL — the agent completes without throwing (timeout not applied because interceptor binding was missed).

- [ ] **Step 3: Fix `hasAnyInterceptorBindings` in `AgenticProcessor.java`**

First, add `CombinedIndexBuildItem indexBuildItem` parameter to `cdiSupport`:
```java
@BuildStep
@Record(ExecutionTime.RUNTIME_INIT)
void cdiSupport(List<DetectedAiAgentBuildItem> detectedAiAgentBuildItems, AgenticRecorder recorder,
        InterceptorResolverBuildItem interceptorResolverBuildItem,
        CombinedIndexBuildItem indexBuildItem,
        BuildProducer<SyntheticBeanBuildItem> syntheticBeanProducer,
        BuildProducer<RequestChatModelBeanBuildItem> requestChatModelBeanProducer) {
```

Then update the call to `hasAnyInterceptorBindings` in `cdiSupport` (around line 473):
```java
boolean hasInterceptorBindings = hasAnyInterceptorBindings(detectedAiAgentBuildItem, interceptorBindings, indexBuildItem.getIndex());
```

Then replace the `hasAnyInterceptorBindings` method entirely:
```java
private static boolean hasAnyInterceptorBindings(DetectedAiAgentBuildItem agent,
        Set<DotName> interceptorBindings, IndexView index) {
    Set<ClassInfo> hierarchy = ValidationUtil.transitiveInterfaces(agent.getIface(), index);

    // Class-level check: walk the full interface hierarchy
    for (ClassInfo classInfo : hierarchy) {
        if (ValidationUtil.hasAnnotation(classInfo.classAnnotations(), interceptorBindings)) {
            return true;
        }
    }

    // Method-level check: check each agentic method, then look up the same method on parent interfaces
    for (MethodInfo method : agent.getAgenticMethods()) {
        if (ValidationUtil.hasAnnotation(method.annotations(), interceptorBindings)) {
            return true;
        }
        for (ClassInfo classInfo : hierarchy) {
            if (classInfo.name().equals(agent.getIface().name())) {
                continue; // already covered by method.annotations() above
            }
            // Look up the matching method on this parent interface by name + parameter types
            org.jboss.jandex.Type[] paramTypes = method.parameterTypes()
                    .toArray(new org.jboss.jandex.Type[0]);
            MethodInfo parentMethod = classInfo.method(method.name(), paramTypes);
            if (parentMethod != null
                    && ValidationUtil.hasAnnotation(parentMethod.annotations(), interceptorBindings)) {
                return true;
            }
        }
    }
    return false;
}
```

Add import at top of file: `import org.jboss.jandex.IndexView;` (likely already present via `CombinedIndexBuildItem` usage).

- [ ] **Step 4: Run the test to verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agentic/deployment -Dtest=F7InterceptorInheritanceTest -Dno-format 2>&1 | tail -10
```
Expected: BUILD SUCCESS.

- [ ] **Step 5: Run existing interceptor test for regression**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agentic/deployment -Dtest=AgentInterceptorTest -Dno-format 2>&1 | tail -5
```
Expected: BUILD SUCCESS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/quarkus-langchain4j add \
  agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticProcessor.java \
  agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/F7InterceptorInheritanceTest.java
git -C /Users/mdproctor/claude/quarkus-langchain4j commit -m "fix(agentic): detect interceptor bindings on parent interface hierarchy

hasAnyInterceptorBindings previously used declaredAnnotations() which
misses class-level and method-level bindings on parent interfaces.
Now traverses the full interface hierarchy via transitiveInterfaces().

Fixes quarkiverse/quarkus-langchain4j#2527 (F-7)"
```

---

## Task 6: F-3 — `@Fallback(fallbackMethod=...)` build error

**Files:**
- Modify: `agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticProcessor.java`
- Create: `agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/validation/FallbackMethodOnAgentTest.java`

- [ ] **Step 1: Write the failing test**

Create `FallbackMethodOnAgentTest.java`:
```java
package io.quarkiverse.langchain4j.agentic.deployment.validation;

import static org.junit.jupiter.api.Assertions.fail;

import org.assertj.core.api.Assertions;
import org.eclipse.microprofile.faulttolerance.Fallback;
import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import dev.langchain4j.agentic.Agent;
import dev.langchain4j.agentic.declarative.ChatModelSupplier;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.service.IllegalConfigurationException;
import dev.langchain4j.service.UserMessage;
import io.quarkiverse.langchain4j.agentic.deployment.Agents;
import io.quarkus.test.QuarkusUnitTest;

/**
 * Verifies F-3: @Fallback(fallbackMethod=...) on an agent interface is a build error.
 * Agent interfaces are dynamic proxies — fallback method name resolution always fails.
 */
public class FallbackMethodOnAgentTest {

    @RegisterExtension
    static final QuarkusUnitTest unitTest = new QuarkusUnitTest()
            .setArchiveProducer(() -> ShrinkWrap.create(JavaArchive.class)
                    .addClasses(BrokenFallbackAgent.class, Agents.FixedResponseChatModel.class))
            .assertException(t -> Assertions.assertThat(t)
                    .isInstanceOf(IllegalConfigurationException.class)
                    .hasMessageContaining("fallbackMethod")
                    .hasMessageContaining("FallbackHandler"));

    public interface BrokenFallbackAgent {
        @UserMessage("{{q}}")
        @Agent(description = "Agent with broken fallback")
        @Fallback(fallbackMethod = "doFallback")
        String answer(String q);

        @ChatModelSupplier
        static ChatModel model() {
            return new Agents.FixedResponseChatModel("ok");
        }
    }

    @Test
    public void test() {
        fail("should never be called — build must fail with IllegalConfigurationException");
    }
}
```

- [ ] **Step 2: Run the test to verify it fails (in the wrong way)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agentic/deployment -Dtest=FallbackMethodOnAgentTest -Dno-format 2>&1 | tail -15
```
Expected: FAIL — `assertException` condition not met (no `IllegalConfigurationException` thrown; agent boots and test body `fail()` is reached).

- [ ] **Step 3: Add the `FALLBACK` DotName constant and `validateFallback` method to `AgenticProcessor.java`**

Add the constant near the top of the class:
```java
private static final DotName FALLBACK =
        DotName.createSimple("org.eclipse.microprofile.faulttolerance.Fallback");
```

Add the `validateFallback` method (alongside the other `validate*` private methods):
```java
private static void validateFallback(ClassInfo iface) {
    for (AnnotationInstance fallback : iface.annotations(FALLBACK)) {
        AnnotationValue fallbackMethod = fallback.value("fallbackMethod");
        if (fallbackMethod != null && !fallbackMethod.asString().isEmpty()) {
            AnnotationTarget target = fallback.target();
            String location = target.kind() == AnnotationTarget.Kind.CLASS
                    ? "class '" + iface.name() + "'"
                    : "method '" + target.asMethod().name() + "' of class '" + iface.name() + "'";
            throw new IllegalConfigurationException(
                    "Agent " + location + " uses @Fallback(fallbackMethod=\""
                    + fallbackMethod.asString() + "\"). "
                    + "Agent interfaces are dynamic proxies — fallback method name resolution "
                    + "always fails at runtime with FaultToleranceDefinitionException. "
                    + "Use FallbackHandler<T> instead: @Fallback(YourFallbackHandler.class)");
        }
    }
}
```

Add the import: `import org.jboss.jandex.AnnotationTarget;` (already present).

- [ ] **Step 4: Wire `validateFallback` into the `validate(DetectedAiAgentBuildItem)` call chain**

In the `validate(DetectedAiAgentBuildItem item)` method (line ~117), add the call:
```java
private void validate(DetectedAiAgentBuildItem item) {
    ClassInfo iface = item.getIface();
    validateActivationCondition(iface);
    validateAgentListenerSupplier(iface);
    validateChatMemoryProviderSupplier(iface);
    validateChatMemorySupplier(iface);
    validateChatModelSupplier(iface);
    validateContentRetrieverSupplier(iface);
    validateErrorHandler(iface);
    validateExitCondition(iface);
    validateFallback(iface);          // F-3: add here
    validateHumanInTheLoop(iface);
    validateOutput(iface);
    validateParallelExecutor(iface);
    validateRetrievalAugmentorSupplier(iface);
    validateToolProviderSupplier(iface);
    validateToolSupplier(iface);
}
```

- [ ] **Step 5: Run the test to verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agentic/deployment -Dtest=FallbackMethodOnAgentTest -Dno-format 2>&1 | tail -10
```
Expected: BUILD SUCCESS (the `assertException` condition is satisfied).

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/quarkus-langchain4j add \
  agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticProcessor.java \
  agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/validation/FallbackMethodOnAgentTest.java
git -C /Users/mdproctor/claude/quarkus-langchain4j commit -m "fix(agentic): build error for @Fallback(fallbackMethod=...) on agent interfaces

Agent interfaces are dynamic proxies. Fallback method name resolution
always fails at runtime with FaultToleranceDefinitionException.
Direct users to use FallbackHandler<T> instead.

Fixes quarkiverse/quarkus-langchain4j#2527 (F-3)"
```

---

## Task 7: F-4 and F-5 — FaultTolerance interaction warnings

**Files:**
- Modify: `agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticProcessor.java`
- Create: `agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/FaultToleranceRetryWarningTest.java`
- Create: `agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/TransactionalRetryWarningTest.java`

- [ ] **Step 1: Write the F-4 boot test**

Create `FaultToleranceRetryWarningTest.java`:
```java
package io.quarkiverse.langchain4j.agentic.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import jakarta.inject.Inject;

import org.eclipse.microprofile.faulttolerance.Retry;
import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import dev.langchain4j.agentic.Agent;
import dev.langchain4j.agentic.declarative.ChatModelSupplier;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.service.UserMessage;
import io.quarkiverse.langchain4j.testing.internal.WiremockAware;
import io.quarkus.test.QuarkusUnitTest;

/**
 * Verifies F-4: @Retry(retryOn=...) with types that won't match AgentInvocationException
 * emits a build-time warning but does NOT fail the build.
 * Warning text is emitted to the build log only (not assertable via QuarkusUnitTest).
 */
public class FaultToleranceRetryWarningTest {

    @RegisterExtension
    static final QuarkusUnitTest unitTest = new QuarkusUnitTest()
            .setArchiveProducer(() -> ShrinkWrap.create(JavaArchive.class)
                    .addClasses(NarrowRetryAgent.class, Agents.FixedResponseChatModel.class))
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.api-key", "test-key")
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.base-url",
                    WiremockAware.wiremockUrlForConfig("/v1"));

    public interface NarrowRetryAgent {
        @UserMessage("{{q}}")
        @Agent(description = "Agent with narrow retryOn")
        @Retry(maxRetries = 1, retryOn = IllegalStateException.class)
        String answer(String q);

        @ChatModelSupplier
        static ChatModel model() {
            return new Agents.FixedResponseChatModel("ok");
        }
    }

    @Inject
    NarrowRetryAgent agent;

    @Test
    void agentBootsSuccessfullyDespiteNarrowRetryOn() {
        // F-4 is a warning, not an error — the app should boot and function.
        // The warning is emitted to the build log (not captured here).
        assertThat(agent).isNotNull();
    }
}
```

- [ ] **Step 2: Write the F-5 boot test**

Create `TransactionalRetryWarningTest.java`:
```java
package io.quarkiverse.langchain4j.agentic.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import org.eclipse.microprofile.faulttolerance.Retry;
import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import dev.langchain4j.agentic.Agent;
import dev.langchain4j.agentic.declarative.ChatModelSupplier;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.service.UserMessage;
import io.quarkiverse.langchain4j.testing.internal.WiremockAware;
import io.quarkus.test.QuarkusUnitTest;

/**
 * Verifies F-5: @Transactional + @Retry on the same agent method emits a build-time
 * warning but does NOT fail the build.
 * Warning text is emitted to the build log only (not assertable via QuarkusUnitTest).
 */
public class TransactionalRetryWarningTest {

    @RegisterExtension
    static final QuarkusUnitTest unitTest = new QuarkusUnitTest()
            .setArchiveProducer(() -> ShrinkWrap.create(JavaArchive.class)
                    .addClasses(TransactionalRetryAgent.class, Agents.FixedResponseChatModel.class))
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.api-key", "test-key")
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.base-url",
                    WiremockAware.wiremockUrlForConfig("/v1"));

    public interface TransactionalRetryAgent {
        @UserMessage("{{q}}")
        @Agent(description = "Agent with transactional+retry combo")
        @Transactional
        @Retry(maxRetries = 1)
        String answer(String q);

        @ChatModelSupplier
        static ChatModel model() {
            return new Agents.FixedResponseChatModel("ok");
        }
    }

    @Inject
    TransactionalRetryAgent agent;

    @Test
    void agentBootsSuccessfullyDespiteTransactionalRetryCombo() {
        // F-5 is a warning, not an error.
        assertThat(agent).isNotNull();
    }
}
```

- [ ] **Step 3: Run both tests to verify they pass (before implementing the warning)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agentic/deployment \
  -Dtest="FaultToleranceRetryWarningTest,TransactionalRetryWarningTest" -Dno-format 2>&1 | tail -10
```
Expected: BUILD SUCCESS — both tests pass without the warning check (the warning doesn't exist yet). These are boot tests only.

- [ ] **Step 4: Add DotName constants and `validateFaultToleranceInteractions` to `AgenticProcessor.java`**

Add constants near the `FALLBACK` constant:
```java
private static final DotName RETRY =
        DotName.createSimple("org.eclipse.microprofile.faulttolerance.Retry");
private static final DotName TRANSACTIONAL =
        DotName.createSimple("jakarta.transaction.Transactional");
private static final Set<DotName> RETRY_SUPERTYPES = Set.of(
        DotName.createSimple("java.lang.RuntimeException"),
        DotName.createSimple("java.lang.Exception"),
        DotName.createSimple("java.lang.Throwable"),
        DotName.createSimple("dev.langchain4j.agentic.agent.AgentInvocationException"));
```

Add the new build step method:
```java
@BuildStep
@Produce(ServiceStartBuildItem.class)
void validateFaultToleranceInteractions(List<DetectedAiAgentBuildItem> agents) {
    for (DetectedAiAgentBuildItem agent : agents) {
        ClassInfo iface = agent.getIface();
        boolean classLevelRetry = iface.classAnnotation(RETRY) != null;
        boolean classLevelTransactional = iface.classAnnotation(TRANSACTIONAL) != null;

        for (MethodInfo method : agent.getAgenticMethods()) {
            // F-4: @Retry(retryOn=...) where none of the retryOn types match AgentInvocationException
            AnnotationInstance effectiveRetry = method.annotation(RETRY);
            if (effectiveRetry == null && classLevelRetry) {
                effectiveRetry = iface.classAnnotation(RETRY);
            }
            if (effectiveRetry != null) {
                AnnotationValue retryOn = effectiveRetry.value("retryOn");
                if (retryOn != null) {
                    boolean hasSupertype = false;
                    for (org.jboss.jandex.Type t : retryOn.asClassArray()) {
                        if (RETRY_SUPERTYPES.contains(t.name())) {
                            hasSupertype = true;
                            break;
                        }
                    }
                    if (!hasSupertype) {
                        log.warnf(
                                "Agent method '%s#%s' uses @Retry(retryOn=...) but agent exceptions are "
                                + "wrapped in AgentInvocationException. The retryOn types will not match "
                                + "the thrown exception. Add AgentInvocationException to retryOn, or "
                                + "remove retryOn to retry on all exceptions.",
                                iface.name(), method.name());
                    }
                }
            }

            // F-5: @Transactional + @Retry — partial commits + stale scope on retry
            boolean methodHasRetry = method.hasAnnotation(RETRY) || classLevelRetry;
            boolean methodHasTransactional = method.hasAnnotation(TRANSACTIONAL) || classLevelTransactional;
            if (methodHasRetry && methodHasTransactional) {
                log.warnf(
                        "Agent method '%s#%s' combines @Transactional and @Retry. "
                        + "AgenticScope is not a JTA resource — on retry, the second attempt re-enters "
                        + "after the first transaction has already closed, leaving partial scope state "
                        + "unrolled. Ensure this combination is intentional.",
                        iface.name(), method.name());
            }
        }
    }
}
```

Ensure `import org.jboss.jandex.AnnotationValue;` is present (it is).

- [ ] **Step 5: Run both tests again to confirm they still pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agentic/deployment \
  -Dtest="FaultToleranceRetryWarningTest,TransactionalRetryWarningTest" -Dno-format 2>&1 | tail -10
```
Expected: BUILD SUCCESS. Check build output for the warning messages (visible in Maven output even if not asserted in tests).

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/quarkus-langchain4j add \
  agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticProcessor.java \
  agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/FaultToleranceRetryWarningTest.java \
  agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/TransactionalRetryWarningTest.java
git -C /Users/mdproctor/claude/quarkus-langchain4j commit -m "feat(agentic): build-time warnings for @Retry and @Transactional misuse

F-4: warn when @Retry(retryOn=...) targets types that will never match
AgentInvocationException wrapper.

F-5: warn when @Transactional and @Retry co-exist on the same agent
method — AgenticScope is not a JTA resource; retries produce partial
commits alongside stale scope state.

Fixes quarkiverse/quarkus-langchain4j#2527 (F-4, F-5)"
```

---

## Task 8: F-1 — @Retry and @CircuitBreaker test cases

**Files:**
- Modify: `agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/FaultToleranceTest.java`

- [ ] **Step 1: Add `FailOnceModel` and `AlwaysFailingModel` inner classes to `FaultToleranceTest.java`**

Add these static inner classes inside `FaultToleranceTest` (after the existing inner interfaces):

```java
/** Chat model that fails the first N calls, then returns a fixed response. */
public static class FailNTimesChatModel implements ChatModel {
    private final java.util.concurrent.atomic.AtomicInteger failsRemaining;
    private final String successResponse;

    public FailNTimesChatModel(int failCount, String successResponse) {
        this.failsRemaining = new java.util.concurrent.atomic.AtomicInteger(failCount);
        this.successResponse = successResponse;
    }

    @Override
    public dev.langchain4j.model.output.Response<dev.langchain4j.data.message.AiMessage> generate(
            java.util.List<dev.langchain4j.data.message.ChatMessage> messages) {
        if (failsRemaining.getAndDecrement() > 0) {
            throw new RuntimeException("deliberate failure for retry test");
        }
        return dev.langchain4j.model.output.Response.from(
                dev.langchain4j.data.message.AiMessage.from(successResponse));
    }
}

/** Chat model that always throws. */
public static class AlwaysFailingChatModel implements ChatModel {
    @Override
    public dev.langchain4j.model.output.Response<dev.langchain4j.data.message.AiMessage> generate(
            java.util.List<dev.langchain4j.data.message.ChatMessage> messages) {
        throw new RuntimeException("always fails");
    }
}
```

- [ ] **Step 2: Add `RetryAgent` and `CircuitBreakerAgent` interfaces to `FaultToleranceTest.java`**

```java
public interface RetryAgent {
    @UserMessage("{{q}}")
    @Agent(value = "Agent that retries on failure", outputKey = "answer")
    @org.eclipse.microprofile.faulttolerance.Retry(maxRetries = 2)
    String answer(String q);

    @ChatModelSupplier
    static ChatModel model() {
        return new FailNTimesChatModel(1, "retried-success");
    }
}

public interface CircuitBreakerAgent {
    @UserMessage("{{q}}")
    @Agent(value = "Agent with circuit breaker", outputKey = "answer")
    @org.eclipse.microprofile.faulttolerance.CircuitBreaker(
            requestVolumeThreshold = 3,
            failureRatio = 0.5,
            delay = 10000,
            delayUnit = java.time.temporal.ChronoUnit.MILLIS)
    String answer(String q);

    @ChatModelSupplier
    static ChatModel model() {
        return new AlwaysFailingChatModel();
    }
}
```

- [ ] **Step 3: Add these agents to the `@RegisterExtension` archive**

Update the existing `setArchiveProducer` call to include the new classes:
```java
static final QuarkusUnitTest unitTest = new QuarkusUnitTest()
        .setArchiveProducer(
                () -> ShrinkWrap.create(JavaArchive.class)
                        .addClasses(FirstAgent.class, SecondAgentWithDelay.class, RouterAgent.class,
                                    RetryAgent.class, CircuitBreakerAgent.class,
                                    FailNTimesChatModel.class, AlwaysFailingChatModel.class,
                                    Agents.FixedResponseChatModel.class))
        ...
```

- [ ] **Step 4: Add `@Inject` fields and test methods**

Add inject fields:
```java
@Inject
RetryAgent retryAgent;

@Inject
CircuitBreakerAgent circuitBreakerAgent;
```

Add test methods:
```java
@Test
void testRetrySucceedsAfterOneFailure() {
    // model fails once, then succeeds — @Retry(maxRetries=2) covers it
    String result = retryAgent.answer("question");
    assertThat(result).isEqualTo("retried-success");
}

@Test
void testCircuitBreakerOpensAfterThreshold() {
    // Trip the circuit: 3 failures with failureRatio=0.5 means 2+ failures in 3 opens circuit
    for (int i = 0; i < 3; i++) {
        assertThrows(AgentInvocationException.class, () -> circuitBreakerAgent.answer("q"));
    }
    // Circuit is now open — next call throws CircuitBreakerOpenException
    assertThat(assertThrows(AgentInvocationException.class, () -> circuitBreakerAgent.answer("q")))
            .hasRootCauseInstanceOf(org.eclipse.microprofile.faulttolerance.exceptions.CircuitBreakerOpenException.class);
}
```

- [ ] **Step 5: Run `FaultToleranceTest`**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agentic/deployment -Dtest=FaultToleranceTest -Dno-format 2>&1 | tail -10
```
Expected: BUILD SUCCESS (all tests including the two new ones pass).

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/quarkus-langchain4j add \
  agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/FaultToleranceTest.java
git -C /Users/mdproctor/claude/quarkus-langchain4j commit -m "test(agentic): add @Retry and @CircuitBreaker test cases

Documents and verifies that MicroProfile FaultTolerance annotations
work correctly on agent interfaces. Closes the untested/undocumented
gap noted in the audit.

Fixes quarkiverse/quarkus-langchain4j#2527 (F-1)"
```

---

## Task 9: L-1 — `@ConfigMapping` skeleton

**Files:**
- Create: `agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/AgenticRuntimeConfig.java`
- Modify: `agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/devui/AgenticDevUIProcessor.java`

- [ ] **Step 1: Write the test that verifies the config property is readable**

Add a test to a new file `AgenticRuntimeConfigTest.java` in the deployment test package:

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
            .setArchiveProducer(() -> ShrinkWrap.create(JavaArchive.class)
                    .addClasses(Agents.ExpertRouterAgent.class))
            .overrideRuntimeConfigKey("quarkus.langchain4j.agent.dev-ui.eager-init", "false")
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.api-key", "test-key")
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.base-url", "http://localhost");

    @Inject
    AgenticRuntimeConfig config;

    @Test
    void eagerInitConfigPropertyIsReadable() {
        assertThat(config.devUi().eagerInit()).isFalse();
    }

    @Test
    void defaultMaxIterationsIsEmpty() {
        assertThat(config.defaultMaxIterations()).isEmpty();
    }
}
```

- [ ] **Step 2: Run the test — expect compile failure (class doesn't exist yet)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agentic/deployment -Dtest=AgenticRuntimeConfigTest -Dno-format 2>&1 | grep "ERROR\|cannot find" | head -5
```
Expected: compile error — `AgenticRuntimeConfig` not found.

- [ ] **Step 3: Create `AgenticRuntimeConfig.java`**

```java
package io.quarkiverse.langchain4j.agentic.runtime;

import java.util.Optional;

import io.quarkus.runtime.annotations.ConfigPhase;
import io.quarkus.runtime.annotations.ConfigRoot;
import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;

@ConfigRoot(phase = ConfigPhase.RUN_TIME)
@ConfigMapping(prefix = "quarkus.langchain4j.agent")
public interface AgenticRuntimeConfig {

    /**
     * Maximum iterations for loop and planner agents.
     * When absent, langchain4j-agentic uses its own default.
     */
    Optional<Integer> defaultMaxIterations();

    /** Dev UI configuration. */
    DevUiConfig devUi();

    interface DevUiConfig {
        /**
         * Whether to eagerly initialise root agents at startup in dev mode.
         * Set to {@code false} in CI environments to avoid unnecessary agent startup latency.
         */
        @WithDefault("true")
        boolean eagerInit();
    }
}
```

- [ ] **Step 4: Wire `eagerInit` in `AgenticDevUIProcessor.enableDevModeMonitoring`**

Add `AgenticRuntimeConfig runtimeConfig` as a parameter to `enableDevModeMonitoring` and gate `eagerlyInitRootAgents` on it:

```java
@BuildStep(onlyIf = IsDevelopment.class)
@Record(ExecutionTime.RUNTIME_INIT)
@Consume(SyntheticBeansRuntimeInitBuildItem.class)
void enableDevModeMonitoring(List<DetectedAiAgentBuildItem> agents,
        AgenticRecorder recorder,
        AgenticRuntimeConfig runtimeConfig) {
    DotName monitoredAgentName = DotName.createSimple(MonitoredAgent.class.getName());
    Set<String> monitoredRootAgentClassNames = filterUserAgents(agents).stream()
            .filter(a -> a.getIface().interfaceNames().stream().anyMatch(dn -> dn.equals(monitoredAgentName)))
            .map(a -> a.getIface().name().toString())
            .collect(Collectors.toSet());
    if (!monitoredRootAgentClassNames.isEmpty()) {
        recorder.enableDevModeMonitoring(monitoredRootAgentClassNames);
        if (runtimeConfig.devUi().eagerInit()) {
            recorder.eagerlyInitRootAgents(monitoredRootAgentClassNames);
        }
    }
}
```

Add import to `AgenticDevUIProcessor.java`:
```java
import io.quarkiverse.langchain4j.agentic.runtime.AgenticRuntimeConfig;
```

- [ ] **Step 5: Run the config test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agentic/deployment,agentic/runtime -Dtest=AgenticRuntimeConfigTest -Dno-format 2>&1 | tail -10
```
Expected: BUILD SUCCESS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/quarkus-langchain4j add \
  agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/AgenticRuntimeConfig.java \
  agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/devui/AgenticDevUIProcessor.java \
  agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticRuntimeConfigTest.java
git -C /Users/mdproctor/claude/quarkus-langchain4j commit -m "feat(agentic): add AgenticRuntimeConfig @ConfigMapping skeleton

Establishes the quarkus.langchain4j.agent.* namespace. Current properties:
- quarkus.langchain4j.agent.default-max-iterations (unused by code yet)
- quarkus.langchain4j.agent.dev-ui.eager-init (gates eagerlyInitRootAgents)

Fixes quarkiverse/quarkus-langchain4j#2527 (L-1)"
```

---

## Task 10: C-6 — File upstream PR to `langchain4j-agentic`

- [ ] **Step 1: Note the change needed**

The fix is in `PlannerBasedInvocationHandler.parallelExecution()` in the upstream `langchain4j-agentic` repository. Find the method (sources JAR at `~/.m2/repository/dev/langchain4j/langchain4j-agentic/1.15.1-beta25/langchain4j-agentic-1.15.1-beta25-sources.jar`). The current loop:
```java
for (CompletableFuture<Object> future : futures) {
    results.add(future.get());
}
```
Should become:
```java
CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).get();
for (CompletableFuture<Object> future : futures) {
    results.add(future.join()); // join() is safe after allOf completes
}
```

- [ ] **Step 2: File a GitHub issue or PR on `langchain4j-agentic`**

Frame it as a platform-agnostic performance improvement per the upstream contribution protocol. This is tracked as C-6 in the audit. No Quarkus code changes.

- [ ] **Step 3: Run all agentic tests to confirm the full C1 implementation is solid**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agentic/deployment,agentic/runtime -Dno-format 2>&1 | tail -20
```
Expected: BUILD SUCCESS, all tests pass.

---

## Self-Review

**Spec coverage:**
- D-1 ✅ Task 1
- C-7 ✅ Task 2
- ValidationUtil helpers ✅ Task 3
- S-3 ✅ Task 4
- F-7 ✅ Task 5
- F-3 ✅ Task 6
- F-4 ✅ Task 7
- F-5 ✅ Task 7
- F-1 ✅ Task 8
- L-1 ✅ Task 9
- C-6 ✅ Task 10

**Type consistency:**
- `ValidationUtil.transitiveInterfaces(ClassInfo, IndexView) → Set<ClassInfo>` — used in Task 4 (S-3) and Task 5 (F-7) ✅
- `ValidationUtil.hasAnnotation(Collection<AnnotationInstance>, Set<DotName>) → boolean` — used in Task 5 (F-7) ✅
- `ALL_CDI_CAPABLE_SUPPLIER_ANNOTATIONS` defined in Task 4, used in same task ✅
- `FALLBACK`, `RETRY`, `TRANSACTIONAL`, `RETRY_SUPERTYPES` constants — defined and used in Tasks 6-7 ✅
- `AgenticRuntimeConfig` — created in Task 9, used in same task's `AgenticDevUIProcessor` change ✅

**Placeholder scan:** No TBDs, all code blocks complete. ✅
