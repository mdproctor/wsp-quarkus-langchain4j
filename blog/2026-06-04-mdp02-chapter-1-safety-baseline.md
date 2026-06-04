# Chapter 1: The Safety Baseline

The Chapter 1 work on the agentic module is shipped. The goal was straightforward: establish a safe baseline before any feature work lands. Eight audit findings covering infrastructure, CDI integration, FaultTolerance safety, and a config namespace. In practice most of them took about as long as expected. Two didn't.

## The annotation inheritance problem

F-7 was audited as a one-word fix — change `declaredAnnotations()` to `annotations()`. Jandex, it turns out, doesn't work that way. `ClassInfo.annotations()` returns all annotations anywhere within the class body, including on methods and fields. It does not traverse parent interfaces. Java's `@Inherited` meta-annotation only applies to class-to-subclass inheritance, not interface extension. Jandex reflects this faithfully: parent interface annotations simply aren't there.

The fix required two parts. First, a BFS traversal using the Jandex index to walk the full interface hierarchy — straightforward once you know what you're actually looking for. Second, a more awkward problem: Arc's synthetic bean interceptor resolver has the same limitation. Even after `hasAnyInterceptorBindings` correctly returns `true` (triggering `injectInterceptionProxy()`), Arc needs to find the interceptor binding annotation on the bean's interface in the Jandex index, not just know that one exists somewhere in the hierarchy.

The fix that works: `AnnotationsTransformerBuildItem`. We added a build step that modifies the Jandex index entry for the agent interface, propagating interceptor bindings from parent interfaces before Arc processes beans. `SyntheticBeanBuildItem.ExtendedBeanConfigurator` has no `addInterceptorBinding()` method — the index is the correct target. Verified in a test using a custom `@Traced` interceptor binding on a parent interface, confirming the interceptor actually fires.

One wrinkle: `AnnotationInstance` objects carry their original `target()` reference when propagated to a child class via `ctx.addAll()`. Arc uses annotation names and values, not the target, so interception works correctly. But any code inspecting `ann.target()` on the child will see the parent's `ClassInfo`. The rebuild pattern (`AnnotationInstance.builder(ann.name()).buildWithTarget(childClassInfo)`) fixes this if it ever matters.

## A FaultTolerance edge case

When we added `@CircuitBreaker` to the FaultTolerance test, the assertion needed adjusting: `CircuitBreakerOpenException` isn't wrapped in `AgentInvocationException`. The SmallRye interceptor fires at a higher CDI priority than the agent invocation handler, so the open-circuit exception escapes the wrapper entirely. Any catch block targeting `AgentInvocationException` to detect an open circuit will silently miss it — catch `CircuitBreakerOpenException` directly instead.

## On naming

The work throughout these documents had been described as "Quarkus-native integration" — shorthand for making the agentic module a proper first-class Quarkus integration using CDI, OTel, Micrometer, and the rest. A colleague pointed out the obvious problem: "Quarkus native" means GraalVM native image. The term was causing the wrong mental model before anyone read the first sentence. Renamed to "first-class Quarkus integration" everywhere — which is what the original goal statement already said, so this was correcting a drift rather than introducing something new.
