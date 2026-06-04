# Protocols Index — quarkus-langchain4j

One rule per file. Self-contained and retrievable independently.
Run `PLATFORM.md` before implementing anything in the `agentic` module.

## quarkus/
| File | Rule |
|------|------|
| [devui-runtime-dev-module.md](quarkus/devui-runtime-dev-module.md) | Dev UI JSON-RPC services belong in runtime-dev modules — production binaries must not include them |

## upstream/
| File | Rule |
|------|------|
| [upstream-contribution-framing.md](upstream/upstream-contribution-framing.md) | Frame langchain4j-agentic improvements as DI-neutral, never Quarkus-specific |

## cdi/
| File | Rule |
|------|------|

## agentic/
| File | Rule |
|------|------|
| [transitive-interface-traversal.md](agentic/transitive-interface-traversal.md) | Always walk the full interface hierarchy (ValidationUtil.transitiveInterfaces) when checking annotations on agent ClassInfo objects |
| [build-time-warning-precision.md](agentic/build-time-warning-precision.md) | Build-time warnings fire on the exact dangerous combination only — not a valid superset |
