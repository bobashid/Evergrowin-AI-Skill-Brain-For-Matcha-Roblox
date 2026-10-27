# CQB Hell

Place id: `98689921952741` (from `matcha_status`, 2026-10-27).
Script: `roblox-matcha/CQB Hell`, current version `v3.0` (weapon mods only).

## The user's Matcha config is the best name source (2026-10-27)

The user exported their Matcha settings as `CQB HELL.cfg` (59,787 bytes, 134 chunks). It is
more trustworthy than a live probe for one reason: the names in it are ones the user saw
spawning and typed in himself, not names read from a possibly-not-yet-loaded server.

Structure: `[ESP]`, `[NPC]` (Entries=17, each `N.Name/Path/MatchMode/MatchString`),
`[SilentAim]`, `[Weapon]`.

### Enemy entries that are correct (14 of the 17 slots)

```
Db, AKM, Tec9, Mosin, RPK, Deagle, Draco, PKM, TT-33, PM, Musket, Nagant
```

Plus `Tec9` listed twice (slots 6 and 8) and `RPK` appearing as a MatchString in slot 6.

### Bugs found in the config - tell the user these

1. **Slot 6 is `Name=Tec9` but `MatchString=RPK`.** A duplicate-name copy/paste. That entry
   does NOT cover Tec9; it duplicates the RPK entry. Fix: set 6.MatchString to `Tec9` or
   delete it (slot 3 already covers Tec9).
2. **Slots 15, 16, 17 are wall props**, not enemies: `Medkit Dispenser`, `Armor Dispenser`,
   `Armor Dispenser`. Three of seventeen slots wasted, and each makes silent aim able to
   lock onto a wall. Delete all three.
3. **Duplicated keys** scattered through the file (`5.MatchMode` twice, `6.Name` twice,
   `8.Path` twice, `15/16/17.Name` three times each). Harmless at load (last wins) but a
   sign of hand-editing; worth a clean re-save.
4. `[SilentAim]` has `TargetPlayers=false` and `IgnoreDead=true`. That matches this game:
   NPCs are gun-named, players are named after the account, and 30 corpses can be on the
   floor. Leave both as they are.
5. `FOV=180` is very wide and `Smoothing=0.35` is heavy. If aim feels like it drags or snaps
   to the wrong target, tighten FOV first (90-120), then lower smoothing.

## Names still missing from the config

Confirmed alive in live probes but absent from the 17 entries:

```
Brian the Hero, EnemyShotgun, MP5, Slam, GV-15, R8-Shorty, VZ-82, VZ-94, Webley,
G18, IWC-577, RG-14, RG-20, WM-106, AK-74, AP5
```

`EnemyShotgun`, `MP5` and `Slam` in particular were in a live probe with real health values,
so they are real enemies the config does not cover.

## READ THIS: probe results are only valid while fully in a server

User-stated (2026-10-27): "i wasnt in the game for some of it". Several weapon-folder probes
in this session were taken while the client was **not fully loaded into a server**. That
invalidates them as evidence. Specifically suspect:

- `SpecialWeapons` -> "0 children"
- `WeaponDrops` -> "0 children"
- `GunModels` shrinking 14 -> 6 -> 8 unique

The shrinking pool is at least as likely to be "map not streamed in yet" as a real per-map
rotation. **Re-probe all of these with the user confirmed in-game before trusting any of it.**

Rule: before treating a `matcha_exec` result about world/RS contents as fact, confirm
`matcha_status` reports `in_game: true` AND the user says they are in a server. An empty
folder read during loading is indistinguishable from a genuinely empty folder.

## Weapon names (UNCONFIRMED - needs a clean in-server read)

`ACS_Engine.GunModels` reads, in order:

| Read | Unique gun names | Trust |
|---|---|---|
| A | 14 unique (41 instances) - `AP5, AKM, AK-74, Mosin, MP5, WM-106` confirmed individually | in-game, but log-truncated after 6 |
| B | 6 unique - `AKM, GV-15, R8-Shorty, VZ-82, VZ-94, Webley` | in-game |
| C | 8 unique - `G18, IWC-577, R8-Shorty, RG-14, RG-20, VZ-82, VZ-94, Webley` | validity unknown |

Containers checked:

```
ReplicatedStorage.SpecialWeapons   -> reported 0 children  (SUSPECT - re-probe in-server)
ReplicatedStorage.WeaponDrops      -> reported 0 children  (SUSPECT - re-probe in-server)
ReplicatedStorage.Weapons          -> absent / empty (guns clone in at runtime)
ReplicatedStorage.Manufacturers    -> 4 children: VZ, MPD, FM, RRG   (brands, not guns)
ReplicatedStorage.DisabledAmmo     -> ammo ATTACHMENT settings Modules, not guns
ACS_Engine.GunModels               -> the live gun pool (authoritative IF read in-server)
ACS_Engine.Attachments             -> attachments, not guns
ReplicatedStorage.PreMade          -> NOT YET OPENED - last unchecked candidate
```

