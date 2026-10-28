# Restaurant Tycoon 3

Place id **119048529960596**. `game.Name` reports **`Ugc`**, so Hub must pin this place in
`PLACES` (done, Hub v3.0). Script: `Restaurant Tycoon 3` in `bobashid/roblox-matcha`.

## Why only the tip jar worked (User-reported 2026-10-27, diagnosed)

User: "nothing except auto tip jar works".

**Tips is the only action that fires a single remote.** Every other action in the captured
traffic fires **two** calls, in this order:

1. `Events.Restaurant.Interactions.Interacted:FireServer(tycoon, promptTable)`
2. `Events.Restaurant.TaskCompleted:FireServer(taskTable)`

v3.6.x sent only step 2, so seat / bill / serve were silent no-ops. The two-call pair is
**Confirmed** for `CollectBill` from the user's own Remote Spy capture. That seat and serve
need step 1 too is **UNVERIFIED** - it is the inference that explains the symptom, not a proof.

**Second candidate cause, also UNVERIFIED:** the captured payload carries `Range = 7.2` and a
`WorldPosition`, so the server may validate distance. If it does, a remote fired from across the
plot fails no matter how correct the payload is. v3.7.1 therefore refuses to fire at targets
farther than `Range` studs and counts them in `skippedOutOfRange`.

## `TaskCompleted` is the universal task remote (Confirmed 2026-10-27)

Three actions share the remote, discriminated only by `Name`, a plain literal string:

| Action | `Name` | Extra fields |
|---|---|---|
| Seat a guest at a table | `"SendToTable"` | `FurnitureModel` = the `Items.Surface` table, `GroupId` |
| Collect a table bill | `"CollectBill"` | `FurnitureModel` = the `Items.Surface` table |
| Serve a dish | `"Serve"` (getgc-confirmed) | `GroupId`, `CustomerId`, `FoodModel` |

**`GroupId` is a STRING**, not a number - the seat capture shows `GroupId = "19"`. It is the
`ClientCustomers` **folder name**. `CustomerId` is the customer **model name**, also a string.

`require(...Enums.Restaurant.Task)` decompiles to `return {}`, so the source never shows these
values; they are runtime strings. `getgc("Serve")` returned 27 string entries including
`"Serve"`, `"Name"`, `"Key"`, `"IsHoldingFoodForNPC"`; `getgc("CollectBill")` returned 4
including `"CollectBill"`.

**A single `n=0` from `getgc` means nothing.** An earlier probe returned 0 for both keys, then a
retry returned 4 and 27. Treat a zero as a failed call and re-probe.

## Captured remote payloads (User-stated 2026-10-27, verbatim from Remote Spy)

**Seat a guest** (in game: click the guest's prompt, THEN click an empty table):

    Events.Restaurant.TaskCompleted:FireServer({
        FurnitureModel = game.Workspace.Tycoons.Tycoon.Items.Surface.T1,
        GroupId = "19",
        Tycoon = game.Workspace.Tycoons.Tycoon,
        Name = "SendToTable" })

**Tips** - one argument, the tycoon folder:

    Events.Restaurant.TipsCollected:FireServer(game.Workspace.Tycoons.Tycoon)

**Bill** - two calls fired together:

    Events.Restaurant.Interactions.Interacted:FireServer(tycoon, {
        Id = "142", Key = "CollectBill", Model = tycoon.Items.Surface.T19,
        InteractionType = "CollectBill", IsCustomerPrompt = true, Range = 7.2,
        WorldPosition = Vector3.new(1204.80, 0.89, 295.99),
        KeyboardKeyCode = Enum.KeyCode.E, HoldDuration = 0, ... })

    Events.Restaurant.TaskCompleted:FireServer({
        Tycoon = tycoon, Name = "CollectBill", FurnitureModel = tycoon.Items.Surface.T19 })

The bill target is the **`Surface` model**, not its `Bill` child.

## Guest system (Confirmed 2026-10-27, live probe)

`Workspace.Tycoons.Tycoon.ClientCustomers` (client-created; absent until guests spawn):

    Folder "14" -> Model "1", Model "2"
    Folder "15" -> Model "1", Model "2"
    Folder "16" -> Model "1", Model "2"
    Folder "17" -> Model "1", Model "2"

Folder name = `GroupId`, model name = `CustomerId`. The models are client-side dummies:
**no Humanoid, no ProximityPrompt, no ClickDetector**, so `fireproximityprompt` can never serve
or seat. Guest data lives in the module's `TycoonStorage` table, not the DataModel.

`Objects.Food` children carry attributes `EatMethod` and `Taken`.

## Bugs I caused, and their fixes

1. **Serving the wrong dish.** `foodList()` skips dishes with `Taken` set. Grabbing sets `Taken`,
   so the just-grabbed dish vanished from the list and v3.4.0 served some *other* dish still on
   the counter - food never left the player's hands. Fixed in v3.5.0 with `held` tracking.
2. **"Auto seat" misread as firing `SeatPrompt`.** The user meant seating a guest who walks in.
   Removed in v3.3.0; the real fix is the captured `SendToTable` payload.
3. **Firing `Dialog` prompts** opened world NPC dialogue (WaveyDavey). 0 `Dialog` prompts exist
   inside the plot. Dialog firing is banned.
4. **Sending `TaskCompleted` without `Interactions.Interacted`** - see the first section.

## Serving preconditions (Confirmed by decompile)

NPC branch requires customer `State == CustomerState.WaitingForDish` and
`GrabFood:IsHoldingFoodForNPC(order, isKid)`, which compares the order against the held dish's
model Name. The order is not exposed in the DataModel, so the script cycles guests round-robin;
only the guest whose order matches the held dish can accept it. 0.25 s client debounce.

## Other confirmed facts (2026-10-27)

- Ownership is an **`ObjectValue` named `Player`**: `plot.Player.Value == LocalPlayer`.
- Currency is `leaderstats` -> `IntValue` named `💸 Cash` (emoji + space), and it rises on its
  own, so a rising balance is not evidence of anything.
- `Items.Furniture.K32.Cash` is **decoration** (16 Models, 0 touch/prompt/detector/attachment).
- Plot interactables total **9**: 1 `DoorPrompt`, 8 `SeatPrompt`. Nothing else.
- Tip jar = MeshPart + `TipsAmount` BillboardGui. `Dialog` prompts in plot: **0**.
- Money drops: `workspace.DropFolder`, auto-collected within `7 + SizeModifier` studs via
  batched `PlayerCollectedDrops:FireServer({ids})`. `getconnections`/`hookfunction` are nil.
- `GrabFood:InvokeServer(foodModel)` **proven live**, returns bare `true` (no info table).

## Script versions

- v1.0.0: keyword prompt scan. No-op.
- v2.0.0 / v2.0.1: grab + Dialog serve; fired world NPC dialogue.
- v3.0.0 / v3.1.0: plot-scoped; grab + SeatPrompt.
- v3.2.0: + tips + bills. v3.3.0: SeatPrompt removed.
- v3.4.0: serve added - served the wrong dish.
- v3.5.0: held-model tracking, guest round-robin.
- v3.6.0 / v3.6.1: seat via `SendToTable`; fixed a malformed local.
- v3.7.0: adds `Interactions.Interacted` before `TaskCompleted` (only tips had worked).
- v3.7.1: range-gates every interaction to `Range` studs and reports `skippedOutOfRange`.

## Still unknown

1. The `Key` / `InteractionType` strings and prompt `Id` for `SendToTable` and `Serve`.
2. Whether the server range-checks `Interactions.Interacted`.
3. The order-taking payload (to match the held dish to the right guest).
