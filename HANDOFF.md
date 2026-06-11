# Handoff — quarkus-langchain4j
_2026-06-11_

## Last Session

Full PR review sweep across all 5 quarkiverse PRs and 5 upstream issues. Addressed all reviewer comments (geoand, mariofusco, phillip-kruger, patriot1burke), removed proxy-dependent validations (F-4/F-5, validateFallback), fixed code quality (BuiltinScope, FQCN, allowlist scope validation, classloader test). #2526 merged. #5394 merged upstream. Filed #2572 (RegisterAiService simplification) from patriot1burke's discussion. Updated all upstream issues with problem/solution code examples. Restacked full chain multiple times as fixes landed.

## Immediate Next Step

Wait for geoand to re-review #2534 — 4 new comments answered this session (classloader fallback, BuiltinScope, agentsWithMcpToolBox, TCCL eager init). Once #2534 merges, the chain unblocks. Meanwhile, start `langchain4j-cdi` analysis for #2572 roadmap (user requested).

## Cross-Module

**Blocked by:**
- `langchain4j-agentic` — Mario's AgenticProcessor rework (compile-time generation replacing dynamic proxies). PRs #2534–#2550 may need rebasing. Config infrastructure unaffected.

## PR Chain Status

| PR | Title | Status | Comments | CI |
|---|---|---|---|---|
| #2526 | invokeAgent allowlist | **MERGED** | — | — |
| #2534 | build-time safety + CDI foundation | Ready for re-review | 4 new replies posted (classloader, BuiltinScope, mcpToolBox, TCCL). Rebased on origin/main. | Pending |
| #2555 | CDI wiring tiers + guardrails (test-only) | Waiting on #2534 | All answered. Description rewritten to clarify test-only. patriot1burke → #2572. | Pending |
| #2544 | parallel context propagation | Waiting on #2534 | All answered. | Pending |
| #2550 | OTel, metrics, CDI events, health | Waiting on #2534 | All answered. Tag rename split offered, awaiting geoand. | Pending |

## Upstream Issues

| # | Title | Status | Action this session |
|---|---|---|---|
| #5394 | SupplierParameterResolver | **MERGED** | Removed ChatSupplierParameterResolver, updated revapi.json |
| #5376 | Pluggable DefaultExecutorProvider | OPEN | Replied to mariofusco's pushback with classloader test link |
| #5377 | Generalise ChatSupplierParameterResolver | CLOSED | Covered by #5394 |
| #5378 | @ParallelExecutor DI params | OPEN (unblocked) | Updated with problem/solution code examples |
| #5399 | Widen AgentConfigurator to workflow builders | OPEN | Updated with problem/solution code examples |
| #5400 | A2AService.setA2AService() setter | OPEN | Updated with problem/solution code examples |

## What's Left

- Forage sweep — 3 entries identified (conditionalDevDependencies gotcha, PR comment technique, BuiltinScope undocumented), not yet submitted · XS · Low
- Code formatting command added to CLAUDE.md (`mvn process-sources -T 1C`) · XS · Low — done
- `CdiChatSupplierParameterResolver` → `CdiSupplierParameterResolver` rename — do when next upstream beta releases with #5394 · S · Low
- C6 PR — create after C4 (#2550) merges. Code on `fork/main` · M · Low
- OTel tag rename — split from #2550 if geoand requests · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #2572 | @RegisterAiService simplification + composition annotations | L | Med | **Breakdown below**. Start in new session. |
| — | `langchain4j-cdi` analysis → roadmap.md | M | Med | User requested comparative analysis for ideas we might adopt |
| — | Rebase PRs on Mario's AgenticProcessor rework | M | Med | When his changes land |
| C7 | Resilient Agents | M | High | Blocked on upstream #5376, #5377 |
| C8 | Persistent Agents | L | High | Future |

## #2572 Breakdown (for next session)

1. **Foundation** — add one direct bean-class attribute to `@RegisterAiService` (e.g. `contentRetriever`), prove pattern end-to-end with test · S · Low
2. **`@RagPipeline`** — first composition annotation, proof-of-concept · M · Med
3. **Remaining @RegisterAiService attributes** — expand bean-class pattern to all supplier attributes, deprecate supplier versions · M · Low
4. **`@MemoryConfig`** — group memory attributes · S · Low
5. **`@ToolConfig`** — group tool attributes · S · Low
6. **`@HybridSearch`** — search strategy composition · M · Med
7. **`@DocumentIngestion`** — write-side ingestion pipeline · M · Med

## References

- Issue: [#2572](https://github.com/quarkiverse/quarkus-langchain4j/issues/2572)
- Epic: [#2549](https://github.com/quarkiverse/quarkus-langchain4j/issues/2549)
- Discussion: [#2559](https://github.com/quarkiverse/quarkus-langchain4j/discussions/2559)
- Blog: `blog/2026-06-09-mdp08-wrapping-upstream.md`
- C6 spec: `specs/2026-06-09-c6-configurable-agents-design.md`
