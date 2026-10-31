# Restaurant Tycoon 3

Place id **119048529960596**. `game.Name` reports **`Ugc`**, so Hub pins this place in `PLACES`
(done, Hub v3.0). Script: `Restaurant Tycoon 3` in `bobashid/roblox-matcha`.

## CONFIRMED WORKING: v6.0.1 money collection (User-stated 2026-10-27)

User: "nice everything there works" after v6.0.1. The bill collector and tips collector are
proven in a live game. v6.1.0 keeps every one of those fixes and adds dish pickup on top.

**The bill call is genuinely TWO calls** - the earlier "no Interactions call is required" line
in this note was wrong and is superseded:

1. `Events.Restaurant.Interactions.Interacted:FireServer(tycoon, {Id="1", ActionText="",
   WorldPosition=<part pos>, GamepadKeyCode=1000, TemporaryPart=Part, KeyboardKeyCode=101,
   Range=7.2, IsCustomerPrompt=true, Key="CollectBill", Model=<Items.Surface.T?>, Part=Part,
   Prompt=1, InteractionType="CollectBill", HoldDuration=0})`
2. `Events.Restaurant.TaskCompleted:FireServer({Tycoon=tycoon, Name="CollectBill",
   FurnitureModel=<same Items.Surface model>})`

**`Interactions` is a FOLDER, not a remote.** Probed live: `X2 inter Folder`, and its children
include `RemoteEvent Interacted`, `InteractionCreated`, `InteractionDestroyed`,
`AllInteractionsDestroyed`, `ClickableFurnitureInteracted`, plus a BindableEvent
`ClickableFurnitureInteractedClient`. `Interacted` must be resolved **inside** that folder.
Firing the Folder itself (`Folder:FireServer`) throws inside pcall and is swallowed - that was
the v6.0.0 defect and the likely crash path.

**Tips is ONE call:** `Events.Restaurant.TipsCollected:FireServer(tycoon)`.

**Dishes is ONE call** (user-captured 2026-10-27):
`TaskCompleted:FireServer({Tycoon=tycoon, Name="CollectDishes", FurnitureModel=Items.Surface.T19})`
- no `Interacted` partner. Target is the `Items.Surface` model holding a **`Trash`** child.

So the shape differs per action - do not assume uniformity:

| Action | Calls | Target child on `Items.Surface` |
|---|---|---|
| Collect bill | `Interacted` THEN `TaskCompleted` | `Bill` |
| Collect dishes | `TaskCompleted` only | `Trash` |
| Collect tips | `TipsCollected` only | n/a (arg is the tycoon) |

## Crash post-mortem: v6.0.0 (2026-10-27)

v6.0.0 crashed the game. It had four defects, all removed in v6.0.1, which then worked:

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

## Live probe facts (Confirmed 2026-10-27)

- `V3 bills 0 trash 5` - five `Trash` surfaces existed at probe time, so dish collection had
  targets. Bills come and go with customers; `trash`/`bills` reading 0 is normal when idle.
- `Z2 Interacted RemoteEvent`, `X4 hasInteracted true`, `X5 tc RemoteEvent`.
- `Y3 bills 1 T19`, `Y4 partsInFirstBillTable 7` - a billed table has real BaseParts under it,
  so a ClassName part lookup succeeds on real targets.
- Ownership: `workspace.Tycoons` children, `ObjectValue` whose `Value` is the LocalPlayer.
- Currency: `leaderstats` -> `IntValue` named `💸 Cash`; rises on its own, so a rising balance
  is not evidence a script worked.
- `TaskCompleted` is a plain `RemoteEvent` (one child). An earlier read showing
  `UnreplicatedEvent` was a garbled response.

## Dead ends (do not retry)

1. **`PlayerGui.CustomerSpeechUI.Adornee` reads NIL** in this VM (`adornee=nil`, `Header.Text=nil`
   on all 3 probes). The reference script derives customer ids from it; unusable here.
2. **A `HumanoidRootPart` reports `ClassName == 'Model'`.** Any "walk up until you find a Model"
   helper stops on the root part and returns the wrong instance.
3. **`Tycoon.ClientCustomers` is often empty** (`cust=0` across many probes, 0 groups when
   checked). Do not build on it as a live guest source without re-probing.
4. Dishes on the counter carry **no `CustomerId`/`GroupId`** - only `EatMethod`, `Taken`,
   `Part:Head`, `UnionOperation:TestNewPlate`, `Model:Contents`. Serve cannot read its target
   from the dish.
5. `findTycon and nil or findTycoon()` - malformed, fixed in v3.6.1.

## Still unknown

1. How to serve food to the right guest (customers are not reliably visible in any probed
   source; see dead ends 1 and 3).
2. `SendToTable` / `TakeOrder` were never confirmed working in a live game.
3. Whether `TeleportRequestedClient` is a BindableEvent (`:Fire`) or a remote (`:FireServer`).

## Script versions

- v1.0.0 keyword scan: no-op. v2.x Dialog serve: fired world NPC dialogue.
- v3.x: SeatPrompt / held-dish / SendToTable attempts, none confirmed.
- v4.0.0: **died at load** - `Cannot use '...' outside of a vararg function` (a `...` inside a
  nested closure in `fire()`). Fixed in v4.0.1 with `packArgs` + `unpackFn(a, 1, a.n)`.
- v4.1.x / v5.x: diagnostics and auto-serve attempts; none worked (customers never found).
- v6.0.0: auto money. **Crashed the user's game.**
- v6.0.1: fixed the Interactions folder, ClassName part lookup, and call rate. **CONFIRMED
  WORKING** - bills + tips.
- v6.1.0: adds auto dish pickup (`CollectDishes`, single call) alongside bills and tips.
