# Restaurant Tycoon 3

Place id **119048529960596**. `game.Name` reports **`Ugc`**, so Hub pins this place in `PLACES`
(done, Hub v3.0). Script: `Restaurant Tycoon 3` in `bobashid/roblox-matcha`.

## CONFIRMED WORKING by the user

- v6.0.1: bills + tips - "nice everything there works" (2026-10-27).
- v6.1.0: dishes - "nice everything works the auto dish pickup and cash+tips".
- v6.2.0: orders - "nice everything worked once again".
- v6.3.0: seating - "nice its functional".

## Kitchen pickup fires NO events (User-stated 2026-10-27)

User: "i think you might be having a hard time picking up the food from the kitchen theres an
easier way, for me its picking them up one by one, this fires no events but its easier to track
what your giving out".

So the pickup step is **client-side only** - there is no remote to reproduce, and the captured
`Interactions.Interacted` with `Key="Serve"` / `InteractionType="OrderCounter"` is NOT needed.
Use **`Restaurant.GrabFood:InvokeServer(foodModel)`** instead - proven working earlier this
session (returned `true` on `Objects.Food.3`).

v6.4.0 wasted a version reproducing the counter interaction and guessing the counter model
(`Items.Surface` child not matching `^T%d+$`). That whole path is removed in v6.5.0.

## Serving = grab, then one TaskCompleted per dish (User-captured 2026-10-27)

Three captures, each pairing a DIFFERENT dish with a DIFFERENT guest:

    TaskCompleted:FireServer({FoodModel=Objects.Food["10"], Name="Serve",
                              Tycoon=tycoon, GroupId="19", CustomerId="2"})
    TaskCompleted:FireServer({FoodModel=Objects.Food["7"],  Name="Serve",
                              Tycoon=tycoon, GroupId="17", CustomerId="2"})
    TaskCompleted:FireServer({FoodModel=Objects.Food["1"],  Name="Serve",
                              Tycoon=tycoon, GroupId="16", CustomerId="2"})

A dish carries no ids of its own (only `EatMethod`, `Taken`, `Part:Head`,
`UnionOperation:TestNewPlate`, `Model:Contents`), so the script grabs a dish then tries guests
in turn - the server accepts only the dish that guest ordered. That is why the three captured
pairs differ per dish.

## Call shapes DIFFER per action - never assume uniformity

| Action | Calls | Target / args |
|---|---|---|
| Collect bill | `Interacted` **THEN** `TaskCompleted` (2) | `FurnitureModel` = `Items.Surface` model holding a `Bill` child |
| Collect dishes | `TaskCompleted` only (1) | `FurnitureModel` = `Items.Surface` model holding a `Trash` child |
| Collect tips | `TipsCollected` only (1) | the tycoon itself |
| Take order | `TaskCompleted` only (1) | `GroupId` + `CustomerId`, both STRINGS |
| Seat a guest | `TaskCompleted` only (1) | `FurnitureModel` = the table, `GroupId` (string) |
| Serve food | `GrabFood:InvokeServer(dish)` then `TaskCompleted 'Serve'` | `FoodModel` + `GroupId` + `CustomerId` |

Sending an extra unverified remote alongside a single-call action is how v6.0.0 crashed the
game. Only send what the capture or a proven call shows.

## Seating: the guest prompt fires NOTHING (User-stated 2026-10-27)

"when i clicked on the prompt above their head it didnt show an event for me to send you but it
only showed the event when i pressed on an empty table".

The in-game flow is: click the guest's prompt (client-side, no remote), THEN click an empty
table, which fires one `TaskCompleted 'SendToTable'`. Nothing from the first click needs
reproducing.

## Captured seat payload (single call)

    TaskCompleted:FireServer({FurnitureModel = Items.Surface.T1, GroupId = "15",
                              Tycoon = tycoon, Name = "SendToTable" })

## `Interactions` is a FOLDER, not a remote

Its children include `RemoteEvent Interacted`, `InteractionCreated`, `InteractionDestroyed`,
`AllInteractionsDestroyed`, `ClickableFurnitureInteracted`, and a `BindableEvent
ClickableFurnitureInteractedClient`. `Interacted` must be resolved **inside** the folder.
Firing the Folder itself throws inside pcall and is swallowed - the v6.0.0 defect and likely
crash path.

## Guest source: `Tycoon.ClientCustomers`

    Folder "8"  -> Model "1", Model "2"
    Folder "9"  -> Model "1", Model "2"
    Folder "10" -> Model "1", Model "2"

**Folder name = `GroupId`, child model name = `CustomerId`**, both opaque strings - never
validate with `tonumber()`; doing so silently dropped every customer in v4.1.1. The captured
pairs (`GroupId="8"/"15"/"16"/"17"/"19"`, `CustomerId="2"`) match this structure.

