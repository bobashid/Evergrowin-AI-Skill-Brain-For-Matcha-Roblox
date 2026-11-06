# ABOVE

Place id `124357835578554`. `game.Name` reports `Ugc` (the creator group), so match by place id, never by name - same trap as Merge Block Arena, Plunder PVP, Restaurant Tycoon 3 and Sell a Million Sushi.

## Confirmed (2026-11-05, PROBED)

- The world is per-player islands: `Workspace.Island_<id>:c1`, with children `Nodes`, `Blocks` and `HUB`.
- `Nodes` held 68 children in the sample: **63 `BerryBush`**, plus one each of `Tree`, `Stone`, `IronOre`, `Coal`, `Daisy Flower`.
- A `BerryBush` node is a Model with children `Berries` (Model), `HB` (Part - the hitbox) and `Icosphere` (MeshPart). `Berries` holds about 12 `MeshPart` children named `Icosphere.003`, positioned at the bush.
- Attribute NAMES seen: on `HB` -> `BushCell`, `CropId`, `Ripe`; on the fruit MeshParts -> `BushCell`, `BushFruitTransparency`, `AuthoredArt`, `AuthoredScale`. Values are read with `GetAttribute(name)`.
- **`GetAttributes()` returns an ARRAY of attribute-name strings in this VM, not a name-to-value map.** A `for k, v in pairs(x:GetAttributes())` loop printed `1  BushCell`, i.e. `k` was the index and `v` the name. Correct pattern: `for _, k in ipairs(x:GetAttributes()) do local v = x:GetAttribute(k) end`.
- Only **3 ProximityPrompts in the whole place**: `Blocks.Workbench`, `Blocks.Furnace`, `HUB.HB`. Berry harvesting is NOT prompt-based.
- `ReplicatedStorage.Remotes` holds the whole network surface. Harvest-relevant RemoteEvents: `RequestHarvest`, `RequestPickup`, `RequestDrop`, `RequestSwing`, `RequestUseItem`, `RequestPollinate`, `RequestEquipSlot`, `RequestSetEquipment`, `RequestConsume`, `HitConfirmed`, `ItemGained`, `ItemDropped`, `BlockBroken`, `InventoryChanged`, `RequestSync`, `RequestCollectionTurnIn`. Storage: `RequestStorageStore`, `RequestStorageWithdraw`, `RequestStorageWithdrawAll`. Machines: `RequestLoadFuel`, `RequestEmptyMachine`, `RequestStopCrafting`, `RequestCraft`, `RequestHandCraft`, `MachineUpdate`. Shop: `RequestShopOpen`, `RequestShopBuy`, `RequestShopSell`. Only one RemoteFunction: `RequestAdminCoins`.
- `ReplicatedStorage.Shared.Config.CropConfig` is a ModuleScript - the per-crop data (ripe timing, yield) lives there.
- Prefabs: `Prefabs.Bushes.BerryBush`, `Prefabs.Resources.Berries` / `JumboBerry` / `BerryJam` / `BerryPie` / `PinkberryPie`, `Prefabs.Tools.WoodPickaxe` / `StonePickaxe` / `IronPickaxe`.

## Open questions (need a dump)

- **`RequestHarvest` argument shape** - cell coordinates, a Vector3, or the node itself? Needs the client script that fires it.
- Whether harvesting needs an equipped tool plus `RequestSwing` (hit-based) instead of a direct `RequestHarvest` call.
- What `HB`'s `BushCell` value actually is (number, Vector2, string) and whether `Ripe` is the authoritative ready flag.

## Blockers hit

- A `Workspace` scan for `Name == 'BerryBush'` returned nothing in one probe while `GetDescendants()` had just listed 63 of them - treat an empty node scan as a failed call and re-probe by island path before concluding anything.
