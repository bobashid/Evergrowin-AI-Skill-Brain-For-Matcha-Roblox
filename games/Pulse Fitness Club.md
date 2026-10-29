# Pulse Fitness Club (place 85646026816836)

`game.Name` reports **`Ugc`** (the creator group), so the title comes from in-game signage
(`PULSE / FITNESS CLUB`, `TRAIN SMART`, `BOXING RING`). Do not name a file after `game.Name`.

- Script: `bobashid/roblox-matcha/Pulse Fitness Club`, current **v1.0**, 17074 bytes.
- Hub: pinned in `PLACES` as of Hub **v3.1**.
- Features: player ESP (box + name + distance), item ESP, loot container ESP.

## Confirmed (2026-10-29, live probes)

- **Players are `Workspace` Models named exactly `R15Rig`, carrying an attribute `userid`.**
  Read it with `model:GetAttribute('userid')`. The local player's id (2251313101) appears on one
  rig; a second rig carried 10194758356; a third carried **-1500000017**, i.e. NPC rigs use a
  negative id. That negative id is a reliable player/NPC split.
- `Players.GetPlayerByUserId` is **nil** in this build (`type(...) == 'nil'`), so names cannot be
  resolved that way. Build `[uid] = p.Name` from `Players:GetPlayers()` instead.
- The character model is **not** parented under the player and is not named after the player;
  it is a plain `R15Rig` in `Workspace`. `lp.Character` did return the rig
  (`Character=R15Rig parent=Workspace`) but must not be relied on for per-frame ESP.
- `Model:GetBoundingBox` and `Model:GetExtentsSize` are **nil** here (`type(...) == 'nil'`, call
  raises `attempt to call missing method`). Build the box from `Head` + `HumanoidRootPart`
  instead. Measured part sizes: Head `4.16 x 3.18 x 3.18`, HumanoidRootPart `1.76 x 2.0 x 0.8`.
- Items: `Workspace.GameSystem.WorldItem` — 42 children, a mix of `Tool` and `Model`. Each has a
  real `BasePart` (`Handle`, `Tape`, `tripo_node_...`), so `Position` is available.
- Loot containers: `Workspace.GameScene.Container` — 113 children.
- 200 `ProximityPrompt` instances exist, on `ContainerInteractionPoint`, `Back`, `Part`,
  `InteractPoint`, `LootContainer` parents (lockers, weapon cabinets, medical boxes, tripods).
- `MarketplaceService:GetProductInfo` is **missing** (`attempt to call missing method`), so the
  game's display title cannot be fetched from inside the VM.
- Real item names exist as `StringValue` values in the game: `Gym Membership Card`,
  `Large Dog Food`, `Dog Food Can`, `Dog Tag` — but **not** under the `WorldItem` children that
  were probed (those Tools report `kids=0`).

## UNVERIFIED

- Item display names: `WorldItem` children expose no `ItemName` descendant in the probes, so v1.0
  labels items with the part name or the instance name (`WorldItem_13`, `Tape`, ...). If the
  labels look wrong, probe for a `BillboardGui`/`TextLabel` on the tool or a `StringValue` one
  level higher, and fix `labelFor()`.
- Whether the server validates range for the container prompts. Not needed for the ESP, matters
  if an auto-loot is ever added — use `fireproximityprompt`, then read state back.

## Open questions

- The game's real title on the Roblox page (the API lookup was denied, so `Pulse Fitness Club`
  is taken from signage). If the user supplies the URL, pin the real name.
