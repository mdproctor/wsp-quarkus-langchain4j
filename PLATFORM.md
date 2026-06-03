# quarkus-langchain4j — Platform Coherence

> **Purpose:** Before implementing anything in the `agentic` module, run the coherence protocol below.
> Every implementation decision is an integration decision. A change that seems local may belong upstream,
> duplicate something in the core module, or break the CDI extension contract.

> **Protocols are living documents — never treat them as dogma.** If implementation reveals a gap or a rule
> that doesn't fit, update the protocol in the same session. A rule that doesn't adapt to new evidence is friction.

The protocols index is at [`protocols/INDEX.md`](protocols/INDEX.md). One file per rule, self-contained.

---

## Coherence Protocol

### Step 1 — Does this belong upstream?

Check [`protocols/upstream/`](protocols/upstream/). Ask:

- Is this a framework-agnostic improvement (better SPI, fixed threading contract, scope checkpoint)?
- Would Spring or Micronaut users benefit from this too?
- Does it require no Quarkus-specific API?

If yes → file upstream first. The Quarkus side waits or uses a temporary workaround.
See [`upstream-contribution-framing.md`](protocols/upstream/upstream-contribution-framing.md).

### Step 2 — Is this CDI-native?

Check [`protocols/cdi/`](protocols/cdi/). Ask:

- Does this use a static method where CDI injection would work?
- Does this wire directly into `AgenticRecorder` where an `AgentListener` CDI bean would work?
- Does this duplicate something the `core` module already provides (guardrails, metrics, OTel)?

If yes → use CDI. The `AgentListener` pipeline is the primary extension point.

### Step 3 — Is this consistent with the chapter plan?

Check `ARC42STORIES.MD` §9.2 Chapter Index. Ask:

- Which chapter does this belong to?
- Does this chapter's hard dependency chain allow it to land now?
- Does this require a foundation layer that hasn't landed yet (e.g. O-3 before O-1)?

If out of order → note the dependency and implement the blocker first.

### Step 4 — Does this tighten or loosen upstream coupling?

The module uses the same `langchain4j-agentic` annotations as the user-facing API. Ask:

- Does this add a new Quarkus-specific annotation that overlaps an upstream one?
- Does this bypass an upstream API in a way that will break on the next beta upgrade?

If yes → reconsider the approach. Tight coupling is intentional; divergence is not.

---

## Key References

| Document | Purpose |
|----------|---------|
| `ARC42STORIES.MD` | 8-chapter delivery plan, layer taxonomy, sequencing rationale |
| `AGENTIC-NATIVE-AUDIT.md` | 42-finding audit — source of truth for what to fix |
| `protocols/INDEX.md` | All standing rules |
