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
| **C-1** Default parallel executor loses CDI/OTel/Security | **Fix in this chapter** — inject `ManagedExecutor` |
| **C-2** `LangChain4jManaged.CURRENT` ThreadLocal not propagated | **Non-issue** — upstream sets it fresh on worker threads via explicit parameter passing in `AgentInvoker.internalInvoke()`, not ThreadLocal inheritance |
| **C-3** CDI request context missing on worker threads | **Subsumed by C-1** — `ManagedExecutor` activates request context automatically |
| **C-4** `@ParallelExecutor` can't inject CDI beans | **Deferred to upstream** — requires generalised `SupplierParameterResolver` SPI ([#5377](https://github.com/langchain4j/langchain4j/issues/5377), [#5378](https://github.com/langchain4j/langchain4j/issues/5378)) |

## Design

### Approach

Inject `ManagedExecutor` (MicroProfile Context Propagation, SmallRye implementation) as the default executor for parallel agents that don't declare `@ParallelExecutor`. Same pattern as C2's CDI auto-wiring for supplier types.

`ManagedExecutor` propagates CDI request context, OTel spans, and `SecurityIdentity` to worker threads automatically.

### Upstream interaction

The upstream `PlannerBasedInvocationHandler.parallelExecution()` fallback is unchanged:

```java
Executor exec = executor != null ? executor : DefaultExecutorProvider.getDefaultExecutorService();
```

Quarkus ensures `executor` is never null for agents without `@ParallelExecutor`, so the fallback to raw virtual threads is never reached.

User-declared `@ParallelExecutor` is respected — the upstream sets it in `ParallelAgentServiceImpl.configureParallel()` before our consumer runs, and our consumer skips the injection when `hasParallelExecutor` is true.

### Changes

**1. `AiAgentCreateInfo`** — add `boolean hasParallelExecutor`

**2. `AgenticProcessor`** — two additions:
- Detect `@ParallelExecutor` on the agent interface (using `ValidationUtil.transitiveInterfaces()` per protocol) and pass `hasParallelExecutor` to `AiAgentCreateInfo`
- Add `ManagedExecutor` as an injection point on the synthetic bean configurator (unconditional, like `AgentListener`)

**3. `QuarkusAgenticContextConsumer`** — in `accept()`, after AgentListener wiring:
```java
if (!aiAgentCreateInfo.hasParallelExecutor()) {
    ManagedExecutor executor = cdiContext.getInjectedReference(ManagedExecutor.class);
    agentBuilder.executor(executor);
}
```

### Edge cases

- **Non-parallel agents** (`@SequenceAgent`, `@LoopAgent`): executor is injected but never used by `parallelExecution()`. No harm, no overhead.
- **Multiple agents in one app**: per-agent `AiAgentCreateInfo` — each has its own `hasParallelExecutor` flag.
- **`@ParallelExecutor` on parent interface**: detected via transitive interface traversal. Quarkus correctly skips injection.

## Testing

**1. Default executor propagation** — `@ParallelAgent` with two sub-agents, one using a `@RequestScoped` CDI bean in a tool. Without the fix: `ContextNotActiveException`. With the fix: request context is active.

**2. User-declared `@ParallelExecutor` respected** — `@ParallelAgent` with explicit `@ParallelExecutor` returning a custom executor. Verify Quarkus does not override it.

**3. SecurityIdentity propagation** — `@ParallelAgent` where a sub-agent tool checks `SecurityIdentity`. Verify identity propagates to worker threads.

**4. OTel span continuity (best effort)** — `@ParallelAgent` invoked inside a traced context. Assert sub-agent spans share the parent trace ID using `InMemorySpanExporter`. Full OTel test infrastructure lands in C4.

## PR framing

Lead with the bug: "parallel agents lose CDI/OTel/Security context on worker threads." The fix is a one-line executor swap that benefits all users of `@ParallelAgent`. Not a chapter milestone — a general correctness fix.

Include preamble linking to ARC42STORIES and audit for context on internal reference codes.

## Deferred work

| Item | Tracked by |
|---|---|
| Pluggable `DefaultExecutorProvider` SPI (global default) | [langchain4j#5376](https://github.com/langchain4j/langchain4j/issues/5376) |
| Generalise `ChatSupplierParameterResolver` to all supplier types | [langchain4j#5377](https://github.com/langchain4j/langchain4j/issues/5377) |
| `@ParallelExecutor` DI parameter injection | [langchain4j#5378](https://github.com/langchain4j/langchain4j/issues/5378) |
