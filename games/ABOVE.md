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

## Riding the cart: `Rider` attribute, not a Seat (2026-11-07, PROBED)

The user wanted the farm gated on "am I in the cart". There is **no Seat or VehicleSeat** in this game, so
none of the usual seat checks work.

- `<island>.Carts` holds `IronRailcart` (a `Model`, 12 children: `MeshPart`s, `WeldConstraint`s, an `HB` Part,
  and a `ProximityPrompt` **as a child of `HB`, not of the cart** - `cart:FindFirstChildOfClass("ProximityPrompt")`
  returns nil, you must look under `HB`).
- Boarding is `fireproximityprompt(cart.HB.ProximityPrompt)`. Measured: it moved the character from 9.6 studs
  to 1.4 studs from the cart, and **the cart gained the attribute `Rider = <UserId>`**.
- **`cart:GetAttribute("Rider") == Players.LocalPlayer.UserId` is the seated check.** Confirmed on the live
  session: `userId=2251313101`, `rider=2251313101`, `match=true`. Both are numbers.
- What does NOT work here: `Humanoid.Sit` reads nil, `Humanoid.SeatPart` reads nil, no `SeatWeld` is added to
  the character, `char.Parent` stays `Workspace`, and no Weld/Motor6D appears in the character's children.
  Distance to the cart is also useless on its own - standing next to it is 1.4 studs too.
- The cart did **not** move along rails during the probe (`cartMoved=0.00` over 1s), so a cart-based farm is a
  stationary-range farm, not a travelling one.

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
- User preference: the berry farmer should collect **only `ItemId == "Berries"`** (what a `BerryBush` yields), not `BerryPie` / `BerryJam` / `BerryBushSeed` / other berry-flavoured items. Use an exact id match, not a `"berr"` substring.

## CORRECTION (2026-11-07, PROBED): berries DO appear as loose items

The earlier note "no loose berry part appears, so berries go straight to inventory and there is nothing to pick up" was **wrong**. It was written from one sample taken before any berry had landed.

- `Workspace.<island>.Items` is the loose-item folder and it grows to **72 children**. `ItemId` values seen in it: `Berries`, `BerryPie`, `BerryJam`, `BerryBushSeed`, `Bread`, `Coal`, `Wheat`, `IronIngot`.
- A berry stack is a **Part** named `Berries` with `ItemId = "Berries"`, `EntityId = "item_<n>"`, and often `Count` on it or on its art children (`Count` values 2, 3, 4, 5, 8, 19 were all seen). `Count = nil` means one.
- Items **drift toward the player** - one item measured 120.9 studs away and 22.5 studs away minutes later with the player standing still. That is why item counts in `Items` drop on their own; do not read a falling count as your own pickup working.
- `Items` is also the cheap source of truth for what is loose: ~40-72 children, versus ~28 s for a `workspace:GetDescendants()` scan of this world (47992 instances). **Never scan the whole workspace here.**

## No ProximityPrompt on bushes or items (2026-11-07, PROBED)

- A full `workspace:GetDescendants()` sweep found **167 `ProximityPrompt` instances and every one is on a machine**: `WheatHarvester` x80, `Bakehouse` x24, `OutputChest` x16, `Toolsmith` x11, `CoalHarvesterI` x11, `IronHarvesterI` x4, `OverflowChest` x4, `Fabricator` x3, `Workbench` x2, `Furnace` x2, `WoodHarvesterI` x2, plus `BuilderBench`, `CookingStation`, `IndustrialFurnace`, `InputChest`, `Recycler`, `StoneHarvesterI`, `HUB.HB` and `IronRailcart.HB`.
- **Zero prompts sit under `Nodes` or `Items`**, so there is nothing to `fireproximityprompt` for a berry. The "prompt on the berry bush" the user reports is the game's own HUD hint driven by `TargetController.hasPickupTarget()`, not a DataModel prompt. Do not spend another session hunting for a prompt instance.
- `RequestSwing` with an **empty Backpack does nothing** (fired 3x at a ripe bush, `Ripe` stayed `true`). The game's `TargetController` maps a `BushCell` part to `"Axe"` or `"Pickaxe"`, so a legitimate harvest needs a tool in hand; the remote `RequestHarvest` is the only route that works bare.

## Verify every fire - `GAP = 0.05` is ~95%, not 100% (2026-11-07, measured + user report)

- Measured: **20 harvest fires 0.05 s apart -> 19 harvested**, and **30 at 0.02 s -> 26**. The drop is per-batch, not per-second.
- Marking a bush "done" the instant you fire it is therefore **wrong**: ~1 in 20 silently missed and never came back until the regrow cycle happened to reach it. That is exactly what "still happening but not that much anymore" looked like from the user's side.
- Correct shape: fire the cluster, `task.wait(0.3)`, then **re-read `HB.Ripe` for exactly the bushes you fired at**. Still `true` -> put it back in the queue and retry (cap at 3 attempts, then park it so one stubborn bush cannot spin). Flipped `false` -> done, and reset its attempt counter.
- Same rule for pickup: after `FireServer(entityId)`, check the Part is still parented; if it is, fire once more before moving on.
- This is the existing RM3 rule ("verify by reading state back, never by assuming") applied to a remote with a measured drop rate.

