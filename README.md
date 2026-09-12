# Plater 8.1 on Legion 7.3.5

<p align="center">
  <img alt="Client" src="https://img.shields.io/badge/client-7.3.5%20(26972)-1b6ec2">
  <img alt="Plater" src="https://img.shields.io/badge/Plater-v8.1.0.222-8a2be2">
  <img alt="Status" src="https://img.shields.io/badge/status-working-2ea44f">
  <img alt="Helper license" src="https://img.shields.io/badge/helper%20code-MIT-lightgrey">
</p>

A patch kit that makes **Plater Nameplates v8.1.0.222** run on the **Legion 7.3.5** client, so you get
NPC colours, scripts, mods and animations on a client that shipped before any of them existed.

---

## What you get

| | |
|---|---|
| **Npc Colors tab** | Per-NPC health bar colours, recorded automatically as you meet mobs in dungeons and raids |
| **Scripting** | Per-spell scripts, triggered by spell name or ID |
| **Modding** | 18 hooks: nameplate lifecycle, cast start/update/stop, combat, health, target, zone |
| **Animations** | The animation editor and per-nameplate effects |
| **Cast bars** | Correct spell, icon and timing, with interrupts clearing the bar immediately |

## What's in here

```
!PlaterCompat/                 helper addon: supplies 8.x functions the 7.3.5 client lacks
mods/LegionDungeonHelper.txt   a Plater mod: important-cast alerts and NPC type colouring
PATCHES.md                     the edits to make to Plater's own files (also listed below)
```

---

## Quick start

