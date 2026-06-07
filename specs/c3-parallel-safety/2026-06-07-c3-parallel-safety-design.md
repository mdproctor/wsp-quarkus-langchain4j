# C3 — Parallel Safety Design Spec

**Date:** 2026-06-07
**Branch:** `c3-parallel-safety`
**Audit refs:** C-1, C-2, C-3, C-4
**Upstream issues:** [#5376](https://github.com/langchain4j/langchain4j/issues/5376), [#5377](https://github.com/langchain4j/langchain4j/issues/5377), [#5378](https://github.com/langchain4j/langchain4j/issues/5378)

---

## Problem

`@ParallelAgent` execution dispatches sub-agents to worker threads via `CompletableFuture.supplyAsync()`. The default executor is `DefaultExecutorProvider.getDefaultExecutorService()` — `Executors.newVirtualThreadPerTaskExecutor()` — raw virtual threads with no context propagation.

On these worker threads:
- `@RequestScoped` CDI beans throw `ContextNotActiveException`
- OTel spans are orphaned (no parent linkage)
- `SecurityIdentity` is null
- Any CDI interceptor relying on active contexts fails silently

This affects every `@ParallelAgent` that doesn't declare an explicit `@ParallelExecutor` — it's a general bug, not a Quarkus-specific gap.

## Scope Reduction

Four audit findings were analysed. Only C-1 requires implementation:

| Finding | Disposition |
|---|---|
| **C-1** Default parallel executor loses CDI/OTel/Security | **Fix in this chapter** — replace `DefaultExecutorProvider` fallback with `ManagedExecutor` via pluggable SPI |
| **C-2** `LangChain4jManaged.CURRENT` ThreadLocal not propagated | **Non-issue** — upstream sets it fresh on worker threads via explicit parameter passing in `AgentInvoker.internalInvoke()`, not ThreadLocal inheritance |
| **C-3** CDI request context missing on worker threads | **Subsumed by C-1** — `ManagedExecutor` activates request context automatically |
| **C-4** `@ParallelExecutor` can't inject CDI beans | **Deferred to upstream** — requires generalised `SupplierParameterResolver` SPI ([#5377](https://github.com/langchain4j/langchain4j/issues/5377), [#5378](https://github.com/langchain4j/langchain4j/issues/5378)) |

## Design

### Approach

Implement upstream issue [#5376](https://github.com/langchain4j/langchain4j/issues/5376) locally — make `DefaultExecutorProvider` pluggable with both a static setter and `ServiceLoader` discovery. Quarkus registers `ManagedExecutor` (MicroProfile Context Propagation, SmallRye implementation) as the global default via a recorder method at `RUNTIME_INIT`.

`ManagedExecutor` propagates CDI request context, OTel spans, and `SecurityIdentity` to worker threads automatically.

### Why not per-agent injection via AgentBuilder?

The original design proposed calling `agentBuilder.executor(managedExecutor)` from `QuarkusAgenticContextConsumer`. This doesn't work for two reasons:

1. `agenticContext.agentBuilder()` returns `AgentBuilder`, which has no `executor()` method. That method lives on `AbstractServiceBuilder` — a separate class hierarchy. `AgentBuilder` is the leaf-agent builder (chatModel, contentRetriever, listener). `AbstractServiceBuilder` is the workflow builder (executor, subAgents).
2. The consumer is called per-sub-agent via `DeclarativeUtil.configureAgent()`, not on the root parallel builder that owns the dispatch executor.

The pluggable `DefaultExecutorProvider` SPI replaces the fallback globally — simpler, correct, and no per-agent wiring needed.

### Upstream interaction

The upstream `PlannerBasedInvocationHandler.parallelExecution()` fallback:

```java
Executor exec = executor != null ? executor : DefaultExecutorProvider.getDefaultExecutorService();
```

Quarkus replaces the `DefaultExecutorProvider` fallback with `ManagedExecutor` via the upstream SPI. When `executor` is null (no `@ParallelExecutor`), the fallback returns `ManagedExecutor` instead of raw virtual threads.

User-declared `@ParallelExecutor` is still respected — `configureParallel()` sets `executor` directly on the builder, so the `executor != null` check short-circuits before the fallback is reached.

### Changes

**1. Upstream (`langchain4j-core`)** — new `ExecutorServiceProvider` SPI:

```java
package dev.langchain4j.spi;

import java.util.concurrent.ExecutorService;

public interface ExecutorServiceProvider {
    ExecutorService get();
}
```

Modify `DefaultExecutorProvider` to check static override first, then `ServiceLoader`, then fall back to `newVirtualThreadPerTaskExecutor()`:
- Static setter: `DefaultExecutorProvider.setDefaultExecutorService(ExecutorService executorService)`
- The static override field must be `volatile` (concurrent reads from agent worker threads)
- `ServiceLoader` discovery is cached in the lazy holder (runs once on first access, not per-call)

**2. Quarkus (`agentic/runtime`)** — one recorder method:
- `AgenticRecorder.registerDefaultExecutorProvider()` at `@RuntimeInit`
- Calls `DefaultExecutorProvider.setDefaultExecutorService(Arc.container().instance(ManagedExecutor.class).get())`

**3. Quarkus (`agentic/deployment`)** — one `@BuildStep`:
- Invokes the recorder method at `RUNTIME_INIT`

**4. Quarkus (`agentic/deployment`)** — `@ParallelExecutor` info message:
- In the existing `validateParallelExecutor()` path, when `@ParallelExecutor` is detected, emit an info log: "Agent X declares @ParallelExecutor — automatic CDI/OTel/Security context propagation is bypassed. Ensure your executor propagates contexts if needed."

### Edge cases

- **Non-parallel agents** (`@SequenceAgent`, `@LoopAgent`): `DefaultExecutorProvider.getDefaultExecutorService()` is only called from `PlannerLoop.parallelExecution()`, which only runs when `agents.size() > 1`. Setting the global default has no effect on sequence/loop agents.
- **Multiple parallel agents in one app**: all share the same `ManagedExecutor` (it's `@ApplicationScoped`). Correct — they should share the managed thread pool.
- **`@ParallelExecutor` on parent interface**: still works. `configureParallel()` detects it via `selectMethod()` which scans the class. The global default is never reached.

## Testing

**1. Default executor propagation** — `@ParallelAgent` with two sub-agents, one using a `@RequestScoped` CDI bean in a tool. Verify request context is active on parallel worker threads — `@RequestScoped` bean is accessible from sub-agent tool without `ContextNotActiveException`.

**2. User-declared `@ParallelExecutor` respected** — `@ParallelAgent` with explicit `@ParallelExecutor` returning a custom executor. Verify Quarkus does not override it. Assert the info message about bypassed context propagation appears in the build log.

**3. SecurityIdentity propagation (best effort)** — `@ParallelAgent` where a sub-agent tool checks `SecurityIdentity`. Verify identity propagates to worker threads. Requires `quarkus-security` test infrastructure (`@TestSecurity` or basic auth). Best effort for C3 — if the agentic test module doesn't already have security test dependencies, defer to a follow-up.

**4. OTel span continuity (best effort)** — `@ParallelAgent` invoked inside a traced context. Assert sub-agent spans share the parent trace ID using `InMemorySpanExporter`. Full OTel test infrastructure lands in C4.

## PR framing

Two PRs:

**Upstream PR (`langchain4j-core`):** Frame per upstream-contribution-framing protocol — "The current `DefaultExecutorProvider` is not pluggable, preventing any DI framework from providing managed executors with context propagation. This PR adds an `ExecutorServiceProvider` SPI and a static override." No Quarkus-specific references.

**Quarkus PR:** "Use the new `ExecutorServiceProvider` SPI to register `ManagedExecutor` as the default executor, fixing CDI/OTel/Security context loss on parallel agent worker threads." Lead with the bug, not the chapter. Include preamble linking to ARC42STORIES and audit for context on internal reference codes.

## Deferred work

| Item | Tracked by | Status |
|---|---|---|
| Pluggable `DefaultExecutorProvider` SPI (global default) | [langchain4j#5376](https://github.com/langchain4j/langchain4j/issues/5376) | **In scope** — implemented locally, proposed upstream |
| Generalise `ChatSupplierParameterResolver` to all supplier types | [langchain4j#5377](https://github.com/langchain4j/langchain4j/issues/5377) | Deferred |
| `@ParallelExecutor` DI parameter injection | [langchain4j#5378](https://github.com/langchain4j/langchain4j/issues/5378) | Deferred (blocked by #5377) |
