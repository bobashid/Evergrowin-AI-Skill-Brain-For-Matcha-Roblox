# Restaurant Tycoon 3

Place id **119048529960596**. `game.Name` reports **`Ugc`** (the creator group's name), so
name matching in Hub cannot find this place - it must be pinned in `PLACES`. Done in Hub v3.0.

Script: `Restaurant Tycoon 3` in `bobashid/roblox-matcha`.

## Confirmed (2026-10-27, live probes via matcha_exec)

- **Ownership is an ObjectValue, not an attribute.** `workspace.Tycoons` holds 8 children all
  named `Folder "Tycoon"`; `GetAttribute('Owner')` returns **nil** on every one. Each has a
  child `ObjectValue` named `Player`, so the owner test is `plot.Player.Value == LocalPlayer`.
- **Currency:** `LocalPlayer.leaderstats` -> `IntValue` named `💸 Cash` (emoji + space).
  Read 124, then 285 minutes later with no manual collecting: the tycoon has passive income.
  A rising balance is therefore NOT proof a script worked. Measure drops disappearing.
- **Tycoon layout:** `<Tycoon>` -> `Locations` (6 CFrameValues), `Player` (ObjectValue),
  `Objects` (Folder: `Food`, `Advertising`, `CookingResources`), `GreyBox`, `Default`, `Items`.
  `Items` has 7 children: BeingPlaced, Furniture (19), Surface (10), Walls (42), Floor (9),
  Config (2), Stairs (0).
- **Cash piles under `Items.Furniture.K32.Cash` are DECORATION, not collectibles.** 16 child
  Models (`Cash1`..`Cash16`), 2 `MeshPart`s each, and inside that subtree: **0
  `TouchTransmitter`, 0 `ProximityPrompt`, 0 `ClickDetector`, 0 `Attachment`**. Real money is
  the drop system below. This is the trap that made v1.0.0 a no-op.
- **Almost no dish objects in the world.** `plate` -> 7 hits, all `TestNewPlate` unions;
  `dirty` -> **0**. Plates/trays on tables are `Attachment`s only. Food models in
  `Objects.Food` are named numerically (seen: `225`), and `Taken` reads **nil** on fresh ones.

## Prompt inventory (Confirmed 2026-10-27, whole-workspace scan)

Only **6** prompts exist for the actions that matter, out of ~350 total:

| Prompt | Count | Parent | Has NPC Humanoid |
|---|---|---|---|
| `IngredientShopPrompt` | 1 | `HumanoidRootPart` | no |
| `GrowAllPrompt` | 1 | `Sign` | no |
| `Dialog` | 4 | `HumanoidRootPart` | **only 1 of the 4** |

Consequence: firing every `Dialog` prompt also pokes three non-NPC ones, which belong to other
UI/NPC flows. A serve loop must filter `Dialog` by "parent's parent has a `Humanoid`" (or by
customer state) before firing, not fire all four.

`workspace.EatingFoods` existed but held **0** children at probe time.

## Remotes (Confirmed 2026-10-27, name scan of ReplicatedStorage)

| Remote | Class | Parent |
|---|---|---|
| `GrabFood` | RemoteFunction | `Restaurant` |
| `GrabEnded` | RemoteEvent | `Restaurant` |
| `CancelGrabRequested` | RemoteEvent | `Restaurant` |
| `TaskCompleted` | RemoteEvent | `Restaurant` |
| `TipsCollected` | RemoteEvent | `Restaurant` |
| `PlayerCollectedDrops` | **UnreliableRemoteEvent** | `Currency` |
| `DropSpawned` / `DropsDespawned` | **UnreliableRemoteEvent** | `Currency` |
| `CollectOfflineEarningsRequested` | RemoteEvent | `Upgrades` |
| `FoodConsumed` / `FoodOrdered` / `FoodInspected` | RemoteEvent | `Food` |
| `ItemCollected` | RemoteEvent | `ScavengerHunt` |

## Money drops - full mechanism (Confirmed 2026-10-27 by decompile)

Module `PlayerScripts.Source.Systems.Currency.CurrencyDrops`.

- Drops are cloned from `ReplicatedStorage.Assets.Models.Currency.DropTemplate` into a folder
  **`workspace.DropFolder`**, not under the tycoon.
- Ids come from the **server**: `Events.Currency.DropSpawned.OnClientEvent(_, dropId, data)`.
- **Collection is proximity-based and automatic.** A `Heartbeat` loop tests
  `MagnitudeSquared(drop.Position, HumanoidRootPart.Position) <= (7 + SizeModifier)^2`, anchors
  the drop, Lerps it to the player over ~1 s, then collects it. No click, no prompt, no key.
- **The remote call:** ids are batched and one deferred call does
  `Events.Currency.PlayerCollectedDrops:FireServer({id1, id2, ...})` - a single array argument.
- Drops self-despawn after 120 s.

**Consequence: drop collection needs no script at all.** The game's own radius test collects for
you. `getconnections` and `hookfunction` are nil in this VM, so ids cannot be intercepted and a
direct fire is not reachable either.

## Grabbing food - full mechanism (Confirmed 2026-10-27 by decompile)

Module `Source.Systems.Restaurant.GrabFood`.

- **`local ok, info = Events.Restaurant.GrabFood:InvokeServer(foodModel)`** - the food Model is
  the only argument.
- Client preconditions, all replicable: `Managers:HasPermission(tycoon)`;
  `Cook:PlayerIsCooking()` false; `food:GetAttribute('Taken')` falsy; every item already held
  must belong to the same tycoon.
- **Hold capacity** = `Upgrades:Has(WaiterCart) and 4 or Upgrades:Has(WaiterTray) and 3 or 1`.
- Candidate food = children of `tycoon.Objects.Food` with no `Taken` attribute, within
  **manhattan distance `|dx| + |dz| < 4`** of the interaction point.
- `GrabEnded.OnClientEvent(plate)` removes the plate from storage; `CancelGrabRequested:FireServer()` cancels.

## Serving food - full mechanism (Confirmed 2026-10-27 by decompile)

Module `Source.Modules.Tasks.ServeFood`. All three variants end in the same remote:

    Events.Restaurant.TaskCompleted:FireServer(task)

where `task` always contains `Name = <Task enum>.Serve`, `Tycoon = tycoon`, `FoodModel = model`,
plus exactly one target discriminator:

| Variant | Extra fields | Precondition the game checks |
|---|---|---|
| Player | `Player = <Player object>` | `GrabFood:IsHoldingFoodForPlayer(player.Name)` |
| NPC | `GroupId = <id>`, `CustomerId = <id>` | `Customers:GetCustomerData(tycoon, groupId, customerId).State == CustomerState.WaitingForDish` **and** `IsHoldingFoodForNPC(order, isKid)` |
| Car (drive-thru) | `CarId = <carModel.Name>` | `DriveThru:GetCarData(tycoon, name)` and some order in `carData.Orders` matching `IsHoldingFoodForCar(order)` |

- Client dispatch is in `Tasks.TaskInputReceived`: branch on `task.CurrentTask.Name == Task.Serve`,
  then `CarModel` -> `CompleteForCar`, `Player` -> `CompleteForPlayer`, else `GroupId` + `CustomerId`
  -> `CompleteForNPC`.
- **There is a 0.25 s debounce** (`v_u_12`, set true then cleared by `task.delay(0.25)`). Replicate
  it or serve calls spam.

## Probe results (Confirmed 2026-10-27, matcha_exec)

- **`GrabFood:InvokeServer(foodModel)` works from a script.** Called on
  `Workspace.Tycoons.Tycoon.Objects.Food.3` and returned **`true`**.
- **It returned only `true` - no `{TargetPlayerName, TargetCarId, KidFlag}` table.** So the
  claim that the grab result tells you which serve variant to build is **not reliable**:
  the info table can be nil. A serve loop must determine its target independently, from
  `Customers` / `DriveThru` / `Players` state rather than from the grab return.
- `tycoon.Objects.Food` held **5** children at one probe and **2** at a later one (models named
  `225`), both with `Taken == nil`.
- **`getconnections` and `hookfunction` are nil in this VM; `getgc` is a function.** Therefore
  `DropSpawned` cannot be hooked and drop ids cannot be intercepted client-side.
- `workspace.DropFolder` existed but held **0** children at probe time.
- `Events.Currency` children confirmed as `DropSpawned`, `DropsDespawned`,
  `PlayerCollectedDrops`, all **`UnreliableRemoteEvent`**.

## The one missing constant (Confirmed 2026-10-27)

`require(ReplicatedStorage.Source.Enums.Restaurant.Task)` decompiles to exactly:

    return {}

So `Task.Serve` cannot be read statically - the enum is populated at runtime, or the decompiler
dropped it. A script therefore cannot write `Name = Task.Serve` from source knowledge alone.

Two zero-guess ways to obtain the real value:

1. **Remote Spy** (Matcha Pro built-in): serve one customer by hand and read the actual
   `TaskCompleted` call - it prints `Name` exactly.
2. **Let the game build the task itself:** firing the customer's own `Dialog` prompt makes the
   game construct the correct task, enum included.

Do not invent a value for `Task.Serve`. A wrong `Name` means `TaskCompleted` silently no-ops.

## Still unknown

1. The real value of `Task.Serve` (capture it, do not guess).
2. Where `TipsCollected` is fired and its argument shape.
3. Whether firing a customer `Dialog` prompt actually starts serving, or only opens dialogue.
   This is the single assumption in script v2.0.0 and it is **UNVERIFIED**.

## Script versions

- v1.0.0: keyword prompt scan. **No-op on this game** - retire it.
- v2.0.0: `GrabFood:InvokeServer` for grabbing (proven), `fireproximityprompt` on `Dialog`
  prompts for serving (unverified), `GrowAllPrompt` and `IngredientShopPrompt` for grow/shop.