## Harvest mechanics measured (2026-11-07, PROBED - the numbers the collector is built on)

This is the section to read before touching any berry script. Every number below came from a
`matcha_exec` probe in the live game, and the two old scripts (`ABOVE` v5.1.0, `Silent Berries`
v4.0.0) failed on exactly these points.

- **The island is much bigger than the old notes say.** `Nodes` held **1281** children with
  **1036 `BerryBush`** and **1034 ripe at once**, and later **1401 `BerryBush`** as the island
  upgraded. Any design that handles one bush per tick is ~150x too slow.
- **The bush count CHANGES while you play.** 1036 -> 1328 -> 1401 in one session. A cached bush list must
  be rebuilt when the live count differs, or every bush added after load is invisible forever.
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
  per-batch one, and **0.05 s is the measured sweet spot** (~95-100%).
- **A warp needs a settle wait before firing, or the server still has your old position.**
  Warp 40 studs, `task.wait(0.30)`, then fire 12 at 0.05 s gaps -> **12/12 harvested**. Warp and
  fire in the same frame -> **0/1**. `Silent Berries` settled 0.06 s and `ABOVE` v5.1.0 did not
  settle at all; that is the second big reason they under-performed.
- The character **slides ~2-3 studs** during the settle wait (it is falling). Read the real
  `HumanoidRootPart.Position` AFTER the settle and use that as the cluster centre, not the
  intended warp point.
- Harvesting sets `HB.Ripe = false` and the `Berries` MeshParts' `Transparency` to 1.
- **Regrow DOES happen** - one cycle moved 12 bushes back to ripe, and a long silent run showed
  `regrown` repeatedly. It is slow enough that an idle-and-rescan design is correct, but a queue
  gated on "empty" will deadlock (see below).
- `leaderstats` holds only `Coins`. There is no client-readable berry count, so a harvest's
  payout cannot be measured from the client - only the `Ripe` flip can.
- `HB:GetAttributes()` can return the same attribute name **repeated** (BushCell / CropId / Ripe
  seven times each on one HB). Read by name with `GetAttribute(name)`; never treat the array as
  a map, and never assume one entry per attribute.
- The pickup input is a **held action**, not a one-shot: `TouchController` calls
  `InputController.setPickupHeld(true, "button")` and `TargetController.hasPickupTarget()` decides
  whether the pickup button shows. So the legitimate route is "hold pickup while a target is in
  reach"; the remote route above is the shortcut that skips the input layer entirely.

## The deadlock that froze v7.1.0 (2026-11-07, diagnosed from the user's own log line)

- The script printed `queued 1` while the island had **612 ripe bushes**. Cause: the regrow /
  re-check pass only ran when the work queue was **empty**, so a single bush it could not reach
  parked the queue at 1 forever, and the ~1030 bushes already marked done were never re-examined.
- Rule: **a periodic re-check must never be gated on another queue being empty.** Gate it on a
  timer instead. An idle-looking script that still reports counters is the signature - the
  counters kept rising while the work queue was stuck at 1.

## Open questions (need a dump)

- Whether `RequestHarvest` actually pays out berries or only clears `Ripe`. `RequestSwing` with an
  empty Backpack does nothing, and there is no client-readable berry count, so this needs the
  user to look at their inventory after a remote harvest.
- What `HB`'s `BushCell` value actually is - observed as a string like `3,0,7` / `1,0,4` / `12,0,8`. It is a grid cell, `cell * 3 = world position`.
- The exact regrow time. Not measured.
- Why a pickup fire is dropped at one distance and accepted at another. Suspected burst dropping rather than a radius, but not proven.

## Blockers hit

- A `Workspace` scan for `Name == 'BerryBush'` returned nothing in one probe while `GetDescendants()` had just listed 63 of them - treat an empty node scan as a failed call and re-probe by island path before concluding anything. Same for `islands 0`: the island model is not always present (respawn / island load), so a probe that finds no island proves nothing about the game.
- `task.wait` in a `matcha_exec` chunk longer than ~0.3 s closes the output capture, so a probe that fires then waits then reports loses its own result. Read `matcha_logs` for the tail instead of re-running the probe.
- A single sample of `island.Items` taken before anything had dropped led to the wrong conclusion that berries never drop. **A folder that is empty now is not a folder that stays empty** - re-check it after the action that is supposed to fill it, before writing the conclusion down.
- A `FindFirstChildOfClass("ProximityPrompt")` on the cart returned nil and briefly looked like "the cart has no prompt". The prompt was a child of `HB`. When a `FindFirstChildOfClass` misses, check the children one level down before concluding the instance does not exist.
