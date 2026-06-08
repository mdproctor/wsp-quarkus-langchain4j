---
layout: post
title: "The body of work so far"
date: 2026-06-08
type: phase-update
entry_type: note
subtype: diary
projects: [quarkus-langchain4j]
tags: [agentic, quarkus, cdi, observability, guardrails]
---

The agentic module started as a thin wrapper around `langchain4j-agentic`. It compiled. It ran agents. But it didn't use any of what makes Quarkus worth choosing: no CDI injection, no build-time validation, no context propagation across threads, no observability at agent boundaries, and no guardrails.

We audited everything — 42 findings across 7 layers — and planned 8 chapters of work. Five are done. This entry ties them together: what each delivers, how they depend on each other, what the reviewers pushed back on, and what we learned along the way.

## The PR stack

Five PRs, each building on the last:

| PR | Chapter | What it delivers |
|---|---|---|
| #2526 | C1a | Allowlist for `invokeAgent` reflection — Dev UI can't invoke arbitrary classes |
| #2534 | C1b | Build-time validation, `@Fallback` error, interceptor inheritance, `@Retry`/`@Transactional` warnings, `AgenticRuntimeConfig` skeleton, Dev UI JSON RPC service gated on dev mode |
| #2542 | C2 + C5 | CDI auto-wiring for all supplier types + guardrail tests proving the SPI chain works |
| #2544 | C3 | `ManagedExecutor` as default parallel executor — CDI, OTel, and Security contexts propagate to worker threads |
| #2550 | C4 | `AgentSpanListener`, `AgentMetricsListener`, `AgentCdiEventListener`, `AgentHealthCheck`, Dev UI topology and execution views |

#2534 must merge first — the others depend on `AgenticProcessor` and `DetectedAiAgentBuildItem` it introduces. #2542 and #2544 are independent of each other but both depend on #2534. #2550 depends on all three.

## What the reviewers said

The auto-wiring approach in #2542 drew the strongest feedback. The concern: unconditionally adding CDI beans for supplier types means an `@AugmentorSupplier` on one agent interface gets applied to every AiService in the deployment, not just that agent. In advanced RAG pipelines where the augmentor calls out to other AiServices for summarisation or query rewriting, auto-wiring becomes a footgun.

This is a valid concern and matches a broader discussion about removing auto-wiring from the Quarkus integration entirely. The supplier pattern itself — static methods returning instances — was also questioned: in a CDI environment, implementations are already beans, and the supplier becomes boilerplate.

On #2534, the reviewer asked for `AgenticJsonRpcService` to move to a `runtime-dev` module so it's excluded from production builds. Done — the class now lives in `agentic/runtime-dev` (`quarkus-langchain4j-agentic-dev`).

On #2550, the reviewer asked why `AiServicesProcessor.java` and `MetricsCountedWrapper.java` were changed. The changes rename metric tags to align with OpenTelemetry semantic conventions (`ai_service.class_name` instead of `aiservice`, `error.type` instead of `exception` + `result`). This is a breaking change for existing dashboards — it may need to move to its own PR.

## Cross-PR fixes

Working across PRs means a fix in one chapter can break another. Two examples from this session:

**Observability listeners broke the MCP module.** `AgentSpanListener` and `AgentMetricsListener` had `@ApplicationScoped` annotations, which caused Arc to discover them via standard bean scanning — even when the capability check in `AgenticObservabilityProcessor` correctly gated their registration. In the MCP module tests, OTel isn't on the classpath, so Arc failed with `UnsatisfiedResolutionException` for `Tracer`. The fix: remove `@ApplicationScoped` from the listener classes and let the build step set scope via `setDefaultScope()`. The listeners only exist as CDI beans when their capability (OTel, Micrometer) is present.

**Metric tag renames broke integration tests.** The OTel semantic convention alignment in `AiServicesProcessor` changed the default metric tags, but `AssistantResourceWithMetricsTest` in the OpenAI integration tests still expected the old names. The test failures were `"Unable to find the requested metrics"` — straightforward once you know the tags changed, but the CI failure message gives no hint about tag names.

## The C5 surprise

C5 (guardrails) was supposed to be the most complex chapter. We expected to build new annotation types, wire them through `AgentBuilder`, and file upstream PRs to unseal `GuardrailResult`. Instead, we traced the Quarkus SPI chain and found that guardrails already work on agents:

`AnnotationsImpliesAiServiceBuildItem` (from C1) → `AiServicesProcessor.determinedImpliedRegisterAiService()` → `DeclarativeAiServiceBuildItem` → guardrail metadata registration → `QuarkusClassMetadataProviderFactory` SPI → `GuardrailsSupport.executeOutputGuardrails`.

Zero production code changes. The whole chapter is tests proving the existing chain works.

## What's pending

All five PRs are open. #2534 is closest to merge — the `runtime-dev` module move was the last blocking item. Once it merges, the others can be rebased and reviewed in sequence.

The auto-wiring concern on #2542 needs a response. The work is valid — CDI resolution for supplier types is the right direction — but the unconditional-registration mechanism may need to become opt-in, matching `@RegisterAiService`'s pattern.

Three upstream issues are filed (#5376, #5377, #5378) for SPI improvements that would simplify the Quarkus wiring. None block current work.
