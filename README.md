# via-layouts
various layouts to organize between OS's

## Editing these layouts

Before changing any definition or `*.layout.json`, read **[VIA-TIPS.md](VIA-TIPS.md)** —
it covers the VIA gotchas that caused real bugs here (lighting errors, stuck-key
macros, layout-option defaults, the bottom-row gap, and the Windows/macOS split).
The same notes are mirrored in [`.github/copilot-instructions.md`](.github/copilot-instructions.md)
for AI assistants.

Each keyboard folder holds per-OS keymap **exports** (loaded via Configure → Save +
Load). Boards without an official VIA listing (e.g. `jelly60/`) also include a
sideloadable **definition** (`*_draft_definition.json`, loaded in the Design tab),
with the keymaps kept under a `layouts/` subfolder.
