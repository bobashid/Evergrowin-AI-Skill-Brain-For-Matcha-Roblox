# Restaurant Tycoon 3

Place id **119048529960596**. `game.Name` reports **`Ugc`**, so Hub must pin this place in
`PLACES` (done, Hub v3.0).

Script: `Restaurant Tycoon 3` in `bobashid/roblox-matcha`.

## Corrections from the user (User-stated 2026-10-27)

- **"Auto seat" does NOT mean firing `SeatPrompt`.** It means accepting a guest's arrival
  prompt. The SeatPrompt toggle was removed in v3.3.0.
- **Firing `Dialog` prompts opened WaveyDavey's dialogue.** There are 0 `Dialog` prompts in
  the plot. Dialog firing is banned.
- **`💸 Cash` rises on its own** (passive income), so a rising balance is not evidence.
- **Grabbing worked, serving did not** - "it picks up the food from the counter but doesnt
  give it to the guest that is requesting that specific food". That drove v3.4.0.

## Guest system (Confirmed 2026-10-27, live probe)

`Workspace.Tycoons.Tycoon` children include **`ClientCustomers`** and **`ClientWorkers`** (both
`Folder`s, created client-side, absent on earlier probes):

    ClientCustomers
      Folder "14"  -> Model "1", Model "2"
      Folder "15"  -> Model "1", Model "2"
      Folder "16"  -> Model "1", Model "2"
      Folder "17"  -> Model "1", Model "2"

- **The folder name IS the `GroupId`, the model name IS the `CustomerId`** - they are integers,
  which is exactly the pair `Tasks.ServeFood` looks up via
  `Customers:GetCustomerData(tycoon, groupId, customerId)`.
- Customer models contain a `BillboardGui` named `CustomerTypeLabel` (and sometimes another
  `BillboardGui`). **No ProximityPrompt, no ClickDetector, and no Humanoid** (`hum=false` on all
  of them) - these are client-side dummies, so `fireproximityprompt` can never serve.
- There is **no `Customers` folder under the tycoon** (`CUSTOMERS -1`); the data lives in the
  client module's `TycoonStorage` table, not in the DataModel.

## Food attributes (Confirmed 2026-10-27, live probe)

`Objects.Food` children carry attributes `1 = EatMethod` and `2 = Taken` (numeric keys, seen
printed as `1=EatMethod,2=Taken`). So `EatMethod` is an attribute on each dish - the likely
key to matching a dish to the guest that requested it.

## Serving food (Confirmed 2026-10-27 by decompile + live probe)

`Events.Restaurant.TaskCompleted:FireServer(task)` with `Name`, `Tycoon`, `FoodModel`, plus one
discriminator: `Player` | (`GroupId` + `CustomerId`) | `CarId`. 0.25 s debounce in the client
dispatch. Client precondition for the NPC branch:
`Customers:GetCustomerData(tycoon, groupId, customerId).State == CustomerState.WaitingForDish`
and `GrabFood:IsHoldingFoodForNPC(order, isKid)`.

v3.4.0 sends `{ Name = 'Serve', Tycoon = tycoon, GroupId = <folder name>,
CustomerId = <model name>, FoodModel = <first dish> }`.

**UNVERIFIED: the `Name = 'Serve'` string.** The captured bill payload uses a literal string
(`"CollectBill"`), so this is very likely `"Serve"`, but it has not been observed. If serving
no-ops, capture one hand-serve with Remote Spy before changing anything else.

**Second open question:** the dish must match the guest's request
(`IsHoldingFoodForNPC(order, isKid)`), and the first dish in `Objects.Food` may not be the
requested one. The match likely keys off the dish's `EatMethod` attribute.

## The most important fact (User-captured 2026-10-27, Remote Spy)

**`TaskCompleted` takes `Name` as a literal string, not an enum value** (`Name = "CollectBill"`).
This retired the earlier "Task.Serve enum is unreadable" blocker.

## Captured remote payloads (User-stated 2026-10-27, verbatim)

Tips:

    Events.Restaurant.TipsCollected:FireServer(game.Workspace.Tycoons.Tycoon)

Bill (two calls fired together):

    Events.Restaurant.Interactions.Interacted:FireServer(tycoon, {
        Id = "142", ActionText = "", WorldPosition = Vector3.new(1204.80, 0.89, 295.99),
        GamepadKeyCode = Enum.KeyCode.Thumbstick1, TemporaryPart = Part,
        KeyboardKeyCode = Enum.KeyCode.E, Range = 7.2, IsCustomerPrompt = true,
        Key = "CollectBill", Model = tycoon.Items.Surface.T19, Part = Part,
        Prompt = 142, InteractionType = "CollectBill", HoldDuration = 0 })

    Events.Restaurant.TaskCompleted:FireServer({
        Tycoon = tycoon, Name = "CollectBill", FurnitureModel = tycoon.Items.Surface.T19 })

The bill target is the **`Surface` model**, not its `Bill` child.
`IsCustomerPrompt = true` is the clue that guest prompts share the `Interactions` remote.

## Other confirmed facts (2026-10-27, live probes)

- Ownership is an **`ObjectValue` named `Player`**, not an attribute: `plot.Player.Value == lp`.
- Currency is `leaderstats` -> `IntValue` named `💸 Cash` (emoji + space).
- `Items` = BeingPlaced, Furniture (19), Surface (10), Walls (42), Floor (9), Config (2), Stairs (0).
- `Items.Furniture.K32.Cash` is **decoration**: 16 Models, 0 touch/prompt/detector/attachment.
- Plot interactables total **9**: 1 `DoorPrompt`, 8 `SeatPrompt`. Nothing else.
- Tip jar `Items.Furniture.K32.Jar` = MeshPart + `TipsAmount` BillboardGui only.
- `Dialog` prompts in plot: **0**. Workspace total ~350 prompts.
- Money drops: `workspace.DropFolder`, auto-collected inside `7 + SizeModifier` studs via a
  batched `PlayerCollectedDrops:FireServer({ids})`. `getconnections`/`hookfunction` are nil.
- `GrabFood:InvokeServer(foodModel)` **proven live**, returns bare `true` (no info table).

## Script versions

- v1.0.0: keyword prompt scan. No-op.
- v2.0.0: grab + Dialog serve + grow + shop. Fired world NPC dialogue.
- v2.0.1: Humanoid filter on Dialog - still wrong.
- v3.0.0: plot-scoped; grab + SeatPrompt + dead Dialog loop.
- v3.1.0: grab + seat, throttled.
- v3.2.0: grab + seat + tips + bills.
- v3.3.0: seat removed (wrong meaning); grab + tips + bills.
- v3.4.0: **serve added** using `ClientCustomers` GroupId/CustomerId; grab + serve + tips + bills.

## Still unknown

1. The `Name = 'Serve'` string (capture a hand-serve).
2. How to match the dish to the guest's request (`EatMethod` attribute is the lead).
3. The guest-accept payload (`Interactions.Interacted`, `IsCustomerPrompt = true`).
4. The order-taking payload.
