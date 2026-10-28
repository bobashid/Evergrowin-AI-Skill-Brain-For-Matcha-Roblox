# Restaurant Tycoon 3

Place id **119048529960596**. `game.Name` reports **`Ugc`**, so Hub pins this place in `PLACES`
(done, Hub v3.0). Script: `Restaurant Tycoon 3` in `bobashid/roblox-matcha`.

## Corrections from the working reference script (User-supplied 2026-10-27)

The user supplied a complete working executor script ("Samuraa1 Hub", RT3 v2.0.0). It settles
four things I had wrong. Treat it as the highest-value dump so far.

1. **NO `Interactions.Interacted` call is required.** Every automation is a single
   `TaskCompleted:FireServer({...})`. v3.7.0's "two-call" theory was wrong - the tip jar worked
   because it is also one call, not because it was the only one-call action.
2. **Dirty dishes are `Name = 'CollectDishes'`**, fired on an `Items.Surface` child that holds a
   **`Trash`** child. It is NOT `GrabFood` on `Objects.Food`.
3. **Bills are `Name = 'CollectBill'`** on an `Items.Surface` child holding a **`Bill`** child.
   Same shape as above, up to 12 per tick.
4. **Serve reads its target from the food model itself:** `CustomerId` and `GroupId` come from
   the dish's own attribute or Value child, falling back to the `CustomerSpeechUI` list. The
   food model carries who ordered it - no guesswork, no round-robin needed.

## `TaskCompleted` task names (Confirmed from reference, 2026-10-27)

| Name | Payload | Notes |
|---|---|---|
| `SendToTable` | `FurnitureModel` = Surface table, `GroupId` | seat a waiting customer |
| `CollectBill` | `FurnitureModel` = Surface table with a `Bill` child | money from table |
| `CollectDishes` | `FurnitureModel` = Surface table with a `Trash` child | dirty dishes |
| `TakeOrder` | `GroupId`, `CustomerId` | plus the CustomerActionReplicated call below |
| `Serve` | `FoodModel`, `CustomerId`, `GroupId` | NPC table service |
| `Serve` | `FoodModel`, `CarId` | drive-thru variant |
| `CollectDriveThruBill` | `CarId` | drive-thru money |

## Other remotes (Confirmed from reference, 2026-10-27)

| Remote | Call |
|---|---|
| `Restaurant.TipsCollected` | `FireServer(tycoon)` |
| `Restaurant.CustomerActionReplicated` | `FireServer(tycoon, 'SetCustomerState', customerId, groupId, 'WaitingForDish')` - **positional, not a table** |
| `Restaurant.GrabFood` | `InvokeServer(foodModel)` |
| `Restaurant.CustomerActionReplicated` | also used for customer state changes |
| `Cook.CookInputRequested` | `FireServer('Interact', workspace, 'Oven')` |
| `Restaurant.GetRudeCustomerDialog` | `InvokeServer(tycoon, customerId, groupId)`, then a second `InvokeServer(..., apologyText)`, then `Dialog.DialogCompleted:FireServer()` |
| `Teleports.TeleportRequestedClient` | `Fire('Tycoon' / 'Plaza' / 'EKEA' / 'Campsite' / 'Farm')` |
| `Restaurant.RestaurantStatusUpdated` | `FireServer(tycoon, 'Open' / 'Closed' / 'Empty')` |
| `Restaurant.RestaurantNameUpdated` | `FireServer(tycoon, name)` |
| `Restaurant.SetMusicRequested` / `BuyMusicRequested` | `FireServer(tycoon, track)` / `FireServer(track)` |
| `DailyRewards.DailyRewardClaimed` | `FireServer()` |
| `Rewards.RewardRequested` | `FireServer('Obejctives')` - the typo is in the game |
| `Shop.CodeSubmitted` | `FireServer('dineanddash')`, `FireServer('UltrawEats')` |
| `Food.PurchaseIngredientRequested` | `FireServer(tycoon, ingredient, shopName)` where shopName is `Supermarket` / `Butcher` / `Fishmonger` / `Bakery` / `FarmShop` |
| `Farming.RequestCropPlant` / `RequestHarvest` / `CropRemoved` | `InvokeServer(crop, 'Tile1'..'Tile8')` / `InvokeServer('TileN')` / `FireServer('TileN')` |
| `Upgrades.UnlockUpgradeRequested` | `FireServer(tycoon, upgradeName)` - **excluded by user instruction** |

