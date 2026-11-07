# ABOVE

Place id `124357835578554`. `game.Name` reports `Ugc` (the creator group), so match by place id, never by name - same trap as Merge Block Arena, Plunder PVP, Restaurant Tycoon 3 and Sell a Million Sushi.

## Confirmed (2026-11-05, PROBED)

- The world is per-player islands: `Workspace.Island_<id>:c1`, with children `Nodes`, `Blocks` and `HUB`.
- `Nodes` held 68 children in the sample: **63 `BerryBush`**, plus one each of `Tree`, `Stone`, `IronOre`, `Coal`, `Daisy Flower`. A later sample had **111 children and 106 `BerryBush`** - the count varies with island upgrades, so always enumerate, never hardcode.
- A `BerryBush` node is a Model with children `Berries` (Model), `HB` (Part - the hitbox) and `Icosphere` (MeshPart). `Berries` holds about 12 `MeshPart` children named `Icosphere.003`, positioned at the bush.
- Attribute NAMES seen: on `HB` -> `BushCell`, `CropId`, `Ripe`; on the fruit MeshParts -> `BushCell`, `BushFruitTransparency`, `AuthoredArt`, `AuthoredScale`. Values are read with `GetAttribute(name)`.
- **`GetAttributes()` returns an ARRAY of attribute-name strings in this VM, not a name-to-value map.** A `for k, v in pairs(x:GetAttributes())` loop printed `1  BushCell`, i.e. `k` was the index and `v` the name. Correct pattern: `for _, k in ipairs(x:GetAttributes()) do local v = x:GetAttribute(k) end`.
- `HB:Ripe` is a real boolean and is authoritative: one probe counted ripe vs unripe bushes and read `true` / `false` cleanly. `Ripe == true` on some bushes while others are `false` proves the flag works - an earlier note that treated it as unreliable was based on reading the `GetAttributes()` array wrongly.
- Only **3 ProximityPrompts in the whole place**: `Blocks.Workbench`, `Blocks.Furnace`, `HUB.HB`. Berry harvesting is NOT prompt-based.
- `ReplicatedStorage.Remotes` holds the whole network surface. Harvest-relevant RemoteEvents: `RequestHarvest`, `RequestPickup`, `RequestDrop`, `RequestSwing`, `RequestUseItem`, `RequestPollinate`, `RequestEquipSlot`, `RequestSetEquipment`, `RequestConsume`, `HitConfirmed`, `ItemGained`, `ItemDropped`, `BlockBroken`, `InventoryChanged`, `RequestSync`, `RequestCollectionTurnIn`. Storage: `RequestStorageStore`, `RequestStorageWithdraw`, `RequestStorageWithdrawAll`. Machines: `RequestLoadFuel`, `RequestEmptyMachine`, `RequestStopCrafting`, `RequestCraft`, `RequestHandCraft`, `MachineUpdate`. Shop: `RequestShopOpen`, `RequestShopBuy`, `RequestShopSell`. Only one RemoteFunction: `RequestAdminCoins`.
- **`RequestHarvest` takes the bush's `HB` Part as its single argument.** User-stated and used live: `game.ReplicatedStorage.Remotes.RequestHarvest:FireServer(game.Workspace["Island_2251313101:c1"].Nodes.BerryBush.HB)`. No tool, no cell, no Vector3.
- `ReplicatedStorage.Shared.Config.CropConfig` is a ModuleScript - the per-crop data (ripe timing, yield) lives there.
- Prefabs: `Prefabs.Bushes.BerryBush`, `Prefabs.Resources.Berries` / `JumboBerry` / `BerryJam` / `BerryPie` / `PinkberryPie`, `Prefabs.Tools.WoodPickaxe` / `StonePickaxe` / `IronPickaxe`.

## Loose berries: identify by the `Count` attribute, never by name (2026-11-05, PROBED + decompile)

