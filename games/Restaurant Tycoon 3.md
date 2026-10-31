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
- **v6.8.0/6.9.0 serve DOES hand out food** - "that version i said that worked did work, just
  tweak it so it works all the time" (2026-10-31).
- v6.11.0: pickup good, but "it felt like it took forever to give the food or like it never even
  gave it out". **That report was correct and it named the bug** - see below.
- v6.12.0 (2026-10-31): serve acceptance read from the tray. Pushed and read back;
  **not yet confirmed in game**.

## SERVE: `TaskCompleted 'Serve'` WORKS - do not "fix" it again (2026-10-31)

Twice I declared the serve path impossible from source reading and was wrong.

**What the decompile does and does not prove.** `Source.Systems.Restaurant.GrabFood.AttemptGrab`
fills its local `Storage` from `Events.Restaurant.GrabFood:InvokeServer(model)`'s return, and
`Source.Systems.Restaurant.Tasks` fires serving from `Tasks:HandleInput` via a real
`UserInputService` MouseButton1 raycast. That is how the **game's own client** does it. It does
**not** prove the server rejects a `TaskCompleted` call named `'Serve'` - and it does not, as
the user's live report shows.

**Rule for this game:** `TaskCompleted 'Serve'` with `FoodModel` + `GroupId` + `CustomerId` is
accepted. Keep it. Never replace it with a camera-click route on the strength of source reading.

## THE ACTUAL BUG IN v6.11.0: `fire()` success is not server acceptance

This is the lesson worth carrying to every game. `pcall(function() r:FireServer(payload) end)`
returns true when the call was **sent**. A RemoteEvent almost never errors. So in 6.11.0:

    local ok = serveToGuest(...)   -- true on the FIRST attempt, always
    if ok then landed = true end   -- retry loop dead on arrival

`GuestTriesPerDish = 3` was never actually used: one blind attempt per dish per pass, aimed at
whatever guest `serveIdx` happened to point at. When that guest was not waiting for that dish
the server refused it silently, the plate stayed in hand, and the next pass tried someone else.
That is precisely "works sometimes and feels slow".

**Fix (v6.12.0):** the witness is the world, not the return value.

    local function stillOnTray(ty, dish)
        return dish.Parent ~= nil and dish:IsDescendantOf(foodFolder(ty))
    end
    -- fire at a guest, wait, then: if not stillOnTray(...) -> accepted

A plate that leaves `Tycoon.Objects.Food` is the only proof the hand-off landed. Each dish is
retried on up to `GuestAttemptsPerDish = 8` different guests until that test passes.

Also in v6.12.0: serve runs in its **own** `task.spawn` loop (a long serve pass could previously
stall bills/dishes/orders/seating, which was the "everything feels slow" part), the payload is
**exactly** the 6.8.0 one (`FoodName` and `FurnitureModel` were guesses added in 6.9.0/6.11.0 and
are removed), and the fixed 0.7 s cadence became 0.15 s after a success / 0.4 s after a miss.

## `GrabFood` internals (decompiled 2026-10-31)

- `AttemptGrab` marks a plate `Taken`, calls `GrabFood:InvokeServer(model)`, pushes
  `{Model, FoodKey = model.Name, Tycoon, TargetPlayerName, TargetCarId, KidFlag}` onto
  `Storage`, then `Tasks:SetTask({Name = Task.Serve})`.
- `IsHoldingFoodForNPC(foodKey, kidFlag)` compares `FoodKey == order` - the order is the plate's
  **model Name** (`food 13`, `food 2`, `food 10` were live).
- Hold capacity = `WaiterCart and 4 or WaiterTray and 3 or 1`. A fresh account holds **one**
  plate, which alone can look like "serve is broken".
- `Objects.Food` children are the plates; grabbed ones carry `Taken`.
- The bottom bar ("Serve Chicken Nuggets to a customer" + Cancel) is `GrabFood:UpdateTaskBottomBar`.

## `require` and `getgc` cannot reach these modules (probed 2026-10-31)

    GF require ok=false   SF require ok=false   Tasks require ok=false

`require(PlayerScripts.Source...GrabFood)` fails (matches `api-memory.md`), so `Storage` cannot
be read from a script. `getgc('TaskInputReceived')` returns 12 entries whose values are strings
while `getgc()` with no key returns **0** entries in this VM. Do not build on either.

## The serve chain in full (decompile-confirmed)

1. `Interactions:Add(tycoon, promptData)` creates a `ProximityPrompt` named `promptData.Id` on the
   temp part in `workspace.Temp`, `Enabled = (Settings:Get(CustomerPrompts) ~= false or not
   IsCustomerPrompt)`. A disabled prompt cannot be fired.
2. `Interact` early-returns if `Tasks:IsBlocked()`, if the task is `SendToTable` or `Cook`, or if
   the task is `Serve` and `Key == "Cook"`; it also has a 0.5 s debounce.
