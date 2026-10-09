# Architecture

Status: draft. Source of truth for the module map is the code itself; this
document maps **where tracker features attach** and the invariants they must keep.

## The shape of the firmware

Felucca is a unity build (no headers between modules; `main.c` includes the
sources in order). Two execution contexts:

- **Audio ISR** (per audio block): renders engines and voices, advances the
  sequencer (`seq.c`), applies motion values, runs FX. It **only reads** bounded
  structures; it never allocates, never touches flash.
- **Main loop**: everything else — UI (`ui_*.c`), input, storage writes, editor
  SysEx, name entry.

Shared state is published between them by convention:

- main-loop writes to pattern/motion data go through `load_begin`/`load_end`
  (undo) and motion publishes under `motion_guard()` (IRQ off/restore).
- engine changes are requested (`eng_req`) and applied by the ISR at a block
  boundary (`voice.c engine_block`, crossfade `xf_*`).

## Module map and attachment points

| Area | File(s) | Tracker change |
| --- | --- | --- |
| Data model | `core.h` | step fields (instrument, pan, delay), pool and order-list structs |
| Sequencer | `seq.c` | row playback, command application, instrument apply, LPB/`DIV` |
| Commands | `motion.c` | per-track command store, capacity, lock/event semantics |
| Song/arrangement | `song_chain.c` | order-list rows, block refs, per-occurrence mute |
| Projects/storage | `project.c`, `storage.c` | FUN10 record, flash map, slot count |
| UI pages | `ui.c`, `ui_graph.c`, `ui_draw.c` | tracker view, order list, matrix, pool pages |
| UI input | `ui_input.c`, `ui_layer.c`, `ui_name.c` | keymaps, hex entry mode, layer integration |
| Parameters | `params.c` | `G_SLOT` range, `DIV` enum additions |
| Protocol | `editor.c`, `web/EDITOR_PROTOCOL.md` | new step byte, motion ops, order list ops, INFO tags |
| Host tests | `tests/` | model round-trips, timing, UI lint, emulator equivalence |

## Change principles

1. **Attach to existing subsystems.** Commands are motion records, not a new
   effect engine. Instruments are preset applications, not a new patch system.
   Arrangement is the chain, not a second song format.
2. **Append-only stored data.** Enum values appended; parameter ids appended
   before `P_E0`/`P_COUNT` as upstream does; formats bump with new magic
   (`FUN10`) and explicit sizes.
3. **Small first.** Each phase must ship independently: a tracker user should get
   value from phase 1 (64 rows) and phase 2 (commands) without phase 3–5.
4. **No behaviour without a test.** Each phase names its host tests and UI render
   checks up front ([testing.md](testing.md)).
5. **Undo is part of the feature.** New destructive edits go through
   `load_begin`/`load_end` (SAVE held undoes).

## Sequencing and timing

- `seq.c` owns the step clock: `P_SLEN` (length), `P_SDIV` (division = LPB), swing,
  gate; per-track phases stay in sync (`chain.carry`).
- Commands are applied at step start, after chance/ratchet resolution, in the same
  place parameter locks are applied today (`motion_step`).
- Instruments are applied **before** the row's note-ons, in the same ISR step.
- The order list is copied into the chain sources **before PLAY** (main loop);
  the ISR only follows indices (`song_chain.c` comment: "The ISR changes only
  their index and four timing parameters").

## Storage path

- Project/pool records live in `.noinit` RAM (fast, survives reset) and are
  mirrored to flash on save; the flash copy is the authority after a cold boot.
- The flash map and its free budget are documented in [formats.md](formats.md);
  moving boundaries is allowed but must not overlap OTA staging or the autosave.
- All writes are CRC-checked records with a validity check on load
  (`storage.c`, `project.c`).

## What must not change

- The audio ISR's read-only, bounded-access property (NFR4).
- The "no flash writes while playing" rule (upstream behaviour; saves stop the
  transport).
- Existing projects/presets continue to load (converted, never silently dropped).
- The per-engine CPU budgets (NFR1) and the linker region limits (NFR2/NFR3).
