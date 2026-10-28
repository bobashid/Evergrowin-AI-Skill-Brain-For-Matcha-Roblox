# Restaurant Tycoon 3

Place id **119048529960596**. `game.Name` reports **`Ugc`**, so Hub must pin this place in
`PLACES` (done, Hub v3.0).

Script: `Restaurant Tycoon 3` in `bobashid/roblox-matcha`.

## Confirmed (2026-10-27, live probes via matcha_exec)

- **Ownership is an ObjectValue, not an attribute.** `workspace.Tycoons` holds 8 children all
  named `Tycoon`; `GetAttribute('Owner')` is nil on every one. Owner test:
  `plot.Player.Value == LocalPlayer`.
- **Currency:** `LocalPlayer.leaderstats` -> `IntValue` named `💸 Cash` (emoji + space). Read
  124 then 285 with no manual collecting: **passive income exists**, so a rising balance is
  not proof a script worked. Measure objects disappearing.
- **Tycoon layout:** `Locations` (6 CFrameValues), `Player` (ObjectValue), `Objects` (Food,
  Advertising, CookingResources), `GreyBox`, `Default`, `Items`.
  `Items` = BeingPlaced, Furniture (19), Surface (10), Walls (42), Floor (9), Config (2), Stairs (0).
- **`Items.Furniture.K32.Cash` is DECORATION.** 16 child Models, 2 MeshParts each, and in that
  subtree: 0 TouchTransmitter, 0 ProximityPrompt, 0 ClickDetector, 0 Attachment. Not money.
- Food models live in `Objects.Food`, named numerically (`225`), `Taken` reads nil when fresh.

## My plot's complete interactable inventory (Confirmed 2026-10-27)

`Items:GetDescendants()` scanned for ProximityPrompt / ClickDetector / TouchTransmitter:

    INTERACT_IN_ITEMS = 9

| What | Count | Path shape |
|---|---|---|
| `DoorPrompt` | 1 | `Items.Furniture.D20.Base.PromptAttachment.DoorPrompt` |
| `SeatPrompt` | 8 | `Items.Furniture.C23/C1.Base.SeatAttachment.SeatPrompt` |

**Nothing else in the plot is interactable.** Specifically:

- **Tip jar** `Items.Furniture.K32.Jar` is a `MeshPart` whose only child is a
  `BillboardGui` named `TipsAmount`. No prompt, no ClickDetector, no TouchTransmitter.
- **Table bill** `Items.Surface.T19.Bill` is a `Model` with 4 children (3 `Part` + `Part Base`).
  No prompt, no ClickDetector, no TouchTransmitter.

**`Dialog` prompts inside my plot: 0.** (`DIALOG_IN_MY_PLOT 0`.) Every `Dialog` in the place
belongs to world NPCs, which is why firing them opened WaveyDavey's dialogue and why an
in-plot-only scan can never take an order.

**Consequence:** "auto take order", "auto pickup tips from jar" and "auto pickup money/check
from table" have **no client-side interaction object**. They are UI or remote driven and their
entry points are not yet identified. Do not ship another guess at them.

## Remotes (Confirmed 2026-10-27)

| Remote | Class | Parent |
|---|---|---|
| `GrabFood` | RemoteFunction | `Restaurant` |
| `GrabEnded` | RemoteEvent | `Restaurant` |
| `CancelGrabRequested` | RemoteEvent | `Restaurant` |
| `TaskCompleted` | RemoteEvent | `Restaurant` |
| `TipsCollected` | RemoteEvent | `Restaurant` |
| `PlayerCollectedDrops` | UnreliableRemoteEvent | `Currency` |
| `DropSpawned` / `DropsDespawned` | UnreliableRemoteEvent | `Currency` |
| `CollectOfflineEarningsRequested` | RemoteEvent | `Upgrades` |
| `FoodConsumed` / `FoodOrdered` / `FoodInspected` | RemoteEvent | `Food` |
| `ItemCollected` | RemoteEvent | `ScavengerHunt` |

`TipsCollected` is the most likely route for the jar and `TaskCompleted` for the bill/order,
but **neither argument shape has been observed**, so neither is usable yet.

## Money drops (Confirmed 2026-10-27 by decompile of `CurrencyDrops`)

- Drops clone from `Assets.Models.Currency.DropTemplate` into **`workspace.DropFolder`**.
- Ids arrive from the server: `Events.Currency.DropSpawned.OnClientEvent(_, dropId, data)`.
- **Collection is automatic by proximity:** Heartbeat test
  `MagnitudeSquared(drop, HumanoidRootPart) <= (7 + SizeModifier)^2`, then a 1 s Lerp pull and
  a batched `PlayerCollectedDrops:FireServer({ids})` (single array argument). 120 s despawn.
- `getconnections` and `hookfunction` are **nil** in this VM (`getgc` is a function), so ids
  cannot be intercepted and a direct fire is unreachable - and unnecessary.

## Grabbing food (Confirmed 2026-10-27)

- `local ok, info = Events.Restaurant.GrabFood:InvokeServer(foodModel)`; food Model is the
  only argument. Preconditions: `Managers:HasPermission(tycoon)`, `Cook:PlayerIsCooking()`
  false, `food:GetAttribute('Taken')` falsy, all held items from the same tycoon.
- Hold capacity = `WaiterCart and 4 or WaiterTray and 3 or 1`.
- Candidates = `tycoon.Objects.Food` children with no `Taken`, `|dx| + |dz| < 4` from the
  interaction point.
- **Proven live:** called on `Objects.Food.3`, returned **`true`**. It returned only `true`,
  **no info table** - so the grab result cannot be used to pick a serve target.

## Serving food (Confirmed 2026-10-27 by decompile of `Tasks.ServeFood`)

All variants end in `Events.Restaurant.TaskCompleted:FireServer(task)` with
`Name = <Task enum>.Serve`, `Tycoon`, `FoodModel`, plus one discriminator:
`Player` | (`GroupId` + `CustomerId`) | `CarId`. There is a **0.25 s debounce**.

**Missing constant:** `require(ReplicatedStorage.Source.Enums.Restaurant.Task)` decompiles to
`return {}`, so `Task.Serve` has no readable value. A wrong `Name` silently no-ops. Capture it
with Remote Spy (serve one customer by hand) - do not invent it.

## Script versions

- v1.0.0: keyword prompt scan. No-op on this game.
- v2.0.0: grab + Dialog serve + grow + shop. Fired world NPC dialogue (WaveyDavey).
- v2.0.1: added a Humanoid filter on Dialog - still wrong, because WaveyDavey has a Humanoid.
- v3.0.0: plot-scoped only; grab (proven) + SeatPrompt + a Dialog order loop that is dead
  code (0 Dialog prompts in the plot).
- v3.1.0: grab + seat only, globally throttled so it cannot spam 8 seats at once; order loop
  removed rather than left pretending.

## Still unknown

1. `Task.Serve` value (Remote Spy).
2. How a seated customer's order is taken - no prompt in the plot, so it is UI/remote.
3. How the tip jar and table bill are collected - neither is interactable.