- The game's own `PlayerScripts.Client.Controllers.LooseCountController` finds loose items with **`CollectionService:GetTagged("LooseItem")`** and reads **`part:GetAttribute("Count")`** (numeric, >1 shows an `xN` billboard). It also filters `part:IsA("BasePart")`.
- **`CollectionService:GetTagged` / `HasTag` / `GetInstanceAddedSignal` do not exist in this VM** (see `learned/api-facts.md`), so the tag route is closed here. Use the attribute instead: a loose berry is a BasePart whose `GetAttribute("Count")` is a number >= 1.
- **Name matching alone produces false positives.** `Workspace.Hub.Decoration.Rock 2` holds dozens of MeshParts named `Icosphere` / `Icosphere.001` at 92-130 studs from spawn - they match any `icosphere` filter and have no `Count`. Reject anything under `Hub`, `Decoration`, `Nodes`, `PlacementGhost` or `Prefabs`.

## CORRECTION (2026-11-07, PROBED): `RequestPickup` takes the `EntityId` STRING, not the Part

This section replaces the old line "`RequestPickup` is fired with the loose berry Part itself", which was **wrong** and cost a whole round trip of "the auto pickup does nothing".

- **`Remotes.RequestPickup:FireServer(<item>:GetAttribute("EntityId"))`** - a string like `"item_4711600"`. Firing it with the Part instance is **accepted and silently ignored**: the Part stays parented, no error, no state change. Measured on one berry Part: fired 5 times with the Part -> `alive=true`; the very next attempt with the EntityId string -> `alive=false`.
- Shape test on one target, 3 fires each, 0.08 s apart: `FireServer()` -> alive, `FireServer(part)` -> alive, **`FireServer(entityId)` -> collected**. `FireServer(part, 1)`, `FireServer(model)` and `FireServer(child)` were not needed once the EntityId form worked.
- **Pickup fires get dropped in bursts too**, like `RequestHarvest`: 4 fires 0.08 s apart collected 3. Use a gap and let the loop retry survivors next tick rather than firing once and assuming success.
- **Distance does not look like the gate.** Collected at 81 studs with no warp; did not collect at 120.9 or 22.5. That pattern is not a radius, so treat pickup as *not* range-checked and retry rather than warp. Consequence: **auto pickup needs no teleport at all** - fire the EntityId from wherever you stand.
- `RequestPickup` also collects the crafted berry items (`BerryPie`, `BerryJam`) sitting in the same folder, not just `Berries`.
- A pickup item can be a **Part with a nested `Model` child** (`Items.Berries` is a Part whose first child is a Model also named `Berries`). Fire at the Part's `EntityId`; do not walk into the child.

## CORRECTION (2026-11-07, PROBED): berries DO appear as loose items

The earlier note "no loose berry part appears, so berries go straight to inventory and there is nothing to pick up" was **wrong**. It was written from one sample taken before any berry had landed.

- `Workspace.<island>.Items` is the loose-item folder and it grows to **72 children**. `ItemId` values seen in it: `Berries`, `BerryPie`, `BerryJam`, `BerryBushSeed`, `Bread`, `Coal`, `Wheat`, `IronIngot`.
- A berry stack is a **Part** named `Berries` with `ItemId = "Berries"`, `EntityId = "item_<n>"`, and often `Count` on it or on its art children (`Count` values 2, 3, 4, 5, 8, 19 were all seen). `Count = nil` means one.
- Items **drift toward the player** - one item measured 120.9 studs away and 22.5 studs away minutes later with the player standing still. That is why item counts in `Items` drop on their own; do not read a falling count as your own pickup working.
- `Items` is also the cheap source of truth for what is loose: ~40-72 children, versus ~28 s for a `workspace:GetDescendants()` scan of this world (47728 instances). **Never scan the whole workspace here.**

## Harvest mechanics measured (2026-11-07, PROBED - the numbers the collector is built on)

This is the section to read before touching any berry script. Every number below came from a
`matcha_exec` probe in the live game, and the two old scripts (`ABOVE` v5.1.0, `Silent Berries`
v4.0.0) failed on exactly these points.

- **The island is much bigger than the old notes say.** `Nodes` held **1281** children with
  **1036 `BerryBush`** and **1034 ripe at once** in this session. Any design that handles one
  bush per tick is ~150x too slow; that is what "isn't very functional" meant.
