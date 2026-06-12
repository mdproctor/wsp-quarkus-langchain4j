---
title: "Convergence and Cleanup"
date: 2026-06-12
sequence: mdp09
tags: [langchain4j-cdi, convergence, RegisterAiService, composition-annotations]
---

# Convergence and Cleanup

Two things happened today. First, the langchain4j-cdi analysis turned into something more interesting than a comparison table. Second, the @RegisterAiService supplier pattern finally died.

## The convergence roadmap

The original task was straightforward: look at langchain4j-cdi, figure out what ideas to adopt. But the interesting question turned out to be different. Three Java projects all integrate langchain4j with CDI — and they've each invented their own annotation surface. langchain4j-cdi has 11 topology annotations that duplicate upstream's. We wrap upstream's annotations and add build-time processing. The user API is different in every case.

The reframing: if langchain4j (upstream) and quarkus-langchain4j converge on shared SPIs and annotation attributes — with Mario already open to improvements — langchain4j-cdi faces a choice between joining the shared path or maintaining a divergent fork. Convergence creates gravity.

The roadmap identifies eight upstream proposals, all framework-agnostic. The centrepiece is adding optional attributes to upstream's `@Agent` annotations (scope, guardrails, listener) using `Class<? extends Annotation>` for scope (no CDI import needed — it's `java.lang.annotation`). This eliminates the reason langchain4j-cdi forked the annotations.

Six convergence limits are real but addressable: config namespaces, build-time vs. runtime validation, error philosophy, thread context propagation, GraalVM native, Dev UI. For each: share the contract, vary the mechanism.

## Killing the supplier pattern

The @RegisterAiService cleanup is the largest annotation surface change in the project's history. 17 sentinel marker classes — each throwing `UnsupportedOperationException("should never be called")` — existed solely to signal "use CDI" or "skip" to the processor. They're gone.

The replacement is a tri-state `Class<?>` model: `void.class` means disabled, the interface type means auto-discover, a concrete class means inject that specific bean. The processor logic went from string-comparison chains against sentinel class names to a clean `switch` on `ComponentResolutionMode`.

The spec design went through three review rounds. Key decisions: drop @MemoryConfig/@ToolConfig (didn't earn their existence), single transformer on @RagPipeline (upstream's QueryTransformer is fan-out, not pipeline), add `@RagPipeline(augmentor = ...)` for pre-built augmentors like EasyRetrievalAugmentor.

Framework changes are committed. The migration of ~100 test files using the old supplier attributes is next session's work — mechanical but large.
