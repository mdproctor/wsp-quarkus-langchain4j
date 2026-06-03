---
id: PP-20260603-upstream-001
title: "Frame langchain4j-agentic improvements as DI-neutral — never as Quarkus-specific requests"
type: rule
scope: upstream
applies_to: "any improvement to langchain4j-agentic proposed as an upstream PR"
severity: required
violation_hint: "A PR titled 'Add Quarkus CDI support' or referencing Arc, SyntheticBeanBuildItem, or @ApplicationScoped — upstream maintainers will reject it as framework-specific pollution"
created: 2026-06-03
---

## Rule

When contributing improvements to `langchain4j-agentic`, frame every change as a platform-independent improvement that benefits all DI frameworks (Spring, Micronaut, Guice, plain Java) — never as a Quarkus-specific request.

## Rationale

`langchain4j-agentic` maintainers will not accept Quarkus-specific code. The same improvements that unblock Quarkus CDI integration also unblock Spring `@Autowired` injection into supplier methods. Framing the PR as "DI-agnostic" makes it acceptable upstream and useful to a wider audience.

## How to apply

**Framing that works:**
- "The current static method constraint makes it impossible for any DI framework to inject managed beans into supplier methods. This PR adds a `SupplierParameterResolver` SPI that integrators can implement."
- "The `LangChain4jManaged.CURRENT` `ThreadLocal` is not propagated to child threads, breaking any framework with context propagation. This PR exposes a pluggable `ContextPropagator` hook."
- "The `AgenticScope` has no checkpoint mechanism, making `@Retry` semantics unsound for any retry library. This PR adds scope snapshotting."

**Framing that fails:**
- "Add Quarkus CDI support for supplier methods"
- "Make `@ParallelExecutor` work with `ManagedExecutor`"
- "Integrate with Quarkus SmallRye Context Propagation"

## What belongs upstream vs workspace

| Change | Where |
|--------|-------|
| `SupplierParameterResolver` SPI (DI-neutral interface) | Upstream `langchain4j-agentic` |
| `CdiSupplierParameterResolver` (Quarkus implementation of that SPI) | This workspace / `quarkus-langchain4j` |
| Scope checkpoint API | Upstream |
| Scope checkpoint wiring in Quarkus recorder | This workspace |
| `CompletableFuture.allOf()` fix in `parallelExecution()` | Upstream |
| `ManagedExecutor` as default executor | This workspace |

## Parallel track discipline

File upstream PRs immediately when a chapter opens — they have long review cycles. Do not wait until the Quarkus-side workaround is implemented. The workaround should be designed to disappear cleanly when the upstream SPI lands.
