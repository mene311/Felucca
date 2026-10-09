# ADR-0011: v1 pattern length is 32 lines (test scope)

- Status: accepted (2026-10-09) — product call
- Deciders: mene311

## Context

Patterns today run 1..64 steps per track (`LEN`, default 16). The MVP matrix
needs short, quickly testable patterns; 32 lines is also a natural tracker page
size (two 16-step banks).

## Decision

- v1 works with **32-line patterns** as the default in tests and demos
  (`LEN` 32), so arrangements are quick to build and review.
- The engine keeps `NSTEP` 64 and per-track `LEN` 1..64, so existing 64-step
  projects load unchanged. **32 is a default, not a hard cap.**
- If a hard 32-line limit was intended, that is a separate compatibility call
  (it would truncate existing 64-step projects) — recorded as an open question.

## Consequences

- The MATRIX and the later tracker view show two 16-step banks for the default.
- No format, motion or storage change: this is a default value, not a new limit.
- Test songs and the acceptance case use 32-line patterns; 64-line remains
  available.

## Alternatives

- Hard `NSTEP` 32: rejected for now — truncates existing projects, shrinks
  capability, and buys little (the RAM saving is real but small).
- Keep default 16: rejected — too short to exercise arrangement.
