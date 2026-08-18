# VIA Tips & Gotchas

Notes for editing the keyboard definitions and layouts in this repo without
re-introducing the bugs that took a while to track down. Read this before
touching any `*.layout.json` or definition file.

If you use an AI assistant (GitHub Copilot, etc.), the same guidance is mirrored
in [`.github/copilot-instructions.md`](.github/copilot-instructions.md) so it is
picked up automatically.

---

## Repo structure

```
via-layouts/
├─ VIA-TIPS.md                         ← this file
├─ .github/copilot-instructions.md     ← same tips, for AI assistants
├─ jelly60/
│  ├─ jelly60_draft_definition.json    ← VIA definition (sideload in Design tab)
│  └─ layouts/
│     ├─ jelly60_windows.layout.json   ← Windows keymap (Configure → Save+Load)
│     └─ jelly60_osx.layout.json       ← macOS keymap
├─ neoErgo/ …
└─ space65/ …
```

Two file *types*, used in two different places in VIA:

| File | What it is | Where it loads in VIA | Writes to board? |
|------|------------|-----------------------|------------------|
| `*_draft_definition.json` | Keyboard **definition** (physical layout, matrix, layout options) | **Design** tab → Load Draft Definition | **No** — only lets VIA recognize/render the board and expose layout options |
| `layouts/*.layout.json` | Keymap **export** (layers + macros) | **Configure** → **Save + Load** → Load | **Yes** — writes layers & macros to EEPROM |

---

## The Jelly60 board

- 60% with a **split / gapped bottom row**: 7u spacebar + two 1.5u keys on each side (winkeyless 7u).
- USB: **vendorId `0x5048`** (PHDesign), **productId `0x6A3C`**. `vendorProductId` in keymap exports = `1346923068` = `0x50486A3C`.
- Matrix: **5 rows × 15 cols = 75 keys per layer**, 13 layers.
- **No lighting / RGB** on this board.
- There is **no official VIA/QMK definition** for it, so it must be **sideloaded** as a draft (Design tab). This is expected — the "draft definition" requirement is not a bug.

---

## Loading the definition (kills the errors)

1. VIA → **Settings** → enable **Show Design tab**.
2. **Design** tab → enable **Use V2 definitions (deprecated)** — our definition is V2 format, it will not load without this.
3. **Load Draft Definition** → pick `jelly60/jelly60_draft_definition.json`.
4. **Configure** tab → connect the board (click the plug/USB icon and pick "JELLY 60 Keyboard" if it keeps "Searching for devices…").

---

## Gotcha #1 — `lighting` must be present, but `"none"` (the error spam)

Symptom: dozens of console errors on connect:

```
Command Name: BACKLIGHT_CONFIG_GET_VALUE
Error: Receiving incorrect response for command
Loading lighting/menu data failed
```

Cause: the definition advertised RGB with `"lighting": { "extends": "qmk_rgblight" }`.
VIA then queries backlight brightness/effect/speed/color; the board doesn't have
lighting and replies `0xFF`, which VIA flags as an incorrect response.

Rules:
- This board has **no lighting** → use exactly `"lighting": "none"`.
- **Do not** delete the `lighting` property — it is **required** by the VIA schema.
  Removing it causes: `error: Object: must have required property 'lighting'`.
- Only use `"extends": "qmk_rgblight"` (or `qmk_backlight`, etc.) if the board
  **actually has** that lighting.

---

## Gotcha #2 — Stuck keys come from bad macros, not (necessarily) hardware

Symptom: keys randomly "stick" (modifiers appear held down).

Cause found here: macro **M3** was written as a **press-and-hold** with no release:

```
{+KC_LCTL}{+KC_LGUI}{+KC_LEFT}{-KC_NO}     ← BROKEN: holds Ctrl+GUI+Left forever
```

`{+KC_X}` = press & hold, `{-KC_X}` = release. `{-KC_NO}` releases *nothing*, so
the three modifiers stay virtually held.

Rules:
- For a normal chord that taps and releases, use the **comma form**:
  `{KC_LCTL,KC_LGUI,KC_LEFT}`.
- Only use `{+..}` / `{-..}` if you deliberately want a hold, and **always pair
  every `{+X}` with a matching `{-X}`**.
- An **empty macro that is still bound to a key** = a dead key. If a key maps to
  `MACRO(n)`, make sure macro `n` is defined.

After fixing macros, verify on the board with **Key Tester** (toolbar). If a key
still sticks or lights without being pressed there, *that* one is genuinely
**hardware** (switch / hot-swap socket / solder).

