# ADR-0012: Matrix cell semantics — inherit, mute gesture B, notes-only mute

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

- Because blank inherits, **stopping a track needs an explicit value**: the cell
  must carry a "stop / none" state distinct from blank (open question Q-M4:
  confirm the marker and its gesture, e.g. `—`).
- Mute is per occurrence; it never alters the referenced pattern.
- Notes-only muting means release and reverb tails and the track's automation
  continue under a mute — audible only at the edges (long tails, mid-row sound
  changes), predictable in the common case.
- The matrix screen must show three states per cell: pattern (A–D), stop, and
  blank-inherit — plus the mute flag.

## Alternatives

- Blank = silence: rejected by this call — more cells to fill.
- EDIT-modifier mutes: rejected — slower, though it saves a key.
- Muting notes together with automation: rejected — "the track keeps moving,
  just unheard" is the chosen model.
