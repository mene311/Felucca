# UI and input

Status: draft. Goal: **keyboard-first editing, knobs scarce** (they are relative
detent encoders — good for value scrub, wrong for entry). No external keyboard is
ever required.

## Screens

| Screen | Status | Contents |
| --- | --- | --- |
| TRACKER | new (phase 2) | vertical row list: note, instrument, velocity, command columns; cursor |
| POOL | phase 1 (replaces PROJECT) | 8–16 slots, letters A–H; save / load / clone / erase / name |
| SONG (order list) | extend (phase 1) | 64 rows, slot letter + repeat; insert / delete / duplicate / move |
| MATRIX | new (phase 4) | rows × 4 tracks of block refs; mute per block |
| MIXER | exists | per-track LEVEL / PAN / REV / MUTE + meters |
| STEP / PATTERN / PATTERNS / AUTO LIST / CHANCE | exist | unchanged pages; PATTERN keeps LEN/DIV/SWG/GATE |

## TRACKER screen layout (240 × 240)

```
┌ header ~30 px ─────────────────────────────────────────────┐
│  pattern A   track 2   ▶ 124   [cursor info]               │
├ grid ~170 px (10 rows × 16 px) ────────────────────────────┤
│ 01 ▪ C-4 02 64 14 0F  —          ← row 1                   │
│ 02   --- -- -- -- —  —          ← empty row                │
│ 03 ▸ D#4 03 7F -- 8B 40         ← cursor row               │
│ …                                                          │
├ footer ~35 px ─────────────────────────────────────────────┤
│ knob cards: the four columns' values + hints (keycaps)     │
└────────────────────────────────────────────────────────────┘
```

Column budget (approx.): gutter 14 + note 34 + inst 18 + vel 18 + cmd1 40 +
cmd2 40 = 164 px — leaves margins. Ten visible rows with a 16 px pitch; the list
scrolls (as the SONG list already does).

## Input model

**White keys** — content:

- In the *note* column: play/enter notes (existing step-recording behaviour:
  REC arms, a key writes the row under the cursor and advances).
- In any *hex* column: the 16 white keys are the 16 hex digits `0`–`F`, one tap
  each; after the second digit the cursor advances (tracker convention).
- Octal-looking values do not exist; every value column is 2 hex digits.

**Black keys** — cursor and edits (by key name, as `ui_name.c` already does;
left/right per octave, additional functions use the second octave's copy):

| Key | Function |
| --- | --- |
| F# | cursor left (column) |
| A# | cursor right (column) |
| G# | cursor down (row), hold repeats |
| D# | cursor up (row), hold repeats |
| C# | clear the cell (undoable) |
| F#5 (11th black key) | insert / delete row modifiers (with EDIT) |

**Buttons** — global actions (existing idioms):

| Button | Tap | Hold |
| --- | --- | --- |
| PLAY | start/stop | — |
| REC | arm / record | REC layer (clear sequence, click, count-in) |
| SAVE | pool page | one-step undo (existing) |
| EDIT | clipboard/tools layer on the tracker screen | — |
| OCT+ / OCT− | confirm / back (existing) | page up / page down? — reserved for phase-2 review |
| SEQ | sequencer pages | SONG (order list) as today |
| HOME | home | menu |

**Knobs** — minimal: the four knobs scrub the value under the cursor (up/down),
as a fallback and for auditioning. No entry depends on them.

## Interaction rules (what makes it a tracker, not a spreadsheet)

1. **Type and move on**: entering a value advances the cursor down one row
   (Renoise behaviour). No confirm step.
2. **A cell is a value, a command is a pair**: `cmd` columns hold `effect` +
   `value` nibbles; empty = no command. Multiple commands per row are separate
   columns (phase 2: two).
3. **Commands are locks or events** — a per-row toggle (kind bit) shown as a
   marker (e.g. `·` lock vs `:` event). The default is *lock* (Renoise-like
   per-line effect); UPPER/lower marker borrowed from tracker conventions.
4. **Copy/paste/insert/delete rows** live in the EDIT layer (hold EDIT): white
   keys pick the operation, the screen hints via keycaps.
5. **Undo** is the platform's one-step `SAVE`-held for the whole edit session.
6. **Preview**: with the transport stopped, the keys audition the selected
   track (existing behaviour); while playing, editing follows the playhead
   (existing "pattern follow" behaviour).
7. **Auto-advance on note entry** uses the existing step-entry model; ties and
   slides keep their current keys (SF_TIE written by holding, etc. — see the
   current STEP page semantics) so muscle memory transfers.

## Keycap hints and LEDs

- The screen labels the black keys' current functions (the existing keycap
  renderer, 27 pills); the tracker mode changes them, so the hint must be
  mode-aware.
- LEDs: the playhead row and the cursor row are distinct; the octave buttons'
  LEDs keep their existing meaning.

## Open questions

- Q-U1. Row pitch: 16 px (10 rows) vs 15 px (11 rows) vs a zoom toggle.
- Q-U2. Where "kind" (lock/event) lives in the row display.
- Q-U3. Whether OCT+/− become row paging on the tracker screen or stay
  confirm/cancel (platform idiom).
- Q-U4. Matrix screen: how many columns/rows fit readably, and which gesture
  selects blocks (EDIT layer vs a dedicated modifier).
