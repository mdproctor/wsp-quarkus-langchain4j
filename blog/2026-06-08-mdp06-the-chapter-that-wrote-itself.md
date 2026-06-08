---
layout: post
title: "The Chapter That Wrote Itself"
date: 2026-06-08
type: phase-update
entry_type: note
subtype: diary
projects: [quarkus-langchain4j]
tags: [quarkus, guardrails, langchain4j-agentic, spi, architecture]
---

## The Chapter That Wrote Itself

I went into Chapter 5 expecting the most complex piece of work in the agentic integration. Agent guardrails — `@InputGuardrails` and `@OutputGuardrails` on agent interfaces — would need new guardrail types, new annotations, an `AgentGuardrailListener`, and upstream PRs to unseal `GuardrailResult`. The ARC42STORIES had it scoped as L4 (Guardrails), high impact, dependent on C2's `AgentListener` auto-discovery.

None of that was right.

## The Wrong Assumptions

The first wrong turn was treating agent guardrails as a separate concept from AiService guardrails. In langchain4j, "guardrail" means validating `UserMessage` and `AiMessage` — LLM interactions, not method inputs and outputs. The upstream `InputGuardrail.validate(UserMessage)` signature doesn't care whether the calling code is an AiService or an agent. It guards the LLM call, wherever that call happens.

The second wrong turn was assuming the agent runtime path (`AgenticServices.createAgenticSystem()`) bypassed the Quarkus guardrail infrastructure entirely. This seemed obvious — agents don't go through `AiServiceMethodImplementationSupport` where `GuardrailsSupport` runs the chain. Different execution path, different wiring, guardrails don't fire. Write the wiring code.

## What Actually Happens

Agents ARE AiServices. `AgenticProcessor.implyAiService()` produces `AnnotationsImpliesAiServiceBuildItem`, which causes `AiServicesProcessor` to process agent interfaces through its full pipeline. `gatherMethodMetadata()` reads `@InputGuardrails` and `@OutputGuardrails` from agent methods. `gatherOutputGuardrails()` extracts `maxRetries`. The metadata lands in `AiServicesRecorder.getMetadata()` keyed by agent class name.

At runtime, `AgentBuilder.build()` creates an inner `AiServices` proxy. The upstream `GuardrailServiceBuilder` reads the metadata via `QuarkusClassMetadataProviderFactory`. Guardrail beans resolve via `QuarkusClassInstanceFactory` (CDI). The `maxRetries` config applies via `QuarkusOutputGuardrailsConfigBuilderFactory`. Every SPI hook connects.

I wrote the first test — `@InputGuardrails(SpyGuardrail.class)` on an agent method, invoke the agent, check the invocation count. It passed immediately.

## langchain4j-cdi Confirmed It

Partway through the design, I discovered [langchain4j-cdi](https://github.com/langchain4j/langchain4j-cdi) — a portable CDI integration for langchain4j. Their `CommonAgentCreator` already calls `agentBuilder.inputGuardrails(instances)` and `agentBuilder.outputGuardrails(instances)`. The upstream `AgentBuilder` has had guardrail support all along — `inputGuardrails()`, `outputGuardrails()`, `inputGuardrailClasses()`, config methods. The agent framework isn't missing guardrails. The Quarkus agentic module just never tested them.

## What C5 Actually Delivered

Eight tests. Zero production code changes.

Input guardrails fire. Output guardrails fire. Reprompting works — the retry loop runs inside the inner `AiServices`. `maxRetries` from the annotation is respected. Missing guardrail beans fail at build time with `DeploymentException`. Guardrail beans survive Arc's dead-code elimination. Multiple guardrails execute in declaration order. Method-level `@InputGuardrails` takes precedence over class-level.

The `AnnotationsImpliesAiServiceBuildItem` coupling is the key. It's what makes agents invisible to `AiServicesProcessor` — the processor doesn't know it's processing an agent, it just sees an interface with annotations. Everything downstream works because nothing downstream needs to know it's an agent.

## What It Doesn't Cover

This is LLM-level validation, not agent-boundary validation. The guardrails fire on the `UserMessage` and `AiMessage` inside the agent's LLM calls. They don't validate the agent's typed method inputs (`Map<String, Object>`) or intercept sub-agent hand-offs in a sequence. Those are genuinely different mechanisms — the types don't match and the abstraction level is different. The ARC42STORIES now reflects this explicitly.

The spec went through five versions. The final one is three pages shorter than the first. Sometimes the right answer is that the work is already done.
