# Data model

Status: draft. All sizes verified against the tree (1.1.5.1). Proposed items are
marked **proposed**.

## Current structures (verified)

```c
/* core.h */
typedef struct {                 /* 11 bytes in RAM; 9 packed on disk (FUN9) */
    uint8_t note[4];             /* up to 4 notes per step (chord) */
    uint8_t n;                   /* notes in use, 0 = empty */
    uint8_t time;                /* ST_NOTE / ST_TIE / ST_REST */
    uint8_t flags;               /* SF_ACCENT | SF_SLIDE | SF_RATCH (bits 3..4) */
    uint8_t vel;                 /* 0..127 */
    uint8_t hit;                 /* drum lanes (bit per lane) */
    uint8_t acc;                 /* accent bit per hit */
    uint8_t probability;         /* 0 = 100%; 1..100; 101 = silent */
} step_t;

typedef struct { uint8_t place, param; int16_t value; } motion_event_t;   /* 4 B */
typedef struct { uint8_t count, on, rsv[2]; motion_event_t event[64]; } motion_store_t; /* 260 B */

typedef struct { uint8_t slot, repeat; } chain_row_t;                     /* 2 B */
typedef struct { uint8_t count, rsv[3]; chain_row_t row[16]; } chain_config_t; /* 20 B; CHAIN_ROWS 16 */
```

- Track: `int16_t p[P_COUNT]` (P_COUNT = 99 in 1.1) + `step_t step[NSTEP]` +
  runtime state; `NSTEP` = 64.
- Song: `select`, `playing`, `rec`, `octave`, globals `g[G_COUNT]`.
- Chain sources: `chain_pattern_t = step_t step[NTRK][NSTEP] + int16_t timing[NTRK][4] + motion_store_t`
  (the pool entry the song borrows from).

Key semantics:

- **Motion** = sparse step automation: one record per `(track, step, param)`,
  either an *automation event* (value holds from its step) or a *lock*
  (value applies on its step only; `MOTION_LOCK` = bit 7 of `param`).
  Capacity: **64 records per project, shared by all four tracks** (`MOTION_MAX`).
- **Chain** = order list; rows reference pool slots (`slot < 4` enforced by
  `chain_valid`), repeat 1..16; `chain_apply` copies the slot's steps/timing/motion
  into the live tracks and the ISR only walks `seq_idx`.
- **Motion is per source**, not per live track: each `chain_pattern_t` carries its
  own store.

## Proposed changes

### 1. Step fields (phase 2/3)

| Field | Type | Default | Notes |
| --- | --- | --- | --- |
| `inst` | `uint8_t` | 0 = current | instrument index into the pool (engine + preset) |
| `pan` | `int8_t or uint8_t` | −1 = track pan | per-row pan |
| `delay` | `uint8_t` | 0 | 0..255 = row slices (see [ui-input.md](ui-input.md)) |

- RAM: 11 → 14 B/step (or 12–13 B with bit-packing; decide in phase 2).
- Disk (FUN10): packed steps grow from 9 → 12 B (packing keeps RAM/disk apart).
- Budget check: 4 tracks × 64 steps: +3 B × 256 = **+768 B per pattern** RAM.

### 2. Commands (phase 2)

- Store: replace the single `motion_store_t` per source with a **per-track store**:

  ```c
  typedef struct { uint16_t count, on; motion_event_t event[MOTION_MAX_T]; } motion_track_t;
  /* proposed: MOTION_MAX_T = 256 per track per pattern */
  ```

- Record: keep 4 B (`place`, `param`, `value`) + kind bit; per-track capacity 256
  (vs 64 shared) is the minimum for tracked density.
- `place` byte holds the step index 0..255; with `NSTEP` 128 it is still a byte.
- RAM per pattern: 4 × (4 + 256×4) = 4,112 B → this is why commands must live in
  the pattern, and patterns on demand (not all cached).

### 3. Pattern pool (phase 1)

Option A — **full records** (current semantics): one `FUN10` record per pattern,
3.6 KB class. Option B — **steps+commands only**, sounds song-wide: ~2.3 KB +
commands per pattern. See [adr/0003](adr/0003-pattern-pool-and-storage.md).

### 4. Order list (phase 1/4)

```c
/* phase 1: 64 whole-pattern refs */
typedef struct { uint8_t slot, repeat; } chain_row_t;              /* 2 B */
typedef struct { uint8_t count, rsv[3]; chain_row_t row[64]; } chain_config_t;  /* 132 B */

/* phase 4: per-track block refs + mute mask */
typedef struct { uint8_t slot[4]; uint8_t repeat; uint8_t mute; } chain_row_t;  /* 6 B */
typedef struct { uint8_t count, rsv[3]; chain_row_t row[64]; } chain_config_t;  /* 388 B */
```

Project record growth: FUN9 chain block is 36 B; 132 B (+96) fits within the
48 spare bytes? No — 132-36 = +96 > 48 → **new format version required** (FUN10)
or re-use of the reserved areas with a version bump. Decide in phase 1.

### 5. Instruments (phase 3)

- Table: `engine` + `preset` index; enumerates ~65 factory sounds + 32 user slots.
- Stored per row as one byte (0 = keep current, 1..N = table entry).
- Application: copy the preset's `P_*` values for that track before note-on;
  cross-engine goes through `eng_req` + fade.

## Budgets

### RAM (measured, stock build)

| Region | Limit | Used (stock) | Notes |
| --- | --- | --- | --- |
| `.ram_text` | 0x6000 (24 KB) | 916 insns | code in RAM |
| `.data + .bss` | 98,304 B | **91,220 B** | ~7 KB spare |
| POOL | 344,064 B | **331,204 B** | ~12.8 KB spare |
| `.noinit` | 15,696 B | 14,592 B (4 slots) | 1.1 KB spare; guard above |

Implication: extra per-pattern caches do **not** fit by default; the plan is
on-demand loading of pool entries with only the active + next pattern live.

### Flash (1 MB)

| Region | Range | Size |
| --- | --- | --- |
| App (XIP) | 0x00120 – ~0x8E0DC | ~580 KB |
| gap (unassigned) | 0x8E0DC – 0x97000 | ~36 KB |
| Projects/pool | 0x97000 – 0x9EFFF | 32 KB (4 × 2 sectors) |
| FM6 patches (copy A) | 0x9F000 | 4 KB |
| Samples | 0xA0000 – 0xDBFFF | 240 KB |
| Presets | 0xDC000 – 0xDFFFF | 16 KB |
| OTA staging | 0xE0000 – 0xE4FFF | 20 KB |
| Autosave | 0xE5000 – 0xE6FFF | 8 KB |
| Settings | 0xFC000 | 4 KB |
| FM6 patches (copy B) | 0xFE000 | 4 KB |

Pool sizing (one 4 KB sector per pattern, single copy): 8 = 32 KB (current
area), 16 = 64 KB, 32 = 128 KB — each step above 8 takes from the sample area.
Dual copies double the numbers.

## Capacity summaries (proposed)

| Resource | Today | Phase 1 | Phase 2 | Phase 4 |
| --- | --- | --- | --- | --- |
| Unique patterns | 4 | 8 (16 steps-only) | 8–16 | 8–16 |
| Order-list rows | 16 | 64 | 64 | 64 (block refs) |
| Repeat per row | 1..16 | 1..255 | 1..255 | 1..255 |
| Commands per pattern | 64 shared | 64 shared | 256/track | 256/track |
| Step fields | note/vel/time/flags/hit/acc/chance | + inst/pan/delay | same | same |
| Pattern length | 64 | 64 | 64 | 64 (128 in phase 5) |
