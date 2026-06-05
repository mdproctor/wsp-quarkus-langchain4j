# Infrastructure, Protocols, and a Proper Maven Fix

A session that started with a reviewer comment and ended with improved tooling for every future project.

## The runtime-dev fix

PR #2526 came back with a clear and correct observation: `AgenticJsonRpcService` shouldn't exist in production binaries at all. Gating the registration with `onlyIf = IsDevelopment.class` prevents the service from being used, but the class is still there.

The fix is a `runtime-dev` Maven module — a separate artifact that's on the deployment classpath in dev mode but excluded from production builds. We created `quarkus-langchain4j-agentic-dev`, moved `AgenticJsonRpcService` there, and wired it so the deployment module pulls it in. The allow-list field moved from `AgenticJsonRpcService` to `DevAgentMonitorHolder` (which stays in `runtime`) so the recorder and the service can both reach it without a circular dependency.

This is now a protocol. Any Quarkus extension implementing Dev UI JSON-RPC should follow the same structure. The protocol is in `protocols/universal/` because it applies to any extension, not just this one.

## The tooling improvements

The session surfaced that skills in `~/.claude/skills/` installed from Hortora have their source in `~/claude/hortora/soredium/`. Editing the installed copy is silently overwritten on the next sync — not obvious, and painful when discovered. Captured in the garden.

The `protocol` skill had hardcoded casehub-specific conventions (`universal/`, `casehub/` subdirectories, `FOUNDATION-INDEX.md`, `HARNESS-INDEX.md`). Looking at how casehub/parent actually structures its protocols revealed the right abstraction: a three-tier model with a navigation-hub top-level `INDEX.md`, sub-indexes per tier with three-column tables (File | Rule Summary | Applies To), and tier directories that reflect the project's own naming. The skill now documents this instead of encoding one project's choices.

The workspace protocols were restructured to match: `universal/` for any-Quarkus-extension rules, `agentic/` for this module, `upstream/` for contribution framing. Three-column indexes throughout.

`work-start` and `work-end` were fixed to check `$WORKSPACE/PLATFORM.md` and `$WORKSPACE/protocols/` — the two-repo layout was previously a workaround rather than a first-class configuration.

## The GARDEN-REFS.md idea

The garden has accumulated a substantial body of Quarkus extension knowledge — CDI gotchas, Maven multi-module patterns, build-step behaviors, FaultTolerance edge cases. None of it was connected to this project. A `GARDEN-REFS.md` in the workspace now curates the most relevant entries across six categories. It's in CLAUDE.md with an instruction to consult it before brainstorming.

The MCP test failures in CI are pre-existing — `McpRegistryClientTest` tries to reach `registry.modelcontextprotocol.io` and times out. Unrelated to the agentic module changes.
