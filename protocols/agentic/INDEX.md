# Agentic Module Protocols

Rules specific to building and extending the `agentic/` module of quarkus-langchain4j.

| File | Rule Summary | Applies To |
|------|-------------|------------|
| [transitive-interface-traversal.md](transitive-interface-traversal.md) | Always walk the full interface hierarchy (ValidationUtil.transitiveInterfaces) when checking annotations on agent ClassInfo objects | AgenticProcessor build steps — any annotation check on detected agent ClassInfo objects |
| [build-time-warning-precision.md](build-time-warning-precision.md) | Build-time warnings fire on the exact dangerous combination only — not a valid superset | AgenticProcessor validateFaultToleranceInteractions and any future @BuildStep validators |
