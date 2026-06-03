# quarkus-langchain4j Workspace

**Project repo:** /Users/mdproctor/claude/quarkus-langchain4j
**Workspace:** ~/claude/public/quarkus-langchain4j
**Workspace type:** public

## Session Start

Run these two before any other work:
```
add-dir /Users/mdproctor/claude/quarkus-langchain4j
add-dir /Users/mdproctor/claude/public/quarkus-langchain4j
```

## Artifact Locations

| Skill | Writes to |
|-------|-----------|
| brainstorming (specs) | `specs/` |
| writing-plans (plans) | `plans/` |
| handover | `HANDOFF.md` |
| idea-log | `IDEAS.md` |
| design-snapshot | `snapshots/` |
| java-update-design / update-primary-doc | `ARC42STORIES.MD` (workspace) |
| adr | `adr/` |
| write-content | `blog/` |

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
- **Workspace** (`~/claude/public/quarkus-langchain4j`) — plans, blog, snapshots, handover
- **Project repo** (`/Users/mdproctor/claude/quarkus-langchain4j`) — source code only

Never rely on CWD for git operations. Always use explicit paths:
```bash
git -C /Users/mdproctor/claude/public/quarkus-langchain4j add <file>    # workspace
git -C /Users/mdproctor/claude/quarkus-langchain4j add <file>           # project
```

## Git Discipline — CLAUDE.md Symlink

**CLAUDE.md in the project repo is a symlink pointing here. Never stage or commit it.**

- It is excluded via `.git/info/exclude` in the project repo (machine-local, not committed)
- If it appears in `git status`, run: `echo "CLAUDE.md" >> /Users/mdproctor/claude/quarkus-langchain4j/.git/info/exclude`
- Never run `git add CLAUDE.md` in the project repo — it will commit the symlink, not the content
- `AGENTIC-NATIVE-AUDIT.md` and `ARC42STORIES.MD` live in the workspace — do not copy them to the project repo

## Navigation

- In workspace: `proj/` → project repo (`ls proj/` to browse the project)
- In project: `wksp/` → workspace (`ls wksp/` to browse the workspace)

---

## Project Type

type: java
workspace: ~/claude/public/quarkus-langchain4j
Issue tracking: declined

---

# Fix: Jlama bootstrap failure with Quarkus 3.32.2

**Do NOT add CLAUDE.md to git or to any staged changes.**

---

## Problem

`quarkus-langchain4j-jlama 0.26.1` fails at `@QuarkusTest` bootstrap with:

```
java.lang.IllegalStateException: Unsupported value type: [ALL-UNNAMED]
  at io.quarkus.bootstrap.json.Json.appendValue(Json.java:541)
  at io.quarkus.bootstrap.app.ApplicationModelSerializer.writeJson(...)
```

## Root Cause

The file `model-providers/jlama/runtime/pom.xml` configures the Quarkus Maven
plugin with:

```xml
<devMode>
    <jvmOptions>
        <add-modules>jdk.incubator.vector</add-modules>
        <enable-preview />
        <enable-native-access>ALL-UNNAMED</enable-native-access>
    </jvmOptions>
</devMode>
```

The Quarkus Maven plugin generates `META-INF/quarkus-extension.properties`
in the runtime JAR containing:

```
dev-mode.jvm-option.std.enable-native-access=ALL-UNNAMED
dev-mode.jvm-option.std.add-modules=jdk.incubator.vector
```

When running `@QuarkusTest`, Quarkus bootstraps the application model from
extension properties — including dev-mode JVM options. It tries to serialize
the app model to a JSON cache via `ApplicationModelSerializer`. The
`Json.appendValue()` method encounters the `enable-native-access=ALL-UNNAMED`
value, resolves `ALL-UNNAMED` to a `java.lang.Module` object, and fails because
it has no case for `Module` type.

## The Fix

The `enable-native-access=ALL-UNNAMED` (and possibly `add-modules=jdk.incubator.vector`)
is only needed on Java < 23. The Panama FFM/Vector API graduated to a standard
feature in Java 23 (JEP 469) and no longer requires `--enable-native-access`
or `--add-modules jdk.incubator.vector` on Java 23+.

**Option A — Remove the JVM options entirely** (if minimum Java is raised to 23):
Remove the `<devMode><jvmOptions>` block from `runtime/pom.xml`.

**Option B — Conditionalize on Java version** (keeps Java 21/22 support):
Use a Maven profile activated on `${java.version}` < 23 to include the
`<devMode><jvmOptions>` block only when needed.

**Option C — Fix in Quarkus core** (correct fix for all extensions):
In `io.quarkus.bootstrap.json.Json.appendValue()`, add a case for `Module`
objects (call `.toString()` on them, or handle the `[ALL-UNNAMED]` constant).
This is a Quarkus bug — the serializer should handle all JVM arg value types.
This fix would be submitted as a Quarkus PR, not in this repo.

## Recommended approach

Start with **Option A** — check if raising the minimum Java version to 23 is
acceptable for this extension (the extension yaml already targets
`minimum-java-version: "21"` but the JARs were built with Java 23).

If Java 21/22 support must be kept, use **Option B**.

Either way, the fix is in:
- `model-providers/jlama/runtime/pom.xml` (the JVM options declaration)

After fixing, build and install locally:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl model-providers/jlama -DskipTests -Dno-format
```

Then test by running the Qhorus examples module which uses this extension:
```bash
cd /Users/mdproctor/claude/quarkus-qhorus
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -pl examples/agent-communication -Dno-format
```

If tests pass and the `Unsupported value type: [ALL-UNNAMED]` error is gone,
the fix is correct. Submit a PR to https://github.com/quarkiverse/quarkus-langchain4j.

## Key files

| File | Role |
|---|---|
| `model-providers/jlama/runtime/pom.xml` | Source of the JVM options (lines ~64-74) |
| `model-providers/jlama/runtime/src/main/java/...` | Runtime code (probably unchanged) |
| `model-providers/jlama/deployment/src/main/java/.../JlamaProcessor.java` | Deployment build steps (probably unchanged) |

## What to avoid

- Do not add CLAUDE.md to git or staged files — it is a symlink to the workspace
- Do not bump the quarkus-langchain4j-jlama version in `quarkus-qhorus/examples/agent-communication/pom.xml` until after testing locally
- The fix is purely in the build metadata — the Java source code likely does not need changes