Provisional union (do not publish as complete until re-probed in-server):

```
AK-74, AKM, AP5, Db, Deagle, Draco, EnemyShotgun, G18, GV-15, IWC-577,
Mosin, MP5, Musket, Nagant, PKM, PM, R8-Shorty, RG-14, RG-20, RPK,
Slam, Tec9, TT-33, VZ-82, VZ-94, Webley, WM-106
```

### Read the current set (run only when in a server)

```lua
local g = game:GetService('ReplicatedStorage').ACS_Engine.GunModels
local seen, out = {}, {}
for _, c in ipairs(g:GetChildren()) do
    if not seen[c.Name] then seen[c.Name] = true; table.insert(out, c.Name) end
end
table.sort(out)
print('COUNT ' .. #out)
print(table.concat(out, ', '))
```

Log output truncates around 100 characters, so print the count on its own line first.

## There is NO global enemy-name folder (confirmed by re-probe)

`ReplicatedStorage.EnemyDB.Enemies` does **not** exist. An earlier entry claiming it held 6
enemy names came from stale `matcha_exec` echoes. Re-probed with different code shapes:

```
ENEMYDB n=27 :: DefaultROBLOXRegen:Script, Humanoid:Humanoid, Body Colors:BodyColors,
                Beams:Folder, Settings:Folder, Shirt:Shirt, ...
EnemyDB:FindFirstChild('Enemies')  -> nil   (NO_ENEMIES)
EnemyDB:FindFirstChild('Data')     -> nil   (NO_DATA)
ReplicatedStorage.Bots children    -> AIG, AIS
Bots:FindFirstChild('BotsConfig')  -> nil   (NO_CFG)
```

`EnemyDB` is an NPC-like **Model** (Humanoid + Body Colors + Settings + Shirt). `Bots` holds
`AIG`/`AIS`, likely bot AI scripts. Neither is a name list.

Rule: when two `matcha_exec` calls disagree and one answers a *different question* than the
one asked, that one is the stale echo. Discard and re-probe with a changed prefix.

### Names that are NOT weapons/enemies - never add these

Observed live with a `Humanoid`, so a naive `Contains` filter catches them:
- `Viewmodel` - the LOCAL PLAYER'S OWN first-person arms, under `workspace.Camera.Viewmodel`.
  Has a Humanoid. Silent aim on it targets yourself.
- `Dummy`, `DummyTwo` - training dummies under `workspace.Start`.
- Player character models, named exactly after the player (`O_CsF`, `FakeAimbotON`).
- Wall props: `Medkit Dispenser`, `Armor Dispenser`, `Grenade Dispenser`,
  `Random Dispencer`, `Ammo Dispencer`.

**Keep `Ignore Players` ON.** Players carry `NameTag` + `FPV_DynamicCam`; NPCs carry a
`Settings` folder. Use **Match Mode = `Name Contains`** with the plain name: `AKM` (body),
`AKMGP` (wrapper) and `AKM_Corpse` (dead) are all caught by one entry.

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
- Whether `SpecialWeapons` / `WeaponDrops` are genuinely empty (reads taken out of server).
- Whether the `GunModels` pool truly rotates per map or was just not streamed in.
- The full 14-name list from round A (log truncation cut it after 6).
- `ReplicatedStorage.PreMade` contents - never opened.
- The ammo NumberValue name inside a live gun.
- Whether the server honours client-side `Firerate` / `ReloadDelay` writes.
- Whether `Bots.AIG` / `Bots.AIS` return a spawn table.
- What the `EnemyDB` Model is for.

## Notes
- A nil `_G[NAME]` does not mean the script failed to load. Check `matcha_logs` for the
  `[NAME] stopped` line: that is my own `stop()` firing.
- `matcha_scripts` / `matcha_search` need `matcha_reindex` after joining.
- Printing many short strings in a `matcha_exec` garbles or truncates the log tail; a long
  `table.concat` gets cut at roughly 100 characters. Print the count on its own line first.
- **A `matcha_exec` call can return a plausible-looking result that is NOT current.** Seen:
  `R5TO8 ...` placeholders, an earlier block echoed verbatim, and answers to a different
  question than asked. Treat a repeat, or a wrong-question answer, as a failed call.
- **`retrieve` can fail on a large `.cfg` while `get_chunks` succeeds.** The cfg is 134 chunks;
  `retrieve` returned malformed citations with no cfg passages, `get_chunks` read it fine.
  Use `get_chunks` in small ranges for structured config files.
- A push can report a sha and size while the file is not there, or return a body that differs
  from what was sent. Always read the path back and compare.