- **`BushCell` is a grid coordinate and `BushCell * 3` is the bush's world position.** Cell
  `-4,2,0` -> `-12, 206.5, 3000`. The bushes sit on a **3-stud grid**, so a 17-stud radius
  covers roughly 100 bushes and one warp can harvest a whole cluster. Bushes never move, so
  cache positions; only `Ripe` needs re-reading.
- **`RequestHarvest` reach is 18 studs** measured from `HumanoidRootPart` (3D magnitude; the
  game's own `Shared.Config.Player.InteractReachStuds` is what the client compares against).
  Measured: 16.84 and 17.54 harvested; **18.07 and 18.75 did not**, silently - no error, no
  attribute change. A radius of 17 is safe.
- **The server drops bursts of fires.** Same standing position, same range, every fire aimed at
  a different ripe bush in range: **92 fired in one frame -> 22 harvested**; **20 fired 0.05 s
  apart -> 19**; **30 fired 0.02 s apart -> 26**. So it is not a per-second cap, it is a
  per-batch one, and **0.05 s is the measured sweet spot** (~95-100%). Firing the whole cluster
  in one frame is the single biggest reason a cluster-based collector looks broken.
- **A warp needs a settle wait before firing, or the server still has your old position.**
  Warp 40 studs, `task.wait(0.30)`, then fire 12 at 0.05 s gaps -> **12/12 harvested**. Warp and
  fire in the same frame -> **0/1**. `Silent Berries` settled 0.06 s and `ABOVE` v5.1.0 did not
  settle at all; that is the second big reason they under-performed.
- The character **slides ~2-3 studs** during the settle wait (it is falling). Read the real
  `HumanoidRootPart.Position` AFTER the settle and use that as the cluster centre, not the
  intended warp point.
- Harvesting sets `HB.Ripe = false` and the `Berries` MeshParts' `Transparency` to 1.
- **No regrow observed** within ~10 minutes of harvesting; harvested bushes stayed `Ripe = false`
  with `Transparency = 1`. Idle-and-rescan is the correct behaviour, not a tight retry.
- `leaderstats` holds only `Coins`. There is no client-readable berry count.
- `HB:GetAttributes()` can return the same attribute name **repeated** (BushCell / CropId / Ripe
  seven times each on one HB). Read by name with `GetAttribute(name)`; never treat the array as
  a map, and never assume one entry per attribute.
- The pickup input is a **held action**, not a one-shot: `TouchController` calls
  `InputController.setPickupHeld(true, "button")` and `TargetController.hasPickupTarget()` decides
  whether the pickup button shows. So the legitimate route is "hold pickup while a target is in
  reach"; the remote route above is the shortcut that skips the input layer entirely.

## Open questions (need a dump)

- Whether harvesting also needs an equipped tool plus `RequestSwing` on top of `RequestHarvest` (it has worked without one, so probably not). The game's own `TargetController` maps a `BushCell` part to `"Axe"` or `"Pickaxe"`, yet a bare `RequestHarvest` from a player with an empty Backpack was accepted - so the server does not enforce the tool.
- What `HB`'s `BushCell` value actually is - observed as a string like `3,0,7` / `1,0,4` / `12,0,8`. It is a grid cell, `cell * 3 = world position`.
- The exact regrow time. Not observed in ~10 minutes.
- Why a pickup fire is dropped at one distance and accepted at another. Suspected burst dropping rather than a radius, but not proven.

## Blockers hit

- A `Workspace` scan for `Name == 'BerryBush'` returned nothing in one probe while `GetDescendants()` had just listed 63 of them - treat an empty node scan as a failed call and re-probe by island path before concluding anything. Same for `islands 0`: the island model is not always present (respawn / island load), so a probe that finds no island proves nothing about the game.
- `task.wait` in a `matcha_exec` chunk longer than ~0.3 s closes the output capture, so a probe that fires then waits then reports loses its own result. Read `matcha_logs` for the tail instead of re-running the probe.
- A single sample of `island.Items` taken before anything had dropped led to the wrong conclusion that berries never drop. **A folder that is empty now is not a folder that stays empty** - re-check it after the action that is supposed to fill it, before writing the conclusion down.
