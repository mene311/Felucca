# Roadmap

Status: draft. Each phase ships independently, on a branch, and is accepted
before the next starts. Phase scope is frozen by the requirements it implements.

## Phase 0 — Foundations (this doc set)

- Deliverables: `docs/tracker/*`, dev loop (host builds on the phone, toolchain
  and emulator on the laptop), the `aa_raster.py` no-Raqm fallback.
- Exit: documents reviewed; an unmodified fork builds (`felucca.fwsc`) and the
  emulator passes `emu_test.mjs`.
- State: documents in review; everything else **done**.

## Phase 1 — Order list and pattern pool

- Goal: long arrangements from a small pool.
- Deliverables:
  - 64-row order list (`chain_config_t`), repeat 1..255, row operations in the UI.
  - Pool of 8 slots (letters A–H); `G_SLOT` range; storage map either 8 × 1 sector
    in place or 8 × 2 sectors with a 32 KB reclaim from the sample area.
  - FUN10 record + conversions; protocol: order-list ops + pool slot range +
    new capability tags.
  - Resolve Q3 (where the extra storage lives).
- Exit: R1 + R2 accepted; host tests green; `ui_render` clean; save/load
  round-trip with a corrupt-sector recovery; documentation updated.

## Phase 2 — Tracker view and command columns

- Goal: type notes and effects row by row.
- Deliverables:
  - TRACKER screen (see [ui-input.md](ui-input.md)) with cursor, scrolling,
    hex entry, keycap hints.
  - Per-track command stores (≥ 256 records/track/pattern), lock/event semantics,
    two command columns per row.
  - Per-row **pan** and **delay** fields (R4); velocity column shown.
  - EDIT-layer clipboard: copy / paste / insert / clear row.
  - Protocol: motion scope change (capability-tagged), step write extension.
- Exit: R3 + R4 accepted; timing tests frame-accurate; golden playback of a
  scripted pattern documented; UI lint/alignment clean.

## Phase 3 — Instruments per row

- Goal: `C-4 XX` — any instrument on any row.
- Deliverables: instrument table (~65 factory + 32 user), per-row `inst` byte,
  apply-before-note-on, cross-engine fade, protocol + format updates.
- Exit: R5 accepted; documented caveat about track-level parameters tested (tail
  morph behaviour verified); no CPU budget regression.

## Phase 4 — Pattern matrix

- Goal: arrange at block level; per-occurrence mute.
- Deliverables: `chain_row_t {slot[4], repeat, mute}`; MATRIX screen; clone-fill
  and alias operations; repeated-slot indicator.
- Exit: R7 accepted; alias edits propagate; mute never alters sources.

## Phase 5 — Timing extensions

- Goal: breakcore granularity.
- Deliverables: `DIV` 1/64, 1/128, 32T, 64T; `NSTEP` 128 (banks, protocol index,
  motion `place`), storage per the chosen model.
- Exit: R6 accepted; a 64-bar stress song at 1/128 plays without drift
  (host test), existing projects unchanged (round-trip golden).

## Risks

| Risk | Impact | Mitigation |
| --- | --- | --- |
| RAM is nearly full (`.data+.bss` ~7 KB spare) | pattern cache/changes don't fit | on-demand pool loading; small-first phases; measure each change against the linker report |
| Flash map conflicts (OTA staging, samples) | bricking risk if overlapped | keep the map table authoritative; never touch OTA staging; validate with `fm1_install.py` dry runs |
| UI space (240×240) | unreadable tracker | prototype screens via `ui_render` before implementing input |
| Protocol drift (editors) | third-party tools break | append-only ops, capability tags, same-commit doc updates |
| Upstream divergence | merge pain as Felucca evolves | keep changes surgical; rebase on `upstream/main` each phase; ADRs record rationale |
| Scope creep | nothing ships | phases frozen by requirements; later ideas go to the ADR/open-questions list |

## Open questions

From [requirements.md](requirements.md) Q1–Q4: pattern record shape, `NSTEP`
banks/protocol encoding, storage location, block refs timing. From
[ui-input.md](ui-input.md) Q-U1–Q-U4: row pitch, kind marker, OCT paging, matrix
gestures.
