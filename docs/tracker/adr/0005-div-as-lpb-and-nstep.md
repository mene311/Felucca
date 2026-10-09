# ADR-0005: DIV is the LPB selector; NSTEP is the span lever

- Status: accepted (2026-10-09)
- Deciders: mene311

## Context

Renoise exposes LPB (lines per beat) separately from tempo; breakcore and
experimental work uses LPB 16–32 for granularity. Felucca has `P_SDIV` (`DIV`),
per track, currently `1/4 1/8 1/16 1/32 8T 16T 1/2 1/1 2BAR 4BAR`, where
`1/16` = LPB 4 and `1/32` = LPB 8. `NSTEP` = 64 fixed.

Finer `DIV` at a fixed `NSTEP` shrinks the musical span of a pattern
(64 steps = 2 bars at 1/32, 1 bar at 1/64, half a bar at 1/128).

## Decision

- Treat `DIV` as the LPB selector. Append `1/64` (LPB 16), `1/128` (LPB 32),
  `32T`, `64T` to the enum (append-only; stored indices keep their meaning).
- Pair it with **`NSTEP` 64 → 128** in phase 5 so fine divisions keep a usable
  span; 256 is out of budget unless the storage model changes radically.
- The BPM range stays 40–240; the half-speed/double-resolution idiom is
  `DIV` finer + BPM lower.

## Consequences

- The `N_DIV` list is shared with the ARP rate and DLY time; they inherit the
  finer steps (acceptable; document it).
- `NSTEP` 128 touches: step arrays (RAM), UI 16-step banks (`pages = (len+15)/16`
  → 8 pages), the protocol step index (still one byte), motion `place` (byte,
  fine), project formats (bigger packed steps), and the chain/order-list
  bounds. Storage per pattern roughly doubles.
- Alias/pool sizing must be recomputed after the change (ADR-0003).

## Alternatives

- A separate global LPB control distinct from `DIV`: rejected — `DIV` already is
  per-track LPB; a second control would duplicate semantics.
- `NSTEP` 256: deferred — steps alone would be 4 × 256 × 11 B = 11 KB per
  pattern in RAM, beyond the measured budget.
