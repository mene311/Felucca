# ADR-0001: Fork and upstream policy

- Status: accepted (2026-10-09)
- Deciders: mene311

## Context

Felucca is actively developed upstream (1.1.5.1 at fork time; frequent releases,
including a planned 1.2 reorganisation). The tracker fork needs its own course but
must not lose the ability to take upstream fixes (engine DSP, storage, emulator,
tests).

## Decision

- Keep `upstream` = `github.com/hugelton/Felucca` and `origin` = the fork; pull
  `upstream/main` before starting each phase.
- All tracker work sits in `docs/tracker/` (docs) and surgical code changes;
  no wholesale rewrites of upstream files.
- Tracker-only code paths are additive (new pages, new fields, new format
  version) so upstream's behaviour is default and reachable.
- The GPL-3.0-only licence is preserved; new files carry the SPDX header.

## Consequences

- Upstream merges stay feasible; conflicts concentrate in `core.h`, `seq.c`,
  `params.c`, `project.c` and the `ui_*` pages.
- Features must be expressible as delimiter-separated attachments to existing
  subsystems (see ADR-0002, ADR-0004), which also keeps the diff reviewable.
- If upstream ever ships compatible tracker features, this fork must be able to
  drop its own in favour of upstream's — hence the requirement that stored data
  stays convertible (ADR-0008).

## Alternatives

- Hard-fork and abandon upstream merges: rejected (loses engine/storage fixes and
  the emulator/test infrastructure).
- Contributing tracker features upstream directly: not chosen now; the design is
  opinionated and the iteration loop is faster here. May be revisited per feature.
