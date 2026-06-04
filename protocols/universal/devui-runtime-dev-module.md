---
id: PP-20260604-ba7333
title: "Quarkus Dev UI JSON-RPC services belong in runtime-dev modules, not runtime"
type: rule
scope: universal
applies_to: "Any Quarkus extension implementing Dev UI JSON-RPC endpoints via JsonRPCProvidersBuildItem"
severity: important
refs:
  - https://quarkus.io/guides/dev-ui#communicating-to-the-backend
violation_hint: "A JSON-RPC service class in src/main/java of the runtime module — the class will be included in production binaries and native images"
created: 2026-06-04
---

Dev UI JSON-RPC service classes must live in a dedicated `runtime-dev` Maven module, not
the main `runtime` module. The `runtime-dev` module is only included in the development
binary — production builds exclude it entirely. Placing a Dev UI service in the main
`runtime` module means the class ships in production JARs and native images, wasting binary
size and exposing reflection-based invocation paths. The build step that registers the
service (`JsonRPCProvidersBuildItem`) should also be gated on `onlyIf = IsDevelopment.class`
as a secondary guard, but the primary protection is the module boundary.
