# ADR-0015: The arrangement loops from the first row

- Status: accepted (2026-10-09) — product call
- Deciders: mene311

## Context

With blank cells inheriting ([adr/0012](0012-matrix-cell-semantics.md)), a track
would keep its last assignment for every following row — so stopping a track was
an open question. Product answer: **tracks don't stop; the arrangement loops from
the first row.**

Today's chain stops at its end (`seq_stop` in `song_chain.c`).

## Decision

- The arrangement **loops**: after the last row, playback returns to the first
  row and continues, running until stopped with PLAY.
- Inherited cell state **resets at the loop point**: each pass starts from the
  first row's explicit assignments. A track with no assignment in the first row
  is silent until it is first assigned within that pass.
- No explicit stop value is needed in cells.
- Ending a piece is done by stopping playback, or by writing the ending into the
  loop itself.

## Consequences

- The matrix keeps three states per cell in practice: pattern (A–D), blank
  (inherit within the pass), and mute (per occurrence) — the stop marker is
  dropped.
- The loop point should be visible in the MATRIX view, and playback should make
  an audible/visual wrap obvious (the existing playhead/row indicator).
- A future "play once / no loop" mode would leave tracks holding to the end —
  note as an option, not v1 scope.

## Alternatives

- Explicit stop marker per cell: rejected by this call.
- Muting every following row to end a track: rejected — painful.
- Inheriting across the loop point: rejected — each pass restarting from the top
  is what makes the loop predictable.
