# Data model

Status: draft. All sizes verified against the tree (1.1.5.1). Proposed items are
marked **proposed**.

## Current structures (verified on Felucca 1.4)

```c
typedef struct {                 /* 11 bytes in RAM; 9 packed on disk */
    uint8_t note[4];             /* up to 4 notes per step */
    uint8_t n, time, flags, vel, hit, acc, probability;
} step_t;

typedef struct { uint8_t place, param; int8_t value; } motion_event_t;       /* 3 B */
typedef struct { uint8_t count, on, rsv[2]; motion_event_t event[128]; } motion_store_t; /* 388 B */
#define MOTION_MAX 128u          /* shared by the four tracks (1.2/FUN10: 64 before) */

#define CHAIN_ROWS 16u
#define CHAIN_SILENT 4u          /* "-": that track plays nothing in the section */
typedef struct { uint8_t slot[NTRK], repeat; } chain_row_t;                  /* 5 B */
typedef struct { uint8_t count, rsv[3]; chain_row_t row[16]; } chain_config_t; /* 84 B */
```

Upstream 1.4 already ships the matrix data model and its SONG-page editing
(section / track / pattern A-D or "-" / repeats; a new section copies the
previous one). Playback stops at the last section — looping
([adr/0015](adr/0015-arrangement-loops.md)) is ours to add.

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

### 4. Matrix rows — provided by upstream 1.4

`chain_row_t { uint8_t slot[NTRK]; uint8_t repeat; }` already gives per-track
pattern selection per section, with `CHAIN_SILENT` ("-") as the silent value;
a new section copies the previous one (inherit-on-create). v1 deltas:

- **Loop** the arrangement ([adr/0015](adr/0015-arrangement-loops.md)).
- **Move / reorder** sections (upstream has add and delete).
- **Mute policy**: "-" silences but forgets which pattern was there; a mute flag
  that preserves the reference is the open choice
  ([adr/0012](adr/0012-matrix-cell-semantics.md)). Rows can be packed to make
  room: slots 0..4 (3 bits each = 12 bits) + repeat (4 bits) + mute (4 bits) =
  3 bytes/row vs 5 today — saving 32 bytes of the 80-byte chain block.

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
| `.data + .bss` | 98,304 B | **94,420 B** (1.4) | ~3.9 KB spare |
| POOL | 344,064 B | **332,100 B** (1.4) | ~12 KB spare |
| `.noinit` | 15,696 B | 4 x 3,840 B slots (1.4) + crash record | guard above; no room to grow |

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

| Resource | Today | v1 (MVP) | v2 | v4 |
| --- | --- | --- | --- | --- |
| Unique patterns | 4 | 4 | 4 | 8–16 |
| Order-list rows | 16 | 16 | 16 | 64 |
| Row shape | `{slot, repeat}` | `{refs, repeat, mute}` | same | same |
| Repeat per row | 1..16 | 1..16 | 1..16 (255 optional) | 1..255 |
| Commands per pattern | 64 shared | 64 shared | 256/track | 256/track |
| Step fields | note/vel/time/flags/hit/acc/chance | unchanged | + inst/pan/delay | same |
| Pattern length | 64 | 64 | 64 | 64 (128 in phase 5) |