1. **Get Plater 8.1.0.222**
   [`v8.1.0.222.zip`](https://github.com/Tercioo/Plater-Nameplates/archive/refs/tags/v8.1.0.222.zip)
   Unzip it into `Interface\AddOns\` and rename the folder to `Plater`.

2. **Set the interface version**
   In `Plater\Plater.toc`, change `## Interface: 80100` to `## Interface: 70300`.

3. **Install the helper addon**
   Copy `!PlaterCompat` into `Interface\AddOns\`, then add this line near the top of `Plater.toc`:
   ```
   ## Dependencies: !PlaterCompat
   ```
   Restart the game completely. New addon folders are only detected at startup.

4. **Apply the patches** in the table below.

5. *(Optional)* **Install the mod**: `/plater` → Modding → New Mod, then paste each section of
   `mods/LegionDungeonHelper.txt` into its matching hook.

---

## The patches

Nine edits across three files. Line numbers are from a clean 8.1.0.222 checkout and drift as you edit,
so search for the text instead.

### `Plater.lua`

<table>
<tr><th>#</th><th>What</th><th>Change</th></tr>
<tr><td>1</td><td><b>Combat log</b><br><sub>~7091</sub></td><td>

`CombatLogGetCurrentEventInfo()` doesn't exist before 8.0; the values arrive as event arguments.

```lua
-- PlaterCLEUParser.Parser = function (self)
PlaterCLEUParser.Parser = function (self, event, ...)
-- ... = CombatLogGetCurrentEventInfo()
   ... = ...
```
</td></tr>
<tr><td>2</td><td><b>Auras</b><br><sub>line 50</sub></td><td>

Legion returns an extra `rank` value second, shifting every aura result by one.

```lua
local UnitAura, UnitBuff, UnitDebuff =
    PlaterCompat.UnitAura, PlaterCompat.UnitBuff, PlaterCompat.UnitDebuff
```
</td></tr>
<tr><td>3</td><td><b>Border</b><br><sub>~2183</sub></td><td>

`SetBorderSizes` / `UpdateSizes` were added to the border template in 8.0. Add after
`plateFrame.unitFrame.healthBar.border = healthBarBorder`:

```lua
Mixin (healthBarBorder, PlaterCompat.BorderMixin)
```
</td></tr>
<tr><td>4</td><td><b>Resource frame</b><br><sub>~3002</sub></td><td>

Moving Blizzard's class resource frames taints the nameplate driver on Legion, which blocks a
protected call. Add as the first line of `Plater.UpdateResourceFrame()`:

```lua
do return end
```
</td></tr>
<tr><td>5</td><td><b>Class power bar</b><br><sub>~2937</sub></td><td>

Same cause. Comment out:

```lua
-- NamePlateDriverFrame.classNamePlatePowerBar:Hide()
```
</td></tr>
<tr><td>6</td><td><b>NPC cache, PvP check</b><br><sub>~2435</sub></td><td>

`GetZonePVPInfo` returns `contested` inside Legion dungeons, so Plater never recorded any NPC.

```lua
-- and not Plater.ZonePvpType
   and Plater.ZoneInstanceType ~= "pvp" and Plater.ZoneInstanceType ~= "arena"
```
</td></tr>
<tr><td>7</td><td><b>Open-world flag</b><br><sub>~1783</sub></td><td>

After a `/reload` inside an instance the flag kept its startup value. In the
`PLAYER_ENTERING_WORLD` handler, under `Plater.ZoneInstanceType = instanceType`:

```lua
IS_IN_OPEN_WORLD = Plater.ZoneInstanceType == "none"
```
</td></tr>
</table>

### `libs\DF\panel.lua`

<table>
<tr><th>#</th><th>What</th><th>Change</th></tr>
<tr><td>8</td><td><b>Auras and cast info</b><br><sub>top of file</sub></td><td>

`UnitCastingInfo` and `UnitChannelInfo` also return an extra value second on Legion, which is why
cast bars never appeared.

```lua
local UnitAura, UnitBuff, UnitDebuff, UnitCastingInfo, UnitChannelInfo =
    PlaterCompat.UnitAura, PlaterCompat.UnitBuff, PlaterCompat.UnitDebuff,
    PlaterCompat.UnitCastingInfo, PlaterCompat.UnitChannelInfo
```
</td></tr>
<tr><td>9</td><td><b>Cast end events</b><br><sub>~7945–8035</sub></td><td>

The cast events carry different arguments, so the cast ID never matched and interrupted casts kept
running. The bar only receives events for its own unit, so the check can go.

```lua
-- if (self.castID == castID) then
   if (true) then

-- if (self.channeling and castID == self.castID) then
   if (self.channeling) then

-- if (self.casting and castID == self.castID and not self.fadeOut) then   (twice)
   if (self.casting and not self.fadeOut) then
```
</td></tr>
</table>

### `Plater_AnimationEditor.lua`

Same combat log fix as #1, around line 51: add `...` to the handler's parameters and read the values
from it instead of calling `CombatLogGetCurrentEventInfo()`.

---

## `!PlaterCompat`

A small standalone addon that supplies what the 7.3.5 client is missing. It never modifies Blizzard
globals or mixins, because doing so taints the secure nameplate code and gets protected calls blocked.

| Provided | Notes |
|---|---|
| `PixelUtil` | Pass-through, no pixel snapping. Borders can look ~1px soft at some UI scales |
| `PlaterCompat.UnitAura` / `UnitBuff` / `UnitDebuff` | Drop Legion's extra `rank` return |
| `PlaterCompat.UnitCastingInfo` / `UnitChannelInfo` | Drop Legion's extra `subText` return |
| `PlaterCompat.BorderMixin` | `SetBorderSizes` / `UpdateSizes` for Legion's 3-layer nameplate border |

## `LegionDungeonHelper`

A Plater mod, four hooks, no extra dependencies.

- **Important casts** across all 14 Legion dungeons: pink cast bar, taller bar, plate flash, and a glow
  around the cast bar for the duration. The 324 spell IDs are the abilities
  [LittleWigs](https://github.com/BigWigsMods/LittleWigs) raises cast warnings for.
- **Non-interruptible casts**: red, taller cast bar.
- **NPC colouring**: bosses purple, casters blue. Casters are learned as mobs are seen casting, plus
  any mob with a mana bar. A colour set by hand in the Npc Colors tab always wins.

---

## Known limitations

- **Personal resource bar options are disabled** (patches 4 and 5). Blizzard's own display still works,
  unstyled, and Plater's scale, alpha and position settings for it do nothing.
- **Borders are not pixel-snapped.** Cosmetic, and only at some UI scales.
- **Wago imports mostly won't work.** Strings decode, but modern scripts call APIs that don't exist on
  7.3.5 and error the moment they run.
- **Lightly tested**: Scripting, Animations, and import/export.
- **Health bar colouring competes with threat colouring.** Pick one.

## Credits

- **[Tercioo](https://github.com/Tercioo/Plater-Nameplates)** and the Details! team, for Plater itself.
- **[LittleWigs](https://github.com/BigWigsMods/LittleWigs)** (GPLv3), source of the dungeon spell IDs.
- **[ViragDevTool](https://github.com/varren/ViragDevTool)**, which found half of these bugs.

## Licence

The helper addon, the mod and these notes are MIT. Plater is not included here and remains under its
own licence; get it from the author's repository.
