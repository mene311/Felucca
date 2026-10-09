# Roadmap

Status: draft. Releases ship in order; each is accepted before the next starts.
Scope is frozen by the requirements it implements.

## Pre-release — Foundations (done)

- Deliverables: `docs/tracker/*`, dev loop (host builds on the phone, toolchain
  and emulator on the laptop), the `aa_raster.py` no-Raqm fallback.
- Exit: documents reviewed; unmodified fork builds (`felucca.fwsc`); emulator
  passes `emu_test.mjs`.
- State: docs in review; everything else **done**.

## v1 (MVP) — Pattern matrix over four patterns

**State: simulator-ready.** Delivered: the matrix capability via upstream 1.4
(per-track section slots, `-` silent, repeats, add/delete, inherit-on-create),
the arrangement **loop** (ADR-0015), and the **FM-1 TRACKER** name
(splash/ABOUT/version `v1.4-tr1`). Host tests green (`tests/run_tests.sh`), the
browser emulator rebuilt and served.

Delivered since: the **MATRIX bird's-eye screen** — a new GLO-family page
(tracker-first: GLO opens it, a second tap opens SONG) drawing sections x tracks
with the slot letters, `-` silent cells, the cursor, repeats and the play mark;
it edits the same data with the same knobs (SEC/TRACK/PAT/REPS). Renders cover
the empty and a mixed 4-section grid (0 lint findings, 0 misalignments); host
tests updated for the tracker-first GLO behaviour.

Still to do for the hardware milestone: row move/reorder, then the flash and the
3:30 hardware test.

- Goal: arrange by repeating, mixing and matching and muting the four existing
  patterns. The MATRIX is the default arrangement surface (ADR-0010). v1 works
  with 32-line patterns (ADR-0011); blank cells inherit, mute is upstream's `-`
  (ADR-0012); playback loops from the first row (ADR-0015).
- Deliverables:
  - Matrix rows: per-track refs (4 bits/track), repeat, per-track mute mask;
    insert / delete / duplicate / move (16 rows, R1).
  - MATRIX screen: 4 track columns × scrolling rows; cell edit, mute toggle,
    repeat; alias/repeat markers (R2).
  - Format `FUNA`: same 3,648-byte size, rows packed 4 B into the record's 48
    spare bytes; exact FUN9 conversion (R3).
  - Protocol: order-list read/write ops + capability tag (R3).
- Exit: R1–R3 accepted; conversion and corrupt-record tests green; UI lint and
  alignment clean; a demo song built from 4 patterns with mixed rows and mutes.

## v2 — Tracker view and commands

- Goal: type notes and effects row by row. The TRACKER view becomes the default
  editing surface (ADR-0010); the current STEP roll stays reachable. Notes hold
  until an OFF row (ADR-0013).
- Deliverables: TRACKER screen (cursor, hex entry, keycap hints); per-track
  command stores (≥ 256/track, lock/event semantics, two command columns);
  velocity/pan/delay columns; EDIT-layer clipboard; motion merge rule for
  mixed-ref rows documented and implemented; protocol and format updates.
- Exit: R4 accepted; frame-accurate delay test; golden playback of a scripted
  pattern; UI lint/alignment clean.

## v3 — Instruments per row

- Goal: `C-4 XX` — any instrument on any row.
- Deliverables: instrument table (~65 factory + 32 user), per-row `inst` byte,
  apply-before-note-on, cross-engine fade, protocol + format updates.
- Exit: R5 accepted; documented tail-morph caveat verified; no CPU budget
  regression.

## v4 — Capacity

- Goal: more patterns and longer arrangements, once the matrix is proven.
- Deliverables: pool 8–16 slots, order list 64 rows; storage decision (Q3)
  resolved per [adr/0003](adr/0003-pattern-pool-and-storage.md); protocol and
  format updates.
- Exit: R6 accepted; pool round-trip and recovery tests; flash map documented
  and within budget.

## v5 — Timing

- Goal: breakcore granularity.
- Deliverables: `DIV` 1/64, 1/128, 32T, 64T; `NSTEP` 128 (banks, protocol index,
  motion `place` encoding — today `track<<6 | step`); storage per the chosen
  model.
- Exit: R7 accepted; 64-bar stress song at 1/128 without drift (host test);
  existing projects unchanged (round-trip golden).

## Risks

| Risk | Impact | Mitigation |
| --- | --- | --- |
| RAM nearly full (`.data+.bss` ~7 KB spare) | new features don't fit | MVP consumes no RAM growth (spare bytes only); measure every change against the linker report |
| Matrix UI on a 240×240 screen | unusable arrangement screen | prototype via `ui_render` before implementing input; 4 columns is the simplest possible grid |
| Motion merge limits with mixed rows (64 records shared) | dense songs drop commands | documented merge rule in MVP; per-track capacity in phase 2 |
| Flash map conflicts | bricking risk | keep the map table authoritative; never touch OTA staging; validate installs with `fm1_install.py` dry runs |
| Upstream divergence | merge pain | surgical changes, rebase each phase, ADRs record rationale |
| Scope creep | nothing ships | phases frozen by requirements; new ideas go to open questions |

## Open questions

- Q-M1..Q-M4 ([requirements.md](requirements.md)): row count for the MVP, matrix
  gestures, mute scope, empty-cell semantics.
- Q1–Q4 for later phases: pattern record shape, `NSTEP` banks, storage location;
  block-ref timing answered by [adr/0009](adr/0009-mvp-pattern-matrix.md).
