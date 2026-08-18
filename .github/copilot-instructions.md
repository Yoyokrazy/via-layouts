# Copilot instructions for `via-layouts`

This repo stores [VIA](https://usevia.app) keyboard **definitions** and **keymap
layouts**, organized per OS. Full details and rationale are in
[`VIA-TIPS.md`](../VIA-TIPS.md) — read it before editing.

## What the files are

- `*/*_draft_definition.json` — VIA **definition** (matrix, physical layout,
  layout options). Sideloaded in VIA's **Design** tab. Does **not** write to the board.
- `*/layouts/*.layout.json` — VIA keymap **export** (layers + macros). Loaded via
  **Configure → Save + Load**. **Writes** to the board's EEPROM.
- Windows and macOS keymaps are **separate files** and must stay independent.

## Hard rules (these caused real bugs — don't regress them)

1. **`lighting` is required.** Use `"lighting": "none"` when the board has no RGB.
   Never delete the property (schema error: *must have required property 'lighting'*).
   Only use `"extends": "qmk_rgblight"`/`qmk_backlight` if the board truly has lighting,
   otherwise VIA spams `BACKLIGHT_CONFIG_GET_VALUE` / "Loading lighting data failed" errors.
2. **No sticky macros.** Use the chord form `{KC_LCTL,KC_LGUI,KC_LEFT}`. Never emit
   `{+KC_X}` (hold) without a matching `{-KC_X}` (release) — an unreleased hold is what
   makes keys "stick." Never leave a key bound to an empty `MACRO(n)`.
3. **Layout-option default = index 0.** VIA has no "default option" field; it reads the
   board's EEPROM and falls back to index 0. To change the default, reorder the choices in
   `layouts.labels` so the desired one is first, and renumber the KLE `group,option` legends
   to match.
4. **Overlap stacked bottom-row options.** Put the default option's row physically first,
   then prefix every other option row with `{"y":-1}` so all variants render at the same
   position (no vertical gap). Verify in the Design tab.
5. **Never change matrix coords.** In a keymap string `"row,col\n\n\ngroup,option"`, only
   edit the `group,option` (after `\n\n\n`). The top-left `row,col` is physical wiring.
6. **OS awareness.** macOS uses `KC_LGUI` (Cmd) where Windows uses `KC_LCTL`; `KC_PSCR`
   does nothing on macOS (use a `Cmd+Shift+4` macro instead).

## Before you commit

- Validate every edited JSON file parses (a stray trailing comma breaks the whole file).
- Load the definition in the Design tab and confirm: no console errors, no layout gap.
- Load the matching `layouts/*.layout.json` onto the board, then check **Key Tester**.
- Keep Windows and macOS edits in separate commits/diffs where practical.
