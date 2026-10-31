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
  tweak it so it works all the time" (2026-10-31). See the correction note below.
- v6.11.0 (2026-10-31): 6.9.0 serve path with a dedicated call budget and multi-guest retry.
  Pushed and read back; **not yet confirmed in game**.

## SERVE: `TaskCompleted 'Serve'` WORKS - do not "fix" it again (2026-10-31)

This is the correction of record for this game. Twice now I have declared the serve path
impossible from source reading and been wrong.

**What the decompile does and does not prove.** `Source.Systems.Restaurant.GrabFood.AttemptGrab`
fills its local `Storage` from `Events.Restaurant.GrabFood:InvokeServer(model)`'s return, and
`Source.Systems.Restaurant.Tasks` fires serving from `Tasks:HandleInput` via a real
`UserInputService` MouseButton1 raycast. That is how the **game's own client** does it. It does
**not** prove the server rejects a `TaskCompleted` call named `'Serve'` - and it does not, as
the user's live report shows.

**Rule for this game:** `TaskCompleted 'Serve'` with `FoodModel` + `GroupId` + `CustomerId` is
accepted. Keep it. Never replace it with a camera-click route on the strength of source reading.

**Why it was intermittent, and the actual fix (v6.11.0):**

1. Serve shared **one** token bucket (8 calls / 0.5 s) with bills, dishes, orders and seating.
   A busy kitchen starved the serve calls. Serve now has its own bucket (`ServeBudget = 10`).
2. Each dish got **exactly one** guest attempt, then an 8 s park. Now each dish is retried on up
   to `GuestTriesPerDish = 3` different guests, and the cooldown is keyed per **dish+guest pair**
   (3 s), so one refusal does not block a dish against everybody else. A dish that fails every
   try is parked only 3 s.
3. Cadence: `ServeEvery` 1.2 -> 0.7, `ServeGap` 0.12 -> 0.1, `MaxServesPerPass` 6.
4. Tray-aware: if `Objects.Food` children carry a `Taken` attribute, only those are served;
   otherwise every plate is tried. Pickup only fires when nothing is held.

Nothing about the payload changed - the payload was correct in 6.8.0.

## `GrabFood` internals (still useful, decompiled 2026-10-31)

- `AttemptGrab` marks a plate `Taken`, calls `GrabFood:InvokeServer(model)`, pushes
  `{Model, FoodKey = model.Name, Tycoon, TargetPlayerName, TargetCarId, KidFlag}` onto
  `Storage`, then `Tasks:SetTask({Name = Task.Serve})`.
- `IsHoldingFoodForNPC(foodKey, kidFlag)` compares `FoodKey == order` - so the customer's order
  is that plate's **model Name** (`food 13`, `food 2`, `food 10` were the live names).
- Hold capacity = `WaiterCart and 4 or WaiterTray and 3 or 1`. A fresh account holds **one**
  plate, which alone can look like "serve is broken" when the second dish never leaves.
- `Objects.Food` children are the plates; grabbed ones carry `Taken`.
- The bottom bar text ("Serve Chicken Nuggets to a customer" + Cancel) is
  `GrabFood:UpdateTaskBottomBar` - seeing it means the grab succeeded.

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

## Guests and orders - treat matching as required (decompile)

`ServeFood.CompleteForNPC` calls `GrabFood:IsHoldingFoodForNPC(order, isKid)`, which compares
`FoodKey == order`. The user says wrong orders are accepted; the code says the plate's Name must
equal the order. v6.11.0 therefore retries a dish on several guests instead of assuming either
way - one of the two is right and the retry covers both.

## Call shapes DIFFER per action - never assume uniformity

| Action | Calls | Target / args |
|---|---|---|
| Collect bill | `Interacted` **THEN** `TaskCompleted` (2) | `FurnitureModel` = `Items.Surface` with a `Bill` child |
| Collect dishes | `TaskCompleted` only (1) | `FurnitureModel` = `Items.Surface` with a `Trash` child |
| Collect tips | `TipsCollected` only (1) | the tycoon itself |
| Take order | `TaskCompleted` only (1) | `GroupId` + `CustomerId`, both STRINGS |
| Seat a guest | `TaskCompleted` only (1) | `FurnitureModel` = a `T<n>` table, `GroupId` (string) |
| Serve food | `fireproximityprompt` (tray) THEN `TaskCompleted 'Serve'` per dish | `FoodModel`, `GroupId`, `CustomerId` |

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

**Customers carry NO attributes** (probed 2026-10-31: group 87 and group 89 both had zero
attributes, and the customer models have zero too). Their order / state is NOT readable from the
instance tree; it lives in the server-side `Customers` module. Position must come from the
`HumanoidRootPart` / `Head` part, not from attributes.

## The real cause of "slow" and "only sometimes": one attempt per pass (User-diagnosed)

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
    on the LOCAL signal inside `Interact`. This one is genuine; it is about the *grab*, not the
    serve.
12. **`getgc` cannot reach `TaskInputReceived`**, and `require` cannot load the modules.
13. Camera-aim + `mouse1click` serve (v6.10.0) - the user discarded it and it was never needed;
    `TaskCompleted 'Serve'` works. Do not build this again.

## Script versions

- v1.0.0 keyword scan: no-op. v2.x Dialog serve: fired world NPC dialogue.
- v4.0.0: **died at load** - `Cannot use '...' outside of a vararg function`. Fixed in v4.0.1.
- v6.0.0: auto money. **Crashed the user's game.**
- v6.0.1: Interactions folder, ClassName parts, call rate. **WORKING** bills + tips.
- v6.1.0: + dishes. **WORKING.** v6.2.0: + orders. **WORKING.**
- v6.8.0: seat only `^T%d+$`, tray pickup without text filter. **WORKING - the baseline**, and
  it does hand out food.
- v6.9.0: 6.8.0 + pickup-then-re-read. Serve works but only intermittently.
- v6.10.0: click-path serve. **Discarded** - replaced a working call with a guess.
- v6.11.0 (2026-10-31): serve keeps the `TaskCompleted 'Serve'` call, gets its own call budget,
  retries each dish on up to 3 guests, per-pair 3 s cooldown, faster cadence. Pushed and read
  back; **not yet confirmed in game**.
- v7.1.0 / v8.0.0: multi-target per pass, split budgets, click-serve. **Discarded** by the user.
