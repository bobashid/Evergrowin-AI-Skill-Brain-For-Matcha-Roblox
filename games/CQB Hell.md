# CQB Hell

Place id: `98689921952741` (from `matcha_status`, 2026-10-27).
Script: `roblox-matcha/CQB Hell`, current version `v3.0` (weapon mods only).

## Enemy names - the authoritative list (59, from the user's own config)

Supersedes every earlier partial list. The user pasted the full JSON config (2026-10-27);
its `npc_order` had 59 real entries. Earlier "17 entries" and "cfg unreadable" notes were
wrong: `retrieve` failed on the attachment and only 5 of 134 chunks were ever served, which
is a tooling limit, not the file's contents.

```
416, AK-12, AK27, AKM, AN-94, AR, Brian the Hero, DB, DW-15, Deagle, Draco,
FAMAS, FM-9, FME, G17, G18, G3, Heavy2, KT12, Lee Enfield, M14, M202, M9,
MAT49, MD45, MP5, MPD45, Mosin, Musket, Nagant, PKM, PM, PPK, Phyton,
R-500S, R8-Shorty, RG-14, RG-20, RPK, STEN, SVD, Saiga, ScoutB, Serbu,
Shotgun, Slam, Smiley, Spectre, Standard4, TT-33, Tec9, USP, VAL, VSK,
VZ-16, VZ-82, VZ-94, VZS-74U, p226
```

So NPC names are gun-model names, not a small fixed set - 59 and counting. This is why a
hardcoded script list was never the right approach, and why the user's own config is the best
source: it is what he saw spawning.

## Six bugs found in that config (fixed version delivered)

1. Stray `""` entry: `match_mode: 0`, `match_string: ""`. Worst case - an empty Contains string
   can match **every** model including the local `Viewmodel`.
2. `VAL` had `match_mode: 1` and empty `match_string` -> matched nothing.
3. `VZ-16` had empty `match_string` -> matched nothing.
4. `p226` had `match_string` and `path` of `p226_Corpse` -> matched only dead bodies.
5. `MPD45` was the only entry with `enable_box: true, enable_distance: true,
   enable_skeleton: false` among 58 identical ones - leftover, not a preference.
6. `npc_order` did not equal the `npc_map` keys because of the `""` entry.

Left deliberately untouched: `DB` vs live `Db`, `DW-15` vs probed `GV-15` (possible
misspellings - both spellings would be safe to include), and `is_hostile: false` on all 59
(unknown semantics in Matcha; flipping blind could mark enemies as allies).

The config schema, entry template and validation checklist now live in `matcha-npc-cfg.md`.

## Weapon settings are NumberValues, not GC tables

```
<gun>.Settings.<Weapon>.Firerate      NumberValue  values seen: 0.1, 0.15, 0.2, 0.4, 1
<gun>.Settings.<Weapon>.ReloadDelay   NumberValue  value seen: 0.2
```

Different engine family from Realistic Home Defense, where ACS wraps every settings table in
`LockSettings`/`table.freeze` so GC writes are inert. Here they are plain `NumberValue`s - no
`getgc`, no `applygc`, no unfreeze step. v3.0 walks the LOCAL character
(`workspace[LocalPlayer.Name]`) so it never writes a corpse's gun.

UNVERIFIED: the ammo field name inside a live gun; whether the server honours client-side
`Firerate`/`ReloadDelay` writes or recomputes them on its own tick.

## Where the NPCs are in the world

Under a folder named `People`, but **that folder moves** - workspace root one probe,
`Surroundings.People` the next, and `workspace:FindFirstChild('People')` returned `false`
while three NPCs were alive. Find them by walking ANCESTORS for an instance named `People`,
never by a fixed path. Each NPC is wrapped `People.<Name>GP.<Name>` and the wrapper has no
Humanoid - draw the inner model and drop a candidate whose parent is already a candidate.

## Health is a sentinel, not a value

NPC and statue Humanoids report `5000000000 / 5000000000` - the game's "invincible" marker.
Hide HP text and skip the bar when `MaxHealth >= 1e6`. **Statues**
(`Surroundings.Lobby.Model.Statue`) share the sentinel and must be excluded by ancestor.

## Player vs NPC

- players -> has `NameTag`, has `FPV_DynamicCam`, no `Settings`
- NPCs -> no `NameTag`, no `FPV_DynamicCam`, has `Settings`

Name is never consulted; NPC names are gun names and collide with player names. Character
models are DIRECT `workspace` children named after the player, ~20 per server.
`LocalPlayer.Character` is intermittent here, so resolve by `workspace[LocalPlayer.Name]`.

## Names that are NOT enemies - never add these

`Viewmodel` (local player's own arms, has a Humanoid), `Dummy`, `DummyTwo`,
`Medkit Dispenser`, `Armor Dispenser`, `Grenade Dispenser`, `Random Dispencer`,
`Ammo Dispencer`.

## Other structure
- Corpses are `<Name>_Corpse` with `Humanoid.Health == 0`; 30 can be on the floor at once.
- Ammo pickups are `Part`s named `10 Ammo` at `workspace` root, 5 `Decal`s + `ProximityPrompt`
  + `RefilSound` + `Config` Script.
- Heal pickups clone from `ReplicatedStorage.Heal.HealModels`: 7 `MeshPart`s with prompts -
  `Bandage`, `Medkit`, `Big Medkit`, `Regen Stim`, `Health Stim`, `Combat Stim`, `Coffee`.
- Equipped weapons are `Tool` children of a character.
- Rooms: `Room1_19`, `Room1_11`, `Room1_7`, `Room1_17`, `Room1_16`, `Room1_B`, `RoomA_2`,
  `RoomA_5`, `RoomA_6`, `RoomA_7`; `ProximityPrompt` doors; `REVIVE` prompts on characters.

## UNVERIFIED
- The ammo NumberValue name inside a live gun (no gun equipped while probing).
- Whether the server honours client-side `Firerate` / `ReloadDelay` writes.
- Whether `SpecialWeapons` / `WeaponDrops` are genuinely empty (reads taken out of server).
- Whether the gun pool truly rotates per map or was just not streamed in.
- `ReplicatedStorage.PreMade` contents - never opened.
- What `Bots.AIG` / `Bots.AIS` are, and whether either returns a spawn table.
- Whether `ReplicatedStorage.Characters` template names (`capri`, `chemist`, ...) are ever
  used as bot display names - no live NPC has been seen using one.

## Notes
- A nil `_G[NAME]` does not mean the script failed to load. Check `matcha_logs` for the
  `[NAME] stopped` line: that is my own `stop()` firing.
- `matcha_scripts` / `matcha_search` need `matcha_reindex` after joining.
- Printing many short strings in one `matcha_exec` garbles or truncates the log tail; a long
  `table.concat` gets cut at roughly 100 characters. Print the count on its own line first.
- **A `matcha_exec` call can return a plausible-looking result that is NOT current.** Seen:
  `R5TO8 ...` placeholders, an earlier block echoed verbatim, and answers to a different
  question than asked. Treat a repeat, or a wrong-question answer, as a failed call.
- **A large attachment can be unreadable while `list_attachments` still lists it.** `retrieve`
  returned malformed citations for `CQB HELL.cfg` and only 5 of 134 chunks came back from
  `get_chunks` before the file stopped appearing. Asking the user to paste the JSON was what
  actually worked - do that earlier for a config of this size.
- A push can report a sha and size while the file is not there, or return a body that differs
  from what was sent. Always read the path back and compare.
