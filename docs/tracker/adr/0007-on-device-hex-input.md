# ADR-0007: On-device hex input from the 16 white keys

- Status: accepted (2026-10-09)
- Deciders: mene311

## Context

Tracker entry is keyboard-first: digits are typed, the cursor advances. The
product may not depend on an external keyboard. The FM-1 has 27 binary keys
(16 white + 11 black), 12 buttons, 2 octave buttons and 8 relative detent
encoders. The firmware already runs a key-only text mode: `ui_name.c` maps 16
white keys to character groups with multi-tap and black keys to
cursor/space/delete/mode, with hold-to-repeat.

## Decision

- In value columns, the **16 white keys are the 16 hex digits `0`–`F`**, one tap
  per digit; the second digit advances the cursor down (tracker behaviour).
- **Black keys** are cursor and edit functions (left/right, up/down,
  clear), named per key as in `ui_name.c`; clipboard/tools live in a held
  `EDIT` layer so the key count suffices.
- **Knobs are a fallback** (value scrub/audition), never a dependency.
- Hex is sufficient: every value is 2 nibbles; 4 hex digits cover a
  command+value pair; the alphabet is 16 symbols.

## Consequences

- The input model is consistent with existing firmware idioms (keycap hints,
  quick layers, hold-to-repeat), minimising new input code.
- Multi-tap is not needed for hex (single tap per digit), which is faster than
  the existing name entry and matches tracker flow.
- Note entry and hex entry share the white keys; entering a *note* vs a *digit*
  is decided by the cursor column (and can be locked by a mode if needed).
- The screen must show which mode the keys are in (keycap hints, LEDs).

## Alternatives

- Two knobs as hi/lo nibble selectors: slower than typing and requires looking
  away from the grid; kept only as a fallback.
- Black-key-only hex (11 keys): rejected — cannot address 16 symbols without
  multi-tap.
