---
layout: post
title: "Making agents visible"
date: 2026-06-08
type: phase-update
entry_type: note
subtype: diary
projects: [quarkus-langchain4j]
tags: [observability, opentelemetry, micrometer, cdi-events]
---

## The gap upstream

The upstream `langchain4j-agentic` library provides an `AgentListener` SPI and an `AgentMonitor` that collects execution traces into an in-memory tree. But that's where it stops. No OTel spans. No Micrometer metrics. No health checks. No CDI event bridge. The `AgentMonitor` feeds an HTML report generator for dev tooling — useful for debugging, but invisible to production monitoring.

A supervisor agent dispatching three sub-agents, each making two LLM calls, produces six flat sibling spans with no parent agent span. The trace is useless for understanding what happened.

## Four beans, four capabilities

The design is straightforward once you accept the constraint: each observability concern is a separate `AgentListener` CDI bean, conditionally registered at build time based on which Quarkus extensions are present.

`AgentSpanListener` injects `Tracer` and opens a span named `langchain4j.agent.<agentName>` in `beforeAgentInvocation`. Making the span current means child agent invocations and tool executions automatically nest underneath — the core module's `ToolSpanWrapper` already creates `langchain4j.tools.*` spans, so those appear as children without any additional wiring. I deliberately excluded tool spans from the agent listener to avoid duplicates.

`AgentMetricsListener` records three metrics: `gen_ai.agent.invocations` (counter), `gen_ai.agent.duration` (timer), and `gen_ai.agent.tool.executions` (counter). The counter uses an `error.type` tag — `none` for success, the exception class name for failures — so a single metric drives both success rate and error rate alerting.

`AgentCdiEventListener` fires `AgentStartedEvent`, `AgentCompletedEvent`, and `AgentErrorEvent` via synchronous `Event.fire()`. I chose synchronous delivery because `@ObservesAsync` silently loses OTel span context — a gotcha I'd already catalogued in the garden from earlier work. Users who need async dispatch can declare `@ObservesAsync` on their observer method and CDI handles the thread handoff.

`AgentHealthCheck` is a `@Readiness` probe that resolves every agent CDI bean at readiness check time. If any fails to resolve, the pod reports DOWN. It's readiness only — runtime health (LLM provider down, latency spikes) belongs to metrics and alerting, not a health check that would restart the container.

## The build-time pattern

All four beans are registered via `AdditionalBeanBuildItem` in a new `AgenticObservabilityProcessor`, gated on `Capability.OPENTELEMETRY_TRACER`, `MetricsCapabilityBuildItem`, and `Capability.SMALLRYE_HEALTH` respectively. The CDI event listener is unconditional — CDI events are always available.

I initially considered `SyntheticBeanBuildItem` but the Quarkus extension guide is clear: synthetic beans don't support injection. Since every listener bean needs `@Inject` for `Tracer`, `MeterRegistry`, or `Event<>`, `AdditionalBeanBuildItem` is the right mechanism. This matches the core module's `ListenersProcessor` pattern exactly.

All listeners set `inheritedBySubagents()` to `true`. Without this, sub-agents would be invisible — no spans, no metrics, no events for anything below the root agent.

## The Dev UI pivot

The existing Dev UI pages render HTML blobs from upstream's `HtmlReportGenerator` inside sandboxed iframes. The topology and execution views are opaque — can't be filtered, sorted, or themed with Quarkus Dev UI components.

Claude replaced both with structured JSON endpoints and Vaadin grids. The backend now walks the `AgentInstance` hierarchy and `MonitoredExecution` trees directly, serialising them as `JsonObject` trees. The frontend renders them in native Vaadin grids with columns for agent name, type, duration, token count, and iteration index.

The hot-reload fix was surgical: `DevAgentMonitorHolder` uses static `CopyOnWriteArrayList` fields that accumulate stale monitors across reload cycles. A `ShutdownContext` callback clears them before each restart.

## Specs on issues

A process change came out of this session: design specs now attach to GitHub issues, not PRs. Each chapter gets an issue carrying its spec in a collapsible `<details>` block. PRs reference the issue — `Design spec: attached to #N` — instead of duplicating the content. An epic (#2549) tracks all eight chapters with ARC42STORIES embedded as a living document that gets updated as chapters land.
