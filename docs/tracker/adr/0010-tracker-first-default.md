# ADR-0010: Tracker-first default interface

- Status: accepted (2026-10-09) — product call
- Deciders: mene311

## Context

The fork exists for tracker users. The open question was whether the tracker
surfaces (MATRIX in v1, TRACKER view in v2) should be a mode users switch into,
or the firmware's default interface. Product answer: **default**.

## Decision

- The fork is **tracker-first**: the MATRIX is the default arrangement surface
  (v1); the TRACKER view is the default editing surface (v2).
- There is **no enable switch / no "tracker mode"**: the tracker surfaces are part
  of the normal page family and the default navigation leads with them.
- Felucca's existing pages stay reachable and unchanged (upstream behaviour is
  preserved, per [ADR-0001](0001-fork-and-upstream-policy.md)); the current STEP
  roll remains available as a secondary view after the tracker view lands.
- Documentation, onboarding and release notes present the firmware as a tracker.

## Consequences

- **Quality bar**: the tracker surfaces are the first thing a user sees; they
  cannot ship rough. This strengthens the "renders for sign-off before input"
  approach in the roadmap.
- Navigation changes: entry points (HOME / SEQ) lead to the tracker surfaces;
  upstream pages become secondary but keep their existing content.
- Upstream merges: page-family lists and input maps gain entries — conflicts in
  `ui.c` / `ui_input.c` are likely; keep the changes additive.
- The official firmware (V15) remains the supported way back, so a user who
  prefers stock Felucca is not trapped.

## Alternatives

- Opt-in mode: rejected — it halves the value for the target audience, doubles
  the documentation, and makes the default experience a synth with a hidden
  tracker rather than a tracker.
- Tracker-only (removing Felucca pages): rejected — needless divergence, loses
  features, breaks upstream mergeability.
