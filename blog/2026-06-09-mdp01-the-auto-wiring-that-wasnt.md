---
layout: post
title: "The auto-wiring that wasn't"
date: 2026-06-09
type: pivot
entry_type: note
subtype: diary
projects: [quarkus-langchain4j]
tags: [agentic, cdi, quarkus, auto-wiring]
---

The CDI auto-wiring in `AgenticProcessor` looked right. For each agent, `detectCdiSuppliers()` checked whether a static supplier method existed. If not, it queried CDI for a `@Default`-qualified bean of that type. If exactly one existed, it wired it in. Per-agent. Explicit fallback logic.

The problem was what "per-agent" meant in practice. If you have one `RetrievalAugmentor` bean in the deployment — say, for your RAG pipeline agent — every other agent that didn't declare a `@ContentRetrieverSupplier` also got that augmentor. Silently. The code was per-agent in structure but global in effect.

Two reviewers flagged this independently, from production experience with the same pattern on `@RegisterAiService`. Advanced RAG pipelines compose multiple AiServices — the augmentor calls out to other services for query rewriting and summarisation. Auto-wiring the augmentor into those inner services creates circular dependencies or just wrong behaviour. The fix for `@RegisterAiService` was explicit opt-in/opt-out properties. We'd built the same footgun.

## What already worked

The fix came from tracing what was already in the codebase rather than designing something new.

`@RegisterAiService` already works on agent interfaces. `AnnotationsImpliesAiServiceBuildItem` causes `AiServicesProcessor.determinedImpliedRegisterAiService()` to create a bare synthetic `@RegisterAiService` for every agent. But if the user puts an explicit `@RegisterAiService` on their agent interface, the `alreadyHandled` set in `AiServicesProcessor` skips the implied one. The user's properties win. This was already there — we just hadn't connected it.

`@CdiBean` on supplier method parameters already worked too. A `@ChatModelSupplier` method can take a `@CdiBean`-annotated parameter, and `CdiChatSupplierParameterResolver` resolves it from CDI at runtime. The supplier method stays portable — it's a valid langchain4j-agentic declaration — and `@CdiBean` adds the Quarkus behaviour on top.

## Three tiers

Removing `detectCdiSuppliers()` and the `CdiSupplierType` enum left a clean three-tier model:

**Tier 1** is `@Agent` with static suppliers only. No CDI involvement. The interface works identically in plain langchain4j, langchain4j-cdi, or Quarkus. This is the portable baseline.

**Tier 2** adds `@CdiBean` on supplier method parameters. The supplier method is still the declaration site, but the parameter gets resolved from CDI at build time. Currently limited to `@ChatModelSupplier` — the upstream parameter resolver SPI (`ChatSupplierParameterResolver`) only covers that one supplier type.

**Tier 3** adds `@RegisterAiService` to the agent interface. Full Quarkus auto-wiring with explicit controls — each supplier type can be specified or opted out with `NoRetrievalAugmentorSupplier.class` and friends. Not portable, but familiar to anyone who's used `@RegisterAiService` on AiServices.

## The upstream bottleneck

`@CdiBean` is the cleanest of the three tiers — explicit, per-parameter, portable. But it only works on `@ChatModelSupplier` because that's the only supplier type where langchain4j-agentic accepts method parameters. Every other supplier annotation (`@ContentRetrieverSupplier`, `@ChatMemorySupplier`, `@RetrievalAugmentorSupplier`, etc.) enforces `static` + no-args at the framework level.

The SPI that makes `@CdiBean` work — `ChatSupplierParameterResolver` — is already generalised in concept. It resolves parameters by type and annotation. But it's only wired into the code path for chat model suppliers. Extending it to all supplier types is a mechanical change in langchain4j-agentic's `DeclarativeUtil`: the same resolver call that runs for `@ChatModelSupplier` needs to run for the other supplier annotations instead of rejecting parameters outright.

We filed this as langchain4j issue #5377. It has no traction yet — just one CC, no code. Submitting a PR with the implementation would move it faster. Until it lands, a user who wants CDI-resolved content retrievers or memory providers has to use tier 3 — `@RegisterAiService` with its Quarkus-specific properties. Tier 2 is the better design, but tier 3 is the one that works today for all supplier types.