3. `GrabFood` on the LOCAL `Interacted` signal, `Key == "Serve"`: iterates `Objects.Food`
   children without `Taken`, keeps those with `|dx| + |dz| < 4` of the prompt's model pivot, then
   `AttemptGrab` each.

## Counters are `K<n>`, tables are `T<n>` - never mix them (Confirmed by log)

    Surface children: T19, T19, T1, T1, K3, K16, K28, K28, K16, K3, T1, T1, T1

`K3`/`K16`/`K28` are kitchen counters; `T1`/`T19` are tables. Seating must require `^T%d+$`.
Counters are only the pickup target (`^K%d+$`).

## There is NO per-dish pickup prompt - only the GROUP (tray) pickup

User-stated: "there is no prompt for singular food pickup only group pickup". Remove any
`ActionText` filter: `fireproximityprompt` on **every** `ProximityPrompt` under
`workspace.Temp`. Probe 2026-10-31 found **5** prompts there while a tray was held.

## `Prompt` is an INDEX BY LIVE PROMPT ID (Confirmed by probe)

The id is not stable (`140`, `172`, `176` seen). Never hardcode it.

## Guests and orders - treat matching as required (decompile)

`ServeFood.CompleteForNPC` calls `GrabFood:IsHoldingFoodForNPC(order, isKid)`, which compares
`FoodKey == order`. The user says wrong orders are accepted; the code says the plate's Name must
equal the order. v6.12.0 retries a dish on several guests, which covers both readings.

## Call shapes DIFFER per action - never assume uniformity

| Action | Calls | Target / args |
|---|---|---|
| Collect bill | `Interacted` **THEN** `TaskCompleted` (2) | `FurnitureModel` = `Items.Surface` with a `Bill` child |
| Collect dishes | `TaskCompleted` only (1) | `FurnitureModel` = `Items.Surface` with a `Trash` child |
| Collect tips | `TipsCollected` only (1) | the tycoon itself |
| Take order | `TaskCompleted` only (1) | `GroupId` + `CustomerId`, both STRINGS |
| Seat a guest | `TaskCompleted` only (1) | `FurnitureModel` = a `T<n>` table, `GroupId` (string) |
| Serve food | `fireproximityprompt` (tray) THEN `TaskCompleted 'Serve'` per dish | `FoodModel`, `Name`, `Tycoon`, `GroupId`, `CustomerId` - nothing else |

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

`CancelGrabRequested` is the game's own "Cancel" button - it clears `Storage` and resets the
task. Use it to unstick a held tray; do not spam it.

## Guest source: `Tycoon.ClientCustomers`

Folder name = `GroupId`, child model name = `CustomerId`, both opaque strings - **never** validate
with `tonumber()`; it silently dropped every customer in v4.1.1. Empty is normal when idle.

**Customers carry NO attributes** (probed 2026-10-31: groups 87 and 89 had zero attributes, and
the customer models had zero too). Their order / state is NOT readable from the instance tree.
Position comes from the `HumanoidRootPart` / `Head` part, never from attributes.

## The real cause of "slow": one attempt per pass (User-diagnosed 2026-10-27)

"you try the task once, you need to spam them on different objectives to make it faster".
Fix = try many targets per pass, and give the heaviest feature its **own** call budget (a single
shared bucket starves it). v6.12.0 also gives serve its own **thread**, not just its own budget.

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
    on the LOCAL signal inside `Interact`. Genuine; it is about the *grab*, not the serve.
12. **`getgc` cannot reach `TaskInputReceived`**, and `require` cannot load the modules.
13. Camera-aim + `mouse1click` serve (v6.10.0) - discarded, never needed.
14. Treating `fire()`'s pcall result as "the server accepted it" (v6.11.0) - this is the bug that
    made every retry loop a no-op. Read the tray instead.

## Script versions

- v1.0.0 keyword scan: no-op. v2.x Dialog serve: fired world NPC dialogue.
- v4.0.0: **died at load** - `Cannot use '...' outside of a vararg function`. Fixed in v4.0.1.
- v6.0.0: auto money. **Crashed the user's game.**
- v6.0.1: Interactions folder, ClassName parts, call rate. **WORKING** bills + tips.
- v6.1.0: + dishes. **WORKING.** v6.2.0: + orders. **WORKING.**
- v6.8.0: seat only `^T%d+$`, tray pickup without text filter. **WORKING - the baseline.**
- v6.9.0: 6.8.0 + pickup-then-re-read. Serve works but only intermittently.
- v6.10.0: click-path serve. **Discarded** - replaced a working call with a guess.
- v6.11.0: own call budget + 3 guest retries - but the retry loop was dead because `fire()`
  always returned true. User: "felt like it took forever... or like it never even gave it out".
- v6.12.0 (2026-10-31): acceptance read from `Objects.Food` (plate leaving = accepted), up to 8
  guest attempts per dish, own serve thread, exact 6.8.0 payload, 0.15 s after a hit.
  Pushed and read back; **not yet confirmed in game**.
- v7.1.0 / v8.0.0: multi-target per pass, split budgets, click-serve. **Discarded** by the user.
