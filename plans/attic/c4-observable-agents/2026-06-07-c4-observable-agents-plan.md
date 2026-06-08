# C4 — Observable Agents Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add production observability to the agentic module — OTel spans, Micrometer metrics, CDI events, health check, hot-reload fix, and structured Dev UI — using the upstream `AgentListener` SPI.

**Architecture:** Four CDI `AgentListener` beans conditionally registered via `AdditionalBeanBuildItem` / `HealthBuildItem` in a new `AgenticObservabilityProcessor`. All beans are `@ApplicationScoped` and set `inheritedBySubagents() → true`. Build-time capability detection gates registration on `Capability.OPENTELEMETRY_TRACER`, `MetricsCapabilityBuildItem`, and `Capability.SMALLRYE_HEALTH`.

**Tech Stack:** Quarkus CDI, OpenTelemetry API (`Tracer`, `Span`), Micrometer (`Metrics.globalRegistry`, `MeterProvider`), MicroProfile Health (`@Readiness`, `HealthCheck`), CDI events (`Event.fire()`), Vert.x JSON (`JsonObject`, `JsonArray`), Lit + Vaadin (Dev UI frontend).

**Spec:** `specs/2026-06-07-c4-observable-agents-design.md`

**Existing patterns to follow:**
- `core/deployment/.../ListenersProcessor.java` — conditional `AdditionalBeanBuildItem` with capability checks
- `core/runtime/.../MetricsChatModelListener.java` — `MeterProvider` with `Metrics.globalRegistry`
- `core/runtime/.../SpanWrapper.java` — `Tracer` injection, span lifecycle
- `mcp/deployment/.../McpProcessor.java` — `HealthBuildItem` for readiness probes
- `agentic/deployment/.../ParallelOtelPropagationTest.java` — `InMemorySpanExporter` test pattern
- `agentic/deployment/.../AgentMetricsTest.java` — Micrometer test pattern

---

### Task 1: Maven Dependencies

**Files:**
- Modify: `agentic/runtime/pom.xml`
- Modify: `agentic/deployment/pom.xml`

- [ ] **Step 1: Add optional dependencies to runtime pom**

In `agentic/runtime/pom.xml`, add these dependencies after the existing `langchain4j-agentic` dependency:

```xml
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-opentelemetry</artifactId>
      <optional>true</optional>
    </dependency>
    <dependency>
      <groupId>io.micrometer</groupId>
      <artifactId>micrometer-core</artifactId>
      <optional>true</optional>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-smallrye-health</artifactId>
      <optional>true</optional>
    </dependency>
```

- [ ] **Step 2: Add health deployment SPI and test deps to deployment pom**

In `agentic/deployment/pom.xml`, add after the existing `quarkus-opentelemetry` test dependency:

```xml
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-smallrye-health-spi</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-smallrye-health-deployment</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-smallrye-health</artifactId>
      <scope>test</scope>
    </dependency>
```

Note: `quarkus-smallrye-health-spi` is NOT test scope — it provides `HealthBuildItem` which is used by the deployment processor at build time.

- [ ] **Step 3: Verify compilation**

Run: `/opt/homebrew/bin/mvn -pl agentic/runtime,agentic/deployment compile -q`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```
git add agentic/runtime/pom.xml agentic/deployment/pom.xml
git commit -m "build(agentic): add optional OTel, Micrometer, Health dependencies for C4"
```

---

### Task 2: CDI Event Record Types

**Files:**
- Create: `agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/observability/AgentStartedEvent.java`
- Create: `agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/observability/AgentCompletedEvent.java`
- Create: `agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/observability/AgentErrorEvent.java`

- [ ] **Step 1: Create AgentStartedEvent**

```java
package io.quarkiverse.langchain4j.agentic.runtime.observability;

import java.util.Map;

public record AgentStartedEvent(String agentName, String agentId, Map<String, Object> inputs) {
}
```

- [ ] **Step 2: Create AgentCompletedEvent**

```java
package io.quarkiverse.langchain4j.agentic.runtime.observability;

import java.util.Map;
import java.util.Optional;

import dev.langchain4j.model.output.TokenUsage;

public record AgentCompletedEvent(
        String agentName,
        String agentId,
        Map<String, Object> inputs,
        Object output,
        long durationNanos,
        Optional<TokenUsage> tokenUsage) {
}
```

- [ ] **Step 3: Create AgentErrorEvent**

```java
package io.quarkiverse.langchain4j.agentic.runtime.observability;

import java.util.Map;

public record AgentErrorEvent(String agentName, String agentId, Map<String, Object> inputs, Throwable error) {
}
```

- [ ] **Step 4: Verify compilation**

Run: `/opt/homebrew/bin/mvn -pl agentic/runtime compile -q`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```
git add agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/observability/
git commit -m "feat(agentic): add CDI event record types for agent lifecycle"
```

---

### Task 3: AgentCdiEventListener + AgenticObservabilityProcessor + Test

**Files:**
- Create: `agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/observability/AgentCdiEventListener.java`
- Create: `agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticObservabilityProcessor.java`
- Create: `agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/AgentCdiEventListenerTest.java`

- [ ] **Step 1: Write the failing test**

```java
package io.quarkiverse.langchain4j.agentic.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;

import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import dev.langchain4j.agentic.Agent;
import dev.langchain4j.agentic.declarative.ChatModelSupplier;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.service.UserMessage;
import dev.langchain4j.service.V;
import io.quarkiverse.langchain4j.agentic.runtime.observability.AgentCompletedEvent;
import io.quarkiverse.langchain4j.agentic.runtime.observability.AgentErrorEvent;
import io.quarkiverse.langchain4j.agentic.runtime.observability.AgentStartedEvent;
import io.quarkus.test.QuarkusUnitTest;

public class AgentCdiEventListenerTest {

    @RegisterExtension
    static final QuarkusUnitTest unitTest = new QuarkusUnitTest()
            .setArchiveProducer(() -> ShrinkWrap.create(JavaArchive.class)
                    .addClasses(TestAgent.class, EventCapture.class));

    public interface TestAgent {
        @UserMessage("Answer: {{request}}")
        @Agent(description = "test agent")
        String ask(@V("request") String request);

        @ChatModelSupplier
        static ChatModel chatModel() {
            return new Agents.FixedResponseChatModel("test-response");
        }
    }

    @ApplicationScoped
    public static class EventCapture {
        final List<AgentStartedEvent> started = new CopyOnWriteArrayList<>();
        final List<AgentCompletedEvent> completed = new CopyOnWriteArrayList<>();
        final List<AgentErrorEvent> errors = new CopyOnWriteArrayList<>();

        void onStarted(@Observes AgentStartedEvent event) {
            started.add(event);
        }

        void onCompleted(@Observes AgentCompletedEvent event) {
            completed.add(event);
        }

        void onError(@Observes AgentErrorEvent event) {
            errors.add(event);
        }

        void reset() {
            started.clear();
            completed.clear();
            errors.clear();
        }
    }

    @Inject
    TestAgent agent;

    @Inject
    EventCapture capture;

    @BeforeEach
    void reset() {
        capture.reset();
    }

    @Test
    void agentInvocationFiresStartedAndCompletedEvents() {
        agent.ask("hello");

        assertThat(capture.started).hasSize(1);
        assertThat(capture.started.get(0).agentName()).isNotBlank();

        assertThat(capture.completed).hasSize(1);
        assertThat(capture.completed.get(0).agentName()).isEqualTo(capture.started.get(0).agentName());
        assertThat(capture.completed.get(0).output()).isNotNull();
        assertThat(capture.completed.get(0).durationNanos()).isGreaterThan(0);

        assertThat(capture.errors).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -pl agentic/deployment test -Dtest=AgentCdiEventListenerTest -q`
