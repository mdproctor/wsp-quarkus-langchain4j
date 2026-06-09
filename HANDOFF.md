# Handoff — quarkus-langchain4j
_2026-06-09_

## Last Session

Implemented C6 (Configurable Agents) — per-agent config overrides, config expression resolution, Vert.x A2A transport. Then reconstructed all 5 PR branches as self-contained, standalone-compilable commits. Each PR now has a family header, ARC42 references removed, George's feedback addressed. Upstream issues filed (#5399, #5400). Mario tagged on #2534 about his AgenticProcessor rework.

## Immediate Next Step

Wait for #2526 CI to go green (reconstructed branch pushed, should trigger CI). If it passes, ping George for review. If it fails, check the build log — the branch was verified locally.

## Cross-Module

**Blocked by:**
- `langchain4j-agentic` — Mario's AgenticProcessor rework (compile-time generation replacing dynamic proxies). Our PRs #2534–#2550 may need rebasing once his changes land. Config infrastructure (AgenticRuntimeConfig, build-time name extraction) is unaffected.

## What's Left

- OTel tag rename PR — split from #2550 per George's feedback. `AiServicesProcessor` + `MetricsCountedWrapper` aligning with `gen_ai.*` semantic conventions · S · Low
- C6 PR — create after C4 (#2550) merges. Code is on `fork/main` (commits `129da8b`–`d0afde3`) · M · Low
- Code formatting command — user mentioned a PR comment with specific formatting instructions, couldn't locate it. `mvn process-sources` runs formatter + impsort and all branches pass · XS · Low
- Upstream PR #5394 — SupplierParameterResolver generalisation, submitted, awaiting review · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Rebase PRs on Mario's AgenticProcessor rework | M | Med | When his changes land; config infra stays, wrappers may change |
| C7 | Resilient Agents | M | High | Blocked on upstream #5376, #5377 |
| C8 | Persistent Agents | L | High | Future |

## References

- PR chain: #2526 (ready) → #2534 → #2555 → #2544 → #2550 (all draft)
- Epic: [#2549](https://github.com/quarkiverse/quarkus-langchain4j/issues/2549)
- Upstream: [#5399](https://github.com/langchain4j/langchain4j/issues/5399), [#5400](https://github.com/langchain4j/langchain4j/issues/5400)
- C6 spec: `specs/2026-06-09-c6-configurable-agents-design.md`
- Blog: `blog/2026-06-09-mdp08-wrapping-upstream.md`
- Garden: 5 entries (fluent-chain-escape, config-ghost-entry, supervisor-spi-bypass, spi-wrapping-technique, produce-builditem)
