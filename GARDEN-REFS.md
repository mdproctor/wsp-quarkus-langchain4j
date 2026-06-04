# Garden Reference — quarkus-langchain4j

Curated garden entries relevant to this project. Consult before writing specs or
brainstorming. Entries are in `~/.hortora/garden/`; search with:
`git -C ~/.hortora/garden show HEAD:<domain>/<GE-ID>.md`

---

## Quarkus Extension Development (Build Steps, Recorders, Dev UI)

| GE | Title |
|----|-------|
| [jvm/GE-20260604-3ae124](jvm/GE-20260604-3ae124.md) | ConfigPhase.RUN_TIME config cannot be injected as @BuildStep parameter — use RuntimeValue<T> in recorder |
| [jvm/GE-20260604-9d91f9](jvm/GE-20260604-9d91f9.md) | Propagate inherited interceptor bindings to synthetic CDI beans via AnnotationsTransformerBuildItem |
| [jvm/GE-20260604-709d74](jvm/GE-20260604-709d74.md) | AnnotationInstance.target() carries stale parent ClassInfo when propagated to child via ctx.addAll() |
| [jvm/GE-20260604-76c3f9](jvm/GE-20260604-76c3f9.md) | String DotNames for optional library annotations in Quarkus deployment modules |
| [jvm/GE-20260512-b3f32a](jvm/GE-20260512-b3f32a.md) | @IfBuildProperty/@UnlessBuildProperty evaluated at augmentation only — QuarkusTestProfile properties have no effect on bean activation |
| [jvm/GE-20260512-c30f52](jvm/GE-20260512-c30f52.md) | @QuarkusIntegrationTest in the runtime module causes class loading failures — separate integration-tests/ module required |
| [jvm/GE-20260531-bd4b53](jvm/GE-20260531-bd4b53.md) | quarkus-langchain4j-anthropic fails with 'Run time configuration cannot be consumed in Build Steps' on Quarkus 3.33+ |

## CDI and Interceptors

| GE | Title |
|----|-------|
| [jvm/GE-20260512-0fe012](jvm/GE-20260512-0fe012.md) | CDI fireAsync() inside @Transactional dispatches immediately — observer can run before the triggering transaction commits |
| [jvm/GE-20260512-6887c9](jvm/GE-20260512-6887c9.md) | @ObservesAsync + @Transactional on the same method is unreliable — delegate transactional logic to a separate bean |
| [jvm/GE-20260512-a9ad9f](jvm/GE-20260512-a9ad9f.md) | Raw ExecutorService drops CDI context — @Transactional silently broken on background threads |
| [jvm/GE-20260512-e552f7](jvm/GE-20260512-e552f7.md) | @ApplicationScoped bean state persists across @QuarkusTest classes — tests pass in isolation but fail in suite |
| [jvm/GE-20260512-c246b0](jvm/GE-20260512-c246b0.md) | Test Quarkus CDI SPI implementations with @Alternative static inner classes — Mockito cannot be injected as CDI beans |
| [jvm/GE-20260512-66d997](jvm/GE-20260512-66d997.md) | Panache static methods bypass CDI @Alternative stores — returns empty results silently |

## Concurrency and Context Propagation

| GE | Title |
|----|-------|
| [jvm/GE-20260501-56e179](jvm/GE-20260501-56e179.md) | ThreadLocal set on calling thread is invisible inside CompletableFuture.supplyAsync() |
| [jvm/GE-20260428-f075ef](jvm/GE-20260428-f075ef.md) | Race-free CompletableFuture per-item pattern for CDI async event tests |

## FaultTolerance and Agent Integration

| GE | Title |
|----|-------|
| [jvm/GE-20260604-5bb2e7](jvm/GE-20260604-5bb2e7.md) | CircuitBreakerOpenException escapes AgentInvocationException wrapper in Quarkus langchain4j agents |

## Maven / Multi-Module Build

| GE | Title |
|----|-------|
| [java/GE-0064](java/GE-0064.md) | Maven aggregator pom requires `<packaging>pom</packaging>` — omitting it causes a cryptic build failure |
| [java/GE-0067](java/GE-0067.md) | Maven ignores non-Java files in `src/main/java/` — resources must be in `src/main/resources/` |
| [java/GE-0144](java/GE-0144.md) | Maven incremental build passes but `NoClassDefFoundError` at runtime — stale `.class` files |
| [java/GE-0158](java/GE-0158.md) | Use `mvn compile` to enumerate all call sites when changing a Java record signature |
| [java/GE-20260416-7ec461](java/GE-20260416-7ec461.md) | Maven `-am -Dtest=ClassName` propagates test filter to all upstream modules — 'No tests matching pattern' on unrelated modules |
| [java/GE-20260417-96accd](java/GE-20260417-96accd.md) | Maven multi-module cycle: adding a module as test-scope dep when it already depends on you |
| [java/GE-20260418-93f8b2](java/GE-20260418-93f8b2.md) | Maven duplicate dependency declarations — test scope silently overrides compile scope |

## Quarkus Test Patterns

| GE | Title |
|----|-------|
| [jvm/GE-20260512-493c90](jvm/GE-20260512-493c90.md) | @QuarkusTest classes named *IT.java silently report 0 tests — maven-failsafe collects them |
| [jvm/GE-20260512-50b394](jvm/GE-20260512-50b394.md) | Use @TestTransaction + unique identifiers to prevent @Scheduled interference in Quarkus tests |
| [jvm/GE-20260520-48e1d4](jvm/GE-20260520-48e1d4.md) | @BuildStep producing ExcludedTypeBuildItem silently not invoked during Quarkus test augmentation from workspace deployment module |

---

*Update this file when adding garden entries relevant to this project.*
*Search the full garden: `git -C ~/.hortora/garden grep -i "<keyword>" HEAD -- '*.md' ':!GARDEN.md'`*
