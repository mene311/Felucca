# Felucca Tracker — documentation

A tracker-oriented fork of [Felucca](https://github.com/hugelton/Felucca), the
custom firmware for the M-VAVE FM-1. This directory is the engineering record:
requirements, architecture, data model, formats, UI/input, testing and the
decision log. **Code follows these documents — not the other way round.**

- Upstream: `github.com/hugelton/Felucca` (GPL-3.0-only; this fork stays GPL-3.0)
- This fork: `github.com/mene311/Felucca`
- Baseline: Felucca 1.1.5.1

## Status

| Phase | What | State |
| --- | --- | --- |
| 0 | Design docs (this set) | draft — under review |
| 0 | Dev loop (phone host builds, laptop toolchain, browser emulator) | **done** |
| 1 (MVP) | **Pattern matrix over the 4 existing patterns**: per-track refs, repeat, per-occurrence mute, MATRIX screen ([adr/0009](adr/0009-mvp-pattern-matrix.md)) | not started |
| 2 | Tracker view + hex command columns over the motion system | not started |
| 3 | Per-step instrument byte | not started |
| 4 | Capacity: pattern pool 8–16, order list 64 rows | not started |
| 5 | `DIV` extensions (1/64, 1/128) + `NSTEP` 128 | not started |

## Documents

| Doc | Contents |
| --- | --- |
| [requirements.md](requirements.md) | goals, non-goals, numbered requirements with acceptance criteria |
| [architecture.md](architecture.md) | module map, task/ISR split, where tracker features attach, invariants |
| [data-model.md](data-model.md) | current structs and sizes, proposed model, RAM/flash budgets |
| [formats.md](formats.md) | project formats (FUN9 → FUN10), SysEx protocol changes, versioning policy |
| [ui-input.md](ui-input.md) | screens, keymaps, hex entry, screen budget, interaction rules |
| [testing.md](testing.md) | test infrastructure, definition of done, dev commands |
| [roadmap.md](roadmap.md) | phases, entry/exit criteria, risks, open questions |
| [adr/](adr/) | architecture decision records (the "why" behind each choice) |

## Glossary

- **Step / row** — one of the 64 time slots of a track's sequence (`NSTEP`).
  "Row" is the tracker term; "step" is Felucca's.
- **Pattern** — a track's 64-step sequence with its `LEN` / `DIV` / `SWG` / `GATE`.
  In this fork's pool, one pattern entry holds the sequences of all four tracks.
- **Slot** — a numbered entry of the pattern pool (Felucca's project slots A–D
  today, 8–16 in this fork).
- **Order list / song** — the arrangement: rows referencing pool entries with
  repeat counts (`chain_config_t` today, `SONG` page in the UI).
- **Block** — one track's content inside one pattern slot (Pattern Matrix term).
- **Command** — a typed `xx yy` value pair in a row next to the note; resolves to a
  motion record.
- **Lock / event** — the two motion record kinds: a lock applies on its row only,
  an event sets the value and it holds (`MOTION_LOCK`, `motion.c`).
- **Instrument** — an engine + preset pair selectable per row (proposed).
- **LPB** — lines per beat. Felucca's `DIV` is the per-track equivalent.

## Conventions (upstream rules this fork follows)

1. **Stored enums are append-only** — stored indices keep their meaning; new
   values are appended (e.g. `N_DIV`, `N_QUOTE` and the parameter table).
2. **New firmware reads old stores; old firmware refuses new ones** — format
   magic/size checks are explicit (see [formats.md](formats.md)).
3. **No flash writes while playing** — saves stop the transport first.
4. **User-data edits are undoable** — the `SAVE`-held one-step undo
   (`load_begin` / `load_end`); destructive actions go through dialogs.
5. **The audio ISR only reads bounded structures** — main-loop writes publish under
   the IRQ guard (`motion_guard`, `fm1_icfg`).
6. **Regressions are checked, not assumed** — golden audio hashes, CPU/target
   budgets, UI layout lint and alignment (see [testing.md](testing.md)).
