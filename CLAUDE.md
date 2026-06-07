# quarkus-langchain4j Workspace
**Name:** quarkus-langchain4j

**Project repo:** /Users/mdproctor/claude/quarkus-langchain4j
**Workspace:** ~/claude/public/quarkus-langchain4j
**Workspace type:** public

## Goal

Make the `agentic` module of `quarkus-langchain4j` a first-class Quarkus integration.

The module currently wraps `langchain4j-agentic` faithfully but without leveraging
Quarkus: CDI injection is blocked by static supplier methods, parallel agents lose
context propagation, no OTel spans exist at agent boundaries, guardrails are absent,
and no config namespace exists. 42 findings catalogued in `AGENTIC-NATIVE-AUDIT.md`.

**Approach:** enhance the existing module in-place — same `langchain4j-agentic`
annotations as user-facing API, proper Quarkus wiring underneath. Tight coupling
to upstream; track manually per beta release. Planned as 8 chapters in `ARC42STORIES.MD`.

**Parallel track:** contribute framework-agnostic SPI improvements upstream to
`langchain4j-agentic` (supplier parameter resolver, threading contract, scope
checkpoint) — framed as platform-independent improvements, not Quarkus-specific requests.

**Key artifacts (workspace):**
- `AGENTIC-INTEGRATION-AUDIT.md` — 42-finding audit, source of truth for what to fix
- `ARC42STORIES.MD` — 8-chapter delivery plan, 7 layers, chapter sequencing rationale
- `PLATFORM.md` — coherence protocol: run before implementing anything in the agentic module
- `protocols/INDEX.md` — standing rules (upstream framing, CDI patterns, chapter sequencing)
- `GARDEN-REFS.md` — curated universal garden entries relevant to this project (Quarkus extension, CDI, Maven, FaultTolerance); **consult before brainstorming or writing specs**

---

## Session Start

Run these two before any other work:
```
add-dir /Users/mdproctor/claude/quarkus-langchain4j
add-dir /Users/mdproctor/claude/public/quarkus-langchain4j
```

Then verify the project CLAUDE.md symlink exists:
```bash
ls -la /Users/mdproctor/claude/quarkus-langchain4j/CLAUDE.md
```

If it is missing (e.g. after a fresh clone or `git clean`), re-create it:
```bash
ln -sf /Users/mdproctor/claude/public/quarkus-langchain4j/CLAUDE.md /Users/mdproctor/claude/quarkus-langchain4j/CLAUDE.md
echo "CLAUDE.md" >> /Users/mdproctor/claude/quarkus-langchain4j/.git/info/exclude
```

The symlink must always remain unstaged — never run `git add CLAUDE.md` in the project repo.

**Never add `wksp` to `.gitignore` in the project repo.** The `wksp` symlink pointing to the workspace is machine-local. Use `.git/info/exclude` instead (already configured). Adding it to `.gitignore` pollutes upstream PRs with a personal workspace artifact.

## Artifact Locations

**All generated methodology artifacts stay in the workspace. The project repo contains source code only.**

| Skill | Writes to |
|-------|-----------|
| brainstorming (specs) | `specs/` (workspace) |
| writing-plans (plans) | `plans/` (workspace) |
| handover | `HANDOFF.md` (workspace) |
| idea-log | `IDEAS.md` (workspace) |
| design-snapshot | `snapshots/` (workspace) |
| java-update-design / update-primary-doc | `ARC42STORIES.MD` (workspace) |
| adr | `adr/` (workspace) |
| write-content | `blog/` (workspace) |

**Never promote specs, plans, ADRs, snapshots, or blog entries to the project repo.** These are workspace artifacts — they would pollute upstream PRs with session methodology content that reviewers don't want and the project repo doesn't own.

## Structure

- `HANDOFF.md` — session handover (single file, overwritten each session)
- `IDEAS.md` — idea log (single file)
- `specs/` — brainstorming / design specs
- `plans/` — implementation plans
- `snapshots/` — design snapshots with INDEX.md
- `adr/` — architecture decision records with INDEX.md
- `blog/` — project diary entries with INDEX.md
- `design/` — epic journals (created by work-start)

## Git Discipline — Two Repos

Two git repositories are active in every session:
- **Workspace** (`~/claude/public/quarkus-langchain4j`) — plans, blog, snapshots, handover, specs, ADRs
- **Project repo** (`/Users/mdproctor/claude/quarkus-langchain4j`) — source code only