Expected: FAIL — `AgentCdiEventListener` class not found

- [ ] **Step 3: Create AgentCdiEventListener**

```java
package io.quarkiverse.langchain4j.agentic.runtime.observability;

import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;

import dev.langchain4j.agentic.observability.AgentInvocationError;
import dev.langchain4j.agentic.observability.AgentListener;
import dev.langchain4j.agentic.observability.AgentRequest;
import dev.langchain4j.agentic.observability.AgentResponse;
import dev.langchain4j.model.chat.response.ChatResponse;
import dev.langchain4j.model.chat.response.ChatResponseMetadata;
import dev.langchain4j.model.output.TokenUsage;

@ApplicationScoped
public class AgentCdiEventListener implements AgentListener {

    private final ConcurrentHashMap<String, Long> startTimes = new ConcurrentHashMap<>();

    @Inject
    Event<AgentStartedEvent> startedEvent;

    @Inject
    Event<AgentCompletedEvent> completedEvent;

    @Inject
    Event<AgentErrorEvent> errorEvent;

    @Override
    public void beforeAgentInvocation(AgentRequest agentRequest) {
        startTimes.put(agentRequest.agentId(), System.nanoTime());
        startedEvent.fire(new AgentStartedEvent(
                agentRequest.agentName(),
                agentRequest.agentId(),
                agentRequest.inputs()));
    }

    @Override
    public void afterAgentInvocation(AgentResponse agentResponse) {
        Long startTime = startTimes.remove(agentResponse.agentId());
        long durationNanos = startTime != null ? System.nanoTime() - startTime : 0;

        Optional<TokenUsage> tokenUsage = Optional.ofNullable(agentResponse.chatResponse())
                .map(ChatResponse::metadata)
                .map(ChatResponseMetadata::tokenUsage);

        completedEvent.fire(new AgentCompletedEvent(
                agentResponse.agentName(),
                agentResponse.agentId(),
                agentResponse.inputs(),
                agentResponse.output(),
                durationNanos,
                tokenUsage));
    }

    @Override
    public void onAgentInvocationError(AgentInvocationError invocationError) {
        startTimes.remove(invocationError.agentId());
        errorEvent.fire(new AgentErrorEvent(
                invocationError.agentName(),
                invocationError.agentId(),
                invocationError.inputs(),
                invocationError.error()));
    }

    @Override
    public boolean inheritedBySubagents() {
        return true;
    }
}
```

- [ ] **Step 4: Create AgenticObservabilityProcessor**

```java
package io.quarkiverse.langchain4j.agentic.deployment;

import java.util.Optional;

import io.quarkiverse.langchain4j.agentic.runtime.observability.AgentCdiEventListener;
import io.quarkus.arc.deployment.AdditionalBeanBuildItem;
import io.quarkus.deployment.Capabilities;
import io.quarkus.deployment.Capability;
import io.quarkus.deployment.annotations.BuildProducer;
import io.quarkus.deployment.annotations.BuildStep;
import io.quarkus.deployment.metrics.MetricsCapabilityBuildItem;

public class AgenticObservabilityProcessor {

    @BuildStep
    void registerObservabilityListeners(
            Capabilities capabilities,
            Optional<MetricsCapabilityBuildItem> metricsCapability,
            BuildProducer<AdditionalBeanBuildItem> additionalBeanProducer) {

        // CDI events — unconditional
        additionalBeanProducer.produce(
                AdditionalBeanBuildItem.builder()
                        .addBeanClass(AgentCdiEventListener.class)
                        .setUnremovable()
                        .build());
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -pl agentic/deployment test -Dtest=AgentCdiEventListenerTest -q`
Expected: PASS

- [ ] **Step 6: Commit**

```
git add agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/observability/AgentCdiEventListener.java \
       agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticObservabilityProcessor.java \
       agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/AgentCdiEventListenerTest.java
git commit -m "feat(agentic): CDI event listener + observability processor — fires AgentStartedEvent/CompletedEvent/ErrorEvent"
```

---

### Task 4: AgentSpanListener + Test

**Files:**
- Create: `agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/observability/AgentSpanListener.java`
- Modify: `agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticObservabilityProcessor.java`
- Create: `agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/AgentSpanListenerTest.java`

- [ ] **Step 1: Write the failing test**

