# Restaurant Tycoon 3

Place id **119048529960596**. `game.Name` reports **`Ugc`** (the creator group's name), so Hub
must pin this place in `PLACES`. Done in Hub v3.0.

Script: `Restaurant Tycoon 3` in `bobashid/roblox-matcha`.

## Corrections from the user (User-stated 2026-10-27)

- **"Auto seat" does NOT mean firing `SeatPrompt`.** The user meant **accepting a guest's
  arrival prompt when they walk into the restaurant**. v3.0-v3.2 fired `SeatPrompt` and were
  wrong; the toggle was removed in v3.3.0. The guest-accept payload has **not** been captured
  yet, so nothing may guess at it.
- **"It keeps activating dialogue for WaveyDavey"** - firing `Dialog` prompts hit world NPCs,
  not customers. There are 0 `Dialog` prompts inside the plot. Dialog firing is banned.
- **`💸 Cash` rises on its own** (passive income), so a rising balance is not evidence.

## The single most important fact (User-captured 2026-10-27, Remote Spy)

**`TaskCompleted` takes `Name` as a literal string, not an enum value.** The captured bill
payload is `Name = "CollectBill"`. This supersedes the earlier "Task.Serve enum is unreadable"
blocker: the names are plain strings, so any task can be built if its string is known.
Known string so far: `"CollectBill"`. Not yet captured: the guest-accept, order-taking and
serve names.

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

**`IsCustomerPrompt = true` in that payload is the clue for guest-seating:** the guest arrival
prompt almost certainly goes through the same `Interactions.Interacted` remote with its own
`Key`/`InteractionType` string. Capture it - do not guess it.

## Confirmed (2026-10-27, live probes via matcha_exec)

- **Ownership is an ObjectValue, not an attribute.** `workspace.Tycoons` holds 8 children all
  named `Tycoon`; `GetAttribute('Owner')` returns nil on every one. Owner test:
  `plot.Player.Value == LocalPlayer`.
- **Currency:** `LocalPlayer.leaderstats` -> `IntValue` named `💸 Cash` (emoji + space).
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
a prompt, ClickDetector or TouchTransmitter - both are reachable only by **remote**.

**`Dialog` prompts inside my plot: 0** (`DIALOG_IN_MY_PLOT 0`).

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

- Drops clone into **`workspace.DropFolder`**; ids come from the server via
  `Events.Currency.DropSpawned.OnClientEvent(_, dropId, data)`.
- **Collection is automatic by proximity:** `MagnitudeSquared(drop, HumanoidRootPart) <=
  (7 + SizeModifier)^2`, then a ~1 s Lerp pull and a batched
  `PlayerCollectedDrops:FireServer({ids})` (single array argument). 120 s despawn.
- `getconnections` and `hookfunction` are **nil** in this VM (`getgc` is a function).

## Grabbing food (Confirmed 2026-10-27)

- `local ok, info = Events.Restaurant.GrabFood:InvokeServer(foodModel)`; the food Model is the
  only argument. Preconditions: `Managers:HasPermission(tycoon)`, `Cook:PlayerIsCooking()` false,
  `food:GetAttribute('Taken')` falsy, all held items from the same tycoon.
- Hold capacity = `WaiterCart and 4 or WaiterTray and 3 or 1`.
- **Proven live:** called on `Objects.Food.3`, returned **`true`**, and **no info table**.

## Serving food (Confirmed 2026-10-27 by decompile of `Tasks.ServeFood`)

All variants end in `Events.Restaurant.TaskCompleted:FireServer(task)` with `Name`, `Tycoon`,
`FoodModel`, plus one discriminator: `Player` | (`GroupId` + `CustomerId`) | `CarId`. There is
a **0.25 s debounce** in the client dispatch.

Given the captured `Name = "CollectBill"`, `Name` is a literal string here too. The serve
string is probably `"Serve"` but is **UNVERIFIED** - capture it before shipping it.

## Script versions

- v1.0.0: keyword prompt scan. No-op on this game.
- v2.0.0: grab + Dialog serve + grow + shop. Fired world NPC dialogue (WaveyDavey).
- v2.0.1: Humanoid filter on Dialog - still wrong, WaveyDavey has a Humanoid.
- v3.0.0: plot-scoped; grab + SeatPrompt + dead Dialog order loop.
- v3.1.0: grab + seat only, throttled.
- v3.2.0: grab + seat + tips + bills.
- v3.3.0: **SeatPrompt removed** (wrong meaning of "auto seat"); grab + tips + bills only.

## Still unknown

1. **The guest-accept payload** (`Interactions.Interacted`, `IsCustomerPrompt = true`, own
   `Key`/`InteractionType`). This is what the user actually wants from "auto seat".
2. The order-taking payload.
3. The serve `Name` string (probably `"Serve"`).
