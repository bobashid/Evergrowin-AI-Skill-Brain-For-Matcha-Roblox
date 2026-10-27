# CQB Hell

Place id: `98689921952741` (from `matcha_status`, 2026-10-27).
Script: `roblox-matcha/CQB Hell`, current version `v1.3`.

## Features
Weapon / NPC / Heal / Ammo ESP, one tab, four toggles, range + transparency sliders.

## Confirmed structure (probes 2026-10-27)

### How to tell a player from an NPC (the important one)

**NPC models are named after the gun they carry**, and names repeat:
`RPK`, `PM`, `Slam`, `Deagle`, `AKM`, `Tec9`, `Mosin`, `MP5`, `PKM`, `Draco`, `EnemyShotgun`.
Two `PM`, three `Slam`, four `Tec9` can be alive at the same time. A name-based exclude
list therefore hides real NPCs the moment a player's Name or DisplayName collides with a
weapon name (user-reported bug, fixed in v1.3).

**Use a marker, not a name.** Probe of 8 live humanoid models:
- players (`O_CsF`, `FakeAimbotON`) -> `tagY fpvY setx` = has `NameTag`, has `FPV_DynamicCam`, no `Settings`
- NPCs (`RPK`, `Slam`, `Deagle`, `PM`) -> `tagx fpx setY` = no `NameTag`, no `FPV_DynamicCam`, has `Settings`

So: `isPlayer = hasChild('NameTag') or hasChild('FPV_DynamicCam')`. Name is never consulted.

### Placement
- Most NPCs are `workspace` children, **but some are parented to a Room model** (probe saw a
  `PM` whose parent was `Room1_19`, not `Workspace`). Collect with `workspace:GetDescendants()`, not
  `GetChildren()`, or those NPCs are invisible to the ESP.
- Corpses are `<Name>_Corpse` with `Humanoid.Health == 0`; 30 can be on the floor at once.
- Health reads reliably: `100/100`, `150/150`, `200/200`, `250/250`, `2117/3000` (PKM).

### Items
- **Ammo pickups** are `Part`s named `10 Ammo` at `workspace` root, 5 `Decal`s + `ProximityPrompt`
  + `RefilSound` + `Config` Script.
- **Static props that are NOT pickups**: `Ammo Dispencer`, `Medkit Dispencer`, `Grenade Dispencer`,
  `Armor Dispencer` (x3), `Random Dispencer` - all `Model`s with a `ClickDetector` on `.Button`.
  Filter any name containing `dispenc`.
- **Heal pickups** clone from `ReplicatedStorage.Heal.HealModels`: 7 `MeshPart`s, each with a prompt -
  `Bandage`, `Medkit`, `Big Medkit`, `Regen Stim`, `Health Stim`, `Combat Stim`, `Coffee`.
- **Equipped weapons** are `Tool` children of a character (`O_CsF.Musket`, `FakeAimbotON.Eagle.50`).
- `ReplicatedStorage.DisabledAmmo` holds `ModuleScript`s (`0Ammo`, `5Ammo`, `15Ammo`, `60Ammo`,
  `71Ammo`, `SlugAmmo`) plus matching `Model`s. A decompiled `15Ammo` is an ACS attachment
  settings table (`SightZoom`, `DamageMod`, `camRecoil`, `MinSpread`, `Drum`, `Smol`), not a count.

### Map
- Rooms: `Room1_19`, `Room1_11`, `Room1_7`, `Room1_17`, `Room1_16`, `Room1_B`, `RoomA_2`,
  `RoomA_5`, `RoomA_6`, `RoomA_7`. Each has `Spawner15s` parts and a `BotSpawn` script.
- Doors: `ProximityPrompt` on `<Room>.Wooden Door.Base` and `Room1_B.Door`.
- Characters have a `REVIVE` ProximityPrompt.
- `ReplicatedStorage.Characters` has 20 NPC templates (`nobody`, `operator_alpha`, `chemist`, ...).

## UNVERIFIED
- A dropped heal keeping its `HealModels` name at workspace root (none was on the floor while probing).
- A dropped weapon keeping its gun name at workspace root (only equipped `Tool`s were seen).
- Every player character carrying `NameTag` + `FPV_DynamicCam` (2 of 2 observed).

## Notes
- `matcha_scripts` / `matcha_search` need `matcha_reindex` after joining.
- `matcha_decompile` by path can return `instance not found`; `decompile(instance)` inside `matcha_exec` works.
- Printing many short strings in one `matcha_exec` garbles the log tail; use one `table.concat`ed string.
- A push can report a sha and size while the file is not there. Always read the path back.