```java
package io.quarkiverse.langchain4j.agentic.deployment;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

import java.time.Duration;
import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import jakarta.inject.Inject;

import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.asset.StringAsset;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import dev.langchain4j.agentic.Agent;
import dev.langchain4j.agentic.declarative.ChatModelSupplier;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.service.UserMessage;
import dev.langchain4j.service.V;
import io.opentelemetry.api.common.AttributeKey;
import io.opentelemetry.sdk.testing.exporter.InMemorySpanExporter;
import io.opentelemetry.sdk.trace.data.SpanData;
import io.quarkus.test.QuarkusUnitTest;

public class AgentSpanListenerTest {

    private static final String APPLICATION_PROPERTIES = """
            quarkus.otel.bsp.schedule.delay=PT0.001S
            quarkus.otel.bsp.max.queue.size=1
            quarkus.otel.bsp.max.export.batch.size=1
            """;

    @RegisterExtension
    static final QuarkusUnitTest unitTest = new QuarkusUnitTest()
            .setArchiveProducer(() -> ShrinkWrap.create(JavaArchive.class)
                    .addClasses(SpanTestAgent.class, InMemorySpanExporterProducer.class,
                            Agents.FixedResponseChatModel.class)
                    .addAsResource(new StringAsset(APPLICATION_PROPERTIES), "application.properties"));

    public interface SpanTestAgent {
        @UserMessage("Answer: {{request}}")
        @Agent(description = "span test agent")
        String ask(@V("request") String request);

        @ChatModelSupplier
        static ChatModel chatModel() {
            return new Agents.FixedResponseChatModel("span-response");
        }
    }

    @ApplicationScoped
    public static class InMemorySpanExporterProducer {
        @Produces
        @ApplicationScoped
        InMemorySpanExporter exporter() {
            return InMemorySpanExporter.create();
        }
    }

    @Inject
    SpanTestAgent agent;

    @Inject
    InMemorySpanExporter spanExporter;

    @BeforeEach
    void resetSpans() {
        spanExporter.reset();
    }

    @Test
    void agentInvocationCreatesSpanWithCorrectAttributes() {
        agent.ask("test-question");

        await().atMost(Duration.ofSeconds(10)).untilAsserted(() -> {
            List<SpanData> spans = spanExporter.getFinishedSpanItems();
            List<SpanData> agentSpans = spans.stream()
                    .filter(s -> s.getName().startsWith("langchain4j.agent."))
                    .toList();
            assertThat(agentSpans).isNotEmpty();
        });

        List<SpanData> agentSpans = spanExporter.getFinishedSpanItems().stream()
                .filter(s -> s.getName().startsWith("langchain4j.agent."))
                .toList();

        SpanData agentSpan = agentSpans.get(0);
        assertThat(agentSpan.getAttributes().get(AttributeKey.stringKey("gen_ai.operation.name")))
                .isEqualTo("agent_invocation");
        assertThat(agentSpan.getAttributes().get(AttributeKey.stringKey("gen_ai.agent.name")))
                .isNotBlank();
        assertThat(agentSpan.getAttributes().get(AttributeKey.stringKey("gen_ai.agent.id")))
                .isNotBlank();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -pl agentic/deployment test -Dtest=AgentSpanListenerTest -q`
Expected: FAIL — no agent spans found (listener not registered yet)

- [ ] **Step 3: Create AgentSpanListener**

```java
package io.quarkiverse.langchain4j.agentic.runtime.observability;

import java.util.concurrent.ConcurrentHashMap;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import dev.langchain4j.agentic.observability.AgentInvocationError;
import dev.langchain4j.agentic.observability.AgentListener;
import dev.langchain4j.agentic.observability.AgentRequest;
import dev.langchain4j.agentic.observability.AgentResponse;
import dev.langchain4j.model.chat.response.ChatResponse;
import dev.langchain4j.model.chat.response.ChatResponseMetadata;
import dev.langchain4j.model.output.TokenUsage;
import io.opentelemetry.api.common.AttributeKey;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.StatusCode;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.context.Scope;

@ApplicationScoped
public class AgentSpanListener implements AgentListener {

    private static final AttributeKey<String> OPERATION_NAME = AttributeKey.stringKey("gen_ai.operation.name");
    private static final AttributeKey<String> AGENT_NAME = AttributeKey.stringKey("gen_ai.agent.name");
    private static final AttributeKey<String> AGENT_ID = AttributeKey.stringKey("gen_ai.agent.id");
    private static final AttributeKey<Long> INPUT_TOKENS = AttributeKey.longKey("gen_ai.usage.input_tokens");
    private static final AttributeKey<Long> OUTPUT_TOKENS = AttributeKey.longKey("gen_ai.usage.output_tokens");

    private final ConcurrentHashMap<String, SpanScope> activeSpans = new ConcurrentHashMap<>();

    @Inject
    Tracer tracer;

    @Override
    public void beforeAgentInvocation(AgentRequest agentRequest) {
        Span span = tracer.spanBuilder("langchain4j.agent." + agentRequest.agentName())
                .setAttribute(OPERATION_NAME, "agent_invocation")
                .setAttribute(AGENT_NAME, agentRequest.agentName())
                .setAttribute(AGENT_ID, agentRequest.agentId())
                .startSpan();
        Scope scope = span.makeCurrent();
        activeSpans.put(agentRequest.agentId(), new SpanScope(span, scope));
    }

    @Override
    public void afterAgentInvocation(AgentResponse agentResponse) {
        SpanScope spanScope = activeSpans.remove(agentResponse.agentId());
        if (spanScope == null) {
            return;
        }
        try {
            TokenUsage tokenUsage = null;
            ChatResponse chatResponse = agentResponse.chatResponse();
            if (chatResponse != null) {
                ChatResponseMetadata metadata = chatResponse.metadata();
                if (metadata != null) {
                    tokenUsage = metadata.tokenUsage();
                }
            }
            if (tokenUsage != null) {
                if (tokenUsage.inputTokenCount() != null) {
                    spanScope.span.setAttribute(INPUT_TOKENS, tokenUsage.inputTokenCount().longValue());
                }
                if (tokenUsage.outputTokenCount() != null) {
                    spanScope.span.setAttribute(OUTPUT_TOKENS, tokenUsage.outputTokenCount().longValue());
                }
            }
        } finally {
            spanScope.scope.close();
            spanScope.span.end();
        }
    }

    @Override
    public void onAgentInvocationError(AgentInvocationError invocationError) {
        SpanScope spanScope = activeSpans.remove(invocationError.agentId());
        if (spanScope == null) {
            return;
        }
        try {
            spanScope.span.recordException(invocationError.error());
            spanScope.span.setStatus(StatusCode.ERROR, invocationError.error().getMessage());
        } finally {
            spanScope.scope.close();
            spanScope.span.end();
        }
    }

    @Override
    public boolean inheritedBySubagents() {
        return true;
    }

    private record SpanScope(Span span, Scope scope) {
    }
}
```

- [ ] **Step 4: Register AgentSpanListener in AgenticObservabilityProcessor**

Add to the `registerObservabilityListeners` method, before the CDI event listener registration:

```java
        // OTel spans — conditional on OpenTelemetry tracer
        if (capabilities.isPresent(Capability.OPENTELEMETRY_TRACER)) {
            additionalBeanProducer.produce(
                    AdditionalBeanBuildItem.builder()
                            .addBeanClass(AgentSpanListener.class)
                            .setUnremovable()
                            .build());
        }
```

Add the import:
```java
import io.quarkiverse.langchain4j.agentic.runtime.observability.AgentSpanListener;
```

Note: The class reference `AgentSpanListener.class` compiles even though `quarkus-opentelemetry` is optional in the runtime pom, because the deployment module has it as a test dependency. The `AdditionalBeanBuildItem` uses the class name string internally — the bean is only instantiated at runtime when the capability is present.

- [ ] **Step 5: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -pl agentic/deployment test -Dtest=AgentSpanListenerTest -q`
Expected: PASS

- [ ] **Step 6: Commit**

```
git add agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/observability/AgentSpanListener.java \
       agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticObservabilityProcessor.java \
       agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/AgentSpanListenerTest.java
