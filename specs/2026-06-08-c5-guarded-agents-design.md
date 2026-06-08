# C5 — Guarded Agents: Design Spec

**Branch:** `c5-guarded-agents`
**Date:** 2026-06-08
**Status:** Draft v5

---

## Problem

Upstream `AgentBuilder` supports guardrails programmatically — `inputGuardrails()`, `outputGuardrails()`, `inputGuardrailClasses()`, `outputGuardrailClasses()`, and config methods that pass through to the agent's internal `AiServices.builder()`. The standard `InputGuardrail`/`OutputGuardrail` interfaces work. No new types needed.

The upstream **declarative** path (`DeclarativeUtil.configureAgent()`) does NOT read `@InputGuardrails`/`@OutputGuardrails` from agent interfaces. However, Quarkus's `AiServicesProcessor` already processes agent interfaces as AiServices (via `AnnotationsImpliesAiServiceBuildItem` → `determinedImpliedRegisterAiService()` → `DeclarativeAiServiceBuildItem`). This means:

- `gatherMethodMetadata()` reads `@InputGuardrails`/`@OutputGuardrails` from agent methods
- `gatherOutputGuardrails()` extracts `maxRetries` from the annotation (falling back to `quarkus.langchain4j.guardrails.max-retries` config)
- The metadata is stored in `AiServicesRecorder.getMetadata()` keyed by the agent class name
- Class-level guardrails from `DeclarativeAiServiceBuildItem` are also populated

At runtime, `AgentBuilder.build()` creates an internal `AiServices` proxy. The upstream `GuardrailServiceBuilder` uses three Quarkus SPIs:
- `QuarkusClassMetadataProviderFactory` — reads guardrail annotations from `AiServicesRecorder.getMetadata()`
- `QuarkusOutputGuardrailsConfigBuilderFactory` — reads `quarkus.langchain4j.guardrails.max-retries` config
- `QuarkusClassInstanceFactory` — resolves guardrail beans via CDI

**The SPI chain means guardrails on agent methods should already work today without any C5 code changes.** Every hop in the chain connects:

1. **Build time:** `AiServicesProcessor.findDeclarativeServices()` processes agents via `AnnotationsImpliesAiServiceBuildItem` → `determinedImpliedRegisterAiService()`. `gatherMethodMetadata()` extracts `@InputGuardrails`/`@OutputGuardrails` into `InputGuardrailsLiteral`/`OutputGuardrailsLiteral`. Stored in `AiServicesRecorder.getMetadata()` at `RUNTIME_INIT`.
2. **Runtime context creation:** `AgentBuilder.build()` → `AiServiceContext.create(agentServiceClass)` → `GuardrailService.builder(agentServiceClass)` → `GuardrailServiceBuilder(agentServiceClass)`.
3. **Guardrail build (lazy, on first access):** `guardrailServiceBuilder.build()` → `QuarkusClassMetadataProviderFactory.getNonStaticMethodsOnClass(agentServiceClass)` → reads `AiServicesRecorder.getMetadata().get(agentClassName)` → finds `AiServiceClassCreateInfo` → iterates `methodMap()` → `getAnnotation(method, InputGuardrails.class)` → returns `InputGuardrailsLiteral` → `computeInputGuardrails(annotation)` → `getGuardrails(annotation)` → `ClassInstanceLoader.getClassInstance()` → `QuarkusClassInstanceFactory` → `CDI.current().select(clazz).get()`.
4. **Config:** `computeConfig(outputGuardrails)` → `OutputGuardrailsConfig.builder()` → `QuarkusOutputGuardrailsConfigBuilderFactory` SPI → `QuarkusOutputGuardrailsConfig` → reads `quarkus.langchain4j.guardrails.max-retries`.
5. **No double execution:** `computeInputGuardrailsForAiServiceMethod()` checks `inputGuardrailsAndConfigSetOnBuilder()` first — false (nothing set on the builder by `QuarkusAgenticContextConsumer`), so it falls through to the SPI metadata path. Only ONE source of guardrails is active.

Phase 1 is positioned as **confirmation** of this traced analysis, not speculative discovery.

---

## Design Decision: LLM-Level Guardrails, Not Agent-Boundary Guardrails

ARC42STORIES (§5, L4) originally described C5 as delivering **agent-boundary** guardrails via an `AgentGuardrailListener` that fires in `beforeAgentInvocation(AgentRequest)` / `afterAgentInvocation(AgentResponse)`. This would validate typed method inputs (`Map<String, Object>`) and agent outputs (`Object`) at the agent lifecycle boundary.

This spec delivers a **different** thing: LLM-level guardrails wired through `AgentBuilder` → inner `AiServices`. These validate `UserMessage` before the LLM call and `AiMessage` after — the same boundary as AiService guardrails.

**Why this is the right first step:**

