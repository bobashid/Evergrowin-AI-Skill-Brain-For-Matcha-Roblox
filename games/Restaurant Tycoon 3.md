# Restaurant Tycoon 3

Place id **119048529960596**. `game.Name` reports **`Ugc`**, so Hub pins this place in `PLACES`
(done, Hub v3.0). Script: `Restaurant Tycoon 3` in `bobashid/roblox-matcha`.

## CONFIRMED WORKING by the user

- v6.0.1: bills + tips - "nice everything there works" (2026-10-27).
- v6.1.0: dishes - "nice everything works the auto dish pickup and cash+tips".
- v6.2.0: orders - "nice everything worked once again".
- v6.8.0: seating + tray pickup - "nice everything functional".
- v6.8.0 is the user's baseline: "this was the most functional do a roll back just fix the
  handing out food problem" (2026-10-31).
- v6.9.0 (pickup-then-re-read): **pickup works, hand-out still does not** - user: "it picks
  them up well but not giving" (2026-10-31). This is what killed the read-order theory.

## THE DECOMPILE THAT SETTLES IT (2026-10-31) - serve is not a remote at all

Two modules decompiled live this session. Read both before touching serve again.

### `Source.Systems.Restaurant.GrabFood.AttemptGrab(tycoon, foodModel)`

    if not p32:GetAttribute("Taken") then
        local v34 = p32.Name                        -- FoodKey IS the model's Name
        local v35, v36 = Events.Restaurant.GrabFood:InvokeServer(p32)
        if v35 then
            table.insert(Storage, { Model = p32, FoodKey = v34, Tycoon = p31,
                TargetPlayerName = v36.TargetPlayerName,
                TargetCarId      = v36.TargetCarId,
                KidFlag          = v36.KidFlag })
            ... Tasks:SetTask({ Name = Task.Serve, Tycoon = p31 })

and `IsHoldingFoodForNPC(foodKey, kidFlag)` accepts an entry only when
`FoodKey == order` and the target fields match. Consequences:

- **The tray and the target come from the server's `GrabFood:InvokeServer` return.** No script
  can write that list, so no payload we build can ever be "held".
- `TaskCompleted 'Serve'` is not a serve call. **This is why 6.4.0-7.1.0 and 6.9.0 all picked up
  fine and handed out nothing.** The payload was never the problem, and there is no payload fix.
- `Objects.Food` children are the plates; the grabbed ones carry a `Taken` attribute. Tray count
  = children with `Taken == true`. Do not count them as "food on counter".

### `Source.Systems.Restaurant.Tasks`

- Serving fires `Tasks.TaskInputReceived`, which is fired **only** by
  `Tasks:HandleInput(part, position)`.
- `HandleInput` is reached from `OnInputBegan` (a real `UserInputService.InputBegan`
  MouseButton1/Touch raycast, excluded against the character, CollisionGroup `Tasks`) or from
  `OnSpeechBubbleClicked`.
- `HandleInput` rejects a target when `dx^2 + dz^2 > 1024` or `|dy| > 32` relative to the
  player's `HumanoidRootPart`, and has a 0.25 s debounce.
- `HandleInput` then classifies the hit instance: player / customer (`GetCustomer`) / food /
  furniture / car, and fires `TaskInputReceived` with the matching ids. For a customer it passes
  `GroupId`, `CustomerId`, `Tycoon` straight from the instance tree - we never supply them.

**So the only working serve route is: grab via `fireproximityprompt`, then aim the camera at a
customer and click.** That is what v6.10.0 does.

## `require` and `getgc` cannot reach these modules (probed 2026-10-31)

    GF require ok=false   SF require ok=false   Tasks require ok=false

`require(PlayerScripts.Source...GrabFood)` fails (matches `api-memory.md`: require cannot load
instances), so `Storage` cannot be read or written from a script. And
`getgc('TaskInputReceived')` returns 12 entries whose values are strings while `getgc()` with no
key returns **0** entries in this VM, so the Signal cannot be fired directly. Both routes dead.