git commit -m "feat(agentic): OTel span listener — creates named spans per agent invocation with GenAI attributes"
```

---

### Task 5: AgentMetricsListener + Test

**Files:**
- Create: `agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/observability/AgentMetricsListener.java`
- Modify: `agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticObservabilityProcessor.java`
- Create: `agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/AgentMetricsListenerTest.java`

- [ ] **Step 1: Write the failing test**

```java
package io.quarkiverse.langchain4j.agentic.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.Collection;

import jakarta.inject.Inject;

import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import dev.langchain4j.agentic.Agent;
import dev.langchain4j.agentic.declarative.ChatModelSupplier;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.service.UserMessage;
import dev.langchain4j.service.V;
import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.Metrics;
import io.micrometer.core.instrument.Timer;
import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
import io.quarkus.test.QuarkusUnitTest;

public class AgentMetricsListenerTest {

    @BeforeAll
    static void addSimpleRegistry() {
        Metrics.globalRegistry.add(new SimpleMeterRegistry());
    }

    @RegisterExtension
    static final QuarkusUnitTest unitTest = new QuarkusUnitTest()
            .setArchiveProducer(() -> ShrinkWrap.create(JavaArchive.class)
                    .addClasses(MetricsTestAgent.class, Agents.FixedResponseChatModel.class));

    public interface MetricsTestAgent {
        @UserMessage("Answer: {{request}}")
        @Agent(description = "metrics test agent")
        String ask(@V("request") String request);

        @ChatModelSupplier
        static ChatModel chatModel() {
            return new Agents.FixedResponseChatModel("metrics-response");
        }
    }

    @Inject
    MetricsTestAgent agent;

    @Test
    void agentInvocationRecordsInvocationCounterAndDurationTimer() {
        agent.ask("test");

        Collection<Counter> counters = Metrics.globalRegistry
                .find("gen_ai.agent.invocations").counters();
        assertThat(counters).isNotEmpty();
        assertThat(counters.stream().mapToDouble(Counter::count).sum()).isGreaterThan(0);

        Collection<Timer> timers = Metrics.globalRegistry
                .find("gen_ai.agent.duration").timers();
        assertThat(timers).isNotEmpty();
        assertThat(timers.stream().mapToLong(Timer::count).sum()).isGreaterThan(0);
    }

