# Restaurant Tycoon 3

Place id **119048529960596**. `game.Name` reports **`Ugc`**, so Hub must pin this place in
`PLACES` (done, Hub v3.0). Script: `Restaurant Tycoon 3` in `bobashid/roblox-matcha`.

## Two bugs I caused, and their fixes

1. **Serving the wrong dish.** `foodList()` skips dishes with `Taken` set. Grabbing sets `Taken`,
   so the just-grabbed dish disappeared from the list and v3.4.0 served some *other* dish still
   on the counter - so nothing ever left the player's hands. Fixed in v3.5.0 by remembering the
   grabbed model in `held` and serving that exact model.
2. **Firing `Dialog` prompts / `SeatPrompt`.** Both were wrong interpretations (see corrections).

## Corrections from the user (User-stated 2026-10-27)

- **"Auto seat" does NOT mean firing `SeatPrompt`.** It means accepting a guest's arrival
  prompt. The SeatPrompt toggle was removed in v3.3.0.
- **Firing `Dialog` prompts opened WaveyDavey's dialogue.** There are 0 `Dialog` prompts in the
  plot. Dialog firing is banned.
- **`💸 Cash` rises on its own** (passive income), so a rising balance is not evidence.
- **"Grabbing worked, serving did not"** drove v3.4.0 and then v3.5.0.

## `TaskCompleted` names are literal strings (Confirmed 2026-10-27)

- **User-captured:** `TaskCompleted:FireServer({Tycoon=tycoon, Name="CollectBill", FurnitureModel=Surface.T19})`.
- **Memory-confirmed:** `getgc("Serve")` returned **27** string entries including `"Serve"`,
  `"Name"`, `"Key"`, `"IsHoldingFoodForNPC"`, `"BuyFurniture"`. `getgc("CollectBill")`
  returned 4 entries including `"CollectBill"` and `"Key"`.
- **So `Name = 'Serve'` is correct.** The earlier `getgc` probe returning `n=0` was a failed
  call, not evidence of absence - a stale echo. Do not trust a single zero result from `getgc`.
- `require(...Enums.Restaurant.Task)` decompiles to `return {}`; the values are runtime strings,
  which is why the source read never showed them.

## Captured remote payloads (User-stated 2026-10-27, verbatim)

Tips - one argument, the tycoon folder:

    Events.Restaurant.TipsCollected:FireServer(game.Workspace.Tycoons.Tycoon)

Bill - two calls fired together:

    Events.Restaurant.Interactions.Interacted:FireServer(tycoon, {
        Id = "142", Key = "CollectBill", Model = tycoon.Items.Surface.T19,
        InteractionType = "CollectBill", IsCustomerPrompt = true, Range = 7.2,
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

**Folder name = `GroupId`, model name = `CustomerId`** - the integer pair that
`Customers:GetCustomerData(tycoon, groupId, customerId)` expects. The models are client-side
dummies: **no Humanoid, no ProximityPrompt, no ClickDetector**, so `fireproximityprompt` can
never serve. Guest data lives in the module's `TyonoStorage`/`TycoonStorage` table, not the
DataModel - there is no `Customers` folder under the tycoon.

`Objects.Food` children carry attributes: `EatMethod` and `Taken`.

## Serving (Confirmed 2026-10-27 by decompile + probe)

`TaskCompleted:FireServer(task)` where task = `Name='Serve'`, `Tycoon`, `FoodModel`, plus one
discriminator: `Player` | (`GroupId` + `CustomerId`) | `CarId`. 0.25 s client debounce.
NPC branch precondition: customer `State == WaitingForDish` and
`GrabFood:IsHoldingFoodForNPC(order, isKid)`, which compares `order` against the **held dish's
model Name** (`FoodKey`). The order is not exposed in the DataModel, so v3.5.0 cycles guests
round-robin; only the guest whose order matches the held dish can accept it.

## Other confirmed facts (2026-10-27)

- Ownership is an **`ObjectValue` named `Player`**: `plot.Player.Value == LocalPlayer`.
- Currency is `leaderstats` -> `IntValue` named `💸 Cash` (emoji + space).
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
- v3.2.0: + tips + bills (user-captured payloads).
- v3.3.0: SeatPrompt removed.
- v3.4.0: serve added via ClientCustomers - **served the wrong dish**.
- v3.5.0: held-model tracking; serves the dish actually in hand, guest round-robin.

## Still unknown

1. The guest-accept payload (`Interactions.Interacted`, `IsCustomerPrompt = true`).
2. The order-taking payload.
3. Whether `Interactions.Interacted` is also required for bills (v3.5.0 sends `TaskCompleted` only).
