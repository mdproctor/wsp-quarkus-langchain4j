# C3 Parallel Safety Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the hardcoded `DefaultExecutorProvider` in langchain4j-core with a pluggable SPI, then register Quarkus `ManagedExecutor` as the default — fixing CDI/OTel/Security context loss on `@ParallelAgent` worker threads.

**Architecture:** Two-layer change. Upstream (langchain4j-core): add `ExecutorServiceProvider` SPI and static override to `DefaultExecutorProvider` with three-tier priority (volatile override → lazy ServiceLoader → virtual thread fallback). Quarkus (agentic module): one recorder method registers `ManagedExecutor` via the static setter at `RUNTIME_INIT`.

**Tech Stack:** langchain4j-core 1.15.1, Quarkus CDI (`Arc`), MicroProfile Context Propagation (`ManagedExecutor`), SmallRye, JUnit 5 + `QuarkusUnitTest`

**Spec:** `~/claude/public/quarkus-langchain4j/specs/c3-parallel-safety/2026-06-07-c3-parallel-safety-design.md`

---

### File Map

| File | Action | Purpose |
|---|---|---|
| `agentic/runtime/src/main/java/.../spi/ExecutorServiceProvider.java` | Create | SPI interface — temporary local copy until upstream #5376 merges |
| `agentic/runtime/src/main/java/.../runtime/DefaultExecutorProviderOverride.java` | Create | Static setter + ServiceLoader + fallback (local override of upstream class) |
| `agentic/runtime/src/main/java/.../runtime/AgenticRecorder.java` | Modify | Add `registerDefaultExecutorProvider()` `@RuntimeInit` method |
| `agentic/deployment/src/main/java/.../deployment/AgenticProcessor.java` | Modify | Add `@BuildStep` calling recorder; add info log in `validateParallelExecutor()` |
| `agentic/deployment/src/test/java/.../deployment/ParallelContextPropagationTest.java` | Create | Test 1: CDI request context active on worker threads |
| `agentic/deployment/src/test/java/.../deployment/ParallelExecutorRespectedTest.java` | Create | Test 2: user `@ParallelExecutor` not overridden + info message |
| `agentic/deployment/src/test/java/.../deployment/ParallelOtelPropagationTest.java` | Create | Test 4: OTel span continuity (best effort) |

**Note on upstream SPI locality:** The `ExecutorServiceProvider` SPI and the `DefaultExecutorProvider` modification live locally in this repo for now. When upstream accepts #5376, we delete our local copies and depend on the upstream version. The Quarkus recorder code is identical either way.

**Note on Test 3 (SecurityIdentity):** The agentic deployment module has no `quarkus-security` test dependency. Adding it and `@TestSecurity` infrastructure is out of scope for C3. Deferred to a follow-up.

---

### Task 1: Create local ExecutorServiceProvider SPI interface

**Files:**
- Create: `agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/spi/ExecutorServiceProvider.java`

- [ ] **Step 1: Create the SPI interface**

