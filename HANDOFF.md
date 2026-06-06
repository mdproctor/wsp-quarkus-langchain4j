# Handoff — quarkus-langchain4j
_2026-06-06_

## Last Session

Completed Chapter 2 (CDI-Native Agents). Build-time auto-wiring for 4 supplier types + AgentListener global CDI discovery + S-2 qualifier fix + scope validation + `agentsWithMcpToolBox` cleanup. Squashed to 2 commits, pushed to fork, PR #2542 opened on quarkiverse. CLAUDE.md updated with remote topology rule (PRs only to blessed repo). Two garden entries submitted (standalone RAG skip, MCP ToolProvider collision). Jlama fix merged (PR #2374).

## Immediate Next Step

Start **Chapter 3 (Parallel Safety)** — centres on `ManagedExecutor` replacing `DefaultExecutorProvider` (audit refs C-1 through C-4). Requires an upstream `AgentExecutorProvider` SPI contribution to langchain4j-agentic. If starting C3: `work-start`, file the upstream PR first (per upstream-contribution-framing protocol).

## What's Left

- PR #2526 (C1 — invokeAgent allowlist) awaiting review · S | Low
- PR #2534 (C1 — quick wins + safety) awaiting review · M | Low
- PR #2542 (C2 — CDI auto-wiring) awaiting review · M | Low

## What's Next

| # | Chapter | Audit refs | Notes |
|---|---------|------------|-------|
| C3 | Parallel Safety | C-1 through C-4 | Needs upstream `AgentExecutorProvider` SPI PR |
| C4 | Observable Agents | O-1, O-2, O-4, O-5, D-4 | Depends on C2 + C3 |
| C5 | Guarded Agents | A-2, G-1 | Depends on C2 |
| PR2a | Upstream ParameterResolver | S-4 | Generalise `ChatSupplierParameterResolver` to all supplier types |

## References

- Workspace: `~/claude/public/quarkus-langchain4j/`
- Project: `/Users/mdproctor/claude/quarkus-langchain4j/`
- Full chapter plan: `ARC42STORIES.MD` (C1 ✅, C2 ✅)
- C2 spec: `specs/c2-cdi-native-agents/`
- C2 plan: `plans/attic/c2-cdi-native-agents/`
- Garden refs: `GARDEN-REFS.md`
- Protocols: `protocols/INDEX.md`
