# Felucca Tracker — design notes

> Draft, 2026-10-09. Design decisions taken while turning this Felucca fork (1.1.5.1)
> toward a tracker workflow. Nothing here is implemented yet except the dev-loop
> enabling patch listed at the bottom ("Dev loop").

Baseline: fork of [Felucca](https://github.com/hugelton/Felucca) (M-VAVE FM-1 custom
firmware), **GPL-3.0-only** — this fork stays GPL-3.0.

The engine is already a step sequencer: 4 tracks x 64 steps (`NSTEP`), per-track
`LEN` / `DIV` / `SWG` / `GATE`, step flags (accent / slide / ratchet), per-step
chance, and 64 sparse motion records per project (`firmware/src/motion.c`) as
*automation events* (value holds) and *parameter locks* (value on that step only,
`MOTION_LOCK` in `core.h`). The SysEx editor protocol (`web/EDITOR_PROTOCOL.md`)
already exposes step writes and motion ops — an external tracker front-end can be
built before any firmware change.

## Target model (decisions)

1. **Arrangement-first.** A song is an order list of pattern references over a small
   reusable pattern pool.
   - Pattern pool: **8 unique** to start; **up to 16** with the storage change below.
   - Order list (SONG page): **16 rows today -> ~64**, each row
     `pattern x repeat` (repeat is a uint8; extend 1..16 to 1..255).
   - Pattern content is shared by reference: rows are aliases; editing a pattern
     updates every row that uses it.
2. **Commands per row (the tracker core).** Typed effect commands next to the note
   resolve to motion records: *lock* applies on that row only, *event* holds.
   - Upfront/default columns: velocity exists (`step_t.vel`); **pan** and **delay**
     need per-step fields (delay does not exist per note at all today — only global
     swing/gate).
   - Other commands target any `motion_param`-eligible parameter by id (sound
     parameters; transport, routing and discrete engine changes stay excluded).
   - Capacity: motion is **one record per (track, step, param)** and **64 records per
     project shared by all 4 tracks** — the pool must become per-track and much
     larger for tracked density.
