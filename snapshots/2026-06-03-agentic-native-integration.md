# quarkus-langchain4j agentic module — Design Snapshot
**Date:** 2026-06-03
**Topic:** Quarkus-native agentic integration — initial audit and delivery plan
**Supersedes:** *(none — first snapshot)*
**Superseded by:** *(leave blank — filled in if this snapshot is later superseded)*

---

## Where We Are

The `agentic` module wraps `langchain4j-agentic` (`1.15.1-beta25`) as a
Quarkus extension but without genuine Quarkus integration: CDI injection
is blocked by static supplier methods, parallel agents lose context propagation,
no OTel spans exist at agent boundaries, guardrails are absent, and no config
namespace exists. A 42-finding audit (`AGENTIC-NATIVE-AUDIT.md`) catalogued
every gap. An 8-chapter delivery plan (`ARC42STORIES.MD`) structures the work
as vertical slices through 7 defined layers. Chapter 1 has not yet started.

## How We Got Here

| Decision | Chosen | Why | Alternatives Rejected |
|---|---|---|---|
| Enhance in-place vs new module | Enhance existing `agentic/` module | Same `langchain4j-agentic` annotations mean two modules would classpath-conflict | New `quarkus-agentic` module |
| Coupling to upstream | Tight — same annotations, track manually | Loose coupling requires parallel annotation sets; bypass requires full re-implementation | Loose coupling, bypass |
| Workspace location | `~/claude/public/quarkus-langchain4j/` | Methodology artifacts must not enter the project git | In-project alongside source |
| CLAUDE.md handling | Symlink project→workspace, `.git/info/exclude` | Project policy forbids committing CLAUDE.md | Committed CLAUDE.md, separate files |
| Protocol store | `protocols/` in workspace, `PLATFORM.md` as entry point | Mirrors casehub/parent pattern; keeps rules out of source repo | In-project `docs/protocols/` |

## Where We're Going

**Next steps:**
- C1 (Quick Wins & Safety) — first PR: D-2 allow-list fix, F-3/F-4/F-5 build-time warnings, one-line fixes (F-7, C-6, C-7, D-1, S-3)
- File upstream PRs to `langchain4j-agentic`: `SupplierParameterResolver` SPI, scope checkpoint, Qute-compatible PlannerAgent template
- C2 (CDI-Native Agents) — `AgentListener` CDI auto-discovery unblocks O-1, O-2, A-2

**Open questions:**
- Does the upstream `langchain4j-agentic` maintainer accept the `SupplierParameterResolver` SPI PR? Determines whether S-4 can be addressed without a Quarkus-side workaround.
- Will C3 (`ManagedExecutor` default) need an opt-in config flag to avoid silently changing threading behaviour for existing users?
- Can `@InputGuardrails` on agent interfaces be wired entirely via `AgentListener` on the Quarkus side, or does it require an upstream hook?
- Is `langchain4j-agentic` beta API stable enough across releases that tight coupling is safe, or should a compatibility shim be planned?

## Linked ADRs

*(No ADRs created yet — key decisions captured in `ARC42STORIES.MD` §10)*

## Context Links

- Audit: `AGENTIC-NATIVE-AUDIT.md` — 42 findings, priority table, phased plan
- Delivery plan: `ARC42STORIES.MD` — 8 chapters, 7 layers, layer × chapter matrix
- Coherence: `PLATFORM.md` — run before implementing anything in the agentic module
- Protocols: `protocols/INDEX.md` — standing rules
