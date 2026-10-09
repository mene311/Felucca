# Formats and protocol

Status: draft. Rule zero (upstream behaviour, kept here): **new firmware reads old
stores; old stores are never written back; old firmware refuses newer stores by
magic/size hash.** The SysEx protocol is extended append-only with capability tags.

## Project record formats (flash / `.noinit`)

| Format | Size (B) | Added |
| --- | --- | --- |
| FUN1 | 688 | first stored format |
| FUN2 | 2,552 | modulation matrix |
| FUN3 | 2,584 | SLICER |
| FUN4 | 2,680 | drum grid |
| FUN5 | — | (read-only) grid without chain |
| FUN6 | — | + 36-byte song chain |
| FUN7 | 3,388 | 89 parameters |
| FUN8 | 3,584 | self-contained FM6 patches |
| FUN9 | **3,648** | DRUM lane levels (P_COUNT 99) |
| **FUN10** | *proposed* | tracker: per-row fields, 64-row order list, per-track command stores |

Storage layout today (`storage.c`):

- `ST_SECTOR` = 4096; each object keeps **two sector copies** (A/B); a save writes
  both, a load takes the valid one (CRC).
- Project/pool objects: `0x97000 + obj * 2 * ST_SECTOR + copy * ST_SECTOR`
  (4 objects × 8 KB = 32 KB), FM6 patch copies at `0x9F000` / `0xFE000`,
  autosave at `0xE5000`/`0xE6000`.
- A record larger than one sector payload must either span sectors with an
  explicit size field (as today) or move to a counted multi-sector record —
  decide with FUN10.

## FUN10 sketch (to be finalised in phase 1)

Proposed layout (all little-endian, as today; exact offsets pinned by a
`_Static_assert` like the current formats):

```
magic u32 (next free tag after "FUN9" 0x46554E39: use "FUNA" 0x46554E41)
size/version u32
globals g[G_COUNT].i16   select, parts, phys, flags
per track ×4:
  p[P_COUNT].i16
  engine u8, preset u8
  step[NSTEP] packed: 12 B each (notes, time, flags, vel, chance, ratchet,
                                hits/acc, inst, pan, delay)
chain: count u8, rsv[3]; rows ×64 { slot u8, repeat u8 }          (phase 1)
       later ×64 { slot[4] u8, repeat u8, mute u8 }               (phase 4)
motion: per track { count u16, on u8, rsv; events ×N {place u8, param u8, value i16} }
FM6 patches ×4 ×128
name[?]  checksum u32
```

- Size budgeting (single pattern): steps 4×64×12 = 3,072 B; params/patches as
  today; motion 4×256×4 ≈ 4 KB → a full record is ~8–9 KB, i.e. **three 4 KB
  sectors** per copy, or the steps+commands-only split from
  [adr/0003](adr/0003-pattern-pool-and-storage.md).
- Alternative (recommended by the ADR): the **pattern pool stores steps +
  commands only** (~3–5 KB), sounds stay in one song-wide state block. This keeps
  one pattern inside one sector pair and removes per-pattern sound duplication.

## SysEx protocol (`web/EDITOR_PROTOCOL.md`)

Current relevant parts (v7 / 1.1):

- `STEP_GET` / `STEP_SET` / `TRACK_STEP`: 8-byte legacy, 11-byte grid, 12-byte
  with chance, 13-byte with ratchet.
- `MOTION` (64): query / on / clear / set event / delete / set lock / clear locks /
  query with kinds. Reply carries `(step, id, value, kind)`.
- `PROJECT` (9): `slot 0..3` — load / save / query.
- `INFO` (1) carries capability tags: `uiCaps`, motion (`4D 01 64 01`),
  ratchet, parameter locks (`4C 01 01`), MENU settings count, etc.
- `BACKUP_*` (65–67): object list with sizes and CRCs.

Proposed extensions (append-only, each behind a new INFO tag):

| Tag / change | Meaning |
| --- | --- |
| step write 14 B | + `inst` (phase 3); 15–16 B when `pan`/`delay` land |
| `4F 01 01` (example) | order list: query, set row, insert, delete, move, set repeat |
| pool slot range | `PROJECT` slot 0..7 (phase 1); the byte already allows it |
| motion per track | `MOTION` scope changes from per-project to per-track; new capability tag; old ops keep replying byte-for-byte for track 0 semantics? — **must be specified before phase 2** |
| pattern transfer | new objects in `BACKUP_*` for the pool (one per slot) with sizes/crcs |

Compatibility rules:

1. Existing editors (Felucca-WebApp) MUST keep working for everything they know;
   unknown capability tags are ignored by design.
2. Removing or changing the meaning of an existing op is forbidden; supersede
   instead.

## Flash map (proposed phase 1 allocation)

| Region | Range | Size | Change |
| --- | --- | --- | --- |
| App | 0x00120 – ~0x8E0DC | ~580 KB | unchanged |
| gap | 0x8E0DC – 0x97000 | ~36 KB | candidate for pool growth (Q3) |
| Pool | 0x97000 – 0x9EFFF | 32 KB | 8 patterns single-copy (or grow) |
| FM6 copy A | 0x9F000 | 4 KB | unchanged |
| Samples | 0xA0000 – 0xDBFFF | 240 KB | shrinks if pool > 32 KB |
| Presets | 0xDC000 – 0xDFFFF | 16 KB | unchanged |
| OTA staging | 0xE0000 – 0xE4FFF | 20 KB | must stay clear |
| Autosave | 0xE5000 – 0xE6FFF | 8 KB | unchanged |
| Settings | 0xFC000 | 4 KB | unchanged |
| FM6 copy B | 0xFE000 | 4 KB | unchanged |

Sizing per option: 8 patterns single copy = 32 KB; 16 = 64 KB; 32 = 128 KB;
dual copies double. A steps-only pattern (~2.5–5 KB) fits one sector per copy.

## Version exposure

- `FELUCCA_VERSION` (splash / ABOUT) and the package identity (FM-1_9XY) keep
  upstream's scheme; the fork's tracker features must be visible in `INFO` so a
  connected editor can adapt without guessing.