## The serve chain in full (decompile-confirmed)

1. `Interactions:Add(tycoon, promptData)` creates a `ProximityPrompt` named `promptData.Id` on the
   temp part in `workspace.Temp`, `Enabled = (Settings:Get(CustomerPrompts) ~= false or not
   IsCustomerPrompt)`. A disabled prompt cannot be fired.
2. `Interact` early-returns if `Tasks:IsBlocked()`, if the task is `SendToTable` or `Cook`, or if
   the task is `Serve` and `Key == "Cook"`; it also has a 0.5 s debounce.
3. `GrabFood` on the LOCAL `Interacted` signal, `Key == "Serve"`: iterates `Objects.Food`
   children without `Taken`, keeps those with `|dx| + |dz| < 4` of the prompt's model pivot, then
   `AttemptGrab` each. Hold capacity `WaiterCart and 4 or WaiterTray and 3 or 1`.
4. Serving: `ServeFood` on `TaskInputReceived` requires `CurrentTask.Name == Serve`, then
   `CompleteForNPC` checks the customer is `WaitingForDish` **and**
   `IsHoldingFoodForNPC(order, isKid)` - i.e. `FoodKey == order`.

## Counters are `K<n>`, tables are `T<n>` - never mix them (Confirmed by log)

    Surface children: T19, T19, T1, T1, K3, K16, K28, K28, K16, K3, T1, T1, T1

`K3`/`K16`/`K28` are kitchen counters; `T1`/`T19` are tables. v6.7.0 logged
`seated: group 24 -> K3` because `freeTables()` accepted any `Items.Surface` child. Seating must
require `^T%d+$`. Counters are only the pickup target (`^K%d+$`).

## There is NO per-dish pickup prompt - only the GROUP (tray) pickup

User-stated: "there is no prompt for singular food pickup only group pickup". v6.7.0 filtered
prompts on `ActionText == 'Serve'` and logged `prompt: 0`. Remove the filter:
`fireproximityprompt` on **every** `ProximityPrompt` under `workspace.Temp`. Probe 2026-10-31
found **5** prompts there while a tray was held.

## `Prompt` is an INDEX BY LIVE PROMPT ID (Confirmed by probe)

    W2 temp Folder 2 / W3 part Part 1 / W4 ProximityPrompt 176 / W5 total 1

The id is not stable (`140`, `172`, `176` seen). Never hardcode it.

## Guests accept the WRONG order - user-stated, but the code disagrees

User: "they dont care if you give them the wrong order btw". `ServeFood.CompleteForNPC` calls
`GrabFood:IsHoldingFoodForNPC(order, isKid)` which compares `FoodKey == order`. With the
decompile in hand, treat order matching as **required**.

## Call shapes DIFFER per action - never assume uniformity

| Action | Calls | Target / args |
|---|---|---|
| Collect bill | `Interacted` **THEN** `TaskCompleted` (2) | `FurnitureModel` = `Items.Surface` with a `Bill` child |
| Collect dishes | `TaskCompleted` only (1) | `FurnitureModel` = `Items.Surface` with a `Trash` child |
| Collect tips | `TipsCollected` only (1) | the tycoon itself |
| Take order | `TaskCompleted` only (1) | `GroupId` + `CustomerId`, both STRINGS |
| Seat a guest | `TaskCompleted` only (1) | `FurnitureModel` = a `T<n>` table, `GroupId` (string) |
| Serve food | **no remote exists** - grab prompt, then aim + mouse click | - |

## Seating: the guest prompt fires NOTHING (User-stated)

Only clicking an empty table fires the single `TaskCompleted 'SendToTable'`.

    TaskCompleted:FireServer({FurnitureModel = Items.Surface.T1, GroupId = "15",
                              Tycoon = tycoon, Name = "SendToTable" })

