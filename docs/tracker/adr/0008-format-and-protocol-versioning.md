# ADR-0008: Format and protocol versioning

- Status: accepted (2026-10-09)
- Deciders: mene311

## Context

Felucca stores projects in versioned CRC-checked records (FUN1–FUN9) and speaks a
versioned SysEx editor protocol whose `INFO` reply carries capability tags. Its
rule: newer firmware reads older stores; older firmware refuses newer ones.

## Decision

- New records use the next free magic tag (`FUNB` after upstream 1.4's `FUNA`), an explicit
  size, and a `_Static_assert` pinning the layout, as upstream does.
- Conversions are one-way and lossless where possible; fields that cannot map are
  explicitly defaulted and documented (e.g. locks on parameters an older layout
  lacks).
- The SysEx protocol is extended **append-only**: new ops and new capability tags;
  existing byte layouts never change meaning.
- Every format/protocol change updates `web/EDITOR_PROTOCOL.md` in the same
  commit (see testing.md, definition of done).
- Stored enum additions are append-only (e.g. `DIV`); parameter ids are appended
  before `P_E0` as upstream does so count-based mapping keeps working.

## Consequences

- Downgrades (returning to official V15 or an older Felucca) lose tracker data —
  documented; users are warned by the installer's existing "Return to official
  V15" flow.
- Editors that do not know the new tags keep working for everything else.
- Each phase pays a small permanent tax: conversion code and format constants
  stay in the tree.

## Alternatives

- Reusing spare bytes inside FUN9 without a new version: rejected — silent
  incompatibility with older firmware, no way to refuse safely.
- A separate side-file for tracker data: rejected — breaks the single-record
  save/load model, backup/restore, and the autosave.
