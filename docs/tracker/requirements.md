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

## v1 (MVP) — Pattern matrix: remaining deltas

> **Re-based on Felucca 1.4 ([adr/0016](adr/0016-rebase-on-1.4.md)):** upstream
> already provides per-track section slots, `CHAIN_SILENT` (-), repeats,
> add/delete and inherit-on-create, plus SONG-page editing. The requirements
> below are what the fork adds; *(upstream)* marks what is kept for context.

### R1 — Matrix rows

- R1.1 The pattern pool stays **4 patterns (A–D)** *(upstream)*; no storage or
  pool changes.
- R1.2 Sections reference, per track, one of the 4 patterns or the silent value
  *(upstream: `slot[NTRK]`, `CHAIN_SILENT`)*.
- R1.3 Mix and match: a section may take track 1 from A, track 2 from C, and so
  on *(upstream)*.
- R1.4 **Mute policy (open):** upstream's `-` silences a track in a section but
  forgets which pattern it played. Decide whether v1 adds a **mute flag** that
  preserves the reference (gesture B on the MATRIX screen,
  [adr/0012](adr/0012-matrix-cell-semantics.md)); scope stays **notes only**.
- R1.4b New sections copy the previous one *(upstream)*; **playback loops from
  the first section** and each track's state resets per pass
  ([adr/0015](adr/0015-arrangement-loops.md)).
- R1.5 Repeat per section, 1..16 *(upstream)*.
- R1.6 Add/delete/repeat *(upstream)*; **move/reorder** is ours. Row count
  stays 16 for v1 — the 3:30 acceptance fits with room to spare (16 x 16 x 4 s
  ~ 17 minutes at 32 lines, 1/16, 120 BPM).
- R1.9 v1 works with **32-line patterns** as the default (existing 64-step
  projects still load; 32 is a default, not a cap —
  [adr/0011](adr/0011-v1-pattern-length.md)).
- R1.7 References are aliases: rows sharing a pattern share content; editing the
  pattern changes every row that uses it. Playback loops from the first row until
  stopped.
- R1.8 **Acceptance**: a **3:30 song built from the 4 patterns at 32 lines**
  plays on hardware; the arrangement **loops from the first section**; a mixed
  section plays each track from its own slot; two sections sharing a pattern
  both change when it changes; host tests green; `ui_render` clean.
- R1.10 **Loop** ([adr/0015](adr/0015-arrangement-loops.md)): playback wraps to
  the first section instead of stopping (upstream stops), resetting each track's
  state per pass.

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

- R4.0 **Note semantics** ([adr/0013](adr/0013-notes-hold-until-off.md),
  [adr/0014](adr/0014-note-columns.md)): each track carries 1..4 note columns.
  A column is monophonic — a new note cuts the previous note in that column
  unless glide is on, in which case it glides from it (existing slide /
  `P_GLIDE` path). A note otherwise holds until an OFF row in its column; empty
  rows do not end it. Chords are written across columns on the same row. OFF is
  per column. `step_t` gains an OFF state and the glide flag; TIE/REST are
  redefined in the v2 data-model update.
- R4.1 A TRACKER screen with rows and typed values: 1..4 note columns per track
  (add/remove), instrument, velocity, command columns, and the OFF entry.
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

### Blocking v1

- **Q-M1. Song rows for v1: 16 or 64?** 16 rows is free (current record); 64
  needs a bigger format change (record grows). Default if unanswered: **16**.
- **Q-P1. Upstream base.** Build v1 on Felucca 1.1.5.1 now, or wait for
  upstream's planned 1.2 reorganisation (fewer pages)? Building now risks a
  larger merge later; waiting delays v1. Default if unanswered: build now on
  1.1.5.1, merge 1.2 when it lands.
- **Q-P2. Hardware testing.** v1 needs at least one flash test on a real FM-1
  (the emulator proves logic, not the panel). Who flashes: the maintainer via
  the browser installer, or driven from the laptop?
- **Q-P3. Naming.** What the fork/release is called publicly and in ABOUT
  (candidate: “Felucca Tracker”, crediting upstream as required by GPL).
- **Q-P4. Two mutes.** The matrix mute (per occurrence, notes only) and the
  mixer's track MUTE (persistent) are different things — confirm that the
  mixer page keeps its mute unchanged, and that the mockups must distinguish
  them visually.

### Settled during v1 review

- Q-M2 (matrix gestures) → settled in the MATRIX renders for sign-off;
  Q-U1–Q-U5 in [ui-input.md](ui-input.md) are the same class.
- Q-M3 (mute scope) → **notes only** ([adr/0012](adr/0012-matrix-cell-semantics.md)).
- Q-M4 (empty cells) → **inherit**, with the arrangement looping from row 1
  ([adr/0012](adr/0012-matrix-cell-semantics.md), [adr/0015](adr/0015-arrangement-loops.md)).

### Engineering decisions (maintainer, unless objected)

- Command default kind: **per-row lock** (Renoise-like), shown by a marker;
  *event* (value holds) is the explicit alternative.
- Instrument list composition: ~65 factory sounds + 32 user presets, one byte.
- Existing MIXER page is enough for v1 — no new mixer work.

Later releases: Q1–Q4 (pattern record shape, `NSTEP` banks, storage location,
block-ref timing).