1. The upstream `AgentBuilder` already supports it. The Quarkus SPI chain may already make it work.
2. The same guardrail beans work on both AiServices and agents — zero new API for developers.
3. Reprompting works because the retry loop runs inside the inner `AiServices` `GuardrailService`.
4. `maxRetries` config is handled by the existing `QuarkusOutputGuardrailsConfigBuilderFactory` SPI.

**What this does NOT deliver (deferred):**

Agent-boundary guardrails — validating typed method inputs before the agent starts, validating sub-agent output before passing to the next agent in a sequence, applying security filters to the agent's final typed output (as distinct from individual LLM responses). These require new guardrail types or an `AgentListener`-based mechanism. Deferred.

**ARC42STORIES must be rewritten** to reflect this change.

---

## Coverage Against A-2

| A-2 bullet | Level | This spec delivers? | Notes |
|---|---|---|---|
| Validate or sanitise inputs to an agent call | Agent boundary | **No** | Inputs are typed method args, not `UserMessage`. LLM-level guardrails fire on the message constructed from those args. |
| Validate sub-agent output before passing to next agent | Agent boundary | **No** | Sequence hand-off is inside the workflow pipeline. LLM guardrails fire on each sub-agent's individual LLM calls. |
| Reprompt when output fails a business rule | LLM level | **Yes** | Inner `AiServices` retry loop handles reprompting. `OutputGuardrail.reprompt()` works. |
| Apply security/PII filters at agent boundaries | Partial | **Partial** | PII filters fire on every `UserMessage`/`AiMessage` within the agent, not on the agent's typed method return value. |

---

## Implementation

### Phase 1: Empirical Verification

Before writing runtime wiring code, verify whether the existing SPI chain already makes guardrails work on agents:

**Test:** Create an agent interface with `@InputGuardrails(TestGuardrail.class)` and a CDI guardrail bean. Invoke the agent. Verify the guardrail's `validate()` method is called.

**If guardrails fire:** C5 is a validation-and-testing chapter. No `QuarkusAgenticContextConsumer` changes needed. No `AiAgentCreateInfo` field additions needed. Proceed to Phase 2.

**If guardrails do NOT fire:** Identify exactly where the SPI chain breaks. Add a targeted fix at the break point. Do NOT pass instances explicitly via `agentBuilder.inputGuardrails()` — that risks double execution if the SPI chain is partially active.

### Phase 2: Build-Time Validation (Expected to already work)

`AiServicesProcessor.validateGuardrails()` validates guardrail beans for all interfaces it processes. Since agents produce `DeclarativeAiServiceBuildItem` and are processed in the same loop, agent guardrail beans should already be validated — the same `SynthesisFinishedBuildItem` step iterates all `AiServicesMethodBuildItem` entries, including those from agent interfaces.

**Confirm:** Put `@InputGuardrails(NonExistentGuardrail.class)` on an agent interface. Verify the build fails with `DeploymentException("Missing guardrail bean: ...")`.

If validation already works (expected): no new `@BuildStep` needed.

If not: add a targeted `@BuildStep` in `AgenticProcessor`.

### Phase 3: Unremovable Beans (Expected to already work)

The same `AiServicesProcessor` loop that processes guardrail beans for AiServices also processes agent interfaces. Bean unremovability should already be handled since agents share the processing path.

**Confirm:** Reference a guardrail bean only from an agent interface (no AiService references). Verify it survives Arc's dead-code elimination.

If it does (expected): no changes needed.

If not: add `UnremovableBeanBuildItem.beanTypes()` for agent-referenced guardrail classes.

### Phase 4: Test Suite

Comprehensive tests regardless of which phases required code changes:

1. **Agent with `@InputGuardrails`** — verify guardrails fire before inner LLM call
2. **Agent with `@OutputGuardrails`** — verify guardrails fire after inner LLM call
3. **Agent with output guardrail reprompt** — verify retry loop works through `AgentBuilder` → inner `AiServices` path
4. **Agent with `@OutputGuardrails(maxRetries = 5)`** — verify annotation value is respected
5. **Agent with `quarkus.langchain4j.guardrails.max-retries` config** — verify config value applies
6. **Agent with missing guardrail bean** — verify build-time `DeploymentException`
7. **Agent with class-level + method-level guardrails** — verify method-level precedence
8. **Agent without guardrails** — verify no change in behavior
9. **`@SequenceAgent` where sub-agent has guardrails but parent doesn't** — verify guardrails fire only on sub-agent's LLM calls
10. **Multiple guardrails `@InputGuardrails({A.class, B.class})`** — verify execution order matches declaration order
11. **Same guardrail bean on both AiService and agent** — verify no cross-contamination or double execution

### Phase 5: Documentation and ARC42STORIES Updates

Update these ARC42STORIES sections:

