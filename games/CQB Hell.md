# CQB Hell

Place id: `98689921952741` (from `matcha_status`, 2026-10-27).
Script: `roblox-matcha/CQB Hell`, current version `v1.0`.

## Features
Weapon / NPC / Heal / Ammo ESP, one tab, four toggles, range + transparency sliders.

## Confirmed structure (probes 2026-10-27)

- **Player characters** are `workspace` children with a `Humanoid`: `O_CsF`, `FakeAimbotON` (the LocalPlayer's character name equals the player name here). Exclude them by checking `Players:GetPlayers()[i].Character == inst`.
- **NPCs** are root `Model`s with a `Humanoid`. Names: `Tec9`, `AKM`, `EnemyShotgun`, `Draco`, `Mosin`, `MP5`, plus `Draco` at partial HP. Health reads fine: `100/100`, `150/150`, `200/200`, `46.2/200`.
- **Corpses** are the same model renamed `<Name>_Corpse` with `Humanoid.Health == 0` (e.g. `Tec9_Corpse`, `AKM_Corpse`, `Draco_Corpse`). There can be many at once (10+), so a corpse filter is needed or the ESP is noise.
- **Equipped weapons** are `Tool` children of a character: `O_CsF.Musket`, `FakeAimbotON.Eagle.50`, `O_CsF.Makarov`, `FakeAimbotON.WM-106`.
- **Ammo pickups** in the world are `Model`s named `10 Ammo` at `workspace` root, each with two `Part` children both named `10 Ammo`, and a `ProximityPrompt` child. Several can exist at once.
- **Heal pickups** are cloned from `ReplicatedStorage.Heal.HealModels`. That folder holds `MeshPart`s: `Bandage`, `Medkit`, `Big Medkit`, `Regen Stim`, `Coffee` (and the `Heal` folder also has `Tool`s named `Medkit`, `Big Medkit`, `Bandage`, `Regen Stim`, `Health Stim`, `Combat Stim`, `Coffee`). Each `HealModels` MeshPart carries a `ProximityPrompt`, a `Put Item In ServerStorge` script and a `CustomPromptConf` Configuration (Beam / Highlight settings).
- **Ammo templates** live in `ReplicatedStorage.DisabledAmmo` as `ModuleScript`s (`0Ammo`, `5Ammo`, `15Ammo`, `60Ammo`, `71Ammo`, `SlugAmmo`) plus matching `Model`s. A decompiled `15Ammo` is an ACS attachment settings table (`SightZoom`, `DamageMod`, `camRecoil`, `MinSpread`, `Drum`, `Smol`, ...), not an ammo count.
- **Other world objects** seen: `Chemlight 10x`, `Crayon 3x`, `Armor Dispencer` (x3, has a `ClickDetector` on `.Button`), `Random Dispencer` (ClickDetector on `.Button`), `wonder table.PaperMain`.
- **Doors** use `ProximityPrompt` on `<Room>.Wooden Door.Base` and `Room1_B.Door`. Characters have a `REVIVE` ProximityPrompt.
- Rooms are `Room1_19`, `Room1_11`, `Room1_7`, `Room1_17`, `Room1_16`, `Room1_B`; each room has `Spawner15s` parts and a `BotSpawn` script.
- Character models carry `Health` and `Armor` Scripts and an `ACS_Client.Kit` folder; `ACS_Client.Stances.MedKit` is a `BoolValue`.

## UNVERIFIED
- Whether a dropped heal appears at `workspace` root under its own name (only `Heal.HealModels` was inspected; no heal was on the floor during probing).
- Whether a dropped weapon keeps the gun model name at `workspace` root; only equipped `Tool`s were seen.
- Ammo pickup name may vary with amount (`5 Ammo`, `15 Ammo`); the script matches any name containing `ammo`, so this is covered either way.

## Notes
- `matcha_scripts` / `matcha_search` need `matcha_reindex` after joining; the index went stale mid-session (`index not found`).
- `matcha_decompile` by path failed with `instance not found` for `Room1_19.BotSpawn` (the part existed but the script was not indexed yet); `matcha_exec` + `decompile(instance)` inside the VM is the reliable route.
- Several `matcha_exec` calls returned garbled or duplicated log lines when printing many short strings; printing one `table.concat`ed string per call was cleaner.
