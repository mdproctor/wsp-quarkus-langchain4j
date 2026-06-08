# Handoff — quarkus-langchain4j
_2026-06-08_

## Last Session

Completed Chapter 4 (Observable Agents). Four CDI `AgentListener` beans — OTel spans, Micrometer metrics, CDI events, health check — conditionally registered via `AdditionalBeanBuildItem` in `AgenticObservabilityProcessor`. Dev UI HTML blobs replaced with structured JSON + Vaadin grids. Hot-reload fix via `ShutdownContext`. PR #2550 opened on quarkiverse. Epic #2549 created with ARC42STORIES embedded. Chapter issues created (#2545–#2548) with specs attached in `<details>` toggles. PRs updated to reference issues instead of carrying inline specs. Specs-on-issues convention added to CLAUDE.md.

## Immediate Next Step

Start **Chapter 5 (Guarded Agents)** or **Chapter 6 (Configurable Agents)** — both are unblocked. C5 adds `@InputGuardrails`/`@OutputGuardrails` via `AgentListener` pipeline. C6 adds `quarkus.langchain4j.agent.*` config namespace and A2A URL resolution.

## What's Left

- PR [#2526](https://github.com/quarkiverse/quarkus-langchain4j/pull/2526) (C1 — invokeAgent allowlist) awaiting review · S · Low
- PR [#2534](https://github.com/quarkiverse/quarkus-langchain4j/pull/2534) (C1 — quick wins + safety) awaiting review · M · Low
- PR [#2542](https://github.com/quarkiverse/quarkus-langchain4j/pull/2542) (C2 — CDI auto-wiring) awaiting review · M · Low
- PR [#2544](https://github.com/quarkiverse/quarkus-langchain4j/pull/2544) (C3 — parallel safety) awaiting review · M · Low
- PR [#2550](https://github.com/quarkiverse/quarkus-langchain4j/pull/2550) (C4 — observable agents) awaiting review · L · Low

## What's Next

| # | Chapter | Scale | Complexity | Notes |
|---|---------|-------|------------|-------|
| C5 | Guarded Agents | M | Med | @InputGuardrails/@OutputGuardrails via AgentListener |
| C6 | Configurable Agents | M | Med | Config namespace, A2A URL resolution, Vert.x WebClient |
| C7 | Resilient Agents | M | High | Blocked on upstream scope checkpoint (#5376, #5377) |

## References

- Epic: [#2549](https://github.com/quarkiverse/quarkus-langchain4j/issues/2549)
- Issues: [#2545](https://github.com/quarkiverse/quarkus-langchain4j/issues/2545) (C1), [#2546](https://github.com/quarkiverse/quarkus-langchain4j/issues/2546) (C2), [#2547](https://github.com/quarkiverse/quarkus-langchain4j/issues/2547) (C3), [#2548](https://github.com/quarkiverse/quarkus-langchain4j/issues/2548) (C4)
- C4 spec: `specs/2026-06-07-c4-observable-agents-design.md`
- C4 plan: `plans/2026-06-07-c4-observable-agents-plan.md`
- Blog: `blog/2026-06-08-mdp06-making-agents-visible.md`
- Garden entry: `GE-20260608-401287` (@Readiness qualifier gotcha)
