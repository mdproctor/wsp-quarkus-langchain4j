---
layout: post
title: "The quarkus-langchain4j agentic module: taking stock"
date: 2026-06-03
type: phase-update
entry_type: note
subtype: diary
projects: [quarkus-langchain4j]
tags: [quarkus, langchain4j, agentic, cdi, observability]
---

The `agentic` module in `quarkus-langchain4j` has always struck me as a missed opportunity. It wraps `langchain4j-agentic` faithfully — same annotations, same execution model — but without any of the Quarkus integration that makes the `core` module genuinely useful. It registers CDI beans and calls it done.

I brought Claude in to verify this properly. Working through the module with IntelliJ's semantic index — navigating into the beta JAR sources, tracing every reference, following the type hierarchy — we turned up 42 specific gaps. The audit is now at `AGENTIC-NATIVE-AUDIT.md`.

The most revealing finding wasn't any single gap. It was the pattern: wherever the upstream `langchain4j-agentic` library is platform-independent, the Quarkus module imported that constraint wholesale rather than replacing it with something Quarkus-native.

The static supplier methods are the clearest example — `@ChatModelSupplier`, `@ToolsSupplier`, `@ChatMemorySupplier`, and five others. A reinvented DI system. Instead of `@Inject ChatModel`, you write a static method on the interface. The upstream library enforces this because it can't assume any DI framework. There's no good reason for the Quarkus module to do the same, given that `SyntheticBeanBuildItem` can resolve all of it at build time. The `@CdiBean` annotation on parameters is the band-aid: a workaround for a constraint we never needed to import.

Guardrails are completely absent. The `core` module has a full CDI-native guardrail system — `InputGuardrail`, `OutputGuardrail`, `ToolInputGuardrail`, `ToolOutputGuardrail` — all as CDI beans, with injection, metrics, events, and build-time validation. Agents get none of it. The `@ErrorHandler` annotation gives you a static method that fires after failure and can retry or rethrow, but can't modify inputs, can't reprompt, and can't inject a CDI bean into the recovery logic. That's not a guardrail; that's a catch block.

The upstream library has an `AgentListener` SPI with rich lifecycle hooks — before and after invocation, before and after tool execution, scope events. The Quarkus module never discovers CDI beans implementing it. So there's no OTel tracing at agent boundaries, no Micrometer metrics per agent, no MicroProfile Health probe. Users who want observability are on their own.

The plan is to enhance the module in-place. Same user-facing annotations, proper Quarkus wiring underneath. Eight chapters, vertical slices through seven layers. Chapter 1 handles quick wins and security fixes. Chapter 2 makes CDI injection first-class. Chapters 4 and 5 add observability and guardrails. `ARC42STORIES.MD` has the full plan including the chapter sequencing rationale.

There's a parallel track for upstream contributions. Some of what's missing isn't Quarkus-specific — no general-purpose DI resolver hook in `DeclarativeUtil` beyond the one for `@ChatModelSupplier`, no scope checkpoint for safe `@Retry` semantics, `ThreadLocal`-based context that breaks on parallel execution. Those go upstream as framework-agnostic improvements. Spring users would benefit from the same changes, which makes the case for acceptance considerably easier.

The workspace is at `github.com/mdproctor/wsp-quarkus-langchain4j` if you want to follow the work.
