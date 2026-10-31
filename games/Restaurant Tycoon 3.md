# Restaurant Tycoon 3

Place id **119048529960596**. `game.Name` reports **`Ugc`**, so Hub pins this place in `PLACES`
(done, Hub v3.0). Script: `Restaurant Tycoon 3` in `bobashid/roblox-matcha`.

## CONFIRMED WORKING by the user

- v6.0.1: bills + tips - "nice everything there works" (2026-10-27).
- v6.1.0: dishes - "nice everything works the auto dish pickup and cash+tips".
- v6.2.0: orders - "nice everything worked once again" (2026-10-27).

v6.3.0 keeps all of that code unchanged, tightens scan intervals, and adds seating on top.

## Call shapes DIFFER per action - never assume uniformity (Confirmed 2026-10-27)

| Action | Calls | Target / args |
|---|---|---|
| Collect bill | `Interacted` **THEN** `TaskCompleted` (2) | `FurnitureModel` = `Items.Surface` model holding a `Bill` child |
| Collect dishes | `TaskCompleted` only (1) | `FurnitureModel` = `Items.Surface` model holding a `Trash` child |
| Collect tips | `TipsCollected` only (1) | the tycoon itself |
| Take order | `TaskCompleted` only (1) | `GroupId` + `CustomerId`, both STRINGS |
| Seat a guest | `TaskCompleted` only (1) | `FurnitureModel` = the table, `GroupId` (string) |

Sending an extra unverified remote alongside a single-call action is how v6.0.0 crashed the
game. Only send what the capture shows.

## Seating: the guest prompt fires NOTHING (User-stated 2026-10-27)

User: "when i clicked on the prompt above their head it didnt show an event for me to send you
but it only showed the event when i pressed on an empty table after to seat them".

So the in-game flow is: click the guest's prompt (purely client-side, no remote), THEN click an
empty table, which fires the single `TaskCompleted` call. **Nothing needs to be reproduced from
the first click** - seating is one call, and only `GroupId` (from `ClientCustomers`) plus the
table model are needed. This is why seating is buildable at all.

## Captured payloads (user, Remote Spy)

**Seat a guest** (single call):

    Events.Restaurant.TaskCompleted:FireServer({
        FurnitureModel = game.Workspace.Tycoons.Tycoon.Items.Surface.T1,
        GroupId = "15",
        Tycoon = game.Workspace.Tycoons.Tycoon, Name = "SendToTable" })

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
The captured pairs (`GroupId="8"`, `CustomerId="2"`; `GroupId="15"`) match this structure.

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

**Speed tuning (v6.3.0):** the user asked for faster scanning. Loop `Gap` 0.5 -> 0.2, per-feature
intervals 0.8 -> 0.35, tips 2.0 -> 1.0, seat 0.5. The **token bucket stays at 8 calls / 0.5 s**
on purpose - speeding up the *scan* must not raise the *call* rate that caused the crash.

## Table selection for seating

A `Items.Surface` child is free when it has **no `Bill`**, **no `Trash`**, none of the attributes
`Occupied` / `IsOccupied` / `InUse` / `Taken`, and no `Seat` descendant with an `Occupant`.
Table names match `^T%d+$` (seen: `T1`, `T19`).

## Other live probe facts

- `V3 bills 0 trash 5` - five `Trash` surfaces existed at probe time. Bills/trash reading 0 is
  normal when idle.
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
5. Clicking the guest's own prompt fires **no remote** - do not look for one when seating.

## Still unknown

1. **How to serve food to the right guest.** Serve needs `GroupId`+`CustomerId`+`FoodModel`, and
   the dish carries no ids, so the target must come from the live guest list - and the game only
   accepts the dish that guest ordered. Not solved.
2. Whether `SendToTable` also needs the guest to be in a specific state first. The capture is
   real and single-call; not yet confirmed working in a live game.
3. Whether `TeleportRequestedClient` is a BindableEvent (`:Fire`) or a remote (`:FireServer`).
4. Whether `TakeOrder` also needs `CustomerActionReplicated:FireServer(tycoon,
   'SetCustomerState', customerId, groupId, 'WaitingForDish')`. The reference script fires it;
   the capture shows only `TaskCompleted`, and orders are confirmed working without it.

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
- v6.2.0: adds auto take order (`TakeOrder`, single call, string ids, round-robin + 6 s cooldown).
  **CONFIRMED WORKING.**
- v6.3.0: faster scan intervals + auto seat (`SendToTable`, single call, one group per pass,
  10 s per-group cooldown, free-table check).
