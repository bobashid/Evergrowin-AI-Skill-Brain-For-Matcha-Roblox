# Restaurant Tycoon 3

Place id **119048529960596**. `game.Name` reports **`Ugc`**, so Hub pins this place in `PLACES`
(done, Hub v3.0). Script: `Restaurant Tycoon 3` in `bobashid/roblox-matcha`.

## Two dead ends found by probing the live game (Confirmed 2026-10-27)

1. **`PlayerGui.CustomerSpeechUI.Adornee` reads NIL in this VM.** Probed 3 speech UIs: all
   returned `adornee=nil`, `Header.Text=nil`. The reference script derives customer ids from
   `Adornee`, so that whole path is unusable here. Use
   **`Tycoon.ClientCustomers`** instead: group **folder name = `GroupId`**, child **model name =
   `CustomerId`** (proven live: `Folder "14" -> Model "1", Model "2"`).
2. **A `HumanoidRootPart` reports `ClassName == 'Model'`** in this VM. Any "walk up until you
   find a Model" helper stops on the root part and returns the wrong instance - that is why the
   first Adornee-based attempt produced `groupId = "2"`.

## Why "everything looks on but nothing happens" (Confirmed 2026-10-27)

A live probe of an idle session returned:

    Surface = 10 tables   bills = 0   trash = 0   dishes = 0   TaskCompleted = RemoteEvent

So the collect loops were firing at **empty lists** every tick. Nothing errored, nothing printed,
and no toggle appeared broken. Any "collect" feature can only work once a customer sits, eats
and leaves a `Bill` / `Trash` behind.

Also settled in that probe: `TaskCompleted` is a plain **`RemoteEvent`** (one child, index 3).
An earlier read that showed `UnreplicatedEvent` alongside it was a garbled response, not real.

## Corrections from the working reference script (User-supplied 2026-10-27)

1. **NO `Interactions.Interacted` call is required.** Every automation is a single
   `TaskCompleted:FireServer({...})`. The "two-call" theory was wrong.
2. **Dirty dishes are `Name = 'CollectDishes'`** on an `Items.Surface` child holding a `Trash`
   child. NOT `GrabFood` on `Objects.Food`.
3. **Bills are `Name = 'CollectBill'`** on an `Items.Surface` child holding a `Bill` child.
4. **Serve reads its target from the food model itself** (`CustomerId` / `GroupId` attribute or
   Value child), falling back to the customer list.

## `TaskCompleted` task names (Confirmed from reference, 2026-10-27)

| Name | Payload | Notes |
|---|---|---|
| `SendToTable` | `FurnitureModel` = Surface table, `GroupId` | seat a waiting customer |
| `CollectBill` | `FurnitureModel` = Surface table with a `Bill` child | money from table |
| `CollectDishes` | `FurnitureModel` = Surface table with a `Trash` child | dirty dishes |
| `TakeOrder` | `GroupId`, `CustomerId` | plus the CustomerActionReplicated call |
| `Serve` | `FoodModel`, `CustomerId`, `GroupId` | NPC table service |
| `Serve` | `FoodModel`, `CarId` | drive-thru variant |
| `CollectDriveThruBill` | `CarId` | drive-thru money |

`GroupId` and `CustomerId` are **strings** (the captured seat payload used `GroupId = "19"`).

## Other remotes (Confirmed from reference, 2026-10-27)

| Remote | Call |
|---|---|
| `Restaurant.TipsCollected` | `FireServer(tycoon)` - the one feature that always worked |
| `Restaurant.CustomerActionReplicated` | `FireServer(tycoon, 'SetCustomerState', customerId, groupId, 'WaitingForDish')` - **positional** |
| `Restaurant.GrabFood` | `InvokeServer(foodModel)` |
| `Cook.CookInputRequested` | `FireServer('Interact', workspace, 'Oven')` |
| `Restaurant.GetRudeCustomerDialog` | `InvokeServer(tycoon, customerId, groupId)`, then again with the apology text, then `Dialog.DialogCompleted:FireServer()` |
| `Teleports.TeleportRequestedClient` | `Fire('Tycoon' / 'Plaza' / 'EKEA' / 'Campsite' / 'Farm')` |
| `Restaurant.RestaurantStatusUpdated` | `FireServer(tycoon, 'Open' / 'Closed' / 'Empty')` |
| `Rewards.RewardRequested` | `FireServer('Obejctives')` - the typo is in the game |
| `Shop.CodeSubmitted` | `FireServer('dineanddash')`, `FireServer('UltraWats')` |
| `Food.PurchaseIngredientRequested` | `FireServer(tycoon, ingredient, shopName)` |
| `Upgrades.UnlockUpgradeRequested` | `FireServer(tycoon, upgradeName)` - **excluded by user instruction** |

## Table selection (Confirmed from reference)

A `Items.Surface` child is free when it has **no `Bill`**, **no `Trash`**, none of the attributes
`Occupied` / `IsOccupied` / `InUse` / `Taken`, and no `Seat` descendant with an `Occupant`.
Table names match `^T%d+$`.

## Facts carried over from my own probes

- Ownership: `workspace.Tycoons` children, `ObjectValue` (`Owner` or `Player`) whose `Value` is
  the LocalPlayer. 8 tycoons, exactly one matches.
- Currency: `leaderstats` -> `IntValue` named `💸 Cash`; it rises on its own (passive income), so
  a rising balance is not evidence a script worked.
- Plot interactables: 9 total (1 `DoorPrompt`, 8 `SeatPrompt`). `Dialog` prompts in plot: **0**.
- Money drops: `workspace.DropFolder`; the reference teleports the HumanoidRootPart to each drop
  and back.
- `getgc('Serve')` returns 27 entries including `'Serve'`, `'Name'`, `'Key'`,
  `'IsHoldingFoodForNPC'`. A single `n=0` from `getgc` is a failed call, not proof of absence.

## Script versions

- v1.0.0 keyword prompt scan: no-op. v2.0.0/2.0.1 Dialog serve: fired world NPC dialogue.
- v3.0-v3.3 SeatPrompt misunderstanding, removed. v3.4 served the wrong dish.
- v3.5.0 held-dish tracking. v3.6.x seat via `SendToTable`.
- v3.7.0/3.7.1 wrongly added an `Interactions.Interacted` call and a range gate.
- v4.0.0: ported from the reference. **Died at load**: `Cannot use '...' outside of a vararg
  function` (a `...` inside a nested closure in `fire()`), so nothing ran.
- v4.0.1: fixed the vararg parse error with `packArgs` + `unpackFn(a, 1, a.n)`.
- v4.1.0: diagnostic build. `AutoRun` off by default, per-action OK/FAIL, remote class dump.
- v4.1.1: customers read from `ClientCustomers` (Adornee is nil), seat tries up to 4 free
  tables, counters only increment on a successful fire.

## Still unknown

1. Whether `SendToTable` needs the customer to be in a specific state first.
2. The music track names the server accepts.
3. Whether `TeleportRequestedClient` is a BindableEvent (`:Fire`) or a remote (`:FireServer`) -
   v4.1.x calls both inside pcall.
