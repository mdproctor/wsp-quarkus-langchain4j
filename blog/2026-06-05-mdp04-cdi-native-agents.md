---
layout: post
title: "CDI-Native Agents — Build-Time Auto-Wiring for Agentic Suppliers"
date: 2026-06-05
type: phase-update
entry_type: note
subtype: diary
projects: [quarkus-langchain4j]
tags: [CDI, auto-wiring, AgentListener, langchain4j-agentic]
---

# CDI-Native Agents — Build-Time Auto-Wiring for Agentic Suppliers

The second chapter of the agentic-native work, and the one that makes everything else possible.

## The problem with static suppliers

langchain4j-agentic uses static methods on agent interfaces to configure capabilities — `@ChatModelSupplier`, `@ContentRetrieverSupplier`, `@ChatMemorySupplier`, and so on. This works for a framework-agnostic library. In Quarkus, it's the wrong tool. Static methods can't participate in CDI: no injection, no scoping, no qualifiers, no interception. Every supplier becomes a manual wiring exercise. Test a different retriever? New static method. Two agents sharing the same memory? Wire it yourself.

The existing `@CdiBean` annotation on `@ChatModelSupplier` parameters was a first step — it lets you resolve a CDI bean into a chat model supplier method. But it only worked for chat models, it ignored qualifiers entirely, and it required the static method to exist at all.

## What auto-wiring actually means

The design centres on a distinction that shapes everything downstream: fallback wiring vs additive wiring.

**Fallback:** For `ContentRetriever`, `ChatMemory`, `ChatMemoryProvider`, and `RetrievalAugmentor`, CDI beans are wired only when the agent declares no static supplier for that type. If you want a specific retriever for one agent, declare the static method — it wins unconditionally. If you don't declare one and there's exactly one `@Default` CDI bean of that type in the container, it gets wired automatically. Multiple beans? No auto-wiring, no error — an INFO log tells you why.

**Additive:** `AgentListener` is different. CDI listener beans are global — they fire for every agent, composing alongside any per-agent `@AgentListenerSupplier`. This is the hook for OTel spans, Micrometer metrics, guardrail chains, and CDI event bridges in later chapters.

`ToolProvider` was deliberately excluded. The MCP module registers its own `ToolProvider` CDI bean, and `AgentBuilder.toolProvider()` appends to a list rather than replacing. Auto-wiring a `ToolProvider` either double-wires the MCP bean or causes ambiguity when a user bean coexists with it. The concept of "fallback" doesn't apply to append-semantic setters.

## Build time, not runtime

The detection happens in `AgenticProcessor.cdiSupport()` via `BeanDiscoveryFinishedBuildItem`. For each agent, for each supplier type: does a static supplier method exist on the interface hierarchy? If not, is there exactly one `@Default` CDI bean of that type? If yes, an `Instance<T>` injection point is added to the agent's synthetic bean configurator. At runtime, `QuarkusAgenticContextConsumer` reads the pre-computed `cdiResolvedSuppliers` set and wires each type with a single `getInjectedReference()` call — no `Arc.container().select()`, no bean scanning, no runtime decisions.

`@RequestScoped` and `@SessionScoped` beans are caught at build time with a clear error. Agents are `@ApplicationScoped` synthetic beans created during `@RuntimeInit` — no request context exists.

The qualifier fix for `@CdiBean` was straightforward: filter the parameter's annotations for anything annotated with `@jakarta.inject.Qualifier`, exclude `@CdiBean` itself, and pass the qualifiers to `Arc.container().select()`. `@CdiBean @ModelName("gpt4") ChatModel model` now does what you'd expect.

## What this unblocks

C2 is foundation. `AgentListener` CDI auto-discovery is the mechanism that C4 (Observable Agents) and C5 (Guarded Agents) wire their OTel, metrics, and guardrail listeners through. Without it, those chapters would need their own plumbing. With it, they're CDI beans that implement `AgentListener` and get discovered automatically.

The upstream `ChatSupplierParameterResolver` SPI is currently chat-model-only. A separate upstream PR will generalise it to all supplier types — once that lands, the Quarkus `CdiChatSupplierParameterResolver` becomes `CdiParameterResolver` and qualifiers work everywhere. That's PR2, independent of what shipped here.
