# Requirements

Status: draft for review. Scope follows the MVP decision in
[adr/0009](adr/0009-mvp-pattern-matrix.md): **the pattern matrix over the four
existing patterns comes first**; capacity and timing extensions follow.

Keywords MUST / SHOULD / MAY are deliberate. Acceptance criteria are testable
against the harness in [testing.md](testing.md).

## Problem

Felucca's song is a flat list of pattern references: 16 rows, each playing one
whole pattern. There is no way to combine tracks from different patterns, no way
to mute one track for one occurrence, and the pattern pool cannot be browsed as
an arrangement. Tracker users expect a **pattern matrix**: repeat, mix and match,
and mute per occurrence while arranging.

## Goals

- G0. The tracker surfaces are the **default interface** of the fork — no opt-in
  mode ([adr/0010](adr/0010-tracker-first-default.md)).
- G1. Arrange songs from the **4 existing patterns** with per-track mix-and-match
  and per-occurrence mute (the MVP).
- G2. Row-based editing on the device: notes **and effect commands** per row.
- G3. Per-row FX stacking via the motion system, plus velocity/pan/delay columns.
- G4. Any instrument on any row.
- G5. Fine timing (breakcore granularity) once the arrangement model is settled.

## Non-goals

- N1. Per-voice (polyphonic, overlapping) instruments — see
  [adr/0004](adr/0004-per-step-instrument.md).
- N2. Renoise file compatibility.
- N3. New engines or DSP work beyond what the features need.
- N4. Breaking SysEx changes for existing editors.

## v1 (MVP) — Pattern matrix over four patterns

### R1 — Matrix rows

- R1.1 The pattern pool stays **4 patterns (A–D)**; no storage or pool changes.
- R1.2 Each song row MUST reference, **per track**, one of the 4 patterns
  (whole-pattern reference = all four refs pointing at the same pattern).
- R1.3 Rows MUST support **mix and match**: a row may take track 1 from pattern A,
  track 2 from C, and so on.
- R1.4 Each row MUST carry a **per-track mute mask**; a muted track plays nothing
  for that occurrence and the referenced content is never altered.
- R1.5 Each row MUST carry a repeat count (existing range 1..16; extending to
  1..255 is allowed if free).
- R1.6 Row operations MUST include insert, delete, duplicate (alias), move
  up/down, and set repeat. Row count stays 16 for the MVP (a larger list is
  phase 4).
- R1.7 References are aliases: rows sharing a pattern share content; editing the
  pattern changes every row that uses it.
- R1.8 **Acceptance**: a round-trip save/load of a matrix song; two rows sharing
  one pattern both change when it changes; a muted track is silent for that row
  only; a mixed row plays each track from its own referenced pattern; host tests
  (chain, project, protocol) green; `ui_render` lint and alignment clean.

### R2 — MATRIX screen

- R2.0 The MATRIX MUST be the default arrangement surface of the firmware
  (no enable switch; Felucca pages stay reachable, ADR-0010).

- R2.1 A dedicated screen MUST show the arrangement as **4 track columns × the
  row list**, each cell showing which pattern (if any) that track uses for that
  row, its mute state, and the row's repeat.
- R2.2 Editing MUST allow: set a cell's pattern (A–D or empty), toggle mute,
  set repeat, and move between cells/rows — all from the device keys
  (per [ui-input.md](ui-input.md), no external keyboard).
- R2.3 The screen MUST mark aliased/repeated content so structure is visible at
  a glance.
- R2.4 **Acceptance**: the UI input simulation covers cell edits and mute toggles;
  render lint/alignment pass in every palette; a 16-row matrix is editable without
  dropping frames (draw-cost budget).

### R3 — Storage and protocol

- R3.1 New record magic (`FUNA`) with **the same 3,648-byte size**: the matrix
  rows (packed 4 B: refs u16 + repeat u8 + mute u8) consume the record's 48 spare
  bytes; the FM6 patch offset is unchanged.
- R3.2 FUN9 → FUNA conversion MUST be exact: whole-pattern rows become four equal
  refs with mute 0.
- R3.3 Corrupt records MUST be rejected without losing the previous valid copy.
- R3.4 The SysEx protocol MUST gain order-list operations (read/write rows, mute)
  behind a capability tag; existing ops keep their byte layouts.
- R3.5 **Acceptance**: conversion round-trip test; corrupt-sector recovery test;
  protocol tests; existing editors remain functional.

## v2 — Tracker view and commands

- R4.1 A TRACKER screen with rows and typed values: note, instrument, velocity,
  command columns.
- R4.2 Commands resolve to motion records `(track, row, param, value)` with kind
  *lock* (row only) or *event* (holds); addressable ids are `motion_param()`-eligible.
- R4.3 Motion capacity becomes **per track and ≥ 256 records per pattern**
  (today 64 shared). Mixed-ref rows merge per-track events; the merge rule and
  cap are specified before implementation.
- R4.4 Per-row **pan** and **delay** fields; velocity column shown.
- R4.5 Undo follows the one-step `SAVE`-held model.
- R4.6 Acceptance: lock/event semantics, undo, save/load round-trip, per-track
  capacity, frame-accurate delay test, UI input simulation.

## v3 — Instruments per row

- R5.1 Every row selects an instrument (engine + factory/user preset), one byte.
- R5.2 Applied before the row's note-ons; cross-engine uses the fade path.
- R5.3 Acceptance: timbre change verified from that row on; documented caveat
  that release tails morph (track-level parameters).

## v4 — Capacity (deferred)

- R6.1 Pattern pool 8–16 and order list 64 rows, per
  [adr/0003](adr/0003-pattern-pool-and-storage.md) — only once the MVP matrix is
  accepted and the storage question (Q3) is resolved.
- R6.2 Per-track block storage split (steps+commands vs full records) if needed.

## v5 — Timing

- R7.1 `DIV` gains `1/64` (LPB 16), `1/128` (LPB 32), `32T`, `64T`.
- R7.2 `NSTEP` 64 → 128 paired in the same phase.

## Non-functional requirements

- NFR1. No engine may exceed its CPU budget (existing target/cpu baseline checks).
- NFR2. `.data + .bss` stays within its region (98,304 B; stock 91,220 B).
- NFR3. New data fits the documented flash map with a documented trade-off.
- NFR4. The audio ISR keeps reading only bounded structures; new writes publish
  under the IRQ guard.
- NFR5. Saves remain user-initiated or autosave-bounded.
- NFR6. Every screen passes layout lint and ink alignment; draw cost within budget.
- NFR7. GPL-3.0-only; SPDX headers on new files.

## Open questions

- Q-M1. Row count for the MVP: 16 rows is free; is that enough to prove the
  matrix, or does the MVP need 64 (record grows, bigger format change)?
- Q-M2. MATRIX gestures: which keys select, edit a cell, toggle mute, move rows —
  see [ui-input.md](ui-input.md) Q-U4.
- Q-M3. Mute scope: notes only, or notes + that track's commands (motion) for the
  occurrence?
- Q-M4. Empty cells: a row with a track set to empty plays nothing for that
  track — confirm (vs. inheriting the previous row).
- Q1–Q4 from earlier review still apply to phases 2–5 (pattern record shape,
  `NSTEP` banks, storage location, block-ref timing is now answered by ADR-0009).
