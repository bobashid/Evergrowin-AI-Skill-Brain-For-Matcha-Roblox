# Restaurant Tycoon 3

Place id **119048529960596**. `game.Name` reports **`Ugc`**, so Hub pins this place in `PLACES`
(done, Hub v3.0). Script: `Restaurant Tycoon 3` in `bobashid/roblox-matcha`.

## CONFIRMED WORKING by the user

- v6.0.1: bills + tips - "nice everything there works" (2026-10-27).
- v6.1.0: dishes - "nice everything works the auto dish pickup and cash+tips".
- v6.2.0: orders - "nice everything worked once again".
- v6.8.0: seating + tray pickup - "nice everything functional".

## The real cause of "slow": one attempt per pass, then a cooldown (User-diagnosed 2026-10-27)

User: "you try the task once, you need to spam them on different objectives to make it faster,
for example you held a food for a long time there was only one person to give it to, noone was
requesting food out of 10 yet you took a long time; also for seating it had not enough seats,
then you waited for around 7 seconds to retry"

So v6.9.0's per-feature cooldown was the bug: ONE rejected target cost a whole cooldown window.
**Fix: try many targets per pass.**

- serve: pickup once, then for each dish try up to `ServeTriesPerDish` (3) guests, breaking when
  `dish.Parent == nil` (the dish left the counter).
- seat: up to `SeatTriesPerPass` (8) `(group, table)` PAIRS per pass, with the cooldown keyed on
  `"group>table"` so one busy table never blocks a different pair.
- `SeatCooldown` 10 -> 2 s; `ServeEvery` 0.5 -> 0.35 s; `SeatEvery` 0.25 -> 0.2 s.

**A single shared rate limit also starves the heaviest feature.** v6.8.0 had one bucket of
8 calls / 0.5 s for everything, so orders+seating drained it and `serveToGuest()` kept returning
"rate capped" - the dish stayed in hand. v6.9.0 split it into two buckets; v7.0.0 raised them
to serve 8 / other 12 per 0.5 s (~40 calls/s worst case). Reference points: 16/s was too slow,
~96/s crashed the game.

## Guests accept the WRONG order (User-stated 2026-10-27)

"they dont care if you give them the wrong order btw". No dish->guest matching exists. Any dish
to any guest. This retires the old "how to match dish to guest" unknown entirely.

## Counters are `K<n>`, tables are `T<n>` - never mix them (Confirmed by log)

    Surface children: T19, T19, T1, T1, K3, K16, K28, K28, K16, K3, T1, T1, T1

`K3`/`K16`/`K28` are kitchen counters; `T1`/`T19` are tables. v6.7.0 logged
`seated: group 24 -> K3` because `freeTables()` accepted any `Items.Surface` child. Seating must
require `^T%d+$`. Counters are only the pickup target (`nearestCounter`, `^K%d+$`).

## There is NO per-dish pickup prompt - only the GROUP (tray) pickup

User-stated: "there is no prompt for singular food pickup only group pickup i sent it to you
multiple times". v6.7.0 filtered prompts on `ActionText == 'Serve'` and logged `prompt: 0`. Remove
the filter: `fireproximityprompt` on **every** `ProximityPrompt` under `workspace.Temp`. Because
it is a tray pickup, several dishes are handed out per pass.

## Kitchen pickup is RANGE/PROMPT gated, not payload gated

The captured pickup carries `Range = 9` and a `WorldPosition`, and the prompt only exists at
`workspace.Temp.Part` while standing near a counter. Hand-building the `Interacted` table from
across the map is refused. **Preferred route: `fireproximityprompt`**; the captured `Interacted`
call is only a fallback.

Also: `oneServe()` must serve **even if the pickup fails**, or a dish already in hand is stuck.

## CORRECTION: the kitchen pickup DOES fire an event

v6.5.0 dropped the call on a wrong reading of "it fires no events" and picked nothing up. Rule:
"it fires no events for me" means *the user did not see one*, not *the game sends none*.

    Events.Restaurant.Interactions.Interacted:FireServer(tycoon, {
      HoldDuration = 0.25, Id = "172", ActionText = "Serve",
      WorldPosition = Vector3.new(1208.80, 0.27, 316.02),
      GamepadKeyCode = Enum.KeyCode.Thumbstick2, TemporaryPart = workspace.Temp.Part,
      KeyboardKeyCode = Enum.KeyCode.F, Range = 9, IsCustomerPrompt = true, Key = "Serve",
      Model = tycoon.Items.Surface.K16, Part = workspace.Temp.Part,
      Prompt = workspace.Temp.Part["172"], InteractionType = "OrderCounter" })

## `Prompt` is an INDEX BY LIVE PROMPT ID (Confirmed by probe)

    W2 temp Folder 2 / W3 part Part 1 / W4 ProximityPrompt 176 / W5 total 1

The id is not stable (`140`, `172`, `176` seen). Never hardcode it. v6.4.0 looked for
`workspace.Temp["140"]` - wrong parent AND a hardcoded id.

## Serving = pickup, then one TaskCompleted per dish (User-captured)

    TaskCompleted:FireServer({FoodModel=Objects.Food["10"], Name="Serve",
                              Tycoon=tycoon, GroupId="19", CustomerId="2"})

Same shape for `Food["7"]`->17/2 and `Food["1"]`->16/2. A dish carries no ids of its own (only
`EatMethod`, `Taken`, `Part:Head`, `UnionOperation:TestNewPlate`, `Model:Contents`).