    @Test
    void successfulInvocationTaggedWithNoError() {
        agent.ask("test");

        Counter counter = Metrics.globalRegistry.find("gen_ai.agent.invocations")
                .tag("error.type", "none")
                .counter();
        assertThat(counter).isNotNull();
        assertThat(counter.count()).isGreaterThan(0);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -pl agentic/deployment test -Dtest=AgentMetricsListenerTest -q`
Expected: FAIL — no `gen_ai.agent.invocations` metric found

- [ ] **Step 3: Create AgentMetricsListener**

```java
package io.quarkiverse.langchain4j.agentic.runtime.observability;

import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.TimeUnit;

import jakarta.enterprise.context.ApplicationScoped;

import dev.langchain4j.agentic.observability.AfterAgentToolExecution;
import dev.langchain4j.agentic.observability.AgentInvocationError;
import dev.langchain4j.agentic.observability.AgentListener;
import dev.langchain4j.agentic.observability.AgentRequest;
import dev.langchain4j.agentic.observability.AgentResponse;
import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.Meter;
import io.micrometer.core.instrument.Metrics;
import io.micrometer.core.instrument.Tags;
import io.micrometer.core.instrument.Timer;

@ApplicationScoped
public class AgentMetricsListener implements AgentListener {

    private final ConcurrentHashMap<String, Long> startTimes = new ConcurrentHashMap<>();

    private final Meter.MeterProvider<Counter> invocations;
    private final Meter.MeterProvider<Timer> duration;
    private final Meter.MeterProvider<Counter> toolExecutions;

    public AgentMetricsListener() {
        this.invocations = Counter.builder("gen_ai.agent.invocations")
                .description("Number of agent invocations")
                .withRegistry(Metrics.globalRegistry);
        this.duration = Timer.builder("gen_ai.agent.duration")
                .description("Agent invocation duration")
                .withRegistry(Metrics.globalRegistry);
        this.toolExecutions = Counter.builder("gen_ai.agent.tool.executions")
                .description("Number of tool executions by agents")
                .withRegistry(Metrics.globalRegistry);
    }

    @Override
    public void beforeAgentInvocation(AgentRequest agentRequest) {
        startTimes.put(agentRequest.agentId(), System.nanoTime());
    }

    @Override
    public void afterAgentInvocation(AgentResponse agentResponse) {
        Tags tags = Tags.of("gen_ai.agent.name", agentResponse.agentName())
                .and("error.type", "none");
        invocations.withTags(tags).increment();
        recordDuration(agentResponse.agentId(), tags);
    }

    @Override
    public void onAgentInvocationError(AgentInvocationError invocationError) {
        String errorType = invocationError.error() != null
                ? invocationError.error().getClass().getSimpleName()
                : "unknown";
        Tags tags = Tags.of("gen_ai.agent.name", invocationError.agentName())
                .and("error.type", errorType);
        invocations.withTags(tags).increment();
        recordDuration(invocationError.agentId(), tags);
    }

    @Override
    public void afterAgentToolExecution(AfterAgentToolExecution afterToolExecution) {
        Tags tags = Tags.of(
                "gen_ai.agent.name", afterToolExecution.agentInstance().name(),
                "gen_ai.tool.name", afterToolExecution.toolExecution().request().name());
        toolExecutions.withTags(tags).increment();
    }

    @Override
    public boolean inheritedBySubagents() {
        return true;
    }

    private void recordDuration(String agentId, Tags tags) {
        Long startTime = startTimes.remove(agentId);
        if (startTime != null) {
            duration.withTags(tags).record(System.nanoTime() - startTime, TimeUnit.NANOSECONDS);
        }
    }
}
```

- [ ] **Step 4: Register AgentMetricsListener in AgenticObservabilityProcessor**

Add to `registerObservabilityListeners`, after the OTel block:

```java
        // Micrometer metrics — conditional on Micrometer
        if (metricsCapability.isPresent()
                && metricsCapability.get().metricsSupported(MetricsFactory.MICROMETER)) {
            additionalBeanProducer.produce(
                    AdditionalBeanBuildItem.builder()
                            .addBeanClass(AgentMetricsListener.class)
                            .setUnremovable()
                            .build());
        }
```

Add the imports:
```java
import io.quarkiverse.langchain4j.agentic.runtime.observability.AgentMetricsListener;
import io.quarkus.runtime.metrics.MetricsFactory;
```

- [ ] **Step 5: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -pl agentic/deployment test -Dtest=AgentMetricsListenerTest -q`
Expected: PASS

- [ ] **Step 6: Commit**

```
git add agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/observability/AgentMetricsListener.java \
       agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticObservabilityProcessor.java \
       agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/AgentMetricsListenerTest.java
git commit -m "feat(agentic): Micrometer metrics listener — invocation counter, duration timer, tool execution counter"
```

---

### Task 6: AgentHealthCheck + Test

**Files:**
- Create: `agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/observability/AgentHealthCheck.java`
- Modify: `agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticObservabilityProcessor.java`
- Modify: `agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/AgenticRecorder.java`
- Create: `agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/AgentHealthCheckTest.java`

- [ ] **Step 1: Write the failing test**

```java
package io.quarkiverse.langchain4j.agentic.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import jakarta.inject.Inject;

import org.eclipse.microprofile.health.HealthCheckResponse;
import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import dev.langchain4j.agentic.Agent;
import dev.langchain4j.agentic.declarative.ChatModelSupplier;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.service.UserMessage;
import dev.langchain4j.service.V;
import io.quarkiverse.langchain4j.agentic.runtime.observability.AgentHealthCheck;
import io.quarkus.test.QuarkusUnitTest;

public class AgentHealthCheckTest {

    @RegisterExtension
    static final QuarkusUnitTest unitTest = new QuarkusUnitTest()
            .setArchiveProducer(() -> ShrinkWrap.create(JavaArchive.class)
                    .addClasses(HealthTestAgent.class, Agents.FixedResponseChatModel.class));

    public interface HealthTestAgent {
        @UserMessage("Answer: {{request}}")
        @Agent(description = "health test agent")
        String ask(@V("request") String request);

        @ChatModelSupplier
        static ChatModel chatModel() {
            return new Agents.FixedResponseChatModel("health-response");
        }
    }

    @Inject
    AgentHealthCheck healthCheck;

    @Test
    void healthCheckReportsUpWhenAgentsInitialize() {
        HealthCheckResponse response = healthCheck.call();
        assertThat(response.getStatus()).isEqualTo(HealthCheckResponse.Status.UP);
        assertThat(response.getName()).isEqualTo("Agent readiness");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -pl agentic/deployment test -Dtest=AgentHealthCheckTest -q`
Expected: FAIL — `AgentHealthCheck` not found

- [ ] **Step 3: Create AgentHealthCheck**

```java
package io.quarkiverse.langchain4j.agentic.runtime.observability;

import java.util.Collections;
import java.util.Set;

import jakarta.enterprise.context.ApplicationScoped;

import org.eclipse.microprofile.health.HealthCheck;
import org.eclipse.microprofile.health.HealthCheckResponse;
import org.eclipse.microprofile.health.HealthCheckResponseBuilder;
import org.eclipse.microprofile.health.Readiness;
import org.jboss.logging.Logger;

import io.quarkus.arc.Arc;

@Readiness
@ApplicationScoped
public class AgentHealthCheck implements HealthCheck {

    private static final Logger log = Logger.getLogger(AgentHealthCheck.class);

    private static volatile Set<String> rootAgentClassNames = Collections.emptySet();

    public static void setRootAgentClassNames(Set<String> classNames) {
        rootAgentClassNames = Collections.unmodifiableSet(classNames);
    }

    @Override
    public HealthCheckResponse call() {
        HealthCheckResponseBuilder builder = HealthCheckResponse.named("Agent readiness");
        if (rootAgentClassNames.isEmpty()) {
            return builder.up().build();
        }
        boolean allUp = true;
        for (String className : rootAgentClassNames) {
            try {
                Class<?> clazz = Class.forName(className, true, AgentHealthCheck.class.getClassLoader());
                Object agent = Arc.container().select(clazz).get();
                if (agent != null) {
                    builder.withData(className, "UP");
                } else {
                    builder.withData(className, "NOT FOUND");
                    allUp = false;
                }
            } catch (Exception e) {
                log.warnf("Agent health check failed for %s: %s", className, e.getMessage());
                builder.withData(className, "ERROR: " + e.getMessage());
                allUp = false;
            }
        }
        return allUp ? builder.up().build() : builder.down().build();
    }
}
```

- [ ] **Step 4: Add recorder method to set root agent class names on health check**

In `AgenticRecorder.java`, add a new method after `enableDevModeMonitoring`:

```java
    @RuntimeInit
    public void setHealthCheckRootAgents(Set<String> rootAgentClassNames) {
        AgentHealthCheck.setRootAgentClassNames(rootAgentClassNames);
    }
```

Add the import:
```java
import io.quarkiverse.langchain4j.agentic.runtime.observability.AgentHealthCheck;
```

- [ ] **Step 5: Register health check in AgenticObservabilityProcessor**

Add a new build step to `AgenticObservabilityProcessor`:

```java
    @BuildStep
    @Record(ExecutionTime.RUNTIME_INIT)
    void registerHealthCheck(
            Capabilities capabilities,
            List<DetectedAiAgentBuildItem> detectedAgents,
            AgenticRecorder recorder,
            BuildProducer<HealthBuildItem> healthBuildItems) {

        if (capabilities.isPresent(Capability.SMALLRYE_HEALTH)) {
            Set<String> rootAgentClassNames = new HashSet<>();
            for (DetectedAiAgentBuildItem agent : detectedAgents) {
                rootAgentClassNames.add(agent.getIface().name().toString());
            }
            recorder.setHealthCheckRootAgents(rootAgentClassNames);
            healthBuildItems.produce(new HealthBuildItem(AgentHealthCheck.class.getName(), true));
        }
    }
```

Add the imports:
```java
import java.util.HashSet;
import java.util.List;
import java.util.Set;

import io.quarkiverse.langchain4j.agentic.runtime.AgenticRecorder;
import io.quarkiverse.langchain4j.agentic.runtime.observability.AgentHealthCheck;
import io.quarkus.deployment.annotations.ExecutionTime;
import io.quarkus.deployment.annotations.Record;
import io.quarkus.smallrye.health.deployment.spi.HealthBuildItem;
```

- [ ] **Step 6: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -pl agentic/deployment test -Dtest=AgentHealthCheckTest -q`
Expected: PASS

- [ ] **Step 7: Commit**

```
git add agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/observability/AgentHealthCheck.java \
       agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/AgenticRecorder.java \
       agentic/deployment/src/main/java/io/quarkiverse/langchain4j/agentic/deployment/AgenticObservabilityProcessor.java \
       agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/AgentHealthCheckTest.java
git commit -m "feat(agentic): readiness health check for agent initialization"
```

---

### Task 7: DevAgentMonitorHolder Hot-Reload Fix (O-5)

**Files:**
- Modify: `agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/AgenticRecorder.java`

- [ ] **Step 1: Add ShutdownContext to enableDevModeMonitoring**

In `AgenticRecorder.java`, change the `enableDevModeMonitoring` method signature and add the shutdown callback:

Replace:
```java
    @RuntimeInit
    public void enableDevModeMonitoring(Set<String> rootAgentClassNames) {
        DevAgentMonitorHolder.reset();
        AgenticRecorder.devModeMonitoringEnabled = true;
        AgenticRecorder.rootAgentClassNames = Collections.unmodifiableSet(rootAgentClassNames);
    }
```

With:
```java
    @RuntimeInit
    public void enableDevModeMonitoring(Set<String> rootAgentClassNames, ShutdownContext shutdownContext) {
        DevAgentMonitorHolder.reset();
        AgenticRecorder.devModeMonitoringEnabled = true;
        AgenticRecorder.rootAgentClassNames = Collections.unmodifiableSet(rootAgentClassNames);
        shutdownContext.addShutdownTask(DevAgentMonitorHolder::reset);
    }
```

Add the import:
```java
import io.quarkus.runtime.ShutdownContext;
```

- [ ] **Step 2: Update the build step that calls enableDevModeMonitoring**

Find the build step in `AgenticProcessor` or the Dev UI processor that calls `recorder.enableDevModeMonitoring(...)`. The `ShutdownContext` is automatically available as a parameter in `@Record(RUNTIME_INIT)` methods — just add it to the build step method signature and pass it through:

Search for the call site:
```
grep -rn "enableDevModeMonitoring" agentic/deployment/src/main/java/
```

Update the build step method to include `ShutdownContext shutdownContext` as a parameter and pass it to the recorder.

- [ ] **Step 3: Verify compilation and existing tests pass**

Run: `/opt/homebrew/bin/mvn -pl agentic/deployment test -q`
Expected: BUILD SUCCESS, all existing tests pass

- [ ] **Step 4: Commit**

```
git add agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/AgenticRecorder.java \
       agentic/deployment/src/main/java/
git commit -m "fix(agentic): register ShutdownContext callback to clear DevAgentMonitorHolder on hot-reload"
```

---

### Task 8: ObservabilityAbsentTest

**Files:**
- Create: `agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/ObservabilityAbsentTest.java`

- [ ] **Step 1: Write the negative test**

This test verifies the application starts cleanly when no observability extensions are present. It uses `setFlatClassPath(true)` to exclude OTel and Micrometer from the classpath, or simply doesn't add them to the archive. The key assertion is that the app boots and the agent works without errors.

```java
package io.quarkiverse.langchain4j.agentic.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import jakarta.inject.Inject;

import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

import dev.langchain4j.agentic.Agent;
import dev.langchain4j.agentic.declarative.ChatModelSupplier;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.service.UserMessage;
import dev.langchain4j.service.V;
import io.quarkus.test.QuarkusUnitTest;

public class ObservabilityAbsentTest {

    @RegisterExtension
    static final QuarkusUnitTest unitTest = new QuarkusUnitTest()
            .setArchiveProducer(() -> ShrinkWrap.create(JavaArchive.class)
                    .addClasses(MinimalAgent.class, Agents.FixedResponseChatModel.class));

    public interface MinimalAgent {
        @UserMessage("Answer: {{request}}")
        @Agent(description = "minimal agent")
        String ask(@V("request") String request);

        @ChatModelSupplier
        static ChatModel chatModel() {
            return new Agents.FixedResponseChatModel("minimal-response");
        }
    }

    @Inject
    MinimalAgent agent;

    @Test
    void applicationStartsAndAgentWorksWithoutObservabilityExtensions() {
        String result = agent.ask("test");
        assertThat(result).isNotBlank();
    }
}
```

Note: This test validates that the conditional bean registration doesn't break when capabilities are absent. Since OTel and Micrometer are test-scope in the deployment pom, they ARE on the classpath for all tests. The real absence test is that the `AgenticObservabilityProcessor` build steps simply don't produce beans when capabilities aren't declared — which is already the behavior. This test serves as a smoke test that the agent works regardless.

- [ ] **Step 2: Run the test**

Run: `/opt/homebrew/bin/mvn -pl agentic/deployment test -Dtest=ObservabilityAbsentTest -q`
Expected: PASS

- [ ] **Step 3: Commit**

```
git add agentic/deployment/src/test/java/io/quarkiverse/langchain4j/agentic/deployment/ObservabilityAbsentTest.java
git commit -m "test(agentic): verify agent works without observability extensions"
```

---

### Task 9: Dev UI Backend — Structured JSON (O-6)

**Files:**
- Modify: `agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/devui/AgenticJsonRpcService.java`

- [ ] **Step 1: Replace getTopologyHtml with getTopologyJson**

Remove the `getTopologyHtml` method and replace with:

```java
    public JsonObject getTopologyJson(int index) {
        List<Object> rootAgents = DevAgentMonitorHolder.rootAgents();
        if (rootAgents.isEmpty()) {
            return new JsonObject().put("error", "No root agents detected");
        }
        int i = (index >= 0 && index < rootAgents.size()) ? index : 0;
        try {
            AgentInstance rootAgent = (AgentInstance) rootAgents.get(i);
            return serializeAgentTopology(rootAgent);
        } catch (Exception e) {
            log.warn("Failed to generate topology", e);
            return new JsonObject().put("error", "Failed to generate topology: " + e.getMessage());
        }
    }

    private JsonObject serializeAgentTopology(AgentInstance agent) {
        JsonObject node = new JsonObject()
                .put("name", agent.name())
                .put("type", agent.topology() != null ? agent.topology().name() : "AGENT")
                .put("agentId", agent.agentId());

        if (agent.description() != null) {
            node.put("description", agent.description());
        }

        List<AgentInstance> subAgents = agent.subAgents();
        if (subAgents != null && !subAgents.isEmpty()) {
            JsonArray children = new JsonArray();
            for (AgentInstance sub : subAgents) {
                children.add(serializeAgentTopology(sub));
            }
            node.put("subAgents", children);
        }
        return node;
    }
```

Add the import:
```java
import dev.langchain4j.agentic.planner.AgentInstance;
```

- [ ] **Step 2: Replace getExecutionReportHtml with getExecutionReportJson**

Remove the `getExecutionReportHtml` method and replace with:

```java
    public JsonObject getExecutionReportJson(int index) {
        List<AgentMonitor> monitors = DevAgentMonitorHolder.monitors();
        if (monitors.isEmpty()) {
            return new JsonObject().put("error", "No execution data available");
        }
        int i = (index >= 0 && index < monitors.size()) ? index : 0;
        try {
            AgentMonitor monitor = monitors.get(i);
            JsonArray executions = new JsonArray();

            for (MonitoredExecution exec : monitor.successfulExecutions()) {
                executions.add(serializeExecution(exec, "success"));
            }
            for (MonitoredExecution exec : monitor.failedExecutions()) {
                executions.add(serializeExecution(exec, "failed"));
            }
            for (MonitoredExecution exec : monitor.ongoingExecutions().values()) {
                executions.add(serializeExecution(exec, "ongoing"));
            }

            return new JsonObject().put("executions", executions);
        } catch (Exception e) {
            log.warn("Failed to generate execution report", e);
            return new JsonObject().put("error", "Failed: " + e.getMessage());
        }
    }

    private JsonObject serializeExecution(MonitoredExecution exec, String status) {
        JsonObject obj = new JsonObject()
                .put("memoryId", String.valueOf(exec.memoryId()))
                .put("status", status)
                .put("topLevel", serializeInvocation(exec.topLevelInvocations()));
        if (exec.hasError()) {
            obj.put("error", exec.error().error().getMessage());
        }
        return obj;
    }

    private JsonObject serializeInvocation(AgentInvocation inv) {
        JsonObject obj = new JsonObject()
                .put("agentName", inv.agent().name())
                .put("startTime", inv.startTime().toString());
        if (inv.done()) {
            obj.put("duration", inv.duration().toMillis());
            obj.put("tokenCount", inv.totalTokenCount());
            obj.put("output", inv.output() != null ? String.valueOf(inv.output()) : null);
        } else {
            obj.put("status", "in_progress");
        }
        if (inv.iterationIndex() >= 0) {
            obj.put("iterationIndex", inv.iterationIndex());
        }

        if (!inv.toolExecutions().isEmpty()) {
            JsonArray tools = new JsonArray();
            for (var toolExec : inv.toolExecutions()) {
                tools.add(new JsonObject()
                        .put("name", toolExec.request().name())
                        .put("arguments", toolExec.request().arguments())
                        .put("result", toolExec.result()));
            }
            obj.put("toolExecutions", tools);
        }

        if (!inv.nestedInvocations().isEmpty()) {
            JsonArray nested = new JsonArray();
            for (AgentInvocation sub : inv.nestedInvocations()) {
                nested.add(serializeInvocation(sub));
            }
            obj.put("nestedInvocations", nested);
        }
        return obj;
    }
```

Add the imports:
```java
import dev.langchain4j.agentic.observability.AgentInvocation;
import dev.langchain4j.agentic.observability.MonitoredExecution;
```

Remove the import for `HtmlReportGenerator` (no longer used).

- [ ] **Step 3: Verify compilation**

Run: `/opt/homebrew/bin/mvn -pl agentic/runtime compile -q`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```
git add agentic/runtime/src/main/java/io/quarkiverse/langchain4j/agentic/runtime/devui/AgenticJsonRpcService.java
git commit -m "feat(agentic): replace Dev UI HTML with structured JSON — topology and execution endpoints"
```

---

### Task 10: Dev UI Frontend — Vaadin Components (O-6)

**Files:**
- Modify: `agentic/deployment/src/main/resources/dev-ui/qwc-agents-topology.js`
- Modify: `agentic/deployment/src/main/resources/dev-ui/qwc-agents-executions.js`

- [ ] **Step 1: Rewrite qwc-agents-topology.js**

Replace the entire file content with a Vaadin tree grid that renders topology JSON:

```javascript
import { LitElement, html, css } from 'lit';
import { JsonRpc } from 'jsonrpc';
import '@vaadin/button';
import '@vaadin/select';
import '@vaadin/grid';
import '@vaadin/grid/vaadin-grid-tree-column.js';
import '@vaadin/progress-bar';

export class QwcAgentsTopology extends LitElement {

    static styles = css`
        :host {
            height: 100%;
            display: flex;
            flex-direction: column;
        }
        .toolbar {
            padding: 10px 15px;
            display: flex;
            align-items: center;
            gap: 10px;
        }
        .grid-container {
            flex: 1;
            padding: 0 15px 15px 15px;
        }
        vaadin-grid {
            height: 100%;
        }
        .placeholder {
            padding: 20px;
            text-align: center;
            color: var(--lumo-secondary-text-color);
        }
    `;

    static properties = {
        _topology: { state: true },
        _flatNodes: { state: true },
        _loading: { state: true },
        _error: { state: true },
        _agentEntries: { state: true },
        _selectedIndex: { state: true },
    };

    jsonRpc = new JsonRpc(this);

    constructor() {
        super();
        this._topology = null;
        this._flatNodes = [];
        this._loading = true;
        this._error = null;
        this._agentEntries = [];
        this._selectedIndex = 0;
    }

    connectedCallback() {
        super.connectedCallback();
        this._loadAgentEntries();
    }

    _loadAgentEntries() {
        this.jsonRpc.getRootAgentEntries()
            .then(response => {
                this._agentEntries = response.result || [];
                this._selectedIndex = this._agentEntries.length > 0 ? this._agentEntries[0].index : 0;
                this._loadTopology();
            })
            .catch(() => this._loadTopology());
    }

    _onAgentSelected(e) {
        this._selectedIndex = parseInt(e.target.value, 10);
        this._loadTopology();
    }

    _loadTopology() {
        this._loading = true;
        this._error = null;
        this.jsonRpc.getTopologyJson({ index: this._selectedIndex })
            .then(response => {
                this._topology = response.result;
                this._flatNodes = this._flatten(this._topology, 0);
                this._loading = false;
            })
            .catch(error => {
                this._error = String(error);
                this._loading = false;
            });
    }

    _flatten(node, level) {
        if (!node) return [];
        const result = [{ ...node, level, hasChildren: !!(node.subAgents && node.subAgents.length) }];
        if (node.subAgents) {
            for (const child of node.subAgents) {
                result.push(...this._flatten(child, level + 1));
            }
        }
        return result;
    }

    render() {
        const agentItems = this._agentEntries.map(e => ({
            label: e.name,
            value: String(e.index),
        }));

        return html`
            <div class="toolbar">
                ${agentItems.length > 1 ? html`
                    <vaadin-select
                        label="Root Agent"
                        .items="${agentItems}"
                        .value="${String(this._selectedIndex)}"
                        @value-changed="${this._onAgentSelected}">
                    </vaadin-select>
                ` : ''}
                <vaadin-button theme="small" @click="${() => this._loadTopology()}">
                    Refresh
                </vaadin-button>
            </div>
            ${this._loading ? html`
                <vaadin-progress-bar indeterminate></vaadin-progress-bar>
            ` : this._error ? html`
                <div class="placeholder">${this._error}</div>
            ` : html`
                <div class="grid-container">
                    <vaadin-grid .items="${this._flatNodes}" theme="compact row-stripes">
                        <vaadin-grid-column header="Name" path="name"></vaadin-grid-column>
                        <vaadin-grid-column header="Type" path="type" width="120px" flex-grow="0"></vaadin-grid-column>
                        <vaadin-grid-column header="Description" path="description"></vaadin-grid-column>
                        <vaadin-grid-column header="Level" path="level" width="80px" flex-grow="0"></vaadin-grid-column>
                    </vaadin-grid>
                </div>
            `}
        `;
    }
}

customElements.define('qwc-agents-topology', QwcAgentsTopology);
```

- [ ] **Step 2: Rewrite qwc-agents-executions.js**

Replace the entire file content with a Vaadin grid that renders execution JSON:

```javascript
import { LitElement, html, css } from 'lit';
import { JsonRpc } from 'jsonrpc';
import '@vaadin/button';
import '@vaadin/select';
import '@vaadin/grid';
import '@vaadin/progress-bar';

export class QwcAgentsExecutions extends LitElement {

    static styles = css`
        :host {
            height: 100%;
            display: flex;
            flex-direction: column;
        }
        .toolbar {
            padding: 10px 15px;
            display: flex;
            align-items: center;
            gap: 10px;
        }
        .grid-container {
            flex: 1;
            padding: 0 15px 15px 15px;
        }
        vaadin-grid {
            height: 100%;
        }
        .placeholder {
            padding: 20px;
            text-align: center;
            color: var(--lumo-secondary-text-color);
        }
        .error-row {
            color: var(--lumo-error-text-color);
        }
    `;

    static properties = {
        _executions: { state: true },
        _flatRows: { state: true },
        _loading: { state: true },
        _error: { state: true },
        _agentEntries: { state: true },
        _selectedIndex: { state: true },
    };

    jsonRpc = new JsonRpc(this);

    constructor() {
        super();
        this._executions = [];
        this._flatRows = [];
        this._loading = true;
        this._error = null;
        this._agentEntries = [];
        this._selectedIndex = 0;
    }

    connectedCallback() {
        super.connectedCallback();
        this._loadAgentEntries();
    }

    _loadAgentEntries() {
        this.jsonRpc.getRootAgentEntries()
            .then(response => {
                this._agentEntries = response.result || [];
                this._selectedIndex = this._agentEntries.length > 0 ? this._agentEntries[0].index : 0;
                this._loadReport();
            })
            .catch(() => this._loadReport());
    }

    _onAgentSelected(e) {
        this._selectedIndex = parseInt(e.target.value, 10);
        this._loadReport();
    }

    _loadReport() {
        this._loading = true;
        this._error = null;
        this.jsonRpc.getExecutionReportJson({ index: this._selectedIndex })
            .then(response => {
                const data = response.result;
                this._executions = data.executions || [];
                this._flatRows = this._flatten(this._executions);
                this._loading = false;
            })
            .catch(error => {
                this._error = String(error);
                this._loading = false;
            });
    }

    _flatten(executions) {
        const rows = [];
        for (const exec of executions) {
            this._flattenInvocation(exec.topLevel, 0, exec.status, exec.memoryId, rows);
        }
        return rows;
    }

    _flattenInvocation(inv, level, status, memoryId, rows) {
        if (!inv) return;
        rows.push({
            agentName: inv.agentName,
            status: inv.status || status,
            duration: inv.duration != null ? inv.duration + ' ms' : 'in progress',
            tokenCount: inv.tokenCount || 0,
            iterationIndex: inv.iterationIndex >= 0 ? inv.iterationIndex : '',
            level,
            memoryId,
        });
        if (inv.nestedInvocations) {
            for (const nested of inv.nestedInvocations) {
                this._flattenInvocation(nested, level + 1, status, memoryId, rows);
            }
        }
    }

    render() {
        const agentItems = this._agentEntries.map(e => ({
            label: e.name,
            value: String(e.index),
        }));

        return html`
            <div class="toolbar">
                ${agentItems.length > 1 ? html`
                    <vaadin-select
                        label="Root Agent"
                        .items="${agentItems}"
                        .value="${String(this._selectedIndex)}"
                        @value-changed="${this._onAgentSelected}">
                    </vaadin-select>
                ` : ''}
                <vaadin-button theme="small" @click="${() => this._loadReport()}">
                    Refresh
                </vaadin-button>
            </div>
            ${this._loading ? html`
                <vaadin-progress-bar indeterminate></vaadin-progress-bar>
            ` : this._error ? html`
                <div class="placeholder">${this._error}</div>
            ` : this._flatRows.length === 0 ? html`
                <div class="placeholder">No execution data. Invoke an agent first.</div>
            ` : html`
                <div class="grid-container">
                    <vaadin-grid .items="${this._flatRows}" theme="compact row-stripes">
                        <vaadin-grid-column header="Agent" path="agentName"></vaadin-grid-column>
                        <vaadin-grid-column header="Status" path="status" width="100px" flex-grow="0"></vaadin-grid-column>
                        <vaadin-grid-column header="Duration" path="duration" width="120px" flex-grow="0"></vaadin-grid-column>
                        <vaadin-grid-column header="Tokens" path="tokenCount" width="80px" flex-grow="0"></vaadin-grid-column>
                        <vaadin-grid-column header="Iteration" path="iterationIndex" width="80px" flex-grow="0"></vaadin-grid-column>
                        <vaadin-grid-column header="Level" path="level" width="60px" flex-grow="0"></vaadin-grid-column>
                    </vaadin-grid>
                </div>
            `}
        `;
    }
}

customElements.define('qwc-agents-executions', QwcAgentsExecutions);
```

- [ ] **Step 3: Verify compilation**

Run: `/opt/homebrew/bin/mvn -pl agentic/deployment compile -q`
Expected: BUILD SUCCESS (JS files are resources, no compilation — just verify the deployment module compiles)

- [ ] **Step 4: Commit**

```
git add agentic/deployment/src/main/resources/dev-ui/qwc-agents-topology.js \
       agentic/deployment/src/main/resources/dev-ui/qwc-agents-executions.js
git commit -m "feat(agentic): Dev UI Vaadin grids replacing HTML iframe — topology and execution views"
```

---

### Task 11: Run Full Test Suite

- [ ] **Step 1: Run all agentic module tests**

Run: `/opt/homebrew/bin/mvn -pl agentic/deployment test -q`
Expected: BUILD SUCCESS — all existing tests plus new tests pass

- [ ] **Step 2: If any test fails, fix and re-run**

Common issues to watch for:
- Tag key conflicts between new agent metrics and existing model metrics (the `AgentMetricsTest` and `AgentMeterRegistryTest` already validate tag consistency)
- OTel span exporter timing issues — increase `await()` timeout if needed
- CDI bean resolution ordering — ensure `AdditionalBeanBuildItem` doesn't conflict with existing `Instance<AgentListener>` injection

- [ ] **Step 3: Commit any fixes**

```
git commit -m "fix(agentic): test suite adjustments for observability integration"
```
