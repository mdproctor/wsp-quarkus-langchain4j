---
layout: post
title: "Wrapping upstream until upstream catches up"
date: 2026-06-09
type: phase-update
entry_type: note
subtype: diary
projects: [quarkus-langchain4j]
tags: [agentic, config, a2a, vertx, upstream]
---

## The problem with annotation-hardcoded values

Chapter 6 is about making agent parameters configurable — `maxIterations` on `@LoopAgent`, `a2aServerUrl` on `@A2AClientAgent`. These are compile-time constants baked into annotations. You can't change them without recompiling, which is exactly the problem `application.properties` solves for everything else in Quarkus.

The first instinct was Jandex annotation transformation — rewrite the annotation values at build time before upstream reads them. That doesn't work. Upstream reads annotations via Java reflection at runtime, not via the Jandex index. Transformations in Jandex are invisible to `method.getAnnotation(LoopAgent.class)`.

The second instinct was to bypass the declarative creation path entirely and use the programmatic builder API. That works but means reimplementing upstream's annotation parsing — and keeping it in sync across releases. A maintenance tax for something that should be upstream's responsibility.

## The SPIs that were already there

The answer turned out to be simpler. Upstream already has `AgenticServices.setWorkflowAgentsBuilder()` — a public static method that replaces the factory for all workflow builders. We register a wrapper that delegates everything except `loopBuilder()`, where it wraps the returned builder in a decorator that applies the config override on `build()`.

For A2A URLs, `A2AService` is loaded via ServiceLoader with no public setter. Reflection gets us in — ugly but temporary. We filed [langchain4j#5400](https://github.com/langchain4j/langchain4j/issues/5400) for the one-line setter that removes the hack.

For the A2A HTTP transport, the SDK already designed the SPI we needed. `A2AHttpClientProvider` uses priority-based discovery — the Javadoc even names "VertxA2AHttpClient: priority 100" as the intended extension point. We implement exactly that.

## The fluent chain trap

The decorator around `LoopAgentService` nearly had a subtle bug. Upstream chains fluently:

```java
var builder = loopBuilder(agentServiceClass)
    .subAgents(...)
    .maxIterations(annotation.maxIterations());
builder.build();
```

If any fluent method on the wrapper delegates and returns the *delegate's* return value instead of `this`, the chain escapes. Every subsequent call bypasses the wrapper. The config override in `build()` never fires. It compiles. It runs. It silently doesn't work.

The fix is mechanical: every fluent method delegates the operation AND returns `this`. Sixteen one-liner methods. But missing it would have meant a config system that appears to work in every test except the one that actually chains calls — which is every real use case.

## What's temporary, what stays

The config structure — `AgenticRuntimeConfig` with `@WithParentName`/`@WithDefaults` per-agent overrides — is permanent. It follows the same pattern as every model provider config in the repo. The Vert.x HTTP client is permanent too — clean SPI, no workaround.

The wrappers around `WorkflowAgentsBuilder` and `A2AService` are temporary. We filed [langchain4j#5399](https://github.com/langchain4j/langchain4j/issues/5399) for a workflow-level `AgentConfigurator` that would make them unnecessary. When that lands, the wrappers come out and the config values flow through the proper SPI. Every temporary class carries the upstream issue number in its Javadoc.

One gap we can't close yet: `@SupervisorAgent(maxAgentsInvocations=10)`. The supervisor builder is created directly via `new SupervisorAgentServiceImpl<>()` — it bypasses `WorkflowAgentsBuilder` entirely. The config key is declared but unwired until upstream routes supervisors through the same SPI.
