# CQB Hell

Place id: `98689921952741` (from `matcha_status`, 2026-10-27).
Script: `roblox-matcha/CQB Hell`, current version `v3.0` (weapon mods only).

## Weapon names: the pool ROTATES per map (no static list exists)

There is no folder holding every weapon in the game. `ACS_Engine.GunModels` is rebuilt for
each map, proven by three reads hours apart that barely overlap:

| Read | Unique gun names |
|---|---|
| A | 14 unique (41 instances) - `AP5, AKM, AK-74, Mosin, MP5, WM-106` confirmed individually |
| B | 6 unique - `AKM, GV-15, R8-Shorty, VZ-82, VZ-94, Webley` |
| C | 8 unique - `G18, IWC-577, R8-Shorty, RG-14, RG-20, VZ-82, VZ-94, Webley` |

B and C share only 3 names (`R8-Shorty, VZ-82, VZ-94`). So a list built from one round is
wrong for the next one.

### Containers checked, with what they actually hold

```
ReplicatedStorage.SpecialWeapons   -> exists, 0 children   (empty)
ReplicatedStorage.WeaponDrops      -> exists, 0 children   (empty)
ReplicatedStorage.Weapons          -> absent / empty (guns clone in at runtime)
ReplicatedStorage.Manufacturers    -> 4 children: VZ, MPD, FM, RRG   (brands, not guns)
ReplicatedStorage.DisabledAmmo     -> ammo ATTACHMENT settings Modules, not guns
ACS_Engine.GunModels               -> the live per-map gun pool (authoritative for this map)
ACS_Engine.Attachments            -> attachments, not guns
```

`Manufacturers` (VZ, MPD, FM, RRG) are brand prefixes, which explains names like `VZ-82`
and `VZ-94`; it is not a weapon list itself.

### Union of every weapon name confirmed in ANY round

```
AK-74, AKM, AP5, Db, Deagle, Draco, EnemyShotgun, G18, GV-15, IWC-577,
Mosin, MP5, Musket, Nagant, PKM, PM, R8-Shorty, RG-14, RG-20, RPK,
Slam, Tec9, TT-33, VZ-82, VZ-94, Webley, WM-106
```

27 names. Over-covering costs nothing; a missing name loses kills.

### Read the exact current set every round

```lua
local g = game:GetService('ReplicatedStorage').ACS_Engine.GunModels
local seen, out = {}, {}
for _, c in ipairs(g:GetChildren()) do
    if not seen[c.Name] then seen[c.Name] = true; table.insert(out, c.Name) end
end
table.sort(out)
print(table.concat(out, ', '))
```

Or from the live world (catches enemies whose gun is not in `GunModels`):

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

`EnemyDB` is an NPC-like **Model** (Humanoid + Body Colors + Settings + Shirt), not a
database folder. `Bots` holds `AIG`/`AIS`, likely the bot AI scripts.

Rule learned: when two `matcha_exec` calls disagree, and one answers a *different question*
than the one asked, that one is the stale echo. Discard it and re-probe with a changed
prefix or index range before writing anything into the brain.

### Names that are NOT weapons/enemies - never add these

Observed live with a `Humanoid`, so a naive `Contains` filter catches them:
- `Viewmodel` - the LOCAL PLAYER'S OWN first-person arms, under `workspace.Camera.Viewmodel`.
  Has a Humanoid. Silent aim on it targets yourself.
- `Dummy`, `DummyTwo` - training dummies under `workspace.Start`.
- Player character models, named exactly after the player (`O_CsF`, `FakeAimbotON`).
- Wall props: `Medkit Dispenser`, `Armor Dispenser`, `Grenade Dispenser`,
  `Random Dispencer`, `Ammo Dispencer`.

**Keep `Ignore Players` ON.** Players carry `NameTag` + `FPV_DynamicCam`; NPCs carry a
`Settings` folder. A `Contains` match on a short name can otherwise hit a player.

Use **Match Mode = `Name Contains`** with the plain name: `AKM` (body), `AKMGP` (wrapper)
and `AKM_Corpse` (dead) are all caught by one entry.

## Weapon settings are NumberValues, not GC tables

```
<gun>.Settings.<Weapon>.Firerate      NumberValue  values seen: 0.1, 0.15, 0.2, 0.4, 1
<gun>.Settings.<Weapon>.ReloadDelay   NumberValue  value seen: 0.2
```

