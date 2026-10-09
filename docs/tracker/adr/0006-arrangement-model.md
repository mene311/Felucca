# ADR-0006: Arrangement model — order list now, blocks later

> **Amended 2026-10-09:** the phasing is superseded by
> [ADR-0009](0009-mvp-pattern-matrix.md) — the matrix (block refs + mute) is the
> MVP over the existing four patterns; the order list stays 16 rows for now. The
> data-model decision below (rows as per-track block references, aliases by
> shared references, playback borrows stored content) stands.

- Status: accepted (2026-10-09)
- Deciders: mene311

## Context

Felucca's `SONG` is an order list: rows `{slot, repeat}` referencing the 4-slot
pool (16 rows, repeat 1..16). Renoise's Pattern Matrix is a grid of
patterns × tracks where blocks (one track's content in one pattern) can be moved,
copied and aliased, and individual blocks muted per occurrence.

Structurally, Felucca already stores per-track content per slot
(`chain_pattern_t = step[NTRK][NSTEP] + timing[NTRK][4] + motion`), so the block
model exists underneath the whole-pattern reference.

## Decision

- **Phase 1**: extend the order list to ~64 rows of whole-pattern references
  (`{slot, repeat}`), with insert/delete/duplicate/move in the UI. References are
  aliases: rows sharing a slot share content.
- **Phase 4**: change the row to per-track block references
  `{slot[4], repeat, mute}` — the matrix model — with per-occurrence mute.
- Keep the existing playback rule: the order list borrows stored slots' content
  and plays with the current sounds; live editing locks out while the song runs
  ("STOP TO EDIT").

## Consequences

- Phase 1 ships arrangement value with a minimal format change; phase 4 adds
  matrix power at ~+2–4 B per row and a UI screen.
- Alias semantics stay implicit (shared references), matching Renoise's linked
  copies; the UI should mark repeated slots.
- Motion travels with the pattern content it belongs to, so aliased blocks share
  their commands (unlike Renoise's automation caveat, this fork can be cleaner
  here — decide and document in phase 4).

## Alternatives

- Blocks from the start: rejected — bigger format/UI change before the simpler
  arrangement win.
- Independent row copies (no references): rejected — loses alias editing and
  multiplies storage.