## Call shapes DIFFER per action - never assume uniformity

| Action | Calls | Target / args |
|---|---|---|
| Collect bill | `Interacted` **THEN** `TaskCompleted` (2) | `FurnitureModel` = `Items.Surface` with a `Bill` child |
| Collect dishes | `TaskCompleted` only (1) | `FurnitureModel` = `Items.Surface` with a `Trash` child |
| Collect tips | `TipsCollected` only (1) | the tycoon itself |
| Take order | `TaskCompleted` only (1) | `GroupId` + `CustomerId`, both STRINGS |
| Seat a guest | `TaskCompleted` only (1) | `FurnitureModel` = a `T<n>` table, `GroupId` (string) |
| Serve food | `fireproximityprompt` on the tray prompt (fallback `Interacted`), THEN `TaskCompleted 'Serve'` per dish | `FoodModel` + `GroupId` + `CustomerId` |

## Seating: the guest prompt fires NOTHING (User-stated)

Only clicking an empty table fires the single `TaskCompleted 'SendToTable'`. Nothing from the
guest's own prompt needs reproducing.

    TaskCompleted:FireServer({FurnitureModel = Items.Surface.T1, GroupId = "15",
                              Tycoon = tycoon, Name = "SendToTable" })

## `Interactions` is a FOLDER, not a remote

Children include `RemoteEvent Interacted`, `InteractionCreated`, `InteractionDestroyed`,
`AllInteractionsDestroyed`, `ClickableFurnitureInteracted`, `BindableEvent
ClickableFurnitureInteractedClient`. Resolve `Interacted` **inside** it. Firing the Folder itself
throws inside pcall and is swallowed - the v6.0.0 defect and likely crash path.

## Guest source: `Tycoon.ClientCustomers`

    Folder "8" -> Model "1", Model "2"

Folder name = `GroupId`, child model name = `CustomerId`, both opaque strings - **never** validate
with `tonumber()`; that silently dropped every customer in v4.1.1. Empty is normal when idle.

## Crash post-mortem: v6.0.0

1. `remote('Restaurant','Interactions')` returned the **Folder**; `Folder:FireServer(...)` was
   attempted - the first bill call never fired.
2. `FindFirstChildWhichIsA('BasePart', true)` for part lookup. `IsA` always returns true here,
   so it can return a non-part; sending a Model where a `Part` is expected is a plausible crash
   trigger. **Find parts by `ClassName` only**, skip the target if none is found.
3. **No rate limit** - up to 12 bills x 2 calls every 0.25 s. Added a token bucket.
4. `task.wait` inside the collect loop. Removed; the loop yields only between passes.

## Other live probe facts

- `TaskCompleted` is a plain `RemoteEvent` (one child). An earlier `UnreplicatedEvent` read was
  a garbled response.
- `GrabFood` is a `RemoteFunction` - a helper accepting only RemoteEvent returns nil for it.
- Ownership: `workspace.Tycoons` children, `ObjectValue` whose `Value` is the LocalPlayer.
- Currency: `leaderstats` -> `IntValue` named `💸 Cash`; rises on its own, so a rising balance is
  not evidence. Measure objects disappearing.
- A single `n=0` from `getgc` is a failed call, not proof of absence.

## Dead ends (do not retry)

1. `PlayerGui.CustomerSpeechUI.Adornee` reads **NIL** here. Use `ClientCustomers`.
2. A `HumanoidRootPart` reports `ClassName == 'Model'`; walking up to the first Model stops there.
3. Dishes carry **no `CustomerId`/`GroupId`**.
4. `findTycon and nil or findTycoon()` - malformed, fixed in v3.6.1.
5. Clicking the guest's own prompt fires **no remote**.
6. "It fires no events" for the kitchen pickup was **wrong**.
7. Hand-firing the pickup `Interacted` from across the map does not work - it is range gated.
8. Filtering pickup prompts on `ActionText == 'Serve'` matches nothing.
9. Seating on any `Items.Surface` child puts groups on kitchen counters.
10. One attempt per pass + a long cooldown = 3-7 s stalls. Spam many targets per pass instead.

## Still unknown

1. Whether `TeleportRequestedClient` is a BindableEvent (`:Fire`) or a remote (`:FireServer`).
2. `ProximityPromptStyle` cannot be reproduced as an enum here, so it is omitted.

## Script versions

- v1.0.0 keyword scan: no-op. v2.x Dialog serve: fired world NPC dialogue.
- v3.x: SeatPrompt / held-dish / SendToTable attempts, none confirmed.
- v4.0.0: **died at load** - `Cannot use '...' outside of a vararg function`. Fixed in v4.0.1.
- v6.0.0: auto money. **Crashed the user's game.**
- v6.0.1: Interactions folder, ClassName parts, call rate. **WORKING** bills + tips.
- v6.1.0: + dishes. **WORKING.** v6.2.0: + orders. **WORKING.**
- v6.3.0: faster scan + seat. v6.7.0: prompt pickup, seated onto counters by mistake.
- v6.8.0: seat only `^T%d+$`, tray pickup without text filter. **WORKING.**
- v6.9.0: split call budget (serve 6 / other 10) so serving is not starved.
- v7.0.0: multi-target per pass (serve across 3 guests per dish, seat across 8 group/table
  pairs with a per-pair cooldown), budgets serve 8 / other 12, test buttons removed.
