# OPR Tabletop Simulator Assigner — Project Context

## What this is

A pair of TTS (Tabletop Simulator) Lua mods for playing **One Page Rules** games —
Grimdark Future (mass-battle squads) and Star Quest (hero/skirmish) — without
alt-tabbing to a reference sheet. Right-click a miniature, pick its unit/hero
from a list, and the model gets a colored name, a readable tooltip, and
right-click wound/power tracking. Loosely inspired by Tombola's "OPR AF to TTS"
Steam Workshop mod, but built independently and diverged significantly.

There are **two separate, parallel systems** — one for squad-based units, one
for heroes — because heroes have a meaningfully different stat block (Power,
Str/Dex/Wil) that squads don't.

## Files

| File | Purpose |
|---|---|
| `opr_profile_assigner.lua` | Squad/mob Global script — paste directly into TTS, hand-edit `UNIT_PROFILES` |
| `opr_list_to_lua.sh` | Generates the squad Global script from an army-list text file |
| `opr_hero_assigner.lua` | Hero Global script — paste directly into TTS, hand-edit `HERO_PROFILES` |
| `opr_hero_list_to_lua.sh` | Generates the hero Global script from a hero-list text file |

Each `.sh` generator produces output identical in structure/behavior to its
paired standalone `.lua` file — they're kept in sync by hand whenever one
changes, and should be treated as needing matching edits.

## Core architecture

1. **Global script** holds `UNIT_PROFILES` / `HERO_PROFILES` (name → stats/rules
   data) and, on load/spawn, gives any model with no script of its own a
   right-click menu listing every profile name.
2. **Assignment is portable.** Picking a name doesn't just set a memo string —
   it builds a **complete, self-contained Lua script** for that specific unit
   (stats baked in as literals) and injects it via `object.setLuaScript(script)`
   followed by `object.reload()`. From that point on the model needs nothing
   from Global to function — right-click menu, name, tooltip, wound tracking
   all live in its own script. This means you can save one assigned model
   individually and drop it into a totally different game/session and it
   still works.
3. Global re-checks `object.getLuaScript() == ""` to decide whether a model
   still needs the "assign" menu — this is also how old/previously-assigned
   models from an earlier version of the script get correctly treated as
   "needs reassignment" (they never got a script under the old design).
4. **Wound (and Power, for heroes) tracking applies to your whole current
   selection**, not just the right-clicked model. Each model manages its own
   state independently; to apply Take Wound/Heal Wound/Use Power/Restore Power
   across a multi-selected group, the clicked model calls
   `otherObject.call('adjustWound', {delta = -1})` — TTS's built-in RPC
   mechanism for invoking a named global function on another object's script.
5. **"Measure Range"** (every assigned model, squad and hero alike): select
   one or more target models, then right-click the measuring model and
   choose it. It computes the horizontal (x/z) distance from that model to
   each selected target, assuming **1 TTS world unit = 1 inch** (retune
   `INCHES_PER_UNIT` in the per-model template if your table's scale
   differs), then checks that distance against the range of each of the
   model's own ranged weapons (parsed straight out of its `Equipment:` list
   — a weapon entry needs a `24"`-style quoted range to be checked; entries
   with no quoted range, e.g. `CCW (A2)`, are reported as melee and not
   range-checked, since OPR resolves those in base contact). Results are
   printed to the acting player via `broadcastToColor`, one line per target
   plus one line per weapon.

## Display design

**No custom UI panels, no 3D-attached objects — everything is TTS's own
Name field and Description (tooltip) field, styled with TTS's native BBCode.**
This was a late and important simplification: an earlier version used
`object.UI` (floating health bar, rotation, `<`/`>` buttons) and it caused
repeated, hard-to-diagnose problems (TTS doesn't billboard Object UI to face
the camera; layout-group width bugs; wrong-axis rotation composition). BBCode
in Name/Description does everything needed with none of that fragility.

Confirmed-real BBCode tags (from TTS's own KB,
`kb.tabletopsimulator.com/player-guides/bb-codes/`): `[HEXCODE]...[-]` for
color (hex only, no named colors), `[b][/b]`, `[i][/i]`, `[u][/u]`, `[s][/s]`,
`[sub][/sub]`, `[sup][/sup]`. Works directly in Name, Description, Notebook,
Notecards, and on-screen Notes.

**Name field** (visible without hovering):
```
[b]Wrysprocket[/b]
[73CDE0]Instigator[-] Lv1
[ef4444][b]Q5[/b]+[-] / [0ea5e9][b]D5[/b]+[-]
[b]Tough: [2ecc40]6[/b][-] / [b][2ecc40]9[/b][-]
[b]Power: [a855f7]3[/b][-] / [b][a855f7]5[/b][-]
```
(Class/Level line and Power line only appear if populated — squads have no
Class/Level/Power at all.)

