# Restaurant Tycoon 3

Place id **119048529960596**. `game.Name` reports **`Ugc`**, so Hub pins this place in `PLACES`
(done, Hub v3.0). Script: `Restaurant Tycoon 3` in `bobashid/roblox-matcha`.

## CONFIRMED WORKING by the user

- v6.0.1: bills + tips - "nice everything there works" (2026-10-27).
- v6.1.0: dishes - "nice everything works the auto dish pickup and cash+tips".
- v6.2.0: orders - "nice everything worked once again".
- v6.3.0: seating - "nice its functional".

## Guests accept the WRONG order (User-stated 2026-10-27)

User: "they dont care if you give them the wrong order btw so if i do that pickup method with
holding a whole tray you can just click every but if not and its faster to have a filter thats
fine".

Consequence: the dish->guest **matching problem does not exist**. No per-pair cooldown table, no
`ServeTries` cycling to find the guest who ordered that dish. Serve any dish to any guest. This
retires the long-standing "Still unknown: how to match dish to guest" item entirely.

## Kitchen pickup is RANGE/PROMPT gated, not payload gated (2026-10-27)

User: "still not working whats wrong with picking it up from cross map".

The captured pickup carries `Range = 9` and a `WorldPosition`, and the prompt object only exists
at `workspace.Temp.Part` while standing near a counter. Hand-building the `Interacted` table and
firing it from across the map is refused no matter how correct the table is.

**Preferred route: `fireproximityprompt(prompt)`** on every `ProximityPrompt` under
`workspace.Temp` whose `ActionText == 'Serve'`. That runs whatever client handler the game
attached - including the pickup - with no distance fiction. The captured `Interacted` call is
kept only as a **fallback** when no prompt object is present.

## CORRECTION: the kitchen pickup DOES fire an event

v6.5.0 was built on a wrong reading of "it fires no events" and dropped the call entirely - it
picked nothing up. Rule: "it fires no events for me" means *the user did not see one*, not
*the game sends none*. Probe before dropping a call.

    Events.Restaurant.Interactions.Interacted:FireServer(tycoon, {
      HoldDuration = 0.25, Id = "172", ActionText = "Serve",
      WorldPosition = Vector3.new(1208.80, 0.27, 316.02),
      GamepadKeyCode = Enum.KeyCode.Thumbstick2, TemporaryPart = workspace.Temp.Part,
      KeyboardKeyCode = Enum.KeyCode.F, Range = 9, IsCustomerPrompt = true, Key = "Serve",
      Model = tycoon.Items.Surface.K16, Part = workspace.Temp.Part,
      Prompt = workspace.Temp.Part["172"], InteractionType = "OrderCounter" })

## `Prompt` is an INDEX BY LIVE PROMPT ID (Confirmed by probe)

    W2 temp Folder 2
    W3 part Part 1
    W4 ProximityPrompt 176      <- sole child of workspace.Temp.Part
    W5 total 1

`workspace.Temp.Part["172"]` means "the prompt currently named 172 inside Temp.Part". The id is
**not stable** - seen as `140`, then `172`, then `176`. Never hardcode it: read the live
`ProximityPrompt` child and send that object as `Prompt`, its name as `Id`.
v6.4.0 looked for `workspace.Temp["140"]` - wrong parent AND a hardcoded id.

## Counter models are `K<n>` (Confirmed by probe)

    W7 T19, T19, T1, T1, K3, K16, K28, K28, K16, K3, T1, T1, T1

Tables are `T1`/`T19`; **counters are `K3`/`K16`/`K28`** (capture used `K16`). Match `^K%d+$`,
pick nearest to the character. v6.4.0's "first Surface child that is not T<n>" was an untested
guess.

## Serving = pickup, then one TaskCompleted per dish (User-captured)

    TaskCompleted:FireServer({FoodModel=Objects.Food["10"], Name="Serve",
                              Tycoon=tycoon, GroupId="19", CustomerId="2"})
    TaskCompleted:FireServer({FoodModel=Objects.Food["7"],  Name="Serve",
                              Tycoon=tycoon, GroupId="17", CustomerId="2"})
    TaskCompleted:FireServer({FoodModel=Objects.Food["1"],  Name="Serve",
                              Tycoon=tycoon, GroupId="16", CustomerId="2"})

A dish carries no ids of its own (only `EatMethod`, `Taken`, `Part:Head`,
`UnionOperation:TestNewPlate`, `Model:Contents`). With wrong orders accepted, any dish pairs with
any guest.

## Call shapes DIFFER per action - never assume uniformity

| Action | Calls | Target / args |
|---|---|---|
| Collect bill | `Interacted` **THEN** `TaskCompleted` (2) | `FurnitureModel` = `Items.Surface` with a `Bill` child |
| Collect dishes | `TaskCompleted` only (1) | `FurnitureModel` = `Items.Surface` with a `Trash` child |
| Collect tips | `TipsCollected` only (1) | the tycoon itself |
| Take order | `TaskCompleted` only (1) | `GroupId` + `CustomerId`, both STRINGS |
| Seat a guest | `TaskCompleted` only (1) | `FurnitureModel` = the table, `GroupId` (string) |
| Serve food | `fireproximityprompt` on the Serve prompt (fallback: `Interacted`), THEN `TaskCompleted 'Serve'` per dish | `FoodModel` + `GroupId` + `CustomerId` |

## Seating: the guest prompt fires NOTHING (User-stated)

