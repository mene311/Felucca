# Architecture Decision Records

One file per decision. Format: status, context, decision, consequences,
alternatives. Statuses: `proposed`, `accepted`, `superseded by ADR-XXXX`.

| ADR | Decision | Status |
| --- | --- | --- |
| [0001](0001-fork-and-upstream-policy.md) | Fork and upstream policy — surgical changes, upstream merges stay possible | accepted |
| [0002](0002-commands-as-motion-records.md) | Effect commands are motion records (locks/events), per-track capacity ≥ 256 | accepted |
| [0003](0003-pattern-pool-and-storage.md) | Pattern pool: 8 slots, on-demand loading; evaluate steps+commands-only records | proposed (Q1/Q3) |
| [0004](0004-per-step-instrument.md) | Per-step instrument byte; per-voice instruments rejected | accepted |
| [0005](0005-div-as-lpb-and-nstep.md) | `DIV` is the LPB selector; `NSTEP` 64 → 128 paired with it | accepted |
| [0006](0006-arrangement-model.md) | Order list now (whole-pattern refs), block refs + mute later | accepted; phasing superseded by [0009](0009-mvp-pattern-matrix.md) |
| [0007](0007-on-device-hex-input.md) | Hex entry on the 16 white keys; knobs are a fallback | accepted |
| [0008](0008-format-and-protocol-versioning.md) | New magic (`FUNA`), append-only protocol ops, capability tags | accepted |
| [0009](0009-mvp-pattern-matrix.md) | **MVP = pattern matrix over 4 patterns** (per-track refs, repeat, mute) | accepted |
| [0010](0010-tracker-first-default.md) | Tracker-first default interface — no mode switch | accepted |
| [0011](0011-v1-pattern-length.md) | v1 pattern length is 32 lines (default, not a cap) | accepted |
| [0012](0012-matrix-cell-semantics.md) | Matrix cells: blank inherits, mute gesture B, notes-only | accepted |
| [0013](0013-notes-hold-until-off.md) | Notes hold until an OFF row (v2 playback semantics) | accepted; refined by 0014 |
| [0014](0014-note-columns.md) | Note columns: per-column monophony, chords across columns, glide | accepted |
| [0015](0015-arrangement-loops.md) | The arrangement loops from the first row; no stop marker | accepted |

To add a decision: copy the template of an existing ADR, take the next number,
and add it here.
