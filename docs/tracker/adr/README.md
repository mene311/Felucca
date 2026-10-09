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
| [0006](0006-arrangement-model.md) | Order list now (whole-pattern refs), block refs + mute later | accepted |
| [0007](0007-on-device-hex-input.md) | Hex entry on the 16 white keys; knobs are a fallback | accepted |
| [0008](0008-format-and-protocol-versioning.md) | New magic (`FUNA`), append-only protocol ops, capability tags | accepted |

To add a decision: copy the template of an existing ADR, take the next number,
and add it here.