1. **C5 chapter description** (line ~319) — rewrite to reflect LLM-level guardrails via SPI chain, not `AgentGuardrailListener`
2. **L4 layer detail** (lines ~495-521) — rewrite: remove `AgentGuardrailListener`, describe the `AgentBuilder` → `AiServices` → SPI chain
3. **§5 Building Block View** (line ~103) — remove `AgentGuardrailListener` from L4 container
4. **§6 Runtime View** (lines ~131-141) — remove guardrail references from `AgentListener` chain description
5. **§4 Solution Strategy** (line ~76) — update "C2 before C5" sequencing — C5 no longer depends on C2
6. **L1 layer participation** (line ~396) — review L1's C5 participation
7. **ADR rationale** (line ~642) — amend: "AgentListener fires at the correct granularity for OTel spans, metrics, ~~and guardrails~~." Guardrails use the inner AiServices path, not AgentListener.

---

## Pattern Break Acknowledgement

C2-C4 established `AgentListener` CDI beans as the extension point for agent lifecycle concerns (OTel spans, metrics, CDI events). This spec does NOT use `AgentListener` for guardrails.

**Why:** `InputGuardrail.validate(UserMessage)` operates on LLM-level types. `AgentListener.beforeAgentInvocation(AgentRequest)` operates on agent-boundary types (`Map<String, Object> inputs`). These are different abstraction levels. LLM-level guardrails belong on the inner `AiServices` path, not the `AgentListener` lifecycle.

The `AgentListener` pattern remains correct for observability (C4) and for future agent-boundary guardrails.

---

## C2 Dependency

ARC42STORIES says "C2 before C5" because the planned `AgentGuardrailListener` needed C2's `AgentListener` CDI auto-discovery. This spec doesn't use `AgentListener`. The dependency is removed. C5 depends only on build-time annotation detection and the `AgentBuilder` API, both available now.

---

## Parity Table

| Capability | Pure langchain4j | Quarkus-integrated |
|---|---|---|
| Guardrail interfaces | ✅ `InputGuardrail` / `OutputGuardrail` | ✅ same |
| Declarative annotations | **No** (programmatic `AgentBuilder` only) | ✅ `@InputGuardrails` / `@OutputGuardrails` on agent interfaces |
| Bean resolution | Reflection (`ClassInstanceFactory` SPI) | CDI via `QuarkusClassInstanceFactory` (SPI) |
| Validation | Runtime | Build-time (`DeploymentException`) |
| Max retries config | Programmatic | `@OutputGuardrails(maxRetries=N)` + `quarkus.langchain4j.guardrails.max-retries` config |
| Output reprompting | ✅ (via inner AiServices) | ✅ same |
| Native image | Manual reflection config | Automatic |

---

## Scope

**Possibly zero runtime code changes.** If the existing SPI chain works, C5 is:
- Verify existing behavior via empirical test
- Ensure build-time validation covers agent guardrail beans
- Ensure guardrail beans are marked unremovable
- Comprehensive test suite (11 tests)
- Documentation and ARC42STORIES updates

**If the SPI chain breaks:** Targeted fix at the specific break point. No wholesale `QuarkusAgenticContextConsumer` rewrite. No new `AiAgentCreateInfo` fields.

---

## Architectural Invariant

Agent guardrails depend on `AnnotationsImpliesAiServiceBuildItem` causing `AiServicesProcessor` to process agent interfaces. The entire chain — guardrail metadata registration, build-time validation, unremovable bean marking, and runtime SPI discovery — flows through this coupling. If agent interfaces are ever removed from `AiServicesProcessor`'s scope (e.g., agents get a dedicated processor, the implied-AiService registration is dropped, or the build step is refactored), guardrail metadata, build-time validation, and unremovable marking must be reimplemented in `AgenticProcessor`.

---

## What This Does NOT Deliver

- **Agent-boundary guardrails** — validating typed method inputs/outputs at agent lifecycle boundaries. Deferred.
- **Sub-agent hand-off guardrails** — validating output of one agent before passing to the next in a sequence. Deferred.
- **Tool guardrail changes** — `ToolInputGuardrail`/`ToolOutputGuardrail` unaffected.
- **`GuardrailResult` unsealing** — not needed.

---

## Relationship to langchain4j-cdi

[langchain4j-cdi](https://github.com/langchain4j/langchain4j-cdi)'s `CommonAgentCreator` (`langchain4j-cdi-core/.../CommonAgentCreator.java`, lines 740-786) explicitly passes guardrail instances to `AgentBuilder.inputGuardrails()` / `AgentBuilder.outputGuardrails()`. This bypasses the SPI metadata path and passes instances directly.

Quarkus's approach is different: the guardrail metadata is registered in `AiServicesRecorder.getMetadata()` by `AiServicesProcessor`, and the upstream `GuardrailServiceBuilder` reads it via the `QuarkusClassMetadataProviderFactory` SPI. No explicit instance passing is needed if the SPI chain works — the guardrails are discovered automatically.

This is architecturally cleaner: one registration path (build-time metadata), one discovery path (SPI). No risk of double execution from both metadata-based and instance-based guardrail resolution.
