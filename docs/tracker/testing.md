# Testing and development

Status: draft. Rule: **no feature without a test, no regression without a hash.**

## Existing infrastructure (upstream, reused)

`tests/run_tests.sh` builds host binaries with plain `cc` — **no JieLi toolchain
needed** — and runs:

| Area | What it proves |
| --- | --- |
| regression (`regress.c`, `golden.txt`) | every engine × preset, GM map on DRUM, voice modes, FX sends, a 4-track mix: one hash each |
| health / CPU (`cpu_baseline.txt`, `target_budget.txt`) | clipping, DC, voice release, silence; instructions/sample per preset (+25 % cap); disassembly budget (+10 %) |
| UI renders (`ui_render.c`) | every screen × palette: layout lint, grey/neutral, ink alignment (1 px fails), draw cost, text audit, PNG sheets |
| UI behaviour (`ui_test.c`) | UI sources against stub display/buttons/knobs (loads keep steps, dialogues, undo, recording, drum grid) |
| audio / persistence / editor (`audio_test`, `storage_test`, `project_test`, `editor_test`, `backup_test`) | bounded overload, flash retries, FUN round-trips, MENU settings over the protocol |
| engines (`mod_`, `chord_`, `ratchet_`, `perform_`, `reverb_`, `slice_`, `drum_`, `noise_`, `phys_`, `fm4_`, `fm6_`) | per-engine behaviour and cost |
| input (`input_test`) | key/button debounce, encoders' detents, LED scan, breath |
| fuzz (`fuzz_proj`, `fuzz_ed`, `fuzz_smp`) | malformed project/editor/sample inputs |
| emulator (`emu_test.mjs`) | boot, keys, MIDI, screen, LEDs, saves across instances, **bit-for-bit same song as the native build**, cost |

## Environments

**Laptop (x86-64, full build + tests)**

```sh
cd ~/Projects/felucca-tracker
sh build.sh                      # firmware: build/felucca.fwsc
tests/run_tests.sh               # host tests + UI renders + emulator (needs emcc in PATH)
```

Install context (already done): `python-pillow`, `python-fonttools`, `emscripten`
(pacman), JieLi toolchain at `~/.jieli/toolchain`, AC79 SDK at `~/fw-AC79_AIoT_SDK`.
`emcc` lives at `/usr/lib/emscripten/emcc` → run tests with
`PATH=/usr/lib/emscripten:$PATH tests/run_tests.sh`.

**Phone (aarch64, UI/audio iteration without the toolchain)**

```sh
cd ~/Projects/felucca-tracker/felucca
# regenerate build/gen/* (Python only)
python3 tools/gen_aa_font.py build/gen/ui_fonts.h --preset inter-tight   # needs the aa_raster fallback
… (icons, keycaps, palettes, tables, fm6 patches, samples)
# host binaries
clang -O2 -w -Ibuild/gen -Ifirmware/src -o build/host/hostsim tests/hostsim.c -lm
clang -O1 -w "-DFELUCCA_VERSION=\"$FV\"" -Ibuild/gen -Ifirmware/src -Itests \
      -o build/host/ui_render tests/ui_render.c -lm
./build/host/ui_render build/ui_new build/ui_slot && python3 tests/ui_render.py build/ui_new build/ui_slot
```

**Browser emulator (the interactive bench for typing)**

```sh
PATH=/usr/lib/emscripten:$PATH sh web/emu/build.sh     # → build/emu/
python3 -m http.server 8080 --directory build/emu      # open http://localhost:8080
```

## Definition of done (per change)

1. Feature implemented per its ADR and requirements; SPDX headers on new files.
2. **Host tests** added/extended for the change and green (`tests/run_tests.sh`).
3. **UI**: new/changed screens pass layout lint and ink alignment; PNG sheets
   reviewed for GREY / MONO at minimum.
4. **Formats**: round-trip test (new record ↔ old record conversion); a corrupt
   record is rejected without losing the previous valid copy.
5. **Protocol**: `EDITOR_PROTOCOL.md` updated in the same commit; capability tag
   added; existing ops unchanged.
6. **Golden/CPU**: any hash or budget delta is intentional, explained in the
   commit message, and the budget tables are updated in the same commit.
7. **Emulator**: `node web/emu/emu_test.mjs …` green when the change touches UI,
   input, audio or storage paths.
8. Docs: requirement/ADR references updated; open questions closed or filed.

## Per-phase test matrix

| Release | New tests |
| --- | --- |
| v1 (MVP) | matrix rows: insert/delete/move/duplicate/repeat; 64-row playback order; pool 8 slots; FUN10 round-trip + downgrade refusal; protocol ops |
| v2 | per-track motion capacity; lock vs event semantics at row start; pan/delay timing (frame-accurate); UI input simulation for hex entry (ui_test); golden playback of a scripted pattern |
| v3 | instrument apply before note-on; same-engine vs cross-engine switch (fade path); undo; protocol step byte |
| v5 | timing: alias edits, mute per occurrence, matrix render lint |
| (v4) | capacity: pool 8+, 64 rows |
| v5 | `DIV` 1/64 / 1/128 / triplets timing; `NSTEP` 128 banks, protocol step index, motion `place` under 128 |

## Regression policy

- `golden.txt` and `cpu_baseline.txt` move only with an intentional, documented
  behavioural change — never to make a red test green.
- `target_budget.txt` (static disassembly budget) is the hard CPU ceiling for
  engines; tracker features must not consume engine headroom (NFR1).
- UI alignment failures are treated as bugs, not noise (the test measures ink
  against boxes to 1 px).
