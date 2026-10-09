# ADR-0014: Note columns — per-column monophony, chords by columns, glide

- Status: accepted (2026-10-09) — product call; refines
  [ADR-0013](0013-notes-hold-until-off.md)
- Deciders: mene311

## Context

How simultaneous and successive notes behave on a track: whether a new note cuts
the previous one, how chords are written, and what glide does.

## Decision

- Each track carries **1..4 note columns**. The engine already holds up to four
  notes per step (`step_t.note[4]`), so the limit is the engine's.
- Each column is **monophonic**: a new note in a column **cuts** the previous
  note in that column.
- **Glide on** (per note): instead of cutting, the new note glides from the
  previous one — the existing slide / `P_GLIDE` path.
- **Chords** are written across columns: several notes on the same row, one per
  column (equivalently, several notes in one step).
- Otherwise a note **holds until an OFF** row in its column
  ([adr/0013](0013-notes-hold-until-off.md)); empty rows do not end it.
- **OFF is per column**: it releases what that column holds.

## Consequences

- `seq.c` must track the sounding note per `(track, column)`; the existing
  held-note, tie and slide handling is the base it extends.
- The UI needs **add / remove note columns** per track, and a way to show a subset
  of them on the 240 px screen (Q-U5). Which column has focus drives entry.
- The matrix mute stays notes-only; a muted occurrence triggers no notes, so
  column state is not disturbed.
- Storage is unchanged for the columns themselves (`note[4]` exists); OFF and
  glide are new step states/flags, landing with the v2 format revision.

## Alternatives

- One column per track (monophonic only): rejected — forbids chords, a tracker
  staple.
- Stacking notes in one column: superseded — ambiguous musically and unbounded.
- More than four columns: engine-limited (four notes per step).
