---
layout: post
title: "When You Can't Shadow a JAR Class"
date: 2026-06-07
type: phase-update
entry_type: note
subtype: diary
projects: [quarkus-langchain4j]
tags: [quarkus, classloading, bytecode, parallel-agents, context-propagation]
---

## When You Can't Shadow a JAR Class

The agentic module runs `@ParallelAgent` sub-agents on virtual threads. Raw `Executors.newVirtualThreadPerTaskExecutor()` — no CDI request context, no OTel span linkage, no `SecurityIdentity`. Every `@RequestScoped` bean in a tool throws `ContextNotActiveException` on the worker thread. This affects every parallel agent that doesn't explicitly declare a `@ParallelExecutor` returning a managed executor. A general bug, not a Quarkus-specific gap.

The fix itself is small: register a `ManagedExecutor` as the default parallel executor. MicroProfile Context Propagation handles the rest — request context, OTel, security identity all propagate automatically. The interesting part was getting the executor into the right place.

### The design that didn't work

The upstream `DefaultExecutorProvider` in langchain4j-core is a utility class with a lazy static holder and no setter. I wanted to add one. The first instinct — class shadowing — seemed obvious: create a `dev.langchain4j.internal.DefaultExecutorProvider` in our runtime module with the setter added. Maven compiles it fine. The split-package warning in the Quarkus build log looks cosmetic.

It isn't. The Quarkus classloader doesn't guarantee extension module classes load before library JAR classes. We got `NoSuchMethodError` at runtime — the JVM loaded the upstream version (no setter) instead of ours. Worse, the failure was intermittent: some test classes loaded our version, others loaded the upstream. This makes it look like a test infrastructure issue rather than a fundamental classloading problem.

Claude went further off the path here — when class shadowing failed, it reached for `sun.misc.Unsafe` to overwrite the `static final` field on the upstream class. That works, technically, but it's fragile, non-portable, and the kind of fix that ages badly. I reverted it and we found the right approach.

### BytecodeTransformerBuildItem — the right mechanism

`BytecodeTransformerBuildItem` is Quarkus's standard tool for modifying third-party classes at build time. An ASM `ClassVisitor` adds a `volatile static` override field and a setter method to the upstream `DefaultExecutorProvider`, then prefixes `getDefaultExecutorService()` with an early-return guard that checks the override before falling through to the original holder. The transformation operates on whatever class bytes the Quarkus build pipeline resolved — no split-package ambiguity.

The recorder then calls the added setter via reflection at `RUNTIME_INIT`:

```java
Method setter = DefaultExecutorProvider.class
        .getMethod("setDefaultExecutorService", ExecutorService.class);
setter.invoke(null, Arc.container().instance(ManagedExecutor.class).get());
```

Reflection because the method is added at build time — the compiler doesn't know about it. One call at startup, before any agent is created.

### What the audit findings actually needed

I started with four concurrency findings from the audit (C-1 through C-4). Three turned out to need no work:

C-2 looked like `LangChain4jManaged.CURRENT` (a ThreadLocal carrying the agent scope) needed propagation to worker threads. Tracing the upstream code showed that `AgentInvoker.internalInvoke()` sets it fresh on each worker thread via explicit parameter passing — not ThreadLocal inheritance. The audit finding was misleading.

C-3 (CDI request context missing on workers) is subsumed by C-1 — `ManagedExecutor` activates it automatically.

C-4 (`@ParallelExecutor` can't inject CDI beans) requires an upstream SPI change to generalise `ChatSupplierParameterResolver` to all supplier types. Filed as [langchain4j#5377](https://github.com/langchain4j/langchain4j/issues/5377) and [#5378](https://github.com/langchain4j/langchain4j/issues/5378). The pluggable executor SPI itself is [#5376](https://github.com/langchain4j/langchain4j/issues/5376).

Three upstream issues filed, one actual code change needed. The scope reduction was more valuable than the implementation.