---

## Gotcha #3 — Default layout option = choice index 0 (there is no "default" field)

VIA has **no field** in the definition to pre-select a layout option. VIA reads
the current option from the keyboard's EEPROM and falls back to **index 0** when
unset.

To make "Win Key Less 7U" the default bottom row we **reordered the choices** so
it sits at index 0 in the `labels` array, and renumbered the KLE option legends
to match. Current Space Row order:

```
["Space Row","Win Key Less 7U","ANSI","HHKB","Win Key With 7U","Minila","Double Space Left"]
                 ↑ index 0 = default
```

After changing the default: on the board, **re-select** the option once (it saves
the new index) or **reset EEPROM**, because the board may still hold the old index.

---

## Gotcha #4 — Stacked bottom-row options need `{"y":-1}` or you get a vertical gap

The six Space Row variants live on six separate KLE rows. VIA renders each option
at its **absolute** row position, so any option that isn't physically first shows
up **one or two rows lower**, leaving a gap between Shift and the bottom row.

Fix: put the default option's row **physically first**, then add `{"y":-1}` at the
start of every *other* option row so they all **overlap at the same vertical
position**. There should be **5** `{"y":-1}` decorators (one per non-first option).

Always **verify visually in the Design tab** after editing bottom-row options.

---

## Gotcha #5 — KLE legend format (never touch matrix coords)

Each key string in the `keymap` encodes two things:

```
"row,col\n\n\ngroup,option"
   │              │
   │              └─ bottom-right: layout GROUP,OPTION (which variant it belongs to)
   └─ top-left: switch MATRIX row,col  ← DO NOT CHANGE
```

When reordering layout options you only renumber the **group,option** part (after
the `\n\n\n`). Never change the `row,col` matrix coordinates — that remaps the
physical wiring.

---

## Gotcha #6 — Windows and macOS are separate files, on purpose

Keep `layouts/jelly60_windows.layout.json` and `layouts/jelly60_osx.layout.json`
as **independent** configs. Don't mirror one into the other. Real differences:

- Corner modifiers: Windows uses **`KC_LCTL`** where macOS uses **`KC_LGUI`** (Cmd).
- **`KC_PSCR` (Print Screen) does nothing on macOS.** Use a screenshot **macro**
  instead (`Cmd+Shift+4` = `{KC_LGUI,KC_LSFT,KC_4}`). On Windows `KC_PSCR` works
  (or use a `Win+Shift+S` snip macro).
- Some Fn-layer keys and macros differ per OS (see table below).

---

## Current key map reference

**Fn keys** (base layer): `MO(1)` at matrix (4,2) and `MO(2)` at matrix (4,10) —
the two 1.5u keys immediately beside the spacebar.

**Macros:**

| Macro | Windows | macOS | Purpose |
|-------|---------|-------|---------|
| M0 | `{KC_GRV}` | `{KC_GRV}` | Grave / backtick |
| M1 | `{KC_LGUI,KC_TAB}` | `{KC_LGUI,KC_TAB}` | App/window switcher |
| M2 | `{KC_LALT,KC_SPC}` | `{KC_LALT,KC_SPC}` | Alt/Opt + Space |
| M3 | `{KC_LCTL,KC_LGUI,KC_LEFT}` | `{KC_LCTL,KC_LGUI,KC_LEFT}` | Previous virtual desktop |
| M4 | `{KC_LCTL,KC_LGUI,KC_RGHT}` | `{KC_LCTL,KC_LGUI,KC_RGHT}` | Next virtual desktop |
| M5 | `{KC_LGUI,KC_LCTL,KC_LSFT,KC_P}` | `{KC_LGUI,KC_LSFT,KC_4}` | Win: app hotkey (⊞Ctrl⇧P) / macOS: region screenshot |

---

## Apply / edit workflow (checklist)

1. Edit the JSON in this repo (keep Windows/macOS separate).
2. **Validate JSON** before loading (a trailing comma will break the whole file):
   `python -c "import json;json.load(open('jelly60/layouts/jelly60_windows.layout.json'))"`
3. **Design tab** → load `jelly60_draft_definition.json` (V2 enabled) → check the
   render has **no gap** and the console has **no errors**.
4. **Configure → Save + Load → Load** the matching `layouts/*.layout.json` to
   write layers + macros to the board.
5. If you changed the default layout option, re-select it once on the board.
6. **Key Tester** → press every key; anything stuck there is hardware.
7. Commit & push.
