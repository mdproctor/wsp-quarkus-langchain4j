---
id: PP-20260604-transitive-interface-traversal
title: "Always traverse the full interface hierarchy when checking annotations on agent interfaces"
type: rule
scope: repo
applies_to: "AgenticProcessor build steps — any annotation check on detected agent ClassInfo objects"
severity: important
refs:
  - ../INDEX.md
violation_hint: "Using iface.hasAnnotation() or iface.declaredAnnotations() directly on the agent ClassInfo without calling ValidationUtil.transitiveInterfaces()"
created: 2026-06-04
---

Any annotation check on an agent interface in `AgenticProcessor` must use
`ValidationUtil.transitiveInterfaces(iface, index)` to walk the full parent
interface chain before querying for annotations. Checking the direct `ClassInfo`
only misses annotations declared on parent interfaces — a silent failure that
produces no error but causes wrong behaviour at runtime (missed CDI bean unremovable
marks, missed interceptor bindings, missed FaultTolerance warnings). This rule was
reinforced in three separate places during Chapter 1: S-3 (`markCdiBeanParametersAsUnremovable`),
F-7 (`hasAnyInterceptorBindings`), and F-4/F-5 (`validateFaultToleranceInteractions`).
