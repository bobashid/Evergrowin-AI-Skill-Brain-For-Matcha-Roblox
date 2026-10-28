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
- **No dish or money words in prompt names.** All 350 `ProximityPrompt`/`ClickDetector` in
  `workspace` listed: `RestaurantBoardPrompt`, `SeatPrompt`, `DoorPrompt`, `BuyPrompt`
  (cosmetics), `IngredientShopPrompt`, `GrowAllPrompt`, `Dialog`, `ChefLuigi`, `CreditsPrompt`,
  `EventHistoryMachinePrompt`, plain `ProximityPrompt`. Dish/money keyword scan matched **0**.
- **Almost no dish objects in the world.** `plate` -> 7 hits, all `TestNewPlate` unions;
  `dirty` -> **0**. Plates/trays on tables are `Attachment`s only.

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
you. See the hook limit below - there is no client-side route to a direct fire.

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

Module `Source.Modules.Tasks.ServeFood`. This is the step that converts held food into cash.
All three variants end in the **same remote**:

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
  earlier claim that the grab result tells you which serve variant to build is **not reliable**:
  the info table can be nil (no customer waiting for that dish at that moment). Any serve loop
  must handle `info == nil` and determine the target independently, from `Customers` / `DriveThru`
  / `Players` state rather than from the grab return.
- `tycoon.Objects.Food` had **5** children; the first child with no `Taken` attribute was
  `Objects.Food.3`.
- **`getconnections` and `hookfunction` are nil in this VM; `getgc` is a function.** Therefore
  `DropSpawned` cannot be hooked and drop ids cannot be intercepted client-side. Combined with
  the automatic proximity collection, a direct `PlayerCollectedDrops` fire is **not reachable**
  from a script, and is not needed.
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
2. **Let the game build the task itself:** serving is driven from an interaction on the customer
   (`Interactions.Interacted(tycoon, {Key = "Serve", Model = customerModel})`), and the prompt
   list already shows a `Dialog` prompt on customer HumanoidRootParts. Firing that prompt with
   `fireproximityprompt` makes the game construct the correct task, enum included.

Do not invent a value for `Task.Serve`. A wrong `Name` means `TaskCompleted` silently no-ops.

## Still unknown

1. The real value of `Task.Serve` (capture it, do not guess).
2. Where `TipsCollected` is fired and its argument shape.

## Consequence for script v1.0.0

v1.0.0 discovers prompts by name keyword and fires them. On this game that finds **nothing**,
so it is a no-op. Retire it; do not ship a keyword-guessing version.
