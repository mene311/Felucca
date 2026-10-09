# ADR-0012: Matrix cell semantics — inherit, mute gesture B, notes-only mute

> **Amended 2026-10-09 (later):** upstream 1.4 already provides per-track slots,
> `CHAIN_SILENT` (`−`) and inherit-on-add (a new section copies the previous one).
> What remains open is whether to add a mute flag that preserves the reference
> (`−` loses which pattern was there) — see [ADR-0016](0016-rebase-on-1.4.md).

> **Amended 2026-10-09:** the stop-value question is resolved by
> [ADR-0015](0015-arrangement-loops.md) — the arrangement loops from the first
> row, so cells need no stop marker and inherited state resets each pass.

- Status: accepted (2026-10-09) — product call
- Deciders: mene311

## Context

The v1 matrix needed three rules: what a blank cell does, how mute is toggled,
and what mute silences.

## Decision

- **Blank cell inherits**: a cell left empty keeps the track's previous
  assignment until it is changed. Arrangements are written by exception
  ("change what differs"), and duplicate-row stays available for explicit work.
- **Mute gesture B**: a dedicated black key toggles mute on the selected cell —
  one press, no modifier. Which black key is a UI detail fixed in the MATRIX
  mockups.
- **Mute scope: notes only** — a muted track plays no notes for that occurrence;
  its recorded automation keeps running.

## Consequences

- Because blank inherits, stopping a track needs something to end it: resolved by
  the looping arrangement ([adr/0015](0015-arrangement-loops.md)) — no stop value
  in cells; each pass restarts from the first row.
- Mute is per occurrence; it never alters the referenced pattern.
- Notes-only muting means release and reverb tails and the track's automation
  continue under a mute — audible only at the edges (long tails, mid-row sound
  changes), predictable in the common case.
- The matrix screen must show: pattern (A–D), blank-inherit, and the mute flag
  (no stop state — [adr/0015](0015-arrangement-loops.md)).

## Alternatives

- Blank = silence: rejected by this call — more cells to fill.
- EDIT-modifier mutes: rejected — slower, though it saves a key.
- Muting notes together with automation: rejected — "the track keeps moving,
  just unheard" is the chosen model.
