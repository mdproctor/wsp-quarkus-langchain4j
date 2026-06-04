# Protocols Index — quarkus-langchain4j

Standing rules for this project. Run `PLATFORM.md` before implementing anything in the `agentic` module.

Three tiers: **universal** (any Quarkus extension) → **agentic** (this module) → **upstream** (contribution rules).

---

## Universal Protocols

Rules applicable to any Quarkus extension. See [universal/INDEX.md](universal/INDEX.md) for the full table.

| File | Rule Summary | Applies To |
|------|-------------|------------|
| [universal/devui-runtime-dev-module.md](universal/devui-runtime-dev-module.md) | Dev UI JSON-RPC services belong in runtime-dev modules, not runtime | Any Quarkus extension using JsonRPCProvidersBuildItem |

---

## Agentic Module Protocols

Rules for building and extending `agentic/`. See [agentic/INDEX.md](agentic/INDEX.md) for the full table.

| File | Rule Summary | Applies To |
|------|-------------|------------|
| [agentic/transitive-interface-traversal.md](agentic/transitive-interface-traversal.md) | Walk the full interface hierarchy when checking annotations on agent ClassInfo objects | AgenticProcessor build steps |
| [agentic/build-time-warning-precision.md](agentic/build-time-warning-precision.md) | Build-time warnings fire on the exact dangerous combination only | AgenticProcessor validators |

---

## Upstream Contribution Protocols

Rules for proposing changes to `langchain4j-agentic`. See [upstream/INDEX.md](upstream/INDEX.md) for the full table.

| File | Rule Summary | Applies To |
|------|-------------|------------|
| [upstream/upstream-contribution-framing.md](upstream/upstream-contribution-framing.md) | Frame improvements as DI-neutral, never Quarkus-specific | Any upstream PR to langchain4j-agentic |
