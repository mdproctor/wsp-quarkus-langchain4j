# Design Spec — C1 Quick Wins & Safety
**Date:** 2026-06-04  
**Issue:** quarkiverse/quarkus-langchain4j#2527  
**Branch:** feat/issue-2527-c1-quick-wins-safety  

---

## Scope

Remaining Chapter 1 items after D-2 (shipped in #2526):

| ID | Description | Kind |
|----|-------------|------|
| D-1 | `jsonRpcProvider` build step not gated on dev mode | One-liner fix |
| C-7 | `loadClassSafe` uses TCCL — unreliable on Vert.x / virtual threads | One-liner fix |
| F-7 | `hasAnyInterceptorBindings` misses bindings on parent interfaces | Correctness fix |
| S-3 | `markCdiBeanParametersAsUnremovable` covers only `@ChatModelSupplier` | Loop fix |
| F-3 | `@Fallback(fallbackMethod=...)` always fails at runtime on proxy classes | Build error |
| F-4 | `@Retry(retryOn=...)` targets types that agent wraps in `AgentInvocationException` | Build warning |
| F-5 | `@Transactional + @Retry` on agent method — partial commits + stale scope | Build warning |
| C-6 | `parallelExecution()` sequential `future.get()` loop | Upstream PR |

---

## D-1 — `jsonRpcProvider` dev-mode gate

**File:** `agentic/deployment/.../devui/AgenticDevUIProcessor.java`

Add `(onlyIf = IsDevelopment.class)` to the `@BuildStep` on `jsonRpcProvider`. Three other build steps in the same class already carry this guard. Without it, `AgenticJsonRpcService` is registered as a JSON-RPC provider in production and staging profiles.

**Test:** `@QuarkusUnitTest` that overrides `quarkus.profile=prod` and asserts `AgenticJsonRpcService` is not resolvable via CDI (or that `JsonRPCProvidersBuildItem` is not produced).

Actually: the cleanest test verifies that the application boots in a production-like profile without registering the service. Use `QuarkusUnitTest.overrideConfigKey("quarkus.profile", "prod")` and assert the bean is absent.

---

## C-7 — `loadClassSafe` classloader

**File:** `agentic/runtime/.../AgenticRecorder.java`

Replace `Thread.currentThread().getContextClassLoader()` with `AgenticRecorder.class.getClassLoader()` in:
- `loadClassSafe(AiAgentCreateInfo info)`
- `eagerlyInitRootAgents(Set<String> rootAgentClassNames)` — same TCCL call at line 72

**Rationale:** TCCL is not the deployment classloader on Vert.x I/O threads or virtual threads scheduled by `Executors.newVirtualThreadPerTaskExecutor()`. The recorder's own classloader is always the correct deployment classloader.

**Test:** No viable unit test path — the failure mode requires execution on a Vert.x worker thread. Add code comment citing the failure mode. Do not write a compilation-only test.

---

## F-7 — Inherited interceptor binding detection

**File:** `agentic/deployment/.../AgenticProcessor.java`

### Problem

`hasAnyInterceptorBindings` misses interceptor bindings declared on parent interface class declarations and parent interface methods. Two scenarios:

**Scenario A — class-level on parent:**
```java
@Timeout(1000)
interface BaseAgent { ... }
interface MyAgent extends BaseAgent { ... }  // @Timeout missed
```

**Scenario B — method-level on parent:**
```java
interface BaseAgent {
    @Timeout(1000)
    String chat(String input);
}
interface MyAgent extends BaseAgent {
    @Agent("...")  // @Timeout on BaseAgent.chat() missed
    String chat(String input);
}
```

### Fix

Extract three helpers:

**`transitiveInterfaces(ClassInfo start, IndexView index) → Set<DotName>`**  
BFS walk of the interface hierarchy. Returns the agent interface itself plus all transitive parent interfaces.

**`hasAnnotation(Collection<AnnotationInstance>, Set<DotName>) → boolean`**  
Shared predicate — returns true if any annotation name is in the set.

**`hasAnyInterceptorBindings(DetectedAiAgentBuildItem, Set<DotName>, IndexView) → boolean`**  
New signature adds `IndexView`. Two checks:
1. **Class-level:** for each interface in `transitiveInterfaces`, check `classInfo.classAnnotations()`
2. **Method-level:** for each agentic method, check `method.annotations()` PLUS for each parent interface in the hierarchy, look up the matching method by `classInfo.method(method.name(), method.parameterTypes())` and check its annotations

**Caller change:** `cdiSupport` gains `CombinedIndexBuildItem indexBuildItem` parameter; passes `indexBuildItem.getIndex()` to `hasAnyInterceptorBindings`.

**Test:** Agent extending a base interface where `@Timeout` is on the base interface class AND on a base interface method — verify `hasInterceptorBindings = true` and `injectInterceptionProxy()` is called (observable via correct timeout behavior at runtime).

---

## S-3 — `markCdiBeanParametersAsUnremovable` covers all supplier types

**File:** `agentic/deployment/.../AgenticProcessor.java`

### Problem

Current implementation:
```java
MethodInfo chatModelSupplier = item.getChatModelSupplier();
if (chatModelSupplier == null) continue;
// only checks chatModelSupplier.parameters()
```

Any other supplier type annotated with `@CdiBean` (e.g., `@ContentRetrieverSupplier`, `@AgentListenerSupplier`) will silently produce `UnsatisfiedResolutionException` at runtime.

### Fix

Rewrite to scan all methods on the agent interface for any of the nine supplier annotations, then check parameters for `@CdiBean`:

```
ALL_SUPPLIER_ANNOTATIONS = [
    CHAT_MODEL_SUPPLIER, CHAT_MEMORY_SUPPLIER, CHAT_MEMORY_PROVIDER_SUPPLIER,
    CONTENT_RETRIEVER_SUPPLIER, RETRIEVAL_AUGMENTER_SUPPLIER,
    TOOL_SUPPLIER, TOOL_PROVIDER_SUPPLIER, AGENT_LISTENER_SUPPLIER,
    PARALLEL_EXECUTOR
]

for item in agents:
    for method in item.getIface().methods():
        if method has any ALL_SUPPLIER_ANNOTATIONS:
            for param in method.parameters():
                if param has @CdiBean:
                    produce UnremovableBeanBuildItem(param.type().name())
```

**Test:** Agent with `@ContentRetrieverSupplier static ContentRetriever retrieve(@CdiBean ContentRetriever cr)` — verify app boots without `UnsatisfiedResolutionException`.

---

## F-3, F-4, F-5 — FaultTolerance interaction validator

**File:** `agentic/deployment/.../AgenticProcessor.java`  
**New method:** `validateFaultToleranceInteractions`

```java
@BuildStep
@Produce(ServiceStartBuildItem.class)
void validateFaultToleranceInteractions(
        List<DetectedAiAgentBuildItem> agents,
        CombinedIndexBuildItem indexBuildItem)
```

Uses string-based DotNames to avoid hard compile dependency on SmallRye FaultTolerance or Jakarta Transactions:
```
FALLBACK   = "org.eclipse.microprofile.faulttolerance.Fallback"
RETRY      = "org.eclipse.microprofile.faulttolerance.Retry"
TRANSACTIONAL = "jakarta.transaction.Transactional"
AGENT_INVOCATION_EXCEPTION = "dev.langchain4j.agentic.agent.AgentInvocationException"
```

### F-3 — `@Fallback(fallbackMethod=...)` is a hard build error

Check both class-level and method-level `@Fallback` annotations. When `fallbackMethod` attribute is non-empty, throw `IllegalConfigurationException`:

> "Agent interface 'X' method 'Y' uses @Fallback(fallbackMethod=\"Z\"). Agent interfaces are dynamic proxies — fallback method name resolution always fails at runtime with FaultToleranceDefinitionException. Use FallbackHandler<T> instead: @Fallback(YourFallbackHandler.class)"

**Test:** `@QuarkusUnitTest` with `assertException` — verify `IllegalConfigurationException` with the expected message fragment.

### F-4 — `@Retry(retryOn=...)` warning

Condition: `@Retry` is present on an agent method AND `retryOn` attribute is set AND none of the listed types is `AgentInvocationException`, `RuntimeException`, `Exception`, or `Throwable`.

Log at WARN:
> "Agent method 'X#Y' uses @Retry(retryOn=...) but agent exceptions are wrapped in AgentInvocationException. The retryOn types [Z] will not match the thrown exception. Add AgentInvocationException to retryOn, or remove retryOn to retry on all exceptions."

**Test:** App boots without exception when `@Retry(retryOn=SomeCheckedException.class)` is on an agent method. Warning is build-time only and not assertable via `@QuarkusUnitTest` log capture — test verifies no boot failure; warning is manual build log inspection.

### F-5 — `@Transactional + @Retry` warning

Condition: BOTH `@Transactional` AND `@Retry` are on the same agent method (not `@Transactional` alone — that is valid for read-only agent methods).

Log at WARN:
> "Agent method 'X#Y' combines @Transactional and @Retry. AgenticScope is not a JTA resource — on retry, the second attempt re-enters after the first transaction closed, leaving any partial scope state unrolled. Ensure this combination is intentional."

**Test:** Same pattern as F-4 — app boots without exception; warning is build-time only.

---

## C-6 — Upstream PR to `langchain4j-agentic`

**No Quarkus-side change.** Per `protocols/upstream/upstream-contribution-framing.md`, the `CompletableFuture.allOf()` fix in `PlannerBasedInvocationHandler.parallelExecution()` belongs in the upstream `langchain4j-agentic` repository.

**Work on this branch:** File upstream PR to `langchain4j-agentic` with the fix framed as a platform-agnostic performance improvement ("parallel execution currently waits on submission order, not completion order, wasting wall-clock time when slower tasks are submitted first").

---

## Helper refactoring note

`transitiveInterfaces` and `hasAnnotation` introduced for F-7 are pure utility methods. If S-3 or future build steps also need interface traversal, they should reuse `transitiveInterfaces`. Do not duplicate.

---

## Implementation order

Sequential, each with TDD before coding:

1. D-1 (trivial — establishes test infrastructure baseline)
2. C-7 (code comment only, one-liner fix)
3. S-3 (self-contained loop fix)
4. F-7 (most structural — adds helpers, modifies `cdiSupport` signature)
5. F-3 (new build step — errors)
6. F-4, F-5 (warnings, same build step as F-3)
7. C-6 (upstream PR, filed on this branch)

All items committed together in a single PR (#2527) after code review.
