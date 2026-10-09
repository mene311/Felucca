# ADR-0016: Base is Felucca 1.4; its matrix is inherited

- Status: accepted (2026-10-09) — product call ("build the latest")
- Deciders: mene311

## Context

The fork was designed against 1.1.5.1. Upstream then shipped **Felucca 1.4**
(merged here), which already provides:

- per-track section slots (`chain_row_t { uint8_t slot[4]; uint8_t repeat; }`,
  `CHAIN_SILENT` = a track plays nothing), i.e. the mix-and-match matrix model;
- SONG-page editing: section / track / pattern (A–D, −) / repeats, add-section
  copies the previous section (inherit-on-create);
- FUN10 (`FUNA`): 104 parameters, **128 motion records** (3-byte events), a
  3,840-byte record that fills the storage payload;
- the 1.2 page reorganisation (SONG under GLO).

Everything in the design docs that assumed 1.1.5.1 numbers is obsolete.

## Decision

- **Rebase the fork on upstream 1.4** and maintain it against upstream main
  (merge before each milestone; [adr/0001](0001-fork-and-upstream-policy.md)).
- **Inherit upstream's matrix** (data model + SONG editing) — do not rebuild it.
- Our **v1 deltas** become:
  1. loop the arrangement (upstream stops at the end, [adr/0015](0015-arrangement-loops.md));
  2. a MATRIX bird's-eye screen (4 track columns × sections) over the same data;
  3. row move/reorder (upstream has add and delete);
  4. mute policy: decide between a mute flag that preserves the reference and
     upstream's `−` silent value ([adr/0012](0012-matrix-cell-semantics.md));
  5. branding: **FM-1 TRACKER** (ABOUT/splash; GPL credit to Felucca kept);
  6. a hardware-validated 3:30 song (16 sections × repeats is ~17 minutes of
     capacity at 32-line patterns, 1/16, 120 BPM).
- Later releases (tracker typing, note columns/OFF, per-row instruments,
  capacity, LPB) are unchanged.

## Consequences

- v1 is much smaller than planned: no format work is required for the matrix
  itself; new storage is only needed if the mute flag is chosen (rows can be
  packed to make room: 3 bytes/row vs 5 today).
- All docs are re-based on 1.4 constants (FUN10, 104 params, 128 motion records,
  3,840-byte records, `.noinit` 15,696 B).
- The tracker-specific work (v2+) remains the fork's differentiator.

## Alternatives

- Stay on 1.1.5.1: rejected — upstream moved; rebuilding what 1.4 ships is waste.
- Rebuild the matrix in our own format: rejected — duplicate work, merge pain.
