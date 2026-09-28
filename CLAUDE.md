# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Two parallel Tabletop Simulator (TTS) Lua mods for playing **One Page Rules**
games without alt-tabbing to a reference sheet: one for squad-based units
(Grimdark Future), one for heroes (Star Quest). Heroes get a different stat
block (Power, Str/Dex/Wil) that squads don't have, hence two separate systems
rather than one generalized one.

Full design rationale, BBCode reference, data-format spec, and a running list
of hard-won gotchas live in [opr-tts-project-writeup.md](opr-tts-project-writeup.md) — read it before making
non-trivial changes, especially before touching the Lua template strings or
the `.sh` parsers.

## Files

| File | Purpose |
|---|---|
| `opr_army_list_to_lua/opr_profile_assigner.lua` | Squad/mob Global script — paste directly into TTS, hand-edit `UNIT_PROFILES` |
| `opr_army_list_to_lua/opr_list_to_lua.sh` | Generates the squad Global script from an Army Forge text-export file |
| `opr_hero_assigner.lua` | Hero Global script — paste directly into TTS, hand-edit `HERO_PROFILES` |
| `opr_hero_list_to_lua.sh` | Generates the hero Global script from a hero-list text file |
| `opr_hero_customizer.lua` | Hero Global script with **no generator**: assigns a generic "HERO NAME" template that is edited in-game via the model's "Edit Hero..." menu |

**The two `.sh` generators and their paired `.lua` files are kept in sync by
hand.** They must produce output identical in structure/behavior to the
standalone `.lua` file. A change to one (e.g. a new BBCode field, a new
per-model right-click action) needs the matching edit made in the other three
files.

`opr_hero_customizer.lua` is standalone (no `.sh` pair). Its per-model
script keeps all hero data in the object's saved state (`onSave` /
`onLoad(saved_data)` as JSON) rather than baked-in literals, and edits
go through TTS's `Player.showInputDialog` / `showMemoDialog` /
`showOptionsDialog`. The only thing injected at assignment is
`HERO_DEFAULTS`, serialized into the template's `@@DEFAULTS@@`. It
shares the display code, colors, `adjustWound`/`adjustPower` names and
range measuring with `opr_hero_assigner.lua`'s per-model script, so
changes there may need mirroring here by hand.

**Bump `SCRIPT_VERSION` in `opr_hero_customizer.lua` whenever its
per-model template changes.** The template carries a
`-- opr_hero_customizer script version: N` line; on load/spawn, Global
replaces any customizer script with a lower version, reading the hero's
`script_state` and baking it in as the new script's `DEFAULTS` so no
data is lost. Its tooltip omits Q/D/Tough (already in the Name field),
groups abilities by a trailing `(Str)`/`(Dex)`/`(Wil)` tag, and leaves
ability descriptions to the "Ability Info..." menu (printed to chat).

## Commands

Generate a squad Global script from an Army Forge text export:
```bash
./opr_army_list_to_lua/opr_list_to_lua.sh opr_army_list_to_lua/army_export.txt [output.lua]
```

Generate a hero Global script from a hero-list text file:
```bash
./opr_hero_list_to_lua.sh hero_sheet.txt [output.lua]
```
(defaults to `opr_profile_assigner.lua` / `opr_hero_assigner.lua` respectively
if no output path is given)