3. **Per-step instrument byte** (patch change, Renoise's `C-4 XX`):
   - One byte per step; instrument = engine + factory/user preset (~65 factory
     sounds + 32 user presets ~ well inside one byte).
   - Applies the target preset's parameters before the row's note-on.
   - Caveat: parameters are **track-level**, so release tails morph with the patch.
     Fine for mono lines, bass and drums; wrong for overlapping sustained pads.
   - **Per-voice instruments are rejected for now**: every engine reads
     `track_t.p[]`; per-voice would need per-voice parameter context *and* per-voice
     engine state (FM6 patches, PHYS models, ...) across 13 engines, against the
     enforced per-engine CPU budgets. That is a synth rearchitecture.
   - Engine changes (ANALOG -> FM6) take the existing fade/switch path (`eng_req`,
     `xf_on`): usable between steps, glitchy if switched every step.
4. **`DIV` is the LPB selector** (per track): `1/16` = LPB 4, `1/32` = LPB 8,
   `8T` / `16T` = triplets. Breakcore reach wants `1/64` (LPB 16) / `1/128` (LPB 32)
   and `32T` / `64T` — append-only enum additions (stored indices stay valid).
   Finer `DIV` shrinks the musical span of a fixed 64-step pattern, so
   **`NSTEP` 64 -> 128/256** is the coupled decision (storage x2/x4).
5. **Pattern Matrix layer (later).** `chain.source[slot]` is already per track
   (`step[track][64] + timing[track][4] + motion`), so per-track block references in
   song rows are a small format change: row = 4 block refs + mute mask (~4-6 B
   instead of today's 2 B `{slot, repeat}`).
   - Operations map: move / reorder refs; copy = duplicate a ref (free) vs clone =
     slot write; **alias** = two rows referencing the same block (already implicit);
     **block mute** = 4-bit mask per row (new — today mute is a global track
     parameter); clone-fill = write one block into consecutive rows.
   - "Show identical repeated slots" = compare refs in the view.
   - Motion / commands are pattern content, so they alias with the block naturally.

## Storage & RAM budget (measured)

- `step_t` = 11 B in RAM; disk steps pack to **9 B**; a full self-contained record
  (FUN9) = **3,648 B**; motion store = 260 B.
- RAM: `.noinit` region `0x01C7C000 + 0x3D50` = **15,696 B**; the 4 cached project
  slots use 14,592 B (~1.1 KB spare).
  - It **cannot grow upward**: `0x01C7FD50..0x01C7FFFF` is hardware write-protected
    (boot info / mailbox / vectors, `firmware/hal/fm1_guard.h`).
  - There is an unclaimed 32 KB gap below (`0x01C74000..0x01C7C000`); its owner is
    undocumented — verify before using.
  - 8 cached slots need ~29 KB: either that gap, or **load patterns on demand** from
    flash and cache only the active + next pattern (a working pattern is 4 x 64 x
    11 B = 2.8 KB).
- Flash (1 MB): app region ends ~`0x8E0DC`; user data `0x97000..0xFFFFF` ~ 420 KB:
  projects 32 KB (4 slots x 4 KB sectors x A/B copies), samples 240 KB, presets
  16 KB, OTA staging 20 KB, autosave 8 KB, FM6 patch copies 8 KB, settings 4 KB.
- Pattern pool, one 4 KB sector per pattern: 8 = 32 KB (fits the current projects
  area, loses the A/B copy), 16 = 64 KB, 32 = 128 KB, 64 = 256 KB (each borrows from
  the 240 KB samples area). Dual copies double the numbers.
- **Steps-only patterns** (sounds shared song-wide, tracker-style) = 4 x 64 x 9 B ~
  2.3 KB + commands -> one sector each. This is the model that makes 16-32 patterns
  safe.

## On-device input (no external keyboard)

- **16 white keys = hex digits `0`-`F`**, one tap per digit; 11 black keys =
  functions (field left/right, line up/down, backspace, insert/delete,
  commit/cancel, lock/event toggle).
- Knobs stay a fallback (value scrub / audition) — they are relative detent
  encoders, no absolute state to jump.
- Keys are binary (fixed velocity 100 in `seq.c`), so dynamics are typed columns
  anyway — the tracker convention.
- Precedents already in the firmware: `ui_name.c` keypad (16 white keys, multi-tap
  for 44 characters, black-key functions with hold-to-repeat), quick layers
  (hold a button -> the keys change job), and the keycap hint renderer (27 pills).

## Dev loop

- **Phone (Termux, aarch64)** — no JieLi toolchain needed for this:
  - always re-generate `build/gen/*` headers first (Python only; `pip install
    fonttools` was needed once);
  - `clang` host builds: `tests/hostsim.c` -> WAV renders; `tests/ui_render.c` ->
    every screen x palette as PPM/PNG with layout lint and alignment checks
    (146 screens x 10 palettes, 0 findings on the phone).
  - Requires the `tools/aa_raster.py` no-Raqm fallback committed here (Termux
    Pillow has no libraqm; official builds with Raqm are unaffected).
- **Laptop (x86-64)** — JieLi clang 4.0.1 + AC79 SDK for the installable `.fwsc`
  (`BUILDING.md`); Emscripten for the interactive browser emulator
  (`web/emu/build.sh`), which accepts computer-keyboard input — the bench for
  typing-effect-commands before flashing hardware.
- Renoise manual pages (reference for the model: pattern matrix, effect commands,
  mixer) are kept as local wikitext copies and are **not committed** (copyright).

## Formats / protocol to bump (when implemented)

- Step write: +1 byte (instrument) -> 14 bytes; packed project steps 9 -> 10 B;
  INFO capability tag.
- Order list: 16 -> ~64 rows (format bump), later per-track block refs + mute mask.
- `DIV` enum: append `1/64`, `1/128`, `32T`, `64T` (list shared with ARP rate and
  DLY time).
- `NSTEP` 64 -> 128/256: step arrays, UI 16-step banks (`pages = (len + 15) / 16`),
  protocol step index, motion `place` encoding, project formats.
