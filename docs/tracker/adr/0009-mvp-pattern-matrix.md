# ADR-0009: MVP is the pattern matrix over four patterns

- Release: **v1**, the first public release of the fork (product call).
- Status: accepted (2026-10-09) — supersedes the phasing in
  [ADR-0006](0006-arrangement-model.md)
- Deciders: mene311 (product call)

## Context

The phased plan had "order list first (64 rows), matrix later". The product
review decided the opposite priority: the missing capability is not song length
but **arrangement power** — repeating, mixing and matching and muting the four
existing patterns in a matrix. The pool stays at 4, so no storage expansion is
needed for the MVP.

Two facts make this cheap (verified in the tree):

- The project record has **48 spare bytes** (`project.c`, FUN9 comment). Packed
  matrix rows (refs u16 + repeat u8 + mute u8 = 4 B) × 16 rows + header = 68 B —
  +32 B over today's 36 B chain block, inside the spare. The record keeps its
  **3,648-byte size** and the FM6 patch offset.
- Motion records already carry the track (`place = track << 6 | step`,
  `motion.c`), so per-track references are compatible with the playback path;
  mixed rows merge per-track events into the live store.

## Decision

- **MVP = pattern matrix**: rows reference the four patterns **per track**, with
  a per-track mute mask and repeat; insert/delete/duplicate/move; a MATRIX screen
  on the device.
- Pattern pool stays 4; order list stays 16 rows for the MVP.
- Format: new magic (`FUNA`), same size, exact FUN9 conversion (whole-pattern
  rows → four equal refs, mute 0).
- Tracker typing (commands), instruments, capacity and timing follow in that
  order (requirements R4, R5, R6, R7).

## Consequences

- The headline arrangement feature ships without touching storage budgets or the
  RAM pressure identified in NFR2/NFR3.
- The order-list-first work (64 rows) moves to phase 4 and no longer blocks the
  MVP.
- Per-track refs make the shared 64-record motion store the practical ceiling
  until phase 2 raises it per track; document the merge rule with the MVP.
- The MATRIX screen becomes the first new UI surface and the proof that the
  on-device editing model works before the tracker view is built.

## Alternatives

- Order list first, matrix later (previous plan): rejected by the product review —
  longer lists of whole patterns do not solve arrangement.
- Expand the pool to 8 first: rejected — not needed for the matrix and it spends
  flash/RAM before the arrangement model is proven.