There is no build/lint/test tooling in this repo. **Validate any Lua change
by actually running it through a real Lua interpreter** (`lua5.3`/`luac5.3`),
not just by eyeballing it — several real bugs (missing newlines in string
concatenation, wrong `string.format` arg counts, MoonSharp pattern-complexity
limits) were only caught this way. Note MoonSharp (TTS's Lua engine) is
stricter than standard Lua 5.3 in some respects (see gotcha below), so a
script passing local Lua 5.3 doesn't guarantee it will run in TTS.

## Core architecture

1. **Global script** holds `UNIT_PROFILES` / `HERO_PROFILES` (name → stats/
   rules data) and, on load/spawn, gives any model with no script of its own
   a right-click menu listing every profile name.
2. **Assignment is portable, not just a label.** Picking a name builds a
   **complete, self-contained Lua script** for that specific unit (stats
   baked in as literals) and injects it via `object.setLuaScript(script)`
   followed by `object.reload()` (the reload is required — setting the
   script alone does not run it). From that point the model needs nothing
   from Global — its own right-click menu, name, tooltip, and wound tracking
   all live in its own script — so a saved individual model still works if
   dropped into a different game/session.
3. Global re-checks `object.getLuaScript() == ""` to decide whether a model
   still needs the "assign" menu — this also correctly flags
   previously-assigned models from an older version of the script (which
   never got a per-model script under the old design) as needing
   reassignment.
4. **Wound (and Power, for heroes) tracking applies to the whole current
   selection**, not just the right-clicked model: the clicked model calls
   `otherObject.call('adjustWound', {delta = -1})` — TTS's built-in RPC for
   invoking a named global function on another object's script — on every
   other selected model.
5. There is currently **no re-assignment** once a model has its own script:
   the per-model script deliberately only contains its own unit's data, not
   the full profile table, so it can't switch to a different unit.

## Display design

No custom UI panels, no 3D-attached objects — everything is TTS's native
Name field and Description (tooltip) field, styled with TTS's own BBCode.
An earlier version used `object.UI` and hit repeated fragility (no
billboarding to camera, layout-group width bugs, rotation-axis bugs); BBCode
in Name/Description replaced it entirely.

Confirmed-real BBCode tags: `[HEXCODE]...[-]` for color (**hex only** — named
colors like `[yellow]` do not work), `[b][/b]`, `[i][/i]`, `[u][/u]`,
`[s][/s]`, `[sub][/sub]`, `[sup][/sup]`.

Per-model color constants (defined near the top of each template, easy to
retune): red name `e74c3c`, Quality `ef4444`, Defense `0ea5e9`, Tough
`2ecc40`, Power `a855f7`, abilities `3498db`, Class/Level `73CDE0`.

## Data format (input text files, e.g. `army_export.txt`, `hero_sheet.txt`)

Blank-line-separated blocks. **Squad:**
```
[Nx ]Unit Name [size] Q#+ D#+ | ###pts | rule, rule, Tough(N), ...
weapon, weapon, ...
```
**Hero** (adds Power/Str/Dex/Wil, plus optional ability text after equipment):
```
Name [size] Q#+ D#+ Pow# Str#+ Dex#+ Wil#+ | ###pts | rule, rule, Tough(N), ...
weapon, weapon, ...
Ability Name: full description text, one ability per line, as many as needed
Another Ability: description
```
Only the **first** non-blank line after the stat line is equipment; every
line after that (until the blank separator) becomes ability-description text
folded into `details`. `Tough(N)` in the rules list is pulled out into its
own stat automatically. Equipment/ability parsing is comma/colon-aware but
paren-depth-safe (e.g. `Energy Sword (A6, AP(1), Rending)` stays one item).

When hand-editing the standalone `.lua` files instead of regenerating, use
`details = [[...]]` (double square brackets) for pasted multi-line ability
text so it doesn't need quote-escaping.

The generators can't capture `class`/`level` from the text-list format (no
slot for it) — they emit empty placeholders that need filling in by hand.
Equipment must be exactly one line in the source text.

## Gotcha: confirm which Global script is actually running

The TTS script editor can show newly pasted text while an older Global
script is still what's running (seen once with the hero customizer:
newly assigned models kept getting the previous per-model script; cause
not pinned down — a stale copy source is the leading suspect, and a
loaded save was NOT required for it to work later). The "Error no save
found" message after Save & Play is normal and harmless.
`opr_hero_customizer.lua` prints "Hero Customizer Global loaded
(script vN)" on load, and each per-model upgrade to chat, so what's
running is visible.

## Critical gotcha: MoonSharp pattern limits

TTS's Lua engine is **MoonSharp, not standard Lua**, with a much stricter
limit on pattern-matching complexity than Lua 5.3. A pattern shaped like
`"^(.-)SomeLiteral(.+)$"` (lazy match anchored to a literal appearing later)
threw `"pattern too complex"` in-game on a ~140-character string despite
passing local tests.

**Rule: never use `.-` combined with anchors/later literals.** Use
`string.find(s, "literal", 1, true)` (plain-text, no pattern) + `string.sub()`
instead. This is why `trim()`, equipment/ability splitting, and
Quality/Defense extraction all use hand-rolled `find`/`sub` logic rather than
more "idiomatic" Lua pattern matching. Apply the same rule to any new parsing
logic added to the `.lua` files or `.sh` generators.