Clicking the guest's own prompt fires no remote; only clicking an empty table fires the single
`TaskCompleted 'SendToTable'`. Nothing from the first click needs reproducing.

    TaskCompleted:FireServer({FurnitureModel = Items.Surface.T1, GroupId = "15",
                              Tycoon = tycoon, Name = "SendToTable" })

## `Interactions` is a FOLDER, not a remote

Children include `RemoteEvent Interacted`, `InteractionCreated`, `InteractionDestroyed`,
`AllInteractionsDestroyed`, `ClickableFurnitureInteracted`, `BindableEvent
ClickableFurnitureInteractedClient`. Resolve `Interacted` **inside** it. Firing the Folder itself
throws inside pcall and is swallowed - the v6.0.0 defect and likely crash path.

## Guest source: `Tycoon.ClientCustomers`

    Folder "8"  -> Model "1", Model "2"
    Folder "9"  -> Model "1", Model "2"
    Folder "10" -> Model "1", Model "2"

Folder name = `GroupId`, child model name = `CustomerId`, both opaque strings - **never** validate
with `tonumber()`; that silently dropped every customer in v4.1.1. Empty is normal when no guests
are present.

## Crash post-mortem: v6.0.0

Four defects, all removed in v6.0.1, which then worked:

1. `remote('Restaurant','Interactions')` returned the **Folder**; `Folder:FireServer(...)` was
   attempted - the first bill call never fired.
2. `FindFirstChildWhichIsA('BasePart', true)` for part lookup. `IsA` always returns true here, so
   it can return a non-part; sending a Model where a `Part` is expected is a plausible crash
   trigger. **Find parts by `ClassName` only**, skip the target if none is found.
3. **No rate limit** - up to 12 bills x 2 calls every 0.25 s. v6.0.1 added a token bucket
   (8 calls per 0.5 s) and one target per feature per pass.
4. `task.wait` inside the collect loop. Removed; the loop yields only between passes.

Rule: never fire more than one collect target per pass, never exceed a handful of calls per half
second. It has crashed once already.

**Speed tuning (v6.3.0+):** loop `Gap` 0.2, per-feature 0.35, tips 1.0, seat 0.3, serve 1.0.
The **token bucket stays at 8 calls / 0.5 s** on purpose. Report shows `rateBlocked=`.

## Other live probe facts

- `TaskCompleted` is a plain `RemoteEvent` (one child). An earlier `UnreplicatedEvent` read was a
  garbled response.
- `GrabFood` is a `RemoteFunction` - a helper accepting only RemoteEvent/UnreplicatedRemoteEvent
  returns nil for it (a real v6.4.0 bug, fixed in v6.5.0).
- Ownership: `workspace.Tycoons` children, `ObjectValue` whose `Value` is the LocalPlayer.
- Currency: `leaderstats` -> `IntValue` named `💸 Cash`; rises on its own, so a rising balance is
  not evidence. Measure objects disappearing.
- A single `n=0` from `getgc` is a failed call, not proof of absence.

## Dead ends (do not retry)

1. **`PlayerGui.CustomerSpeechUI.Adornee` reads NIL** in this VM. Use `ClientCustomers`.
2. **A `HumanoidRootPart` reports `ClassName == 'Model'`.** Walking up to the first Model stops
   on the root part.
3. Dishes on the counter carry **no `CustomerId`/`GroupId`**.
4. `findTycon and nil or findTycoon()` - malformed, fixed in v3.6.1.
5. Clicking the guest's own prompt fires **no remote**.
6. **"It fires no events" for the kitchen pickup was WRONG** - see the correction above.
7. Hand-firing the pickup `Interacted` table from across the map does not work - it is range
   gated. Use `fireproximityprompt`.

## Still unknown

1. Whether `TeleportRequestedClient` is a BindableEvent (`:Fire`) or a remote (`:FireServer`).
2. `ProximityPromptStyle` in the serve pickup cannot be reproduced as an enum here, so it is
   omitted - unverified whether the server needs it. (It is also omitted from the prompt route,
   which does not need it at all.)

## Script versions

- v1.0.0 keyword scan: no-op. v2.x Dialog serve: fired world NPC dialogue.
- v3.x: SeatPrompt / held-dish / SendToTable attempts, none confirmed.
- v4.0.0: **died at load** - `Cannot use '...' outside of a vararg function` (a `...` inside a
  nested closure in `fire()`). Fixed in v4.0.1 with `packArgs` + `unpackFn(a, 1, a.n)`.
- v4.1.x / v5.x: diagnostics and auto-serve attempts; none worked.
- v6.0.0: auto money. **Crashed the user's game.**
- v6.0.1: fixed Interactions folder, ClassName part lookup, call rate. **WORKING** bills + tips.
- v6.1.0: + auto dish pickup. **WORKING.**
- v6.2.0: + auto take order. **WORKING.**
- v6.3.0: faster scan intervals + auto seat. **WORKING.**
- v6.4.0: serve via counter interaction - wrong parent, hardcoded prompt id; `GrabFood`
  (RemoteFunction) also resolved to nil.
- v6.5.0: dropped the interaction on a wrong "fires no events" reading - picked nothing up.
- v6.6.0: pickup restored with the live prompt id, counter by `^K%d+$` nearest-to-player.
- v6.7.0: pickup via `fireproximityprompt` on the Serve prompt (remote as fallback); dish->guest
  matching removed since wrong orders are accepted.
