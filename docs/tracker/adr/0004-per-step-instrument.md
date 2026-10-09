# ADR-0004: Per-step instrument, not per-voice

- Status: accepted (2026-10-09)
- Deciders: mene311

## Context

Renoise lets any note column play any instrument (`C-4 XX`), including
overlapping notes with different instruments. Felucca's engines read their
parameters from `track_t.p[]` — one parameter set and one engine per track,
shared by all voices of that track. Engine switching already has a fade path
(`eng_req`, `xf_on`).

## Decision

- Ship **per-row instrument selection**: a one-byte index into an instrument
  table (engine + factory/user preset, ~100 entries).
- Apply the instrument before the row's note-ons: same-engine = parameter copy
  (seamless); cross-engine = the existing fade/switch path.
- **Do not** implement per-voice instruments. Rejected reason: every engine
  (`eng_*.c`, 13 engines) reads track-level parameters; per-voice would need a
  per-voice parameter context and per-voice engine state (FM6 128-byte patches,
  PHYS models, granular, ...) plus per-voice LFO/modulation semantics — against
  the enforced per-engine CPU budgets (NFR1) and the ~7 KB spare `.data+.bss`.

## Consequences

- Documented caveat: changing instrument changes the timbre of everything still
  sounding on that track (release tails morph). Acceptable for mono lines, bass
  and drums; not a pad solution.
- Polyphony with different instruments = use different tracks (4 tracks are 4
  simultaneous instruments); this is also the tracker convention of "instrument
  per channel".
- The instrument byte fits the planned step growth (ADR-0008) and the protocol
  step write extension.

## Alternatives

- Per-voice parameters for a subset of engines: rejected — partial, confusing
  semantics, and still a large DSP change.
- Instrument = engine only (keep the preset): rejected — users expect the patch
  to change with the instrument number.
