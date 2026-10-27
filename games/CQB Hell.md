# CQB Hell

Place id: `98689921952741` (from `matcha_status`, 2026-10-27).
Script: `roblox-matcha/CQB Hell`, current version `v1.4`.

## Features
Weapon / NPC / Heal / Ammo ESP, one tab, four toggles, range + transparency sliders.

## Where the NPCs actually are (the important one)

**`workspace.People` is the NPC container, and every NPC is wrapped one level deep:**

```
Workspace
  People
    Tec9GP          <- Model, NO Humanoid
      Tec9          <- Model, HAS the Humanoid  <- draw here
    MosinGP
      Mosin
```

The wrapper name carries a `GP` suffix; the inner model keeps the plain gun name. Any
collector that only looks at `workspace` root children finds `Tec9GP` (no Humanoid) and
rejects it, which is why a root-only scan shows nothing. Probe: `PEOPLE n=5 :: Tec9GP>Tec9(5000000000/5000000000) | MosinGP>Mosin(5000000000/5000000000) | ...`

## Health is a sentinel, not a value

NPC Humanoids report **`5000000000 / 5000000000`** (5 billion). That is the game's
"invincible / don't show HP" marker, not real health. Consequences:
- a health bar computed from it is always full and always green
- a label reading `5000000000/5000000000` is unreadable and covers the screen
- v1.4 hides HP text and skips the bar when `MaxHealth >= 1e6`, and tints those NPCs purple
  so "invincible" is visually distinct from "full HP"

## Naming

**NPC models are named after the gun they carry**, and names repeat:
`AKM`, `Tec9`, `PM`, `RPK`, `Slam`, `Deagle`, `Mosin`, `MP5`, `PKM`, `Draco`, `EnemyShotgun`,
`Db`, `Brian the Hero`. Two `PM`, three `Slam`, four `Tec9` alive at once is normal.
A name-based exclude list therefore hides real NPCs the moment a player Name or DisplayName
collides with a weapon name (user-reported, fixed in v1.3).

**Players are told apart by marker, not name.** Probe of live humanoid models:
- players (`O_CsF`, `FakeAimbotON`) -> has `NameTag`, has `FPV_DynamicCam`, no `Settings`
- NPCs -> no `NameTag`, no `FPV_DynamicCam`, has `Settings`

So `isPlayer = hasChild('NameTag') or hasChild('FPV_DynamicCam')`. Name is never consulted.

## Other structure
- Corpses are `<Name>_Corpse` with `Humanoid.Health == 0`; 30 can be on the floor at once.
- **Ammo pickups** are `Part`s named `10 Ammo` at `workspace` root, 5 `Decal`s + `ProximityPrompt`
  + `RefilSound` + `Config` Script.
- **Static props that are NOT pickups**: `Ammo Dispencer`, `Medkit Dispencer`, `Grenade Dispencer`,
  `Armor Dispencer` (x3), `Random Dispencer` - all `Model`s with a `ClickDetector` on `.Button`.
  Filter any name containing `dispenc`.
- **Heal pickups** clone from `ReplicatedStorage.Heal.HealModels`: 7 `MeshPart`s, each with a prompt -
  `Bandage`, `Medkit`, `Big Medkit`, `Regen Stim`, `Health Stim`, `Combat Stim`, `Coffee`.
- **Equipped weapons** are `Tool` children of a character.
- `ReplicatedStorage.DisabledAmmo` holds `ModuleScript`s (`0Ammo`, `5Ammo`, `15Ammo`, `60Ammo`,
  `71Ammo`, `SlugAmmo`) plus matching `Model`s. A decompiled `15Ammo` is an ACS attachment
  settings table, not an ammo count.
- Rooms: `Room1_19`, `Room1_11`, `Room1_7`, `Room1_17`, `Room1_16`, `Room1_B`, `RoomA_2`,
  `RoomA_5`, `RoomA_6`, `RoomA_7`; `ProximityPrompt` doors; `REVIVE` prompts on characters.
- `ReplicatedStorage.Characters` holds 20 NPC templates (`nobody`, `operator_alpha`, ...).

## UNVERIFIED
- A dropped heal or dropped weapon keeping its name at `workspace` root (only templates seen).
- Whether `People` also contains the corpse models or only living NPCs.

## Notes
- **A nil `_G[NAME]` does not mean the script failed to load.** Check `matcha_logs` for the
  `[NAME] stopped` line: that is my own `stop()` firing (reloading runs the kill-previous
  line, or the user stopped it). v1.1-v1.3 loaded fine and were stopped, not broken.
- `matcha_scripts` / `matcha_search` need `matcha_reindex` after joining.
- `matcha_decompile` by path can return `instance not found`; `decompile(instance)` inside
  `matcha_exec` works.
- Printing many short strings in one `matcha_exec` garbles the log tail; use one `table.concat`.
- A push can report a sha and size while the file is not there. Always read the path back.