**Different engine family from Realistic Home Defense**, where ACS wraps every settings
table in `LockSettings`/`table.freeze` so GC writes are inert. Here the values are plain
`NumberValue` instances - no `getgc`, no `applygc`, no unfreeze step.

The probe hit only `*_Corpse` models, so the script walks the **local character**
(`workspace[LocalPlayer.Name]`) for `Firerate` / `ReloadDelay`.

Ammo field name is UNVERIFIED - no gun was equipped during probing (`CHARTOOLS n=0`).

## Where the NPCs are in the world

NPCs live under a folder named `People`, but **that folder moves** (workspace root one probe,
`Surroundings.People` the next; `PEOPLE_NOW false` while three NPCs were alive). Find them by
walking ANCESTORS for an instance named `People`, never by a fixed path. Each NPC is wrapped
`People.<Name>GP.<Name>` and the wrapper has no Humanoid - draw the inner model and drop a
candidate whose parent is already a candidate.

## Health is a sentinel, not a value

NPC and statue Humanoids report `5000000000 / 5000000000` (5 billion) - the game's
"invincible / don't show HP" marker. Hide HP text and skip the bar when `MaxHealth >= 1e6`.
**Statues** (`Surroundings.Lobby.Model.Statue`) share the sentinel and must be excluded by
ancestor (`Lobby`, `Statue`).

## Player vs NPC

- players -> has `NameTag`, has `FPV_DynamicCam`, no `Settings`
- NPCs -> no `NameTag`, no `FPV_DynamicCam`, has `Settings`

Character models are DIRECT `workspace` children named after the player, ~20 per server.
`LocalPlayer.Character` is intermittent here, so resolve by `workspace[LocalPlayer.Name]`.

## Other structure
- Corpses are `<Name>_Corpse` with `Humanoid.Health == 0`; 30 can be on the floor at once.
- Ammo pickups are `Part`s named `10 Ammo` at `workspace` root, 5 `Decal`s + `ProximityPrompt`
  + `RefilSound` + `Config` Script.
- Static props that are NOT pickups: `Ammo Dispencer`, `Medkit Dispencer`, `Grenade Dispencer`,
  `Armor Dispencer`, `Random Dispencer` - `Model`s with a `ClickDetector` on `.Button`.
- Heal pickups clone from `ReplicatedStorage.Heal.HealModels`: 7 `MeshPart`s with prompts -
  `Bandage`, `Medkit`, `Big Medkit`, `Regen Stim`, `Health Stim`, `Combat Stim`, `Coffee`.
- Equipped weapons are `Tool` children of a character.
- Rooms: `Room1_19`, `Room1_11`, `Room1_7`, `Room1_17`, `Room1_16`, `Room1_B`, `RoomA_2`,
  `RoomA_5`, `RoomA_6`, `RoomA_7`; `ProximityPrompt` doors; `REVIVE` prompts on characters.

## UNVERIFIED
- The ammo NumberValue name inside a live gun (no gun equipped while probing).
- Whether the server honours client-side `Firerate` / `ReloadDelay` writes.
- Whether `ReplicatedStorage.Characters` template names (`capri`, `chemist`, ...) are ever
  used as bot display names - no live NPC has been seen using one.
- What `Bots.AIG` / `Bots.AIS` are, and whether either returns a spawn table.
- What the `EnemyDB` Model is for (it has a Humanoid and Settings, like an NPC body).
- The full 14-name list from round A (only 6 of it was printed before log truncation).

## Notes
- A nil `_G[NAME]` does not mean the script failed to load. Check `matcha_logs` for the
  `[NAME] stopped` line: that is my own `stop()` firing.
- `matcha_scripts` / `matcha_search` need `matcha_reindex` after joining.
- Printing many short strings in one `matcha_exec` garbles or truncates the log tail; a long
  `table.concat` gets cut at roughly 100 characters. Print one or two values per call.
- **A `matcha_exec` call can return a plausible-looking result that is NOT current.** Seen:
  `R5TO8 ...` placeholders, an earlier block echoed verbatim, and probe results answering a
  different question than asked. Treat a repeat, or a wrong-question answer, as a failed call;
  change the prefix or range and re-run before trusting it.
- A push can report a sha and size while the file is not there, or return a body that differs
  from what was sent. Always read the path back and compare.
