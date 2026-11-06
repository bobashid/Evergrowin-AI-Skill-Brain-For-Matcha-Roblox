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
- `RequestPickup` is fired with the loose berry Part itself.

## Both remotes are range-checked server-side (2026-11-05, user-stated + probe)

- User report: "not working for berry bushes not in range and when in range it picks up extremely slow". Firing `RequestHarvest` from 51 studs changed nothing (`berriesAfterFar` stayed at 12); firing it after moving onto the bush was the working case.
- Consequence: a "silent" cross-map farm is impossible for this game. The only honest route is to move the character to each bush / berry (warp via `HumanoidRootPart.CFrame`), fire, and warp back when idle - which is what `ABOVE` v5.x does.
- `RequestPickup` responds to repeated fires on the same part, so retrying a few times per item at close range is worth doing.

## Open questions (need a dump)

- Whether harvesting also needs an equipped tool plus `RequestSwing` on top of `RequestHarvest` (it has worked without one, so probably not).
- What `HB`'s `BushCell` value actually is - observed as a string like `3,0,7` / `1,0,4` / `12,0,8`.

## Blockers hit

- A `Workspace` scan for `Name == 'BerryBush'` returned nothing in one probe while `GetDescendants()` had just listed 63 of them - treat an empty node scan as a failed call and re-probe by island path before concluding anything. Same for `islands 0`: the island model is not always present (respawn / island load), so a probe that finds no island proves nothing about the game.