```java
package io.quarkiverse.langchain4j.agentic.runtime.spi;

import java.util.concurrent.ExecutorService;

/**
 * Temporary local copy — will be replaced by upstream langchain4j-core SPI
 * when langchain4j/langchain4j#5376 merges.
 */
public interface ExecutorServiceProvider {
    ExecutorService get();
}
```

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/quarkus-langchain4j add agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/spi/ExecutorServiceProvider.java
git -C /Users/mdproctor/claude/quarkus-langchain4j commit -m "feat(agentic): add ExecutorServiceProvider SPI interface (local, pending upstream #5376)"
```

---

### Task 2: Override DefaultExecutorProvider with pluggable fallback

The upstream `DefaultExecutorProvider` is in `langchain4j-core` (package `dev.langchain4j.internal`). We can't modify it directly. Instead, we call its equivalent static setter pattern from a Quarkus recorder — but since the upstream class doesn't have a setter yet, we create a local bridge that replaces the default before any agent is built.

The approach: call `DefaultExecutorProvider`'s static holder indirectly by using the upstream's existing code path. Since `PlannerBasedInvocationHandler.parallelExecution()` does `executor != null ? executor : DefaultExecutorProvider.getDefaultExecutorService()`, and we can't modify the upstream class, we need to ensure our `ManagedExecutor` gets set as the executor on the builder before `parallelExecution()` runs.

**Wait — the spec says to modify `DefaultExecutorProvider` locally.** But `DefaultExecutorProvider` comes from `langchain4j-core` JAR (a transitive dependency). We can shadow it by creating a class with the same fully-qualified name in our module — Java classloading will pick up our version from the application classpath before the JAR version.

**Files:**
- Create: `agentic/runtime/src/main/java/dev/langchain4j/internal/DefaultExecutorProvider.java`

- [ ] **Step 1: Create the shadowed DefaultExecutorProvider**

This class has the same package and class name as the upstream original. It adds a `volatile` static override field and a static setter. The Quarkus module's classloader will load this version instead of the one in langchain4j-core JAR.

```java
package dev.langchain4j.internal;

import static dev.langchain4j.internal.VirtualThreadUtils.createVirtualThreadExecutor;

import java.util.ServiceLoader;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.SynchronousQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

import io.quarkiverse.langchain4j.agentic.runtime.spi.ExecutorServiceProvider;

public class DefaultExecutorProvider {

    private static volatile ExecutorService override;

    private DefaultExecutorProvider() {
    }

    public static void setDefaultExecutorService(ExecutorService executorService) {
        override = executorService;
    }

    public static ExecutorService getDefaultExecutorService() {
        ExecutorService result = override;
        if (result != null) {
            return result;
        }
        return Holder.EXECUTOR_SERVICE;
    }

    private static class Holder {
        private static final ExecutorService EXECUTOR_SERVICE = loadExecutorService();

        private static ExecutorService loadExecutorService() {
            for (ExecutorServiceProvider provider : ServiceLoader.load(ExecutorServiceProvider.class)) {
                return provider.get();
            }
            return createVirtualThreadExecutor(Holder::createPlatformThreadExecutorService);
        }

        private static ExecutorService createPlatformThreadExecutorService() {
            return new ThreadPoolExecutor(0, Integer.MAX_VALUE, 1, TimeUnit.SECONDS, new SynchronousQueue<>());
        }
    }
}
```

- [ ] **Step 2: Build to verify the shadow compiles and doesn't break existing code**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/agentic/pom.xml compile -Dno-format`
Expected: BUILD SUCCESS — the shadowed class is loaded instead of the upstream version.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/quarkus-langchain4j add agentic/runtime/src/main/java/dev/langchain4j/internal/DefaultExecutorProvider.java
git -C /Users/mdproctor/claude/quarkus-langchain4j commit -m "feat(agentic): shadow DefaultExecutorProvider with pluggable SPI (pending upstream #5376)"
```

---

### Task 3: Write test for CDI request context propagation on parallel worker threads

**Files:**
- Create: `agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/ParallelContextPropagationTest.java`

- [ ] **Step 1: Write the failing test**

This test creates a `@ParallelAgent` with two sub-agents. One sub-agent's `@ChatModelSupplier` accesses a `@RequestScoped` CDI bean. Without context propagation, this throws `ContextNotActiveException`. With `ManagedExecutor`, the request context is active.

```java
package io.quarkiverse.langchain4j.agentic.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;

import jakarta.enterprise.context.RequestScoped;
import jakarta.inject.Inject;

import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import dev.langchain4j.agentic.Agent;
import dev.langchain4j.agentic.declarative.ChatModelSupplier;
import dev.langchain4j.agentic.declarative.ParallelAgent;
import dev.langchain4j.data.message.AiMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import dev.langchain4j.model.chat.response.ChatResponse;
import dev.langchain4j.service.UserMessage;
import dev.langchain4j.service.V;
import io.quarkiverse.langchain4j.openai.testing.internal.OpenAiBaseTest;
import io.quarkiverse.langchain4j.testing.internal.WiremockAware;
import io.quarkus.arc.Arc;
import io.quarkus.test.QuarkusUnitTest;

