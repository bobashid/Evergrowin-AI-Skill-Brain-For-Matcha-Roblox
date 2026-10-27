# CQB Hell

Place id: `98689921952741` (from `matcha_status`, 2026-10-27).
Script: `roblox-matcha/CQB Hell`, current version `v3.0` (weapon mods only).

## Enemy names (for Matcha's built-in NPC list / silent aim)

NPC models are named after the gun they carry, and the same name repeats (three `Slam`,
four `Tec9` alive at once is normal). The inner body model carries the plain name; the
outer wrapper adds `GP`, and a dead one adds `_Corpse`. So a Matcha NPC entry with
**Match Mode = Name Contains** and **Match String = the plain name** catches all three
forms (`AKM`, `AKMGP`, `AKM_Corpse`) without extra entries.

Confirmed alive in one probe (`LIVE_NAMES`, 27 humanoid models):

```
AKM, Brian the Hero, Draco, Deagle, Db, EnemyShotgun, Mosin, MP5,
Nagant, PKM, PM, RPK, Slam, Tec9, TT-33, Musket
```

Confirmed from the gun-model pool (`ReplicatedStorage.ACS_Engine.GunModels`, 14 unique):

```
AP5, AKM, AK-74, Mosin, MP5, ...   (the rest duplicate the live names above)
```

Seen in the user's own Matcha NPC list screenshot (adds nothing new, but confirms them):
`Db, AKM, Tec9, Mosin, RPK, Slam, Deagle, Draco, PKM, TT-33, PM, Musket, Nagant,
Brian the Hero`.

**Do NOT add these — they are props, not enemies:** `Medkit Dispenser`,
`Armor Dispenser`, `Grenade Dispenser`, `Random Dispencer`, `Ammo Dispencer`. Their models
sit in the world with a `ClickDetector`, and adding one makes silent aim track a wall.

**Keep `Ignore Players` ON** in the NPC entry. Player characters are direct `workspace`
children named after the player and carry `NameTag` + `FPV_DynamicCam`; NPCs carry a
`Settings` folder instead. A `Contains` match on a short name can otherwise hit a player.

## Weapon settings are NumberValues, not GC tables

Probe 2026-10-27 found real writable instances:

```
<gun>.Settings.<Weapon>.Firerate      NumberValue  values seen: 0.1, 0.15, 0.2, 0.4, 1
<gun>.Settings.<Weapon>.ReloadDelay   NumberValue  value seen: 0.2
```

Paths hit: `Workspace.EnemyShotgun_Corpse.Settings.M4.Firerate`,
`Workspace.Tec9_Corpse.Settings.M4.Firerate`, `Workspace.PKM_Corpse.Settings.M4.Firerate`
(same for `ReloadDelay`). **This is a different engine family from Realistic Home Defense**,
where ACS wraps every settings table in `LockSettings`/`table.freeze` so GC writes are inert.
Here the values are plain `NumberValue` instances and can be written directly - no `getgc`,
no `applygc`, no unfreeze step.

**Important:** the probe hit only `*_Corpse` models, i.e. dead NPCs' guns. Writing those
does nothing for the player. The script therefore walks the **local character**
(`workspace[LocalPlayer.Name]`) for `Firerate` / `ReloadDelay` NumberValues.

Ammo field name is UNVERIFIED - no gun was equipped during probing (`CHARTOOLS n=0`).

## Where the NPCs actually are

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
under `Surroundings.People`.

Each NPC is also wrapped one level deep - `People.<Name>GP.<Name>` - and the wrapper has
no Humanoid. Draw the inner model, and drop a candidate whose parent is already a candidate
so the wrapper and body do not both box.

## Health is a sentinel, not a value

NPC and statue Humanoids report `5000000000 / 5000000000` (5 billion). That is the game's
"invincible / don't show HP" marker.
- a bar computed from it is always full and always green
- a label reading `5000000000/5000000000` is unreadable and covers the screen
- hide HP text and skip the bar when `MaxHealth >= 1e6`
- **Statues** (`Surroundings.Lobby.Model.Statue`) have a Humanoid at 5e9 too and must be
  excluded by ancestor (`Lobby`, `Statue`) or the ESP boxes scenery

## Player vs NPC

Players are told apart by marker, not name:
- players -> has `NameTag`, has `FPV_DynamicCam`, no `Settings`
- NPCs -> no `NameTag`, no `FPV_DynamicCam`, has `Settings`

So `isPlayer = hasChild('NameTag') or hasChild('FPV_DynamicCam')`. Name is never consulted,
because NPC names are gun names and collide with real player names.

Character models are DIRECT `workspace` children named after the player, ~20 per server
(`JuanPiripicho`, `O_CsF`, `FakeAimbotON`, `TwixieKitty7575`, ...), each `100/100`.
`LocalPlayer.Character` is intermittent here, so resolve the character by
`workspace[LocalPlayer.Name]` (probe: `CHAR true name=FakeAimbotON`).

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
- `ReplicatedStorage.Weapons` is EMPTY (guns clone in at runtime); the model pool is
  `ReplicatedStorage.ACS_Engine.GunModels` (41 instances, 14 unique names).
- Rooms: `Room1_19`, `Room1_11`, `Room1_7`, `Room1_17`, `Room1_16`, `Room1_B`, `RoomA_2`,
  `RoomA_5`, `RoomA_6`, `RoomA_7`; `ProximityPrompt` doors; `REVIVE` prompts on characters.
- `ReplicatedStorage.Characters` holds 20 character templates (`nobody`, `operator_alpha`,
  `chemist`, ...) which are NOT the gun-named combat NPCs.

## UNVERIFIED
- The ammo NumberValue name inside a live gun (no gun equipped while probing).
- Whether the server honours client-side `Firerate` / `ReloadDelay` writes, or recomputes
  them on its own tick and overwrites the client copy.
- Whether `GunModels` (14 unique) is the complete NPC name pool, or only the guns currently
  mounted; the live list had names (`Brian the Hero`, `Db`, `TT-33`, `Nagant`, `Musket`)
  that were not all present in that folder.

## Notes
- A nil `_G[NAME]` does not mean the script failed to load. Check `matcha_logs` for the
  `[NAME] stopped` line: that is my own `stop()` firing.
- `matcha_scripts` / `matcha_search` need `matcha_reindex` after joining.
- Printing many short strings in one `matcha_exec` garbles or truncates the log tail;
  a long `table.concat` also gets cut at roughly 100 characters. Print one or two values
  per call when the payload matters.
- A push can report a sha and size while the file is not there, or return a body that
  differs from what was sent. Always read the path back and compare.
