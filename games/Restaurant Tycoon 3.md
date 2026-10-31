# Restaurant Tycoon 3

Place id **119048529960596**. `game.Name` reports **`Ugc`**, so Hub pins this place in `PLACES`
(done, Hub v3.0). Script: `Restaurant Tycoon 3` in `bobashid/roblox-matcha`.

## CONFIRMED WORKING by the user (2026-10-27)

- v6.0.1: bills + tips - "nice everything there works".
- v6.1.0: dishes - "nice everything works the auto dish pickup and cash+tips".

v6.2.0 keeps all of that code unchanged and adds auto take order on top.

## Call shapes DIFFER per action - never assume uniformity (Confirmed 2026-10-27)

| Action | Calls | Target / args |
|---|---|---|
| Collect bill | `Interacted` **THEN** `TaskCompleted` (2) | `FurnitureModel` = `Items.Surface` model holding a `Bill` child |
| Collect dishes | `TaskCompleted` only (1) | `FurnitureModel` = `Items.Surface` model holding a `Trash` child |
| Collect tips | `TipsCollected` only (1) | the tycoon itself |
| Take order | `TaskCompleted` only (1) | `GroupId` + `CustomerId`, both STRINGS |

Sending an extra unverified remote alongside a single-call action is how v6.0.0 crashed the
game. Only send what the capture shows.

## Captured payloads (user, Remote Spy, 2026-10-27)

**Take order** (single call):

    Events.Restaurant.TaskCompleted:FireServer({
        CustomerId = "2", GroupId = "8",
        Tycoon = game.Workspace.Tycoons.Tycoon, Name = "TakeOrder" })

**Collect dishes** (single call):

    Events.Restaurant.TaskCompleted:FireServer({
        Tycoon = tycoon, Name = "CollectDishes",
        FurnitureModel = tycoon.Items.Surface.T19 })

**Collect bill** (two calls, in order):

    Events.Restaurant.Interactions.Interacted:FireServer(tycoon, {
        Id="1", ActionText="", WorldPosition=<part pos>, GamepadKeyCode=1000,
        TemporaryPart=Part, KeyboardKeyCode=101, Range=7.2, IsCustomerPrompt=true,
        Key="CollectBill", Model=<Items.Surface.T?>, Part=Part, Prompt=1,
        InteractionType="CollectBill", HoldDuration=0 })

    Events.Restaurant.TaskCompleted:FireServer({
        Tycoon=tycoon, Name="CollectBill", FurnitureModel=<same Items.Surface model> })

An earlier bill capture used `Id="142"` / `Prompt=142` / `KeyCode E`, so those are just prompt
numbers and key codes, not fixed constants.

**Tips** (single call): `Events.Restaurant.TipsCollected:FireServer(tycoon)`.

## `Interactions` is a FOLDER, not a remote (Confirmed by live probe)

Probed children of `Events.Restaurant.Interactions`: `RemoteEvent Interacted`,
`InteractionCreated`, `InteractionDestroyed`, `AllInteractionsDestroyed`,
`ClickableFurnitureInteracted`, and `BindableEvent ClickableFurnitureInteractedClient`.

`Interacted` must be resolved **inside** that folder. Firing the Folder itself
(`Folder:FireServer`) throws inside pcall and is swallowed - that was the v6.0.0 defect and the
likely crash path.

## Guest source: `Tycoon.ClientCustomers` (Confirmed live 2026-10-27)

    Folder "8"  -> Model "1", Model "2"
    Folder "9"  -> Model "1", Model "2"
    Folder "10" -> Model "1", Model "2"

**Folder name = `GroupId`, child model name = `CustomerId`**, both opaque strings - never
validate them with `tonumber()`; doing so silently dropped every customer in v4.1.1.
The captured take-order pair (`GroupId="8"`, `CustomerId="2"`) matches this structure exactly.

Note: earlier probes found this folder empty (`cust=0`) when no guests were present. Empty is
normal when idle, not a bug.

## Crash post-mortem: v6.0.0 (2026-10-27)

v6.0.0 crashed the game. Four defects, all removed in v6.0.1, which then worked:

1. `remote('Restaurant','Interactions')` returned the **Folder**, and `Folder:FireServer(...)`
   was attempted - the first bill call never fired.