public class ParallelContextPropagationTest extends OpenAiBaseTest {

    @RegisterExtension
    static final QuarkusUnitTest unitTest = new QuarkusUnitTest()
            .setArchiveProducer(() -> ShrinkWrap.create(JavaArchive.class)
                    .addClasses(RequestScopedCounter.class, CountingAgent.class,
                            EchoAgent.class, ParallelCounterAgent.class))
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.api-key", "test-key")
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.base-url",
                    WiremockAware.wiremockUrlForConfig("/v1"));

    @RequestScoped
    public static class RequestScopedCounter {
        private int count = 0;

        public int increment() {
            return ++count;
        }

        public int getCount() {
            return count;
        }
    }

    public interface CountingAgent {
        @UserMessage("Count: {{input}}")
        @Agent(description = "An agent that accesses a request-scoped bean", outputKey = "count")
        String count(@V("input") String input);

        @ChatModelSupplier
        static ChatModel chatModel() {
            // Access the @RequestScoped bean from the worker thread.
            // Without ManagedExecutor, this throws ContextNotActiveException.
            RequestScopedCounter counter = Arc.container().select(RequestScopedCounter.class).get();
            int val = counter.increment();
            return new Agents.FixedResponseChatModel("counted:" + val);
        }
    }

    public interface EchoAgent {
        @UserMessage("Echo: {{input}}")
        @Agent(description = "An agent that echoes input", outputKey = "echo")
        String echo(@V("input") String input);

        @ChatModelSupplier
        static ChatModel chatModel() {
            return new Agents.FixedResponseChatModel("echoed");
        }
    }

    public record ParallelResult(String count, String echo) {
    }

    public interface ParallelCounterAgent {
        @ParallelAgent(outputKey = "result", subAgents = { CountingAgent.class, EchoAgent.class })
        List<ParallelResult> run(@V("input") String input);
    }

    @Inject
    ParallelCounterAgent agent;

    @Test
    void requestScopedBeanAccessibleOnParallelWorkerThreads() {
        // This would throw ContextNotActiveException without ManagedExecutor
        List<ParallelResult> result = agent.run("test");
        assertThat(result).isNotNull();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/agentic/deployment test -Dtest=ParallelContextPropagationTest -Dno-format`
Expected: FAIL — `ContextNotActiveException` because `ManagedExecutor` isn't registered yet.

- [ ] **Step 3: Commit failing test**

```bash
git -C /Users/mdproctor/claude/quarkus-langchain4j add agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/ParallelContextPropagationTest.java
git -C /Users/mdproctor/claude/quarkus-langchain4j commit -m "test(agentic): failing test — @RequestScoped bean inaccessible on parallel worker threads"
```

---

### Task 4: Register ManagedExecutor via recorder and build step

**Files:**
- Modify: `agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/AgenticRecorder.java`
- Modify: `agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticProcessor.java`

- [ ] **Step 1: Add recorder method**

In `AgenticRecorder.java`, add after the existing `registerChatSupplierParameterResolver()` method:

```java
@RuntimeInit
public void registerDefaultExecutorProvider() {
    DefaultExecutorProvider.setDefaultExecutorService(Arc.container().instance(ManagedExecutor.class).get());
}
```

Add the necessary imports:
```java
import dev.langchain4j.internal.DefaultExecutorProvider;
import org.eclipse.microprofile.context.ManagedExecutor;
```

- [ ] **Step 2: Add build step**

In `AgenticProcessor.java`, add a new `@BuildStep` method. Place it near the existing `registerChatSupplierParameterResolver` build step:

```java
@BuildStep
@Record(ExecutionTime.RUNTIME_INIT)
void registerDefaultExecutorProvider(AgenticRecorder recorder) {
    recorder.registerDefaultExecutorProvider();
}
```

- [ ] **Step 3: Run the failing test to verify it now passes**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/agentic/deployment test -Dtest=ParallelContextPropagationTest -Dno-format`
Expected: PASS — `ManagedExecutor` propagates request context to worker threads.

- [ ] **Step 4: Run the full existing test suite to check for regressions**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/agentic/deployment test -Dno-format`
Expected: All existing tests pass — the `ManagedExecutor` replaces raw virtual threads globally, but existing tests don't depend on a specific executor type.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/quarkus-langchain4j add agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/AgenticRecorder.java agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticProcessor.java
git -C /Users/mdproctor/claude/quarkus-langchain4j commit -m "feat(agentic): register ManagedExecutor as default parallel executor via DefaultExecutorProvider SPI"
```

---

### Task 5: Write test for user-declared @ParallelExecutor respected

**Files:**
- Create: `agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/ParallelExecutorRespectedTest.java`

- [ ] **Step 1: Write the test**

This test verifies that when a user declares `@ParallelExecutor`, their custom executor is used instead of `ManagedExecutor`. We do this by using a custom executor that sets a thread-name prefix, then asserting the sub-agent runs on a thread with that prefix.

```java
package io.quarkiverse.langchain4j.agentic.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.concurrent.Executor;
import java.util.concurrent.Executors;
import java.util.concurrent.atomic.AtomicReference;

import jakarta.inject.Inject;

import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import dev.langchain4j.agentic.Agent;
import dev.langchain4j.agentic.declarative.ChatModelSupplier;
import dev.langchain4j.agentic.declarative.ParallelAgent;
import dev.langchain4j.agentic.declarative.ParallelExecutor;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.service.UserMessage;
import dev.langchain4j.service.V;
import io.quarkiverse.langchain4j.openai.testing.internal.OpenAiBaseTest;
import io.quarkiverse.langchain4j.testing.internal.WiremockAware;
import io.quarkus.test.QuarkusUnitTest;

public class ParallelExecutorRespectedTest extends OpenAiBaseTest {

    static final AtomicReference<String> capturedThreadName = new AtomicReference<>();

    @RegisterExtension
    static final QuarkusUnitTest unitTest = new QuarkusUnitTest()
            .setArchiveProducer(() -> ShrinkWrap.create(JavaArchive.class)
                    .addClasses(ThreadCapturingAgent.class, SimpleAgent.class,
                            CustomExecutorParallelAgent.class))
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.api-key", "test-key")
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.base-url",
                    WiremockAware.wiremockUrlForConfig("/v1"));

    public interface ThreadCapturingAgent {
        @UserMessage("{{input}}")
        @Agent(description = "Captures thread name", outputKey = "captured")
        String capture(@V("input") String input);

        @ChatModelSupplier
        static ChatModel chatModel() {
            capturedThreadName.set(Thread.currentThread().getName());
            return new Agents.FixedResponseChatModel("captured");
        }
    }

    public interface SimpleAgent {
        @UserMessage("{{input}}")
        @Agent(description = "Simple echo", outputKey = "echo")
        String echo(@V("input") String input);

        @ChatModelSupplier
        static ChatModel chatModel() {
            return new Agents.FixedResponseChatModel("echoed");
        }
    }

    public record Result(String captured, String echo) {
    }

    public interface CustomExecutorParallelAgent {
        @ParallelAgent(outputKey = "result", subAgents = { ThreadCapturingAgent.class, SimpleAgent.class })
        List<Result> run(@V("input") String input);

        @ParallelExecutor
        static Executor executor() {
            return Executors.newFixedThreadPool(2, r -> {
                Thread t = new Thread(r);
                t.setName("custom-parallel-" + t.getId());
                return t;
            });
        }
    }

    @Inject
    CustomExecutorParallelAgent agent;

    @Test
    void userDeclaredParallelExecutorIsUsed() {
        agent.run("test");
        assertThat(capturedThreadName.get())
                .as("Sub-agent should run on custom executor thread, not ManagedExecutor")
                .startsWith("custom-parallel-");
    }
}
```

- [ ] **Step 2: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/agentic/deployment test -Dtest=ParallelExecutorRespectedTest -Dno-format`
Expected: PASS — user's custom executor is used, thread name starts with "custom-parallel-".

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/quarkus-langchain4j add agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/ParallelExecutorRespectedTest.java
git -C /Users/mdproctor/claude/quarkus-langchain4j commit -m "test(agentic): verify user-declared @ParallelExecutor is respected over ManagedExecutor"
```

---

### Task 6: Add @ParallelExecutor info message

**Files:**
- Modify: `agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticProcessor.java`

- [ ] **Step 1: Add info log in validateParallelExecutor()**

In `AgenticProcessor.java`, modify the `validateParallelExecutor` method to emit an info message when `@ParallelExecutor` is detected. Add after the validation loop body:

```java
private void validateParallelExecutor(ClassInfo iface) {
    DotName annotationToValidate = AgenticLangChain4jDotNames.PARALLEL_EXECUTOR;
    List<AnnotationInstance> instances = iface.annotations(annotationToValidate);
    for (AnnotationInstance instance : instances) {
        if (instance.target().kind() != AnnotationTarget.Kind.METHOD) {
            log.warnf("Unhandled '@%s' annotation: '%s'", annotationToValidate.withoutPackagePrefix(), instance.target());
            continue;
        }
        MethodInfo method = instance.target().asMethod();
        validateStaticMethod(method, annotationToValidate);
        validateNoMethodParameters(method, annotationToValidate);
        validateAllowedReturnTypes(method, Set.of(DotNames.EXECUTOR), annotationToValidate);
        log.infof("Agent '%s' declares @ParallelExecutor — automatic CDI/OTel/Security context propagation is bypassed. "
                + "Ensure your executor propagates contexts if needed.", iface.name());
    }
}
```

- [ ] **Step 2: Run existing @ParallelExecutor validation tests to confirm no regression**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/agentic/deployment test -Dtest=NonStaticReturnParallelExecutorTest,NonExecutorReturnTypeParallelExecutorTest,NonEmptyParametersParallelExecutorTest -Dno-format`
Expected: All pass — the info message doesn't affect validation logic.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/quarkus-langchain4j add agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticProcessor.java
git -C /Users/mdproctor/claude/quarkus-langchain4j commit -m "feat(agentic): info message when @ParallelExecutor bypasses automatic context propagation"
```

---

### Task 7: Write OTel span continuity test (best effort)

**Files:**
- Modify: `agentic/deployment/pom.xml` (add OTel test dependency)
- Create: `agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/ParallelOtelPropagationTest.java`

- [ ] **Step 1: Add quarkus-opentelemetry test dependency**

In `agentic/deployment/pom.xml`, add inside `<dependencies>`:

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-opentelemetry</artifactId>
    <scope>test</scope>
</dependency>
```

- [ ] **Step 2: Write the OTel test**

```java
package io.quarkiverse.langchain4j.agentic.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;

import jakarta.inject.Inject;

import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import dev.langchain4j.agentic.Agent;
import dev.langchain4j.agentic.declarative.ChatModelSupplier;
import dev.langchain4j.agentic.declarative.ParallelAgent;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.service.UserMessage;
import dev.langchain4j.service.V;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.sdk.testing.exporter.InMemorySpanExporter;
import io.opentelemetry.sdk.trace.data.SpanData;
import io.quarkiverse.langchain4j.openai.testing.internal.OpenAiBaseTest;
import io.quarkiverse.langchain4j.testing.internal.WiremockAware;
import io.quarkus.test.QuarkusUnitTest;

public class ParallelOtelPropagationTest extends OpenAiBaseTest {

    @RegisterExtension
    static final QuarkusUnitTest unitTest = new QuarkusUnitTest()
            .setArchiveProducer(() -> ShrinkWrap.create(JavaArchive.class)
                    .addClasses(OtelSubAgentA.class, OtelSubAgentB.class, OtelParallelAgent.class))
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.api-key", "test-key")
            .overrideRuntimeConfigKey("quarkus.langchain4j.openai.base-url",
                    WiremockAware.wiremockUrlForConfig("/v1"));

    public interface OtelSubAgentA {
        @UserMessage("A: {{input}}")
        @Agent(description = "Sub-agent A", outputKey = "a")
        String process(@V("input") String input);

        @ChatModelSupplier
        static ChatModel chatModel() {
            return new Agents.FixedResponseChatModel("result-a");
        }
    }

    public interface OtelSubAgentB {
        @UserMessage("B: {{input}}")
        @Agent(description = "Sub-agent B", outputKey = "b")
        String process(@V("input") String input);

        @ChatModelSupplier
        static ChatModel chatModel() {
            return new Agents.FixedResponseChatModel("result-b");
        }
    }

    public record OtelResult(String a, String b) {
    }

    public interface OtelParallelAgent {
        @ParallelAgent(outputKey = "result", subAgents = { OtelSubAgentA.class, OtelSubAgentB.class })
        List<OtelResult> run(@V("input") String input);
    }

    @Inject
    OtelParallelAgent agent;

    @Inject
    InMemorySpanExporter spanExporter;

    @Inject
    Tracer tracer;

    @BeforeEach
    void resetSpans() {
        spanExporter.reset();
    }

    @Test
    void parallelSubAgentsShareParentTraceId() {
        Span parentSpan = tracer.spanBuilder("test-parent").startSpan();
        try (var scope = parentSpan.makeCurrent()) {
            agent.run("otel-test");
        } finally {
            parentSpan.end();
        }

        List<SpanData> spans = spanExporter.getFinishedSpanItems();
        String parentTraceId = spans.stream()
                .filter(s -> s.getName().equals("test-parent"))
                .map(s -> s.getTraceContext().getTraceId())
                .findFirst()
                .orElse(null);

        assertThat(parentTraceId).as("Parent span should exist").isNotNull();

        // All spans in the trace should share the same trace ID
        List<SpanData> traceSpans = spans.stream()
                .filter(s -> s.getTraceContext().getTraceId().equals(parentTraceId))
                .toList();

        // At minimum: parent + some child spans from the parallel execution
        assertThat(traceSpans.size()).as("Should have parent + child spans in same trace").isGreaterThan(1);
    }
}
```

- [ ] **Step 3: Run test**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/agentic/deployment test -Dtest=ParallelOtelPropagationTest -Dno-format`
Expected: PASS — spans share the same trace ID because `ManagedExecutor` propagates OTel context.

**Note:** If OTel test infrastructure is more complex than expected (missing beans, classpath issues), this test may need adjustment. It's best-effort for C3.

- [ ] **Step 4: Run full test suite**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/agentic/deployment test -Dno-format`
Expected: All tests pass including the new OTel test.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/quarkus-langchain4j add agentic/deployment/pom.xml agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/ParallelOtelPropagationTest.java
git -C /Users/mdproctor/claude/quarkus-langchain4j commit -m "test(agentic): best-effort OTel span continuity test for parallel agents"
```

---

### Task 8: Final integration verification

- [ ] **Step 1: Run the complete agentic module test suite**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/quarkus-langchain4j/agentic test -Dno-format`
Expected: All tests pass — existing workflow tests, CDI auto-wiring tests, fault tolerance tests, plus the three new parallel safety tests.

- [ ] **Step 2: Verify the commit history is clean**

Run: `git -C /Users/mdproctor/claude/quarkus-langchain4j log --oneline -10`
Expected: Clean sequence of commits for C3 work.