## Customer identification (Confirmed from reference, 2026-10-27)

Guests are found through **`PlayerGui.CustomerSpeechUI`**, not through `ClientCustomers`:

- `gui.Adornee` -> walk parents until `ClassName == 'Model'` -> that Model's `Name` is the
  **`CustomerId`**, and `Model.Parent.Name` is the **`GroupId`**.
- The GUI has a `Header` TextLabel. If its text contains **`"table for"`**, that customer is
  waiting to be SEATED; otherwise they are waiting to ORDER.
- This is what makes `SendToTable` and `TakeOrder` target the right guest without guessing.

`ClientCustomers` (folder name = GroupId, model name = CustomerId) still exists and is a valid
fallback, but the speech UI is what the reference actually uses.

## Table selection (Confirmed from reference)

A table is free when its `Items.Surface` child has **no `Bill`**, **no `Trash`**, none of the
attributes `Occupied` / `IsOccupied` / `InUse` / `Taken`, and no `Seat` descendant with an
`Occupant`. Table names match `^T%d+$`.

## Rate limiting in the reference

It uses a token bucket: at most **260 remote calls per 0.5 s** (`n2`), refreshed when
`os.clock()` crosses a 0.5 s boundary. Per-feature gaps: order 0.05 s, seat 0.06 s, give food
adaptive, rude 0.18 s, bills/dishes 0.1 s, tips 0.4 s. Copying the bucket shape is worth it;
the exact numbers are not sacred.

## Facts carried over from my own probes

- Ownership: `workspace.Tycoons` children, ObjectValue (`Owner` or `Player`) whose `Value` is the
  LocalPlayer. The reference checks `Owner` first then any ObjectValue, which is the robust form.
- Currency: `leaderstats` -> IntValue named `💸 Cash`; it rises on its own (passive income), so a
  rising balance is not evidence a script worked.
- Plot interactables: 9 total (1 `DoorPrompt`, 8 `SeatPrompt`). `Dialog` prompts in plot: **0**.
- Money drops: `workspace.DropFolder`; the reference TELEPORTS the HumanoidRootPart to each drop
  and back, which is the only way to reach drops outside the auto-collect radius.
- `getgc('Serve')` returns 27 entries including `'Serve'`, `'Name'`, `'Key'`, `'IsHoldingFoodForNPC'`.
  A single `n=0` from `getgc` is a failed call, not proof of absence.

## Script versions

- v1.0.0 keyword prompt scan: no-op. v2.0.0/2.0.1 Dialog serve: fired world NPC dialogue.
- v3.0-v3.3 SeatPrompt misunderstanding, removed. v3.4 served the wrong dish.
- v3.5.0 held-dish tracking. v3.6.x seat via `SendToTable`.
- v3.7.0/3.7.1 wrongly added an `Interactions.Interacted` call and a range gate.
- v4.0.0: ported from the reference. Adds `CollectDishes` (Trash child), `TakeOrder` +
  `CustomerActionReplicated`, serve targets taken from the food model's own ids, customer
  discovery via `CustomerSpeechUI`, cook via `CookInputRequested`, drive-thru, rude customers,
  drop-money teleport, teleports, restaurant open/close/empty, daily reward, objectives, codes.
  Upgrades excluded by instruction.

## Still unknown

1. The music track names the server accepts (`SetMusicRequested`), beyond the reference's list.
2. Whether `TeleportRequestedClient` is a BindableEvent (`:Fire`) or a remote (`:FireServer`) -
   the reference calls `:Fire`, v4.0.0 calls both inside pcall.