2. `FindFirstChildWhichIsA('BasePart', true)` used for part lookup. `IsA` always returns true in
   this VM, so it can return a non-part; sending a Model where a `Part` is expected is a
   plausible crash trigger. **Find parts by `ClassName` only** (`Part`/`MeshPart`/
   `UnionOperation`/`WedgePart`), and skip the target if none is found.
3. **No rate limit** - up to 12 bills x 2 calls every 0.25 s. v6.0.1 added a token bucket
   (8 calls per 0.5 s) and one target per feature per pass.
4. `task.wait` inside the collect loop. Removed; the loop yields only between passes.

Rule for this game: never fire more than one collect target per pass, and never exceed a
handful of calls per half second. It has crashed once already.

## Other live probe facts

- `V3 bills 0 trash 5` - five `Trash` surfaces existed at probe time, so dish collection had
  targets. Bills/trash reading 0 is normal when idle.
- `Y3 bills 1 T19`, `Y4 partsInFirstBillTable 7` - a billed table has real BaseParts under it,
  so a ClassName part lookup succeeds on real targets.
- Ownership: `workspace.Tycoons` children, `ObjectValue` whose `Value` is the LocalPlayer.
- Currency: `leaderstats` -> `IntValue` named `💸 Cash`; rises on its own, so a rising balance
  is not evidence a script worked.
- `TaskCompleted` is a plain `RemoteEvent` (one child). An earlier read showing
  `UnreplicatedEvent` was a garbled response, not real.
- A single `n=0` from `getgc` is a failed call, not proof of absence.

## Dead ends (do not retry)

1. **`PlayerGui.CustomerSpeechUI.Adornee` reads NIL** in this VM (`adornee=nil`, `Header.Text=nil`
   on all 3 probes). The reference script derives customer ids from it; unusable here.
2. **A `HumanoidRootPart` reports `ClassName == 'Model'`.** Any "walk up until you find a Model"
   helper stops on the root part and returns the wrong instance.
3. Dishes on the counter carry **no `CustomerId`/`GroupId`** - only `EatMethod`, `Taken`,
   `Part:Head`, `UnionOperation:TestNewPlate`, `Model:Contents`. Serve cannot read its target
   from the dish.
4. `findTycon and nil or findTycoon()` - malformed, fixed in v3.6.1.

## Still unknown

1. **How to serve food to the right guest.** Serve needs `GroupId`+`CustomerId`+`FoodModel`, and
   the dish carries no ids, so the target must come from the live guest list - and the game only
   accepts the dish that guest ordered. Not solved.
2. `SendToTable` (seating) was never confirmed working in a live game. The capture exists
   (`Name="SendToTable"`, `FurnitureModel`=table, `GroupId`), but it has not been tested.
3. Whether `TeleportRequestedClient` is a BindableEvent (`:Fire`) or a remote (`:FireServer`).
4. Whether `TakeOrder` also needs `CustomerActionReplicated:FireServer(tycoon,
   'SetCustomerState', customerId, groupId, 'WaitingForDish')`. The reference script fires it;
   the user's capture shows only `TaskCompleted`. v6.2.0 sends only what the capture shows.

## Script versions

- v1.0.0 keyword scan: no-op. v2.x Dialog serve: fired world NPC dialogue.
- v3.x: SeatPrompt / held-dish / SendToTable attempts, none confirmed.
- v4.0.0: **died at load** - `Cannot use '...' outside of a vararg function` (a `...` inside a
  nested closure in `fire()`). Fixed in v4.0.1 with `packArgs` + `unpackFn(a, 1, a.n)`.
- v4.1.x / v5.x: diagnostics and auto-serve attempts; none worked (customers never found).
- v6.0.0: auto money. **Crashed the user's game.**
- v6.0.1: fixed the Interactions folder, ClassName part lookup, and call rate. **CONFIRMED
  WORKING** - bills + tips.
- v6.1.0: adds auto dish pickup (`CollectDishes`, single call). **CONFIRMED WORKING.**
- v6.2.0: adds auto take order (`TakeOrder`, single call, string ids from `ClientCustomers`,
  round-robin with a 6 s per-guest cooldown).