**Description/tooltip** (on hover):
```
Quality 5+ | Defense 5+ | Tough 9
Str 6+ | Dex 5+ | Wil 5+          <- heroes only
Instigated Restoration, Mischievous, ...    (blue)
Equipment:
     Weapon One (12", A2)
     Weapon Two (A2)

[b]Ability Name:[/b]
[sub]Full pasted-in ability description.[/sub]

[b]Next Ability:[/b]
[sub]...[/sub]
```
Stats/equipment text is left uncolored (default white) rather than wrapped in
`[ffffff]` — deliberate choice, not an oversight. The blank line and bold name
between ability entries in the details block were added specifically for
readability after the first version rendered too cramped.

Colors used, defined as named constants near the top of each per-model
template (easy to retune): red name `e74c3c`, Quality `ef4444`, Defense
`0ea5e9`, Tough `2ecc40`, Power `a855f7`, abilities `3498db`, Class/Level
`73CDE0`.

## Data format (army-list text files)

**Squad** (`opr_list_to_lua.sh` input): blank-line-separated blocks, two lines
each:
```
[Nx ]Unit Name [size] Q#+ D#+ | ###pts | rule, rule, Tough(N), ...
weapon, weapon, ...
```

**Hero** (`opr_hero_list_to_lua.sh` input): same idea plus Power/Str/Dex/Wil,
plus optional extra lines after equipment for auto-populated ability text:
```
Name [size] Q#+ D#+ Pow# Str#+ Dex#+ Wil#+ | ###pts | rule, rule, Tough(N), ...
weapon, weapon, ...
Ability Name: full description text, one ability per line, as many as needed
Another Ability: description
```
Only the **first** non-blank line after the stat line is treated as
equipment; every line after that (until the blank separator) is captured as
an ability description and folded into `details`. `Tough(N)` inside the rules
list gets pulled out into its own "Tough N" stat automatically. Equipment/
ability-description parsing is comma/colon-aware but paren-depth-safe — e.g.
`Energy Sword (A6, AP(1), Rending)` stays one item, doesn't shatter on its
internal commas.

Manually editing the standalone `.lua` files works the same way — add entries
to `UNIT_PROFILES`/`HERO_PROFILES` directly, using `details = [[...]]`
(double square brackets, not quotes) for the optional pasted ability text so
multi-line/quoted content doesn't need escaping.

## Important technical gotchas learned the hard way

- **TTS's Lua engine is MoonSharp, not standard Lua**, and it has a much
  stricter limit on pattern-matching complexity than the Lua 5.3 used for
  local testing. A pattern shaped like `"^(.-)SomeLiteral(.+)$"` — lazy match
  anchored to a literal appearing later in the string — threw `"pattern too
  complex"` in-game on a ~140-character string, despite passing every local
  test. **Fix and ongoing rule: never use `.-` combined with anchors/later
  literals.** Use `string.find(s, "literal", 1, true)` (plain-text, no
  pattern) + `string.sub()` instead. This is why `trim()`, equipment/ability
  splitting, and Quality/Defense extraction all use hand-rolled `find`/`sub`
  logic instead of the more "idiomatic" Lua pattern idioms.
- **`[color]` BBCode is hex-only** — `[yellow]`/`[green]` as literal words do
  not work; must be `[eab308]`/`[2ecc40]` etc.
- Object UI panels (when that approach was still in use) don't billboard to
  face the camera, layout groups default `childForceExpandWidth` to true
  unless told otherwise, and rotation composition for "tip up + also spin"
  needed nested outer/inner panels with specific axis assignments taken from
  reading Tombola's actual shipped `mod.lua` — all now moot since the UI
  approach was dropped, but worth knowing if 3D-attached UI ever comes back
  into scope.
- `object.setLuaScript()` needs `object.reload()` afterward to actually take
  effect — setting the script alone doesn't run it.
- When testing/iterating, actually running the generated Lua through a real
  Lua interpreter (`lua5.3`/`luac5.3` are available in the sandbox) catches
  real bugs that eyeballing the code doesn't — several issues (missing
  newlines in string concatenation, wrong string.format arg counts, the
  MoonSharp pattern limit) were only caught this way.

## Known limitations / possible future work

- No re-assignment once a model has its own script — the per-model script is
  deliberately self-contained with just its own unit's data, so it doesn't
  carry the full profile table needed to switch to a different unit. Would
  need to bake the whole table into every model to support this.
- The generator scripts can't capture `class`/`level` from the text-list
  format (no slot for it in the stat line) — they emit empty placeholders
  that need to be filled in by hand afterward.
- Equipment must be exactly one line in the source text; everything after
  it is treated as ability-description text, not more equipment.
- `[sub]` styling on ability descriptions is the current choice for
  hierarchy/size; other options discussed but not implemented: a horizontal
  rule between entries instead of blank-line spacing, coloring ability names
  to match the ability-tag-list blue above them, `[i]` instead of/alongside
  `[sub]`.

## Where things stand

Both squad and hero systems are working end-to-end and confirmed live in TTS:
assignment, portability (save individual model → load in a different game),
colored Name field, tooltip with stats/abilities/equipment/pasted details,
and right-click wound/power tracking across multi-selection. This doc reflects
the current state of all four files as of the end of this session.
