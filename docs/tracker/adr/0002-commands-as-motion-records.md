# ADR-0002: Commands are motion records

- Status: accepted (2026-10-09)
- Deciders: mene311

## Context

A tracker's defining feature is typed effect commands next to the note (`xx yy`
per column). Felucca already has a mechanism that sets a parameter at a step:
motion records (`motion.c`), with two kinds —
**automation events** (value holds) and **parameter locks** (`MOTION_LOCK`,
value applies on its step only). The SysEx protocol already exposes both.

## Decision

Effect commands resolve to motion records `(track, step, param, value, kind)`:

- *lock* = per-row effect (Renoise default for typed commands),
- *event* = value-holds (Renoise's `xyzz` set-and-hold behaviour).
- The addressable set is `motion_param()`-eligible ids (sound parameters);
  transport, routing and discrete engine changes stay excluded as today.
- Storage moves from one shared 64-record pool per project to **per-track stores
  with ≥ 256 records per track per pattern**, because a tracker exhausts 64
  instantly.

## Consequences

- No new effect engine, no new playback path: the ISR keeps reading a bounded
  record array and `motion_step` keeps applying values.
- The "one record per (track, step, param)" restriction stays: the *same* param
  cannot have two values on one row; different params stack via different columns.
- The command UI is presentation over `(param id, value, kind)`; the hex entry
  maps to the parameter table (`params.c`), so no opcode list has to be invented
  or maintained.
- Motion capacity increase has a direct RAM cost (4 B per record); see
  [data-model.md](../data-model.md).

## Alternatives

- A parallel "effect command list" per row (Renoise-style opcode table):
  rejected — duplicates the parameter table, needs a second playback path, and
  would not share undo/locking semantics with existing motion handling.
- Reusing the existing 64-record pool as-is: rejected — too small by more than an
  order of magnitude for tracked density.
