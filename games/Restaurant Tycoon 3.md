# Restaurant Tycoon 3

Place id **119048529960596**. `game.Name` reports **`Ugc`**, so Hub pins this place in `PLACES`
(done, Hub v3.0). Script: `Restaurant Tycoon 3` in `bobashid/roblox-matcha`.

## CONFIRMED WORKING by the user

- v6.0.1: bills + tips - "nice everything there works" (2026-10-27).
- v6.1.0: dishes - "nice everything works the auto dish pickup and cash+tips".
- v6.2.0: orders - "nice everything worked once again".
- v6.8.0: seating + tray pickup - "nice everything functional".

## WHY SERVE CANNOT BE DONE BY FIRING REMOTES (decompile-confirmed 2026-10-27)

This is the most important finding on this game. Read it before touching serve again.

`Source.Systems.Restaurant.Interactions.Interact(tycoon, promptData)` is the client's own
dispatcher. It does **TWO** things, and the order matters:

    p18.Interacted:Fire(p19, p20)          -- LOCAL Signal -> GrabFood listens to this
    v_u_13:FireServer(p19, p20)            -- the remote we captured

`GrabFood.Init` does `v_u_10.Interacted:Connect(function(tycoon, promptData) ... end)` and only
puts food in `Storage` from that **local signal**. So:

- Firing `Interactions.Interacted:FireServer(tycoon, promptData)` from a script tells the SERVER
  something happened, but **never runs GrabFood**, so `Storage` stays empty.
- With empty `Storage`, `IsHoldingFoodForNPC` returns nil, so `ServeFood.CompleteForNPC` returns
  false, and every `TaskCompleted 'Serve'` is refused. This is why v6.4.0-v7.1.0 all failed at
  serve no matter how correct the payload was. **The payload was never the problem.**
- `prompt.Triggered` -> `Interact` -> both calls. So **`fireproximityprompt` is the only route
  that updates local state**, because it goes through the game's own dispatcher.

## The `getgc` route for serving is PROVEN DEAD (probed 2026-10-27)

Serving needs `Tasks.TaskInputReceived`, a Packets.Signal. Probe results:

    K2 getgc('TaskInputReceived') ok=true n=12
    K3 1 string "Player" / 2 string func(0x...) / 4 string "Connect" ...
    K4 nonnil=12   (values are STRINGS, never the Signal object)
    K5 getgc('HandleInput') n=6
    K6 getgc('ServeFood') n=5
    L2 getgc() ok=true n=0      <- whole-heap scan returns NOTHING in this VM

So there is no way to reach the Tasks table or its Signal. Do not retry this.

## The serve chain in full (decompile-confirmed)

1. `Interactions:Add(tycoon, promptData)` creates a `ProximityPrompt` named `promptData.Id`,
   parented to `promptData.Part` (the temp part in `workspace.Temp`), with
   `MaxActivationDistance = Range or 15`, `HoldDuration = promptData.HoldDuration or 0`, and
   **`Enabled = (Settings:Get(CustomerPrompts) ~= false or not IsCustomerPrompt)`**.
   A disabled prompt cannot be fired - if customer prompts are off, nothing works.
2. `Interact` early-returns if `Tasks:IsBlocked()`, or if the current task is `SendToTable`
   or `Cook`, or if the task is `Serve` and `promptData.Key == "Cook"`. It also has a **0.5 s
   debounce** (`v_u_10`) around the whole body.
3. `GrabFood` on the local signal, `Key == "Serve"`: iterates `tycoon.Objects.Food` children with
   no `Taken` attribute, and grabs those whose pivot is within **manhattan distance < 4** of
   `promptData.Model`'s pivot. Then it sets `Tasks:SetTask({Name = Task.Serve, Tycoon = tycoon})`.
4. Hold capacity: `WaiterCart and 4 or WaiterTray and 3 or 1`.
5. Serving: `ServeFood.Init` listens to `Tasks.TaskInputReceived`, requires
   `CurrentTask.Name == Task.Serve`, then `CompleteForNPC(tycoon, groupId, customerId)` checks
   `Customers:GetCustomerData(...).State == CustomerState.WaitingForDish` **and**
   `GrabFood:IsHoldingFoodForNPC(order, isKid)` - which compares `FoodKey == order`.
6. `TaskInputReceived` is fired by `Tasks:HandleInput(part, position)`, called from
   `OnInputBegan` (a mouse click raycast) or `OnSpeechBubbleClicked`. `HandleInput` rejects
   targets where `dx^2 + dz^2 > 1024` or `|dy| > 32`, and has a 0.25 s debounce.

**Consequence:** automating serve means driving the client's own input path:
`fireproximityprompt` to grab, then a camera-aim + mouse click per customer. That is what
v8.0.0 does. Still UNVERIFIED in a live run.

## Counters are `K<n>`, tables are `T<n>` - never mix them (Confirmed by log)

    Surface children: T19, T19, T1, T1, K3, K16, K28, K28, K16, K3, T1, T1, T1

`K3`/`K16`/`K28` are kitchen counters; `T1`/`T19` are tables. v6.7.0 logged
`seated: group 24 -> K3` because `freeTables()` accepted any `Items.Surface` child. Seating must
require `^T%d+$`. Counters are only the pickup target (`^K%d+$`).

## There is NO per-dish pickup prompt - only the GROUP (tray) pickup

User-stated: "there is no prompt for singular food pickup only group pickup". v6.7.0 filtered
prompts on `ActionText == 'Serve'` and logged `prompt: 0`. Remove the filter:
`fireproximityprompt` on **every** `ProximityPrompt` under `workspace.Temp`.