## `Interactions` is a FOLDER, not a remote

Children: `RemoteEvent Interacted`, `InteractionCreated`, `InteractionDestroyed`,
`AllInteractionsDestroyed`, `ClickableFurnitureInteracted`, `BindableEvent
ClickableFurnitureInteractedClient`. Resolve `Interacted` **inside** it.

## `Events.Restaurant` contents (probed 2026-10-31)

    Folder:Interactions, RemoteFunction:GrabFood, RemoteEvent:TaskCompleted,
    RemoteEvent:CancelGrabRequested, Folder:GameStart, Folder:FoodMenu, Folder:Workers,
    RemoteEvent:RestaurantNameUpdated, RestaurantStatusUpdated, TipsCollected,
    SetMusicRequested, BuyMusicRequested, CustomerActionReplicated, FocusChanged,
    GrabEnded, RemoteFunction:GetOwnedMusic, DriveThruActionReplicated,
    DriveThruStatusUpdated, Folder:Customers, GetOwnedCosmetics,
    DeliveryActionReplicated, DeliveryCancelRequested, MopedSpawnRequested, MopedChanged,
    MopedHornRequested

`CancelGrabRequested` is the game's own "Cancel" button - use it to unstick a held tray, and do
not spam it (it clears `Storage` and resets the task).

## Guest source: `Tycoon.ClientCustomers`

Folder name = `GroupId`, child model name = `CustomerId`, both opaque strings - **never** validate
with `tonumber()`; it silently dropped every customer in v4.1.1. Empty is normal when idle.

**Customers carry NO attributes** (probed 2026-10-31: group 87 and group 89 both had zero
attributes, and the customer models have zero too). Their order / state is NOT readable from the
instance tree; it lives in the server-side `Customers` module. Position must be read from
`HumanoidRootPart` (or `Head`) `Position`, not from attributes.

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
12. **`getgc` cannot reach `TaskInputReceived`**, and `require` cannot load the modules.
13. Reading `Objects.Food` before the grab (v6.8.0) or after it (v6.9.0) - **neither matters**,
    because `TaskCompleted 'Serve'` is not a serve call at all.

## Still unknown

1. Whether Matcha's `mouse1click` reaches `UserInputService.InputBegan` while the Roblox window
   is focused. v6.10.0's Report prints `rbxActive`, `clicks`, `inRange` and the tray count so the
   next message can be a diagnosis.
2. Whether `CFrame.lookAt` exists in this VM (v6.10.0 tries `lookAt`, then `CFrame.new(eye,pos)`,
   and prints `aimMode`).
3. Whether the click raycast (CollisionGroup `Tasks`, character excluded) hits the customer model
   or the table in front of it.

## Script versions

- v1.0.0 keyword scan: no-op. v2.x Dialog serve: fired world NPC dialogue.
- v4.0.0: **died at load** - `Cannot use '...' outside of a vararg function`. Fixed in v4.0.1.
- v6.0.0: auto money. **Crashed the user's game.**
- v6.0.1: Interactions folder, ClassName parts, call rate. **WORKING** bills + tips.
- v6.1.0: + dishes. **WORKING.** v6.2.0: + orders. **WORKING.**
- v6.8.0: seat only `^T%d+$`, tray pickup without text filter. **WORKING - the baseline.**
- v6.9.0: 6.8.0 + pickup-then-re-read. Pickup fine, **hand-out still broken** (user).
- v6.10.0 (2026-10-31): serve replaced by the game's own path - `fireproximityprompt` grab, then
  camera aim + `mouse1click` per in-range customer, tray count watched via the `Taken`
  attribute. `TaskCompleted 'Serve'` removed entirely. Pushed and read back; **not yet
  confirmed in game**.
- v7.1.0 / v8.0.0: multi-target per pass, split budgets, click-serve. **Discarded** by the user
  on 2026-10-31 in favour of the 6.8.0 line.
