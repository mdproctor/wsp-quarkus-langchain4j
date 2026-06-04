# Handoff — quarkus-langchain4j
_2026-06-03_

## Last Session

Ran a comprehensive audit of the `agentic` module revealing 42 gaps in Quarkus integration, structured the work as an 8-chapter ARC42STORIES delivery plan, and stood up the companion workspace. The central finding: the module imported upstream `langchain4j-agentic` platform-independence constraints wholesale instead of replacing them with Quarkus-native equivalents — static supplier methods reinvent CDI injection; guardrails are completely absent despite the `core` module having a full CDI-native guardrail system.

## Immediate Next Step

Start **Chapter 1 (Quick Wins & Safety)** — open an issue, create a branch. Audit finding D-2 (unauthenticated reflection in `AgenticJsonRpcService.invokeAgent`) is the priority fix; the allow-list check uses the build-time agent class names already available via `DetectedAiAgentBuildItem`. Remaining C1 items are one-liners (F-7, C-6, C-7, D-1, S-3) plus three build-time warnings (F-3, F-4, F-5).

## What's Left

- File upstream PRs to `langchain4j-agentic` while C1 lands: `SupplierParameterResolver` SPI (S-4), scope checkpoint for `@Retry` safety (F-2), Qute-compatible PlannerAgent template (A-5). Frame as DI-neutral improvements — see `protocols/upstream/upstream-contribution-framing.md`.
- Populate Chapter entries and Layer entries in `ARC42STORIES.MD` §9.3/9.4 as work ships (currently all `🔲`).

## What's Next

| # | Chapter | Layers | Audit refs | Notes |
|---|---------|--------|------------|-------|
| C1 | Quick Wins & Safety | L6, L7 | D-2, F-3–5, F-7, C-6, C-7, D-1, S-3 | Start here — security fix unblocks everything |
| C2 | CDI-Native Agents | L1 | S-1, S-2, O-3, A-1 | `AgentListener` CDI discovery unblocks C4 and C5 |
| C3 | Parallel Safety | L2 | C-1, C-2, C-3, C-4 | Can develop in parallel with C2 |
| C4 | Observable Agents | L3, L7 | O-1, O-2, O-4, O-5, D-4 | Depends on C2 and C3 |
| C5 | Guarded Agents | L4 | A-2, G-1 | Depends on C2 |

Full chapter plan and layer taxonomy: `ARC42STORIES.MD`  
Full finding detail: `AGENTIC-INTEGRATION-AUDIT.md`  
Coherence protocol: `PLATFORM.md`

## References

- Workspace: `~/claude/public/quarkus-langchain4j/`
- Project: `/Users/mdproctor/claude/quarkus-langchain4j/` (fork of github.com/quarkiverse/quarkus-langchain4j)
- Design snapshot: `snapshots/2026-06-03-agentic-native-integration.md`
- Blog entry: `blog/2026-06-03-mdp01-agentic-audit-and-plan.md`