Empty `ClientCustomers` is normal when no guests are present - not a bug.

## Crash post-mortem: v6.0.0

Four defects, all removed in v6.0.1, which then worked:

1. `remote('Restaurant','Interactions')` returned the **Folder**, and `Folder:FireServer(...)`
   was attempted - the first bill call never fired.
2. `FindFirstChildWhichIsA('BasePart', true)` for part lookup. `IsA` always returns true here,
   so it can return a non-part; sending a Model where a `Part` is expected is a plausible crash
   trigger. **Find parts by `ClassName` only**, skip the target if none is found.
3. **No rate limit** - up to 12 bills x 2 calls every 0.25 s. v6.0.1 added a token bucket
   (8 calls per 0.5 s) and one target per feature per pass.
4. `task.wait` inside the collect loop. Removed; the loop yields only between passes.

Rule for this game: never fire more than one collect target per pass, and never exceed a handful
of calls per half second. It has crashed once already.

**Speed tuning (v6.3.0 onwards):** loop `Gap` 0.2, per-feature intervals 0.35, tips 1.0, seat
0.3, serve 1.0. The **token bucket stays at 8 calls / 0.5 s** on purpose - speeding up the
*scan* must not raise the *call* rate that caused the crash. Report shows `rateBlocked=`.

## Other live probe facts

- `TaskCompleted` is a plain `RemoteEvent` (one child). An earlier read showing
  `UnreplicatedEvent` was a garbled response.
- `GrabFood` is a `RemoteFunction` - a helper that only accepts RemoteEvent/UnreplicatedRemote
  Event will silently return nil for it (that was a v6.4.0 bug, fixed in v6.5.0).
- Ownership: `workspace.Tycoons` children, `ObjectValue` whose `Value` is the LocalPlayer.
- Currency: `leaderstats` -> `IntValue` named `💸 Cash`; rises on its own, so a rising balance is
  not evidence a script worked. Measure objects disappearing.
- Bills/trash reading 0 is normal when idle; `trash 5` was seen once at probe time.
- A single `n=0` from `getgc` is a failed call, not proof of absence.

## Dead ends (do not retry)

1. **`PlayerGui.CustomerSpeechUI.Adornee` reads NIL** in this VM. The reference script derives
   customer ids from it; unusable here. Use `ClientCustomers`.
2. **A `HumanoidRootPart` reports `ClassName == 'Model'`.** Any "walk up until you find a Model"
   helper stops on the root part and returns the wrong instance.
3. Dishes on the counter carry **no `CustomerId`/`GroupId`**.
4. `findTycon and nil or findTycoon()` - malformed, fixed in v3.6.1.
5. Clicking the guest's own prompt fires **no remote**.
6. Picking food up from the kitchen fires **no remote**.
7. Guessing the order counter by name shape (`Surface` child not matching `^T%d+$`) was wasted
   work - the pickup needs no model at all.

## Still unknown

1. Whether the server accepts a `Serve` for the wrong guest, or silently drops it. The script
   relies on drops being harmless while it cycles dishes against guests.
2. Whether seating also needs the guest in a specific state. The capture is real and single-call,
   and seating is user-confirmed functional, so this is settled in practice.
3. Whether `TeleportRequestedClient` is a BindableEvent (`:Fire`) or a remote (`:FireServer`).

## Script versions

- v1.0.0 keyword scan: no-op. v2.x Dialog serve: fired world NPC dialogue.
- v3.x: SeatPrompt / held-dish / SendToTable attempts, none confirmed.
- v4.0.0: **died at load** - `Cannot use '...' outside of a vararg function` (a `...` inside a
  nested closure in `fire()`). Fixed in v4.0.1 with `packArgs` + `unpackFn(a, 1, a.n)`.
- v4.1.x / v5.x: diagnostics and auto-serve attempts; none worked.
- v6.0.0: auto money. **Crashed the user's game.**
- v6.0.1: fixed Interactions folder, ClassName part lookup, call rate. **WORKING** bills + tips.
- v6.1.0: + auto dish pickup (`CollectDishes`). **WORKING.**
- v6.2.0: + auto take order (`TakeOrder`). **WORKING.**
- v6.3.0: faster scan intervals + auto seat (`SendToTable`). **WORKING.**
- v6.4.0: auto serve via the captured OrderCounter interaction - unnecessary, the pickup fires
  no events; also had a bug where `GrabFood` (RemoteFunction) resolved to nil.
- v6.5.0: serve via `GrabFood:InvokeServer(dish)` then `TaskCompleted 'Serve'` per dish/guest.
  Counter interaction and counter model guessing removed.
