# Restaurant Tycoon 3

Place id **119048529960596**. `game.Name` reports **`Ugc`** (the creator group's name), so Hub
must pin this place in `PLACES`. Done in Hub v3.0.

Script: `Restaurant Tycoon 3` in `bobashid/roblox-matcha`.

## The single most important fact (User-captured 2026-10-27, Remote Spy)

**`TaskCompleted` takes `Name` as a literal string, not an enum value.** The captured bill
payload is `Name = "CollectBill"`. This supersedes the earlier "Task.Serve enum is unreadable"
blocker: the names are plain strings, so any task can be built if its string is known.
Known string so far: `"CollectBill"`. Not yet captured: the order-taking and serve names.

## Captured remote payloads (User-stated 2026-10-27, verbatim from Remote Spy)

Tips:

    game.ReplicatedStorage.Events.Restaurant.TipsCollected:FireServer(game.Workspace.Tycoons.Tycoon)

One argument - the tycoon folder. No ids, no amounts.

Bill collection, two calls fired together (both captured):

    Events.Restaurant.Interactions.Interacted:FireServer(tycoon, {
        Id = "142", ActionText = "", WorldPosition = Vector3.new(1204.80, 0.89, 295.99),
        GamepadKeyCode = Enum.KeyCode.Thumbstick1, TemporaryPart = Part,
        KeyboardKeyCode = Enum.KeyCode.E, Range = 7.2, IsCustomerPrompt = true,
        Key = "CollectBill", Model = tycoon.Items.Surface.T19, Part = Part,
        Prompt = 142, InteractionType = "CollectBill", HoldDuration = 0 })

    Events.Restaurant.TaskCompleted:FireServer({
        Tycoon = tycoon, Name = "CollectBill", FurnitureModel = tycoon.Items.Surface.T19 })

Note the **target is the furniture model (`Surface.T19`), not its `Bill` child**. The `Bill`
child is how you find it, but the payload wants the parent.

**`Events.Restaurant.Interactions` is a remote I had not catalogued** - it is a child of the
`Restaurant` events folder alongside `GrabFood`/`TaskCompleted`.

## Confirmed (2026-10-27, live probes via matcha_exec)

- **Ownership is an ObjectValue, not an attribute.** `workspace.Tycoons` holds 8 children all
  named `Tycoon`; `GetAttribute('Owner')` returns nil on every one. Owner test:
  `plot.Player.Value == LocalPlayer`.
- **Currency:** `LocalPlayer.leaderstats` -> `IntValue` named `💸 Cash` (emoji + space). Read
  124, then 285 with no manual collecting: **passive income exists**, so a rising balance is not
  proof a script worked. Measure objects disappearing instead.
- **Tycoon layout:** `Locations` (6 CFrameValues), `Player` (ObjectValue), `Objects` (Food,
  Advertising, CookingResources), `GreyBox`, `Default`, `Items`.
  `Items` = BeingPlaced, Furniture (19), Surface (10), Walls (42), Floor (9), Config (2), Stairs (0).
- **`Items.Furniture.K32.Cash` is DECORATION, not money.** 16 child Models, 2 MeshParts each,
  and inside that subtree: 0 TouchTransmitter, 0 ProximityPrompt, 0 ClickDetector, 0 Attachment.
- Food models live in `Objects.Food`, named numerically (seen `225`); `Taken` reads nil when fresh.

## Prompt inventory (Confirmed 2026-10-27, whole-workspace scan)

About 350 `ProximityPrompt`/`ClickDetector` exist in `workspace`. Real names seen:
`RestaurantBoardPrompt`, `SeatPrompt`, `DoorPrompt`, `BuyPrompt` (cosmetics on hats and emotes),
`IngredientShopPrompt`, `GrowAllPrompt`, `Dialog`, `ChefLuigi`, `CreditsPrompt`,
`EventHistoryMachinePrompt`, and several plain `ProximityPrompt`.

A keyword scan for dish/money/trash terms matched **0** of them, which is why v1.0.0 did nothing.

`GrowAllPrompt` sits on a `Sign`; `IngredientShopPrompt` sits on a `HumanoidRootPart`.

## My plot's complete interactable inventory (Confirmed 2026-10-27)

`Items:GetDescendants()` scanned for ProximityPrompt / ClickDetector / TouchTransmitter:

    INTERACT_IN_ITEMS = 9

| What | Count | Path shape |
|---|---|---|
| `DoorPrompt` | 1 | `Items.Furniture.D20.Base.PromptAttachment.DoorPrompt` |
| `SeatPrompt` | 8 | `Items.Furniture.C23/C1.Base.SeatAttachment.SeatPrompt` |

**Nothing else in the plot is interactable as an instance.** The tip jar
`Items.Furniture.K32.Jar` is a `MeshPart` whose only child is a `BillboardGui` named
`TipsAmount`. The table bill `Items.Surface.T19.Bill` is a `Model` with 4 children. Neither has
a prompt, ClickDetector or TouchTransmitter - but both are reachable by **remote**, which is how
the captured payloads above work.

**`Dialog` prompts inside my plot: 0** (`DIALOG_IN_MY_PLOT 0`). Every `Dialog` in the place
belongs to world NPCs, which is why firing them opened WaveyDavey's dialogue.

## Remotes (Confirmed 2026-10-27, name scan of ReplicatedStorage)

| Remote | Class | Parent |
|---|---|---|
| `GrabFood` | RemoteFunction | `Restaurant` |
| `GrabEnded` | RemoteEvent | `Restaurant` |
| `CancelGrabRequested` | RemoteEvent | `Restaurant` |
| `TaskCompleted` | RemoteEvent | `Restaurant` |
| `TipsCollected` | RemoteEvent | `Restaurant` |
| `Interactions` | (remote) | `Restaurant` |
| `PlayerCollectedDrops` | UnreliableRemoteEvent | `Currency` |
| `DropSpawned` / `DropsDespawned` | UnreliableRemoteEvent | `Currency` |
| `CollectOfflineEarningsRequested` | RemoteEvent | `Upgrades` |
| `FoodConsumed` / `FoodOrdered` / `FoodInspected` | RemoteEvent | `Food` |
| `ItemCollected` | RemoteEvent | `ScavengerHunt` |

## Money drops (Confirmed 2026-10-27 by decompile of `CurrencyDrops`)

- Drops clone from `Assets.Models.Currency.DropTemplate` into **`workspace.DropFolder`**.
- Ids arrive from the server: `Events.Currency.DropSpawned.OnClientEvent(_, dropId, data)`.
- **Collection is automatic by proximity:** a Heartbeat test
  `MagnitudeSquared(drop, HumanoidRootPart) <= (7 + SizeModifier)^2`, then a ~1 s Lerp pull and
  a batched `PlayerCollectedDrops:FireServer({ids})` (a single array argument). 120 s despawn.
- `getconnections` and `hookfunction` are **nil** in this VM (`getgc` is a function), so ids
  cannot be intercepted and a direct fire is unreachable - and unnecessary.
- `workspace.DropFolder` existed but held 0 children at probe time.

## Grabbing food (Confirmed 2026-10-27)

- `local ok, info = Events.Restaurant.GrabFood:InvokeServer(foodModel)`; the food Model is the
  only argument. Preconditions: `Managers:HasPermission(tycoon)`, `Cook:PlayerIsCooking()` false,
  `food:GetAttribute('Taken')` falsy, and every item already held must be from the same tycoon.
- Hold capacity = `WaiterCart and 4 or WaiterTray and 3 or 1`.
- Candidates = `tycoon.Objects.Food` children with no `Taken`, within `|dx| + |dz| < 4` of the
  interaction point.
- **Proven live:** called on `Objects.Food.3` and returned **`true`**. It returned only `true`,
  **no info table**, so the grab result cannot be used to pick a serve target.
- `tycoon.Objects.Food` held 5 children at one probe and 2 at a later one.

## Serving food (Confirmed 2026-10-27 by decompile of `Tasks.ServeFood`)

All variants end in `Events.Restaurant.TaskCompleted:FireServer(task)` with
`Name`, `Tycoon`, `FoodModel`, plus exactly one discriminator:
`Player` | (`GroupId` + `CustomerId`) | `CarId`. There is a **0.25 s debounce** in the client
`TaskInputReceived` dispatch - replicate it or calls spam.

Given the captured `Name = "CollectBill"`, `Name` is a literal string here too. The serve string
is most likely `"Serve"`, but it is **UNVERIFIED** - capture it before shipping it.

## Script versions

- v1.0.0: keyword prompt scan. No-op on this game.
- v2.0.0: grab + Dialog serve + grow + shop. Fired world NPC dialogue (WaveyDavey).
- v2.0.1: added a Humanoid filter on Dialog - still wrong, because WaveyDavey has a Humanoid.
- v3.0.0: plot-scoped; grab + SeatPrompt + a Dialog order loop that was dead code.
- v3.1.0: grab + seat only, seat firing throttled; order loop removed.
- v3.2.0: grab + seat + tips (`TipsCollected:FireServer(tycoon)`) + bills
  (`TaskCompleted` with `Name='CollectBill'`, `FurnitureModel` = the `Surface` model that holds a
  `Bill` child). Both payloads user-captured.

## Still unknown

1. The order-taking payload - `Interactions` with some `Key`/`InteractionType` string, not captured.
2. The serve `Name` string (likely `"Serve"`, unverified).
3. Whether `Interactions.Interacted` is needed as well as `TaskCompleted` for bills, or whether
   `TaskCompleted` alone is sufficient. v3.2.0 fires only `TaskCompleted`; if bills do not clear,
   add the `Interactions` call from the captured payload.
