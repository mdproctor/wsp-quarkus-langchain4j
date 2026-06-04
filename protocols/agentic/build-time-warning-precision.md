---
id: PP-20260604-build-time-warning-precision
title: "Build-time warnings fire on the exact dangerous combination, not a valid superset"
type: principle
scope: repo
applies_to: "AgenticProcessor validateFaultToleranceInteractions and any future @BuildStep validators"
severity: guidance
refs:
  - ../INDEX.md
violation_hint: "A warning that fires on @Transactional alone (valid for read-only agent methods) rather than the dangerous @Transactional + @Retry combination"
created: 2026-06-04
---

Before writing a `@BuildStep` validator that emits a build-time warning, identify
the exact precondition that causes the bug — not a superset. A warning that fires on
a valid pattern (e.g., `@Transactional` alone on an agent method) causes warning fatigue:
developers suppress or ignore it globally, making the genuinely dangerous case invisible.
Reserve warnings for patterns that are nearly always wrong. If a pattern is only dangerous
in combination with something else, check for the full combination. F-5 (`@Transactional +
@Retry`) was narrowed from `@Transactional alone` to the full combination for precisely
this reason — `@Transactional` alone on an agent method that reads from a database inside
a transaction is correct and common.