Never rely on CWD for git operations. Always use explicit paths:
```bash
git -C /Users/mdproctor/claude/public/quarkus-langchain4j add <file>    # workspace
git -C /Users/mdproctor/claude/quarkus-langchain4j add <file>           # project
```

## Git Discipline — Remote Topology

The project repo has two remotes:
- **`fork`** (`casehubio/quarkus-langchain4j`) — our fork. Push here always.
- **`origin`** (`quarkiverse/quarkus-langchain4j`) — the blessed upstream repo. **Never push directly.** All changes to the blessed repo go via PR from the fork.

At work-end, the fork push is mandatory. The blessed repo prompt should always be **[R] Open PR**, never **[P] Push directly**.

## Git Discipline — CLAUDE.md Symlink

**CLAUDE.md in the project repo is a symlink pointing here. Never stage or commit it.**

- It is excluded via `.git/info/exclude` in the project repo (machine-local, not committed)
- If it appears in `git status`, run: `echo "CLAUDE.md" >> /Users/mdproctor/claude/quarkus-langchain4j/.git/info/exclude`
- Never run `git add CLAUDE.md` in the project repo — it will commit the symlink, not the content
- `AGENTIC-NATIVE-AUDIT.md` and `ARC42STORIES.MD` live in the workspace — do not copy them to the project repo

## Navigation

- In workspace: `proj/` → project repo (`ls proj/` to browse the project)
- In project: `wksp/` → workspace (`ls wksp/` to browse the workspace)

## Writing Style Guide

**The writing style guide at `~/claude-workspace/writing-styles/blog-technical.md` is mandatory for all blog and diary entries.** Load it in full before drafting. Complete the pre-draft voice classification (I / we / Claude-named) before generating any prose. Do not show a draft without verifying it against the style guide.

---

## Routing

All methodology artifacts live in the workspace. The project repo contains source code only.

| Artifact | Destination | Notes |
|----------|-------------|-------|
| protocols | workspace | /Users/mdproctor/claude/public/quarkus-langchain4j/protocols/ — alongside PLATFORM.md in workspace root |
| design | workspace | /Users/mdproctor/claude/public/quarkus-langchain4j/ARC42STORIES.MD |
| blog | workspace | /Users/mdproctor/claude/public/quarkus-langchain4j/blog/ |
| specs | workspace | /Users/mdproctor/claude/public/quarkus-langchain4j/specs/ |
| plans | workspace | /Users/mdproctor/claude/public/quarkus-langchain4j/plans/ |
| snapshots | workspace | /Users/mdproctor/claude/public/quarkus-langchain4j/snapshots/ |
| adr | workspace | /Users/mdproctor/claude/public/quarkus-langchain4j/adr/ |

**PLATFORM.md** lives at `/Users/mdproctor/claude/public/quarkus-langchain4j/PLATFORM.md` (workspace root). work-start's automatic check looks in `$PROJECT/docs/PLATFORM.md` — it won't find it there. Read it manually at the start of any agentic module work session.

---

## Project Type

type: java
workspace: ~/claude/public/quarkus-langchain4j
Issue tracking: declined

---

## Upstream Contributions

| Issue | Repo | Status | What | Blocks |
|-------|------|--------|------|--------|
| [#5376](https://github.com/langchain4j/langchain4j/issues/5376) | langchain4j/langchain4j | Filed | Pluggable `DefaultExecutorProvider` SPI for parallel agent execution | Nothing — C3 proceeds without it; simplifies Quarkus wiring if accepted |
| [#5377](https://github.com/langchain4j/langchain4j/issues/5377) | langchain4j/langchain4j | Filed | Generalise `ChatSupplierParameterResolver` to all supplier types | Nothing — C3 defers C-4 to this; `@ParallelExecutor` stays static until accepted |
| [#5378](https://github.com/langchain4j/langchain4j/issues/5378) | langchain4j/langchain4j | Filed | Allow `@ParallelExecutor` to accept DI-injected parameters | Blocked by #5377 |
| ~~[#5360](https://github.com/langchain4j/langchain4j/issues/5360)~~ | langchain4j/langchain4j | Closed | Parallel execution waits on submission order rather than completion order | — |