## `Prompt` is an INDEX BY LIVE PROMPT ID (Confirmed by probe)

    W2 temp Folder 2 / W3 part Part 1 / W4 ProximityPrompt 176 / W5 total 1

The id is not stable (`140`, `172`, `176` seen). Never hardcode it.

## Guests accept the WRONG order - user-stated, but the code disagrees

User: "they dont care if you give them the wrong order btw". `ServeFood.CompleteForNPC` calls
`GrabFood:IsHoldingFoodForNPC(order, isKid)` which compares `FoodKey == order`. Unresolved;
treat order matching as **required** until proven otherwise.

## Call shapes DIFFER per action - never assume uniformity

| Action | Calls | Target / args |
|---|---|---|
| Collect bill | `Interacted` **THEN** `TaskCompleted` (2) | `FurnitureModel` = `Items.Surface` with a `Bill` child |
| Collect dishes | `TaskCompleted` only (1) | `FurnitureModel` = `Items.Surface` with a `Trash` child |
| Collect tips | `TipsCollected` only (1) | the tycoon itself |
| Take order | `TaskCompleted` only (1) | `GroupId` + `CustomerId`, both STRINGS |
| Seat a guest | `TaskCompleted` only (1) | `FurnitureModel` = a `T<n>` table, `GroupId` (string) |
| Serve food | **not remotely fireable** - prompt grab then camera click | - |

## Seating: the guest prompt fires NOTHING (User-stated)

Only clicking an empty table fires the single `TaskCompleted 'SendToTable'`.

    TaskCompleted:FireServer({FurnitureModel = Items.Surface.T1, GroupId = "15",
                              Tycoon = tycoon, Name = "SendToTable" })

## `Interactions` is a FOLDER, not a remote

Children: `RemoteEvent Interacted`, `InteractionCreated`, `InteractionDestroyed`,
`AllInteractionsDestroyed`, `ClickableFurnitureInteracted`, `BindableEvent
ClickableFurnitureInteractedClient`. Resolve `Interacted` **inside** it.

## Guest source: `Tycoon.ClientCustomers`

Folder name = `GroupId`, child model name = `CustomerId`, both opaque strings - **never** validate
with `tonumber()`; that silently dropped every customer in v4.1.1. Empty is normal when idle.

## The real cause of "slow": one attempt per pass (User-diagnosed 2026-10-27)

"you try the task once, you need to spam them on different objectives to make it faster".
One rejected target cost a whole cooldown window. Fix = try many targets per pass, and give the
heaviest feature its **own** call budget (a single shared bucket starves it).

## Crash post-mortem: v6.0.0

1. `remote('Restaurant','Interactions')` returned the **Folder**; `Folder:FireServer(...)`
   attempted - the first bill call never fired.
2. `FindFirstChildWhichIsA('BasePart', true)` for part lookup. `IsA` always returns true here,
   so it can return a non-part. **Find parts by `ClassName` only**.
3. **No rate limit** - up to 12 bills x 2 calls every 0.25 s.
4. `task.wait` inside the collect loop.

## Dead ends (do not retry)

1. `PlayerGui.CustomerSpeechUI.Adornee` reads **NIL** here. Use `ClientCustomers`.
2. A `HumanoidRootPart` reports `ClassName == 'Model'`.
3. Dishes carry **no `CustomerId`/`GroupId`**.
4. `findTycon and nil or findTycoon()` - malformed, fixed in v3.6.1.
5. Clicking the guest's own prompt fires **no remote**.
6. "It fires no events" for the kitchen pickup was **wrong** - it does.
7. Hand-firing the pickup `Interacted` from across the map does not work - range gated.
8. Filtering pickup prompts on `ActionText == 'Serve'` matches nothing.
9. Seating on any `Items.Surface` child puts groups on kitchen counters.
10. One attempt per pass + a long cooldown = 3-7 s stalls.
11. **Firing `Interacted:FireServer` alone never fills `GrabFood.Storage`** - the grab happens
    on the LOCAL signal inside `Interact`.
12. **`getgc` cannot reach `TaskInputReceived`.** `getgc('TaskInputReceived')` returns 12 entries
    whose values are strings; `getgc()` with no key returns 0 entries in this VM, so the Tasks
    table cannot be found. Firing the Signal from a script is impossible.

## Still unknown

1. Whether the camera-aim + click serve route actually lands (raycast filter, camera fight).
2. Whether the server re-validates the same preconditions as the client `ServeFood` module.
3. Whether `TeleportRequestedClient` is a BindableEvent (`:Fire`) or a remote (`:FireServer`).

## Script versions

- v1.0.0 keyword scan: no-op. v2.x Dialog serve: fired world NPC dialogue.
- v4.0.0: **died at load** - `Cannot use '...' outside of a vararg function`. Fixed in v4.0.1.
- v6.0.0: auto money. **Crashed the user's game.**
- v6.0.1: Interactions folder, ClassName parts, call rate. **WORKING** bills + tips.
- v6.1.0: + dishes. **WORKING.** v6.2.0: + orders. **WORKING.**
- v6.8.0: seat only `^T%d+$`, tray pickup without text filter. **WORKING.**
- v7.1.0: multi-target per pass, split call budget, test buttons removed.
- v8.0.0: serve via the game's own input path - `fireproximityprompt` to grab (fills Storage)
  then camera-aim + `mouse1click` per customer, with `CancelGrabRequested` if nothing lands.
