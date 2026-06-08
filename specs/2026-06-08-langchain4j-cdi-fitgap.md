# Fit-Gap Analysis: langchain4j-cdi vs quarkus-langchain4j

**Date:** 2026-06-08
**Repository:** https://github.com/langchain4j/langchain4j-cdi
**Local clone:** /Users/mdproctor/claude/langchain4j-cdi

---

## Summary

langchain4j-cdi is a **portable CDI integration** targeting the entire Jakarta EE ecosystem (WildFly, Helidon, Liberty, Payara, GlassFish, Quarkus). It is deliberately server-agnostic — zero `io.quarkus` imports in production code, no `@BuildStep`, no Jandex, no Arc.

quarkus-langchain4j is a **Quarkus-native integration** that exploits build-time processing for validation, dead-code elimination, optimized startup, and GraalVM native image support.

They are complementary, not competing.

---

## Per-Area Comparison

| Area | langchain4j-cdi | quarkus-langchain4j | Gap |
|---|---|---|---|
| **Agent guardrails** | `@RegisterSimpleAgent` only — has `inputGuardrails`/`outputGuardrails` attributes. No other agent topology has them. | C5 plans guardrails on ALL agent types via `@AgentInputGuardrails`/`@AgentOutputGuardrails` | langchain4j-cdi covers one topology; we cover all |
| **Guardrail validation** | Runtime `CDI.current().select()` — missing beans silently warned | Build-time `DeploymentException` — fail fast | Fundamental quality difference |
| **Agent creation** | `AgenticServices.createAgenticSystem()` — same underlying call | Same call in `AgenticRecorder` | Same path, different wiring quality |
| **AgentListener** | Runtime named-bean lookup (`agentListenerName` attribute) | Build-time validation + `Instance<AgentListener>` auto-discovery on all agents | We validate scopes, parameters, types at build time |
| **Telemetry** | ChatModel-level only (`ChatModelListener`) — LLM request metrics | Agent-level OTel spans + Micrometer metrics + CDI events via `AgentListener` | We have agent-level observability; they have LLM-level only |
| **Fault tolerance** | `@RegisterAIService` only — doesn't cover agents | C7 plans agent-specific resilience with scope checkpointing | They don't cover agents |
| **Build-time processing** | Standard CDI BCE — no build-time validation, no dead-code elimination | Quarkus-native `@BuildStep` + Jandex — build-time validation, GraalVM, synthetic beans | Architecturally different; ours is deeper |
| **Configuration** | MicroProfile Config SPI (`dev.langchain4j.cdi.plugin.*` namespace) | `quarkus.langchain4j.agent.*` namespace (C6, planned) | Different namespaces, both property-driven |
| **CDI auto-wiring** | Named-bean lookup per annotation attribute | Build-time CDI supplier detection + `SyntheticCreationalContext` injection | Our approach validates at build time |

---

## Key Files in langchain4j-cdi

- `langchain4j-cdi-core/.../RegisterSimpleAgent.java` — agent annotation with guardrail attributes (lines 78-99)
- `langchain4j-cdi-core/.../CommonAgentCreator.java` — agent creation, calls `AgenticServices.createAgenticSystem()` (line 149)
- `langchain4j-cdi-core/.../CdiLookupHelper.java` — runtime CDI bean resolution for guardrails (lines 93-138)
- `langchain4j-cdi-build-compatible-ext/.../Langchain4JAIServiceBuildCompatibleExtension.java` — CDI BCE, not Quarkus @BuildStep
- `langchain4j-cdi-mp/langchain4j-cdi-telemetry/.../SpanChatModelListener.java` — ChatModel-level OTel only
- `langchain4j-cdi-mp/langchain4j-cdi-fault-tolerance/.../Langchain4JFaultToleranceExtension.java` — @RegisterAIService only (line 46)

---

## Conclusion

**We are not duplicating work.** The two projects serve different audiences with different integration depths:

- langchain4j-cdi: portable CDI, runtime wiring, fallback-and-warn, works everywhere
- quarkus-langchain4j: Quarkus-native, build-time validation, fail-fast, optimized startup, GraalVM native, Dev UI

The runtime wiring patterns are similar (both wrap `langchain4j-agentic`), which is expected. The quality of integration is where they diverge.
