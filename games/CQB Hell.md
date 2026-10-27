# CQB Hell

Place id: `98689921952741` (from `matcha_status`, 2026-10-27).
Script: `roblox-matcha/CQB Hell`, current version `v1.5`.

## Features
Weapon / NPC / Heal / Ammo ESP, one tab, four toggles, range + transparency sliders.

## Where the NPCs actually are (the important one)

NPCs live under a folder named `People`, but **that folder moves**. It is NOT reliably a
direct `workspace` child. Two probes minutes apart:

```
PEOPLE n=5 :: Tec9GP>Tec9(5000000000/5000000000) | MosinGP>Mosin(5000000000/5000000000)
PEOPLE_NOW false
HUMANS n=29 ::
  Surroundings>Lobby>Model>Statue(5000000000/5000000000)
  People>Model>Tec93GP>Tec9(5000000000/5000000000)
  Surroundings>People>MosinGP>Mosin(5000000000/5000000000)
  Surroundings>People>Tec9GP>Tec9(5000000000/5000000000)
  Surroundings>People>MP5GP>MP5(5000000000/5000000000)
```

So: **find NPCs by walking ANCESTORS for an instance named `People`, never by a fixed
path.** `workspace:FindFirstChild('People')` returned `false` while three NPCs were alive
under `Surroundings.People`. v1.4 used the fixed path and drew nothing; v1.5 uses ancestry.

Each NPC is also wrapped one level deep - `People.<Name>GP.<Name>` - and the wrapper has
no Humanoid. Draw the inner model, and drop a candidate whose parent is already a candidate
so the wrapper and body do not both box.

## Health is a sentinel, not a value

NPC and statue Humanoids report `5000000000 / 5000000000` (5 billion). That is the game's
"invincible / don't show HP" marker.
- a bar computed from it is always full and always green
- a label reading `5000000000/5000000000` is unreadable and covers the screen
- v1.5 hides HP text and skips the bar when `MaxHealth >= 1e6`, tinting those NPCs purple
- **Statues** (`Surroundings.Lobby.Model.Statue`) have a Humanoid at 5e9 too and must be
  excluded by ancestor (`Lobby`, `Statue`) or the ESP boxes scenery

## Naming

NPC models are named after the gun they carry, and names repeat:
`AKM`, `Tec9`, `PM`, `RPK`, `Slam`, `Deagle`, `Mosin`, `MP5`, `PKM`, `Draco`, `EnemyShotgun`,
`Db`, `Brian the Hero`. A name-based exclude list hides real NPCs whenever a player Name or
DisplayName collides with a weapon name (user-reported, fixed in v1.3).

Players are told apart by marker, not name:
- players -> has `NameTag`, has `FPV_DynamicCam`, no `Settings`
- NPCs -> no `NameTag`, no `FPV_DynamicCam`, has `Settings`

So `isPlayer = hasChild('NameTag') or hasChild('FPV_DynamicCam')`. Name is never consulted.

## Players
- Character models are DIRECT `workspace` children named after the player, ~20 per server
  (`JuanPiripicho`, `O_CsF`, `FakeAimbotON`, `TwixieKitty7575`, ...), each `100/100`.
- So the NPC collector must never assume "root model with a Humanoid = NPC".

## Other structure
- Corpses are `<Name>_Corpse` with `Humanoid.Health == 0`; 30 can be on the floor at once.
- Ammo pickups are `Part`s named `10 Ammo` at `workspace` root, 5 `Decal`s + `ProximityPrompt`
  + `RefilSound` + `Config` Script.
- Static props that are NOT pickups: `Ammo Dispencer`, `Medkit Dispencer`, `Grenade Dispencer`,
  `Armor Dispencer` (x3), `Random Dispencer` - `Model`s with a `ClickDetector` on `.Button`.
  Filter any name containing `dispenc`.
- Heal pickups clone from `ReplicatedStorage.Heal.HealModels`: 7 `MeshPart`s, each with a prompt -
  `Bandage`, `Medkit`, `Big Medkit`, `Regen Stim`, `Health Stim`, `Combat Stim`, `Coffee`.
- Equipped weapons are `Tool` children of a character.
- `ReplicatedStorage.DisabledAmmo` holds `ModuleScript`s (`0Ammo`, `5Ammo`, `15Ammo`, `60Ammo`,
  `71Ammo`, `SlugAmmo`) plus matching `Model`s. A decompiled `15Ammo` is an ACS attachment
  settings table, not an ammo count.
- Rooms: `Room1_19`, `Room1_11`, `Room1_7`, `Room1_17`, `Room1_16`, `Room1_B`, `RoomA_2`,
  `RoomA_5`, `RoomA_6`, `RoomA_7`; `ProximityPrompt` doors; `REVIVE` prompts on characters.
- `ReplicatedStorage.Characters` holds 20 NPC templates (`nobody`, `operator_alpha`, ...).

## UNVERIFIED
- A dropped heal or dropped weapon keeping its name at `workspace` root (only templates seen).
- Whether `People` can also appear as a grandchild deeper than `Surroundings` (the ancestry
  walk already covers 10 levels, so this should be moot, but it is untested in every map).

## Notes
- A nil `_G[NAME]` does not mean the script failed to load. Check `matcha_logs` for the
  `[NAME] stopped` line: that is my own `stop()` firing (reloading runs the kill-previous
  line, or the user stopped it). v1.1-v1.3 loaded fine and were stopped, not broken.
- `matcha_scripts` / `matcha_search` need `matcha_reindex` after joining.
- `matcha_decompile` by path can return `instance not found`; `decompile(instance)` inside
  `matcha_exec` works.
- Printing many short strings in one `matcha_exec` garbles the log tail; use one `table.concat`.
- A push can report a sha and size while the file is not there. Always read the path back.
