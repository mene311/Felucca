# Requirements

Status: draft for review. Keywords MUST / SHOULD / MAY are used deliberately.
Acceptance criteria are testable; the test harness is described in
[testing.md](testing.md).

## Problem

Felucca is a preset- and knob-oriented instrument. Composition happens by
recording knob moves and holding steps to lock parameters. Tracker users expect a
row-based workflow: type notes and effect commands on numbered lines, arrange
small reusable patterns into long songs, and get fine timing resolution for
breakcore/experimental music — without a computer keyboard in the loop.

## Goals

- G1. Row-based editing on the device: notes **and effect commands** typed per row.
- G2. Arrangement-first songs from a small reusable pattern pool.
- G3. Per-row FX stacking: multiple different parameters per row, plus the
  "upfront" columns (velocity, pan, delay).
- G4. Fine timing: an LPB-style selector reaching breakcore granularity.
- G5. Any instrument on any row (per-row patch change), any instrument on any track.
- G6. No external keyboard: all input on the device's 27 keys / 12 buttons / 8 knobs.

## Non-goals

- N1. Per-voice (polyphonic, overlapping) instruments — rejected in
  [adr/0004](adr/0004-per-step-instrument.md); the engine's parameters are
  track-level.
- N2. Renoise file compatibility (no .xrns import/export).
- N3. New sound engines or DSP changes beyond what the features require.
- N4. Changing the SysEx protocol in incompatible ways for existing editors —
  only append-only extensions with capability tags.

## Functional requirements

### R1 — Order list (arrangement)

- R1.1 The order list MUST hold **≥ 64 rows** (today 16, `CHAIN_ROWS`).
- R1.2 Each row MUST reference a pattern slot and carry a repeat count
  (1..255; today 1..16).
- R1.3 Rows MUST be reference-based aliases: two rows referencing one slot share
  content; editing the slot changes both.
- R1.4 The UI MUST support: insert, delete, duplicate (alias), move up/down,
  and set repeat. Deletion of a row MUST NOT touch the referenced slot.
- R1.5 Playing the order list MUST obey the existing rule: it borrows the stored
  slots' steps and plays with the current sounds.
- R1.6 Acceptance: order list round-trips through save/load; a 64-row list plays
  in the written order; `ui_render` lint and alignment pass; host tests
  (project, chain, protocol) pass.

### R2 — Pattern pool

- R2.1 The pool MUST support **8 unique patterns** at minimum; 16 is the target
  once the storage change in [adr/0003](adr/0003-pattern-pool-and-storage.md)
  lands.
- R2.2 Saving/loading a pool entry MUST be atomic under power loss (CRC + valid
  copy strategy; see [formats.md](formats.md)).
- R2.3 The pool's UI letters stay `A`.. (today `A`..`D`; 8 → `A`..`H`).
- R2.4 Acceptance: fill all slots, power-cycle, all slots intact; corrupt one
  sector and the firmware falls back without bricking.

### R3 — Commands (per-row effects)

- R3.1 A row MUST accept ≥ 2 typed effect commands in addition to the note,
  velocity (existing) and the new pan/delay fields.
- R3.2 A command MUST resolve to a motion record `(track, row, param, value)` with
  kind *lock* (row only) or *event* (holds).
- R3.3 The addressable parameter set is `motion_param()`-eligible ids; transport,
  routing and discrete engine changes stay excluded.
- R3.4 The motion pool MUST become **per track** and hold ≥ 256 records per
  pattern (today 64 per project, shared by 4 tracks).
- R3.5 Undo MUST follow the existing one-step `SAVE`-held model for command edits.
- R3.6 Acceptance: host tests prove lock vs event semantics, undo, save/load
  round-trip, and the per-track capacity; audio is bit-identical to a hand-set
  equivalent when only one command is used (regression hash).

### R4 — Upfront columns

- R4.1 Velocity: existing `step_t.vel` (0..127) MUST be shown and editable as a
  column.
- R4.2 Pan: MUST gain a per-row value (new step field), 0..127, defaulting to the
  track's current pan.
- R4.3 Delay: MUST gain a per-row value dividing the row into slices
  (Renoise: 0..255 per line); default 0.
- R4.4 Acceptance: round-trip through save/load and the protocol; UI shows the
  columns; playback timing verified by host test (frame-accurate).

### R5 — Instruments

- R5.1 Every row MUST be able to select an instrument (engine + factory/user
  preset) — one byte, ~65 factory + 32 user entries.
- R5.2 The change MUST apply before the row's note-on.
- R5.3 Cross-engine changes MUST use the existing fade/switch path; same-engine
  changes SHOULD be a seamless parameter copy.
- R5.4 Acceptance: a row switching instrument changes the timbre from that row
  on; documented caveat: release tails morph (track-level parameters); host test
  verifies the applied values.

### R6 — Timing / LPB

- R6.1 `DIV` MUST gain `1/64` (LPB 16), `1/128` (LPB 32) and `32T`, `64T`
  as appended enum values.
- R6.2 `NSTEP` 64 → **128** MUST be evaluated in the same phase as R6.1, because
  finer `DIV` shrinks the musical span of a fixed-length pattern.
- R6.3 Acceptance: pattern length/step index changes verified by host test;
  existing projects unchanged (round-trip golden).

### R7 — Pattern matrix (phase 4)

- R7.1 A song row MAY reference blocks per track (4 refs) with a mute mask,
  instead of one whole-pattern ref.
- R7.2 Per-occurrence block mute MUST NOT alter the referenced content.
- R7.3 Acceptance: host test — two rows sharing one block alias edits; a muted
  block plays nothing without changing the source; save/load round-trip.

### R8 — Mixer

- R8.1 The existing MIXER page (LEVEL/PAN/REV/MUTE + meters) is the base.
- R8.2 It SHOULD show pre/post-style separation only if the engine gains it;
  otherwise document that the track `LEVEL` is the single gain stage.
- R8.3 Acceptance: no regressions on the mixer page's render tests.

## Non-functional requirements

- NFR1. **CPU**: no engine may exceed its existing per-preset budget by more than
  the enforced margin (`tests/target_budget.txt`, `cpu_baseline.txt`).
- NFR2. **RAM**: total `.data + .bss` MUST stay within the linker region
  (98,304 B; stock build uses 91,220 B). Patterns are the main pressure — the
  default plan is on-demand loading, not a larger cache.
- NFR3. **Flash**: new data MUST fit the defined map (`formats.md`) with a
  documented trade-off against the 240 KB user-sample area.
- NFR4. **ISR safety**: the audio ISR keeps reading only bounded structures; all
  new writes publish under the IRQ guard.
- NFR5. **Flash wear**: saves remain user-initiated or autosave-bounded (the
  upstream autosave policy: when stopped and idle, at most once a minute).
- NFR6. **UI**: every screen passes layout lint and ink-alignment; draw cost
  within the existing budget (`ui_render`).
- NFR7. **Licence**: GPL-3.0-only; every new source file carries the SPDX header.

## Open questions (resolve before the phase that needs them)

- Q1. Pattern records: keep full self-contained records (sounds + steps) or split
  a steps-only pool from the song-wide sound state? ([adr/0003](adr/0003-pattern-pool-and-storage.md))
- Q2. `NSTEP` 128: which UI bank scheme (16-step pages → 8 pages) and protocol
  step-index encoding?
- Q3. Where extra pattern storage physically lives (reclaim the ~36 KB gap below
  the XIP app end vs. shrink the sample area).
- Q4. Should block references (R7) arrive with the matrix view in one phase, or
  stay a whole-pattern ref list until then?
