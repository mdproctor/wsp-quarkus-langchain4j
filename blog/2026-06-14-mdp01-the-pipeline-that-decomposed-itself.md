---
layout: post
title: "The Pipeline That Decomposed Itself"
date: 2026-06-14
type: phase-update
entry_type: note
subtype: diary
projects: [quarkus-langchain4j]
tags: [rag, composition-annotations, design-review]
---

## The Pipeline That Decomposed Itself

The Foundation PR gave `@RegisterAiService` direct bean-class references — clean, type-safe, self-documenting. But `retrievalAugmentor` still took a single `Class<?>` pointing at a monolithic `RetrievalAugmentor` bean. Users were still writing `DefaultRetrievalAugmentor.builder()` by hand, composing retrievers, routers, transformers, aggregators, and injectors inside a CDI bean the framework couldn't see into.

`@RagPipeline` decomposes that into individual attributes. Each one is a CDI bean reference. The build step composes `DefaultRetrievalAugmentor` from them.

```java
@RegisterAiService(modelName = "gpt-4o")
@RagPipeline(
    retrievers = {ProductRetriever.class},
    transformer = HydeTransformer.class
)
public interface ProductAssistant { ... }
```

The interesting design decisions came through review, not through the initial spec.

**Two-state resolution, not tri-state.** The Foundation PR established a three-state model: `void.class` = skip, interface type = auto-discover, concrete class = explicit. The review caught that `@RagPipeline` should never auto-discover — the parent spec's principle was "explicit over auto-discover," and we'd just removed auto-discovery for retrieval augmentors in the Foundation PR. Re-introducing it through a back door would contradict our own decision. All six attributes use two-state: skip or explicit, nothing in between.

**Router and retrievers are mutually exclusive.** I initially had this as a warning — router overrides, retrievers ignored. The review pushed it to an error: there's no valid runtime reason to declare both. The router defines its own retrieval strategy. If you declare retrievers alongside it, you're confused, not configuring. Making it an error forces the choice.

**The injection point contract.** This one would have caused runtime failures. `SyntheticBeanBuildItem` requires every `ctx.getInjectedReference()` call in the `createWith` function to have a matching `.addInjectionPoint()` on the configurator. Without it, the code compiles, the deployment succeeds, and the bean throws at runtime with no useful error. The review caught this gap before any code was written — we added `addRagInjectionPoints()` as a shared helper that mirrors the resolution logic exactly.

Two modes fell out of the design. Companion mode puts `@RagPipeline` on the AI service interface — the augmentor is built inline, no separate bean. Standalone mode puts it on a separate interface, generating an `@ApplicationScoped` `RetrievalAugmentor` that multiple services can reference. The standalone mode is speculative — it might not survive community feedback, but it's cheap to carry and easy to remove.

The implementation itself was straightforward: `RagPipelineProcessor` scans, validates, produces build items. `RagPipelineSupport` is a static utility that both modes call. `AiServicesRecorder` delegates to it. The `retrievalAugmentor` attribute is gone from `@RegisterAiService` — thirteen callers migrated, all tests green.

The part worth remembering isn't the pipeline. It's that the three review cycles found a resolution model contradiction, a validation gap, and a runtime failure — all before any code existed. The spec was the expensive artifact; the implementation was mechanical.
