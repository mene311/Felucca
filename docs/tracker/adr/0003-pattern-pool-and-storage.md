# ADR-0003: Pattern pool and storage shape

- Status: proposed (2026-10-09) — needs Q1/Q3 resolution
- Deciders: mene311

## Context

Today a "pattern" set is a full self-contained project record (FUN9, 3,648 B:
parameters, steps, song chain, motion, FM6 patches). The pool is 4 slots cached
in `.noinit` (15,696 B region, 14,592 B used) and mirrored to 32 KB of flash
(4 × 2 sector copies).

Target: 8 unique patterns (16 later), 64-row order list, ≥ 256 commands per track
per pattern, plus new per-row fields. `.data + .bss` has ~7 KB spare; the POOL
region ~12.8 KB spare.

## Decision

- **Phase 1**: keep the full-record model for the pool, sized for 8 slots
  (single 4 KB sector per slot, no A/B copy) **or** 8 slots with A/B copies by
  reclaiming 32 KB from the sample area. The final choice is Q3.
- **Phase 2+**: evaluate the split model — a **steps+commands-only pattern pool**
  with the sound state (parameters, FM6 patches) stored once per song. This is
  the tracker-native shape: patterns are notes+commands; instruments/sounds are
  song state (see ADR-0004).
- Either way, pool entries are **loaded on demand**; only the active pattern
  (plus the next one, for gapless order-list transitions) is cached in RAM.

## Consequences

- On-demand loading removes the `.noinit` cache growth that would otherwise
  violate NFR2; the cost is a flash read when switching patterns mid-song
  (acceptable at pattern boundaries, must be measured in the ISR budget).
- Single-copy slots trade power-fail safety for capacity; the CRC/valid check
  mitigates but does not eliminate the loss of one slot on an interrupted save.
- The split model halves the per-pattern cost (~2.5–5 KB vs 3.6 KB+) and makes
  16 patterns realistic, but it is a data-model change (song-wide sound state)
  and must not break existing projects (ADR-0008 conversions).

## Alternatives

- Descriptor-only slots (slots as notes, no reuse): rejected — duplicates data.
- Pool in the 32 KB unclaimed gap below `.noinit`: rejected until the gap's
  owner is verified (undocumented in the SDK sources).
- Growing `.noinit` upward: impossible — hardware write-protected region above.
