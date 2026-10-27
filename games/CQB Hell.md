# CQB Hell

Place id: `98689921952741` (from `matcha_status`, 2026-10-27).
Script: `roblox-matcha/CQB Hell`, current version `v3.0` (weapon mods only).

## There is NO global enemy-name folder (corrects an earlier false entry)

Earlier this session I recorded that `ReplicatedStorage.EnemyDB.Enemies` was the game's
enemy catalogue holding 6 names. **That was wrong.** The results it came from were stale
`matcha_exec` echoes. Re-probing with different code shapes returned:

```
ENEMYDB n=27 :: DefaultROBLOXRegen:Script, Humanoid:Humanoid, Body Colors:BodyColors,
                Beams:Folder, Settings:Folder, Shirt:Shirt, ...
EnemyDB:FindFirstChild('Enemies')  -> nil   (NO_ENEMIES)
EnemyDB:FindFirstChild('Data')     -> nil   (NO_DATA)
ReplicatedStorage.Bots children    -> AIG, AIS   (2, not BotsConfig)
Bots:FindFirstChild('BotsConfig')  -> nil   (NO_CFG)
```

So `EnemyDB` is an NPC-like **Model** (Humanoid + Body Colors + Settings + Shirt), not a
database folder, and it has no `Enemies` or `Data` child. `Bots` holds `AIG` and `AIS`
(likely the bot AI scripts), not a config folder.

**No container in `ReplicatedStorage` holds the full enemy name list.** The only name-bearing
things are `ACS_Engine.GunModels` (the gun pool, map-dependent) and `Characters` (20
appearance templates, never seen used as live NPC names).

Rule learned: when two `matcha_exec` calls disagree, and one of them answers a *different
question* than the one asked, that one is the stale echo. Discard it and re-probe with a
changed prefix or index range before writing anything into the brain.

## Enemy names: take them from the live world, not a folder

The reliable source is the live humanoid models. Run this at round start:

```lua
local seen, out = {}, {}
for _, v in ipairs(workspace:GetDescendants()) do
    if v.ClassName == 'Model' and v:FindFirstChildOfClass('Humanoid') then
        local n = (v.Name:gsub('_Corpse$', ''):gsub('GP$', ''))
        if n ~= 'Viewmodel' and n ~= 'Dummy' and n ~= 'DummyTwo' and not seen[n] then
            seen[n] = true; table.insert(out, n)
        end
    end
end
table.sort(out)
print(table.concat(out, ', '))
```

Names confirmed alive in at least one probe (the pool changes per map/round, so no single
list is complete):

```
AKM, Brian the Hero, Db, Deagle, Draco, EnemyShotgun, Mosin, MP5,
Musket, Nagant, PKM, PM, RPK, Slam, Tec9, TT-33
```

Gun names confirmed in `ACS_Engine.GunModels` (enemies when an NPC carries them; the pool
held 41 instances / 14 unique one round and only 6 unique another):

```
AKM, GV-15, R8-Shorty, VZ-82, VZ-94, Webley, WM-106
```

Union for a Matcha NPC list (over-covering costs nothing; a missing name loses kills):

```
AKM, Brian the Hero, Db, Deagle, Draco, EnemyShotgun, GV-15, Mosin, MP5,
Musket, Nagant, PKM, PM, R8-Shorty, RPK, Slam, Tec9, TT-33, VZ-82, VZ-94,
WM-106, Webley
```

Use **Match Mode = `Name Contains`** with the plain name: the same enemy appears as `AKM`
(body), `AKMGP` (wrapper) and `AKM_Corpse` (dead), so one entry covers all three.

### Names that are NOT enemies - never add these

Observed live with a `Humanoid`, so a naive `Contains` filter catches them:
- `Viewmodel` - the LOCAL PLAYER'S OWN first-person arms, under `workspace.Camera.Viewmodel`.
  Has a Humanoid. Silent aim on it targets yourself.
- `Dummy`, `DummyTwo` - training dummies under `workspace.Start`.
- Player character models, named exactly after the player (`O_CsF`, `FakeAimbotON`).
- Wall props: `Medkit Dispenser`, `Armor Dispenser`, `Grenade Dispenser`,
  `Random Dispencer`, `Ammo Dispencer`.

**Keep `Ignore Players` ON.** Player characters carry `NameTag` + `FPV_DynamicCam`; NPCs
carry a `Settings` folder instead. A `Contains` match on a short name can otherwise hit a
player.

### Possible bot display names (UNVERIFIED)

`ReplicatedStorage.Characters` holds 20 templates: `capri, chemist, contract, deserter,
informant, kalyna, maid, michael, nobody, operator_alpha, pavlo, remnant, teto, theif,
thony, thonymaid, time, trench, wolf, yarik`. No live NPC was ever seen using one of these
names - live NPCs are gun-named. If a bot ever appears with one, add it; otherwise these are
only appearance templates.

## Weapon settings are NumberValues, not GC tables

Probe 2026-10-27 found real writable instances:

```
<gun>.Settings.<Weapon>.Firerate      NumberValue  values seen: 0.1, 0.15, 0.2, 0.4, 1
<gun>.Settings.<Weapon>.ReloadDelay   NumberValue  value seen: 0.2
```

Paths hit: `Workspace.EnemyShotgun_Corpse.Settings.M4.Firerate`,
`Workspace.Tec9_Corpse.Settings.M4.Firerate`, `Workspace.PKM_Corpse.Settings.M4.Firerate`
(same for `ReloadDelay`). **Different engine family from Realistic Home Defense**, where ACS
wraps every settings table in `LockSettings`/`table.freeze` so GC writes are inert. Here the
values are plain `NumberValue` instances - no `getgc`, no `applygc`, no unfreeze step.

**Important:** the probe hit only `*_Corpse` models. The script walks the **local character**
(`workspace[LocalPlayer.Name]`) for `Firerate` / `ReloadDelay` NumberValues.

Ammo field name is UNVERIFIED - no gun was equipped during probing (`CHARTOOLS n=0`).

## Where the NPCs actually are in the world

NPCs live under a folder named `People`, but **that folder moves**. Two probes minutes apart:

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

So: **find NPCs by walking ANCESTORS for an instance named `People`, never by a fixed path.**
Each NPC is wrapped one level deep - `People.<Name>GP.<Name>` - and the wrapper has no
Humanoid. Draw the inner model, and drop a candidate whose parent is already a candidate.

## Health is a sentinel, not a value

NPC and statue Humanoids report `5000000000 / 5000000000` (5 billion) - the game's
"invincible / don't show HP" marker.
- a bar computed from it is always full and always green
- hide HP text and skip the bar when `MaxHealth >= 1e6`
- **Statues** (`Surroundings.Lobby.Model.Statue`) have a Humanoid at 5e9 too and must be
  excluded by ancestor (`Lobby`, `Statue`)

## Player vs NPC

- players -> has `NameTag`, has `FPV_DynamicCam`, no `Settings`
- NPCs -> no `NameTag`, no `FPV_DynamicCam`, has `Settings`

`isPlayer = hasChild('NameTag') or hasChild('FPV_DynamicCam')`. Name is never consulted,
because NPC names are gun names and collide with real player names.

Character models are DIRECT `workspace` children named after the player, ~20 per server.
`LocalPlayer.Character` is intermittent here, so resolve the character by
`workspace[LocalPlayer.Name]` (probe: `CHAR true name=FakeAimbotON`).

## Other structure
- Corpses are `<Name>_Corpse` with `Humanoid.Health == 0`; 30 can be on the floor at once.
- Ammo pickups are `Part`s named `10 Ammo` at `workspace` root, 5 `Decal`s + `ProximityPrompt`
  + `RefilSound` + `Config` Script.
- Static props that are NOT pickups: `Ammo Dispencer`, `Medkit Dispencer`, `Grenade Dispencer`,
  `Armor Dispencer` (x3), `Random Dispencer` - `Model`s with a `ClickDetector` on `.Button`.
- Heal pickups clone from `ReplicatedStorage.Heal.HealModels`: 7 `MeshPart`s, each with a prompt -
  `Bandage`, `Medkit`, `Big Medkit`, `Regen Stim`, `Health Stim`, `Combat Stim`, `Coffee`.
- Equipped weapons are `Tool` children of a character.
- Rooms: `Room1_19`, `Room1_11`, `Room1_7`, `Room1_17`, `Room1_16`, `Room1_B`, `RoomA_2`,
  `RoomA_5`, `RoomA_6`, `RoomA_7`; `ProximityPrompt` doors; `REVIVE` prompts on characters.

## UNVERIFIED
- The ammo NumberValue name inside a live gun (no gun equipped while probing).
- Whether the server honours client-side `Firerate` / `ReloadDelay` writes.
- Whether the `ReplicatedStorage.Characters` template names are ever used as bot display names.
- What `Bots.AIG` / `Bots.AIS` are (Script or ModuleScript; children not yet listed).
- What the `EnemyDB` Model is actually for (it has a Humanoid and Settings, like an NPC body).

## Notes
- A nil `_G[NAME]` does not mean the script failed to load. Check `matcha_logs` for the
  `[NAME] stopped` line: that is my own `stop()` firing.
- `matcha_scripts` / `matcha_search` need `matcha_reindex` after joining.
- Printing many short strings in one `matcha_exec` garbles or truncates the log tail;
  a long `table.concat` also gets cut at roughly 100 characters. Print one or two values
  per call when the payload matters.
- **A `matcha_exec` call can return a plausible-looking result that is NOT current.** Seen
  this session: `R5TO8 ...` placeholders, an earlier block echoed verbatim, and probe results
  that answered a different question than the one asked. Treat a repeat of an earlier string,
  or an answer to the wrong question, as a failed call; change the prefix or range and re-run.
- A push can report a sha and size while the file is not there, or return a body that
  differs from what was sent. Always read the path back and compare.
