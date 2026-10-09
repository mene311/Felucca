# ADR-0013: Notes hold until an OFF row

- Status: accepted (2026-10-09) — product call; playback semantics land with v2
- Deciders: mene311

## Context

Today a NOTE step triggers a note and the engine envelope decides the tail; TIE
holds the previous note through a step, REST ends it. Tracker users expect
explicit note endings: a note sounds until you say stop.

## Decision

- A row that triggers notes **holds** them.
- **Empty rows do not end them** — the notes keep sounding across the pattern.
- An explicit **OFF** row releases everything the track is holding.
- A new note does **not** implicitly cut the held ones; held notes accumulate
  until OFF (the track's voice budget, shared across tracks, applies).

## Consequences

- Patterns read like a gate list: note-on rows and OFF rows; endings are
  deliberate and visible.
- `step_t` gains an OFF state (today: `ST_NOTE` / `ST_TIE` / `ST_REST`); the
  TIE/REST semantics of the existing model are redefined in the v2 data-model
  update. Format and protocol changes follow [adr/0008](0008-format-and-protocol-versioning.md).
- The track's polyphony is bounded by the engine (up to 4 notes per step, 8
  shared voices); a pattern that holds too many notes before an OFF steals
  voices — the UI should show held notes.
- The tracker view needs an OFF entry key; the MATRIX mockup work will show the
  row marker.
- Mute (per occurrence, notes-only, [adr/0012](0012-matrix-cell-semantics.md))
  does not release held notes differently: a muted occurrence simply does not
  trigger them.

## Alternatives

- Mono per track (a new note cuts the previous): rejected by this call.
- Envelope-length notes (today's behaviour): kept only as the natural release
  tail after OFF.
