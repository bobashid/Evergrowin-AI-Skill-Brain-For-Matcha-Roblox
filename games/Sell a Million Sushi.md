# Sell a Million Sushi

Place id **125866884667103**. `game.Name` reports **`Ugc`**. Title supplied by the user as the
place URL slug **Sell-a-Million-Sushi**. Script: `Sell a Million Sushi` in `bobashid/roblox-matcha`.

## CONFIRMED by probe (2026-10-31)

- `RemoteEvents` holds **29** children, all genuine `RemoteEvent`. Full list below.
- **Loot parts are named `Loot<n>`.** Live `LootSpawned` parts are `Loot1`..`Loot9`, but
  `ReplicatedStorage.LootModels` holds **24** models up to `Loot24`, so the tier space is 1..24.
- **There is NO `LootTier` attribute.** A workspace-wide attribute scan returned `tierHits=0`.
  A loot part carries only `HolderName` and `SpawnTime`. Tier must come from the NAME:
  `tonumber(name:match("^Loot(%d+)$"))`. A build that only checked the attribute (v1.1.0)
  found 0 loot while 34 parts sat in Plot_1 — that was the bug.
- `LocalPlayer:GetAttribute("assignedPlot")` = `"Plot_1"`. Plots are `Plot_1`..`Plot_6`
  under `workspace.Plots`.
- `LocalPlayer:GetAttribute("CurrentCount")` is a **number** (read 77, 642, 1142).
- `LocalPlayer.leaderstats["Net Worth"].Value` reads **EMPTY** in tests — do not use it as a
  witness. `CurrentCount` is the only reliable number.
- `CollectionService:GetTagged` / `GetTags` / `AddTag` are **nil** in this VM. Any target
  discovery must walk descendants.
- `firetouchinterest` is a **function**; `fireproximityprompt` and `fireclickdetector` exist.
- Loot counts at probe time: Plot_1=34, Plot_3=23, Plot_4=69, Plot_5=5, Plot_2=0, Plot_6=0.

## CROSS-MAP SILENT PICKUP IS MEASURED IMPOSSIBLE (2026-11-02)

Two independent measurements, both on `CurrentCount`:

1. **One single far loot item, fired alone.** Picked the furthest part in `Plot_1.LootSpawned`
   (`Loot1`, tier 1, **d=98**) and fired `LootPickup:FireServer({1}, false)` with nothing else.
   Result: `C0=1142` -> `single-far ok=true C=1142 delta=0`.
2. **The user's own 86-pass run** at `radius=32`, nearest loot `33m`:
   `passes=86 landed=0 missed=86 tiersSent=94`, `CurrentCount 1142 -> 1142`.

Test 1 matters most because it removes every alternative explanation: not batching, not a stale
part, not a wrong tier array. A single-element call at 98 studs does nothing. **The server gates
pickup on distance and the gate is short** — under 33 studs, probably near the client's own
5x15x5 box.

**Consequence, and the thing to remember:** the client's `LootPickupScript` already destroys loot
inside ~5 studs and fires the pickup itself. So a script's scan almost only ever sees loot
*outside* that box — precisely the loot the server rejects. That is why every far-fire build
looked like "works sometimes". It never worked far; the near ones were the game doing it.

**Do not retry cross-map pickup on this game.** Teleport was also removed at the user's request.
The only working shape is: be near the loot (walk), then fire inside a short radius.

## THE PICKUP REMOTE (decompiled `PlayerScripts.LootPickupScript`)

    RemoteEvents.LootPickup:FireServer(<array of LootTier numbers>, <isDeckHand boolean>)
    RemoteEvents.ChestPickup:FireServer("Chest" .. <ChestTier number>)

The game's own client builds that array from `workspace:GetPartBoundsInBox(CFrame.new(HRP.Position),
Vector3.new(5,15,5), params)` with `CollisionGroup = "Loot"` (and `"Chests"`), reads
`part:GetAttribute("LootTier")` off each hit, **destroys the part client-side**, then fires ONE
call with the whole array. `FireServer({3}, false)` is a one-element array, tier 3.

The `EMP_DeckHand` branch adds a second box around the plot's `TroughPart` and sets the boolean
true — that is what the second argument means.

## THE FOUR BUTTONS ARE TOUCH PARTS, NOT PROMPTS

`Plot_1.Sell` is a Model with 7 children, and the interactive scan found **zero `ProximityPrompt`
and zero `ClickDetector`** in the whole subtree — only four `TouchTransmitter` instances named
`TouchInterest`, each parented to a part named `Button`:

| Action | Touch target | Its script |
|---|---|---|
| Deposit fish | `Sell.DepositButton.Button` | `DepositButtonScript` |
| Collect money | `Sell.CollectButton.Button` | `CollectButtonScript` |
| Instant sell | `Sell.InstantSellButton.Button` | `InstantSellButtonScript` |
| Upgrade | `Sell.UpgradeButton.Button` | `UpgradeButtonScript` |

Every part is named `Button`, so identify a button by its **parent model name**, never by the
part name.

## SYNTHETIC TOUCH DOES NOT WORK ON THESE BUTTONS (confirmed 2026-11-02)

Two independent tests, both with a real before/after on `CurrentCount` and the
`OfflineIncome` billboard `AmountText` (which read **713.7K** throughout):

1. **Remote touch, character elsewhere.** `firetouchinterest(hrp, CollectButton.Button, 0)` then
   `(…, 1)`.
   Result: `BEFORE offline=713.7K count=642` -> `AFTER offline=713.7K count=642`.
2. **Touch with the character moved onto the button.** `hrp.CFrame = CFrame.new(bp.X, bp.Y+3, bp.Z)`,
   wait 0.9 s, read, touch, read, return home, read.
   Result: `BEFORE 713.7K/642` -> `ONBTN 713.7K/642` -> `AFTER 713.7K/642` -> `HOME 713.7K/642`.
   Button position: `Vector3(138.0355, -4.68, 43.6823)`. The print confirmed `btn=true`,
   `hrp=true`, `origSaved=true`, so the call sequence ran.

**Conclusion:** the `CollectButton` server script does not act on `firetouchinterest`. Caveat:
whether the CFrame teleport actually *held* is unproven — a server-authoritative game can snap the
character back before the touch lands.

`type(firetouchinterest) == 'function'` only proves the global exists. Exactly the pattern already
recorded for `workspace:Raycast`: listed, callable, inert.

## These scripts CANNOT be decompiled - do not retry

`DepositButtonScript`, `CollectButtonScript`, `InstantSellButtonScript`, `UpgradeButtonScript` and
`Plot_1.OfflineIncome.CollectOfflineIncome` all return **`-- Decompiler: Invalid bytecodes`**.
They are protected server Scripts.

## Offline income is a large uncollected number

`Plot_1.OfflineIncome` is a Model containing `Script:CollectOfflineIncome`, a `BillboardGui`,
`Part1`, `Part2`, `Highlight`. The billboard reads:

    TextLabel:OfflineIncomeTitle = "Offline Fish"
    TextLabel:AmountText        = "713.7K"

Unchanged across every touch attempt. **713.7K is parked there and no client-side mechanism
found so far releases it.**

## Deposit is tier-matched through the trough

`Plot_1.BoatLine.Boat_3` is a **Configuration** with attributes
`Owner, S0, STarget, Slot, T0, Tier, Docked`.
`ReplicatedStorage.FishermanModels` holds `Unit1`..`Unit24`, each with `Tier` and `FishName`.
`TroughPart` is a single `Part` with one `Attachment` and nothing interactive — a **volume
target**, fed by overlap, not clicked.

## ReplicatedStorage layout

Modules: `HelperModule`, `GameAnalytics`, `MusicModuleScript`, `UserInputModuleScript`,
`BigNumberModule`, `FishData`, `SushiStyle`, `KitchenConfig`, `BoatConfig`, `ChefAnimator`,
`FishNameStyle`, `EmployeeAnimator`.
Folders: `RemoteEvents`(29), `Audio`(21), `UI`(0), `Music`(1), `ChestImages`(6),
`SushiNPCs`(22), `BoatModels`(6), `LootModels`(24), `LeaderboardData`(3), `FishermanModels`(24).

### The 29 remotes

`ChangeSongLocal, UpdateCurrencyEvent, UpdateLootEvent, UpdateChestUnlockCost, ToggleMusicMuted,
StopSongLocal, PlaySellSound, PromptBuyCurrency, ShowHelpTextLocal, showGuideArrowLocal,
MessageFeedEvent, UpdatePaidMoneyButtons, LootPickup, UpdateCollection, UpdateTotalCount,
UpdateMoneyMultInfo, PromptBuyMoneyMult, CantAffordEffect, ChestPickup, TeleportHome, DiscardChest,
UpdateChestGui, ChestOpenAnimation, UnlockChest, InstantSellButton, EmployeeToggle, TraderThreshold,
KitchenEvent, BoatEvent`

No `DepositFish`, no `CollectMoney`, no `Sell` remote. Most are server->client updates.

## LocalPlayer attributes (25)

`OnboardStep, assignedPlot, musicMuted, incomeMultsPurchased, incomeMult, KitchenBusy,
KitchenRemainingLog, KitchenRateLog, EMP_Cashier, EMP_DeckHand, EMP_Manager, EMP_FishTrader,
EMP_SousChef, EMP_Greeter, TraderThreshold, CurrentCount, NetWorth, NetWorthLog, Playtime,
GP_AutoMoney, GP_AutoLoot, GP_DoubleLoot, GP_FishTrader, GP_SousChef, GP_Greeter`

`GP_AutoLoot` was **false** — the game's own 5-stud pickup is all the player has without it.

## Dead ends (do not retry)

1. **Guessing `InstantSellButton` argument shapes.** v2.0.0 walked `()`, `(false)`, `(true)` every
   2 s with ~1.3 s of sleeps inside the loop. **It crashed the user's client.**
2. **Decompiling the four button scripts or `CollectOfflineIncome`** — `Invalid bytecodes`.
3. **`CollectionService:GetTagged`** — nil in this VM.
4. **Reading `LootTier` as an attribute** — does not exist; the tier is in the name.
5. **Trusting a loot part disappearing as success** — the game destroys it client-side.
6. **Touching another player's buttons** — owner-validated, reads as griefing.
7. **`firetouchinterest` on `CollectButton.Button`** — zero change, twice, with the call sequence
   confirmed to run.
8. **Cross-map / long-range `LootPickup`.** A single far item at 98 studs returned `delta=0`; an
   86-pass run at 33m+ logged `landed=0`. The server distance-gates pickup. Measured, not assumed.
9. **CFrame teleport to loot.** Removed at the user's request and pointless anyway — the server
   rejects the fire, so teleporting only moved the character for nothing.
10. **Using `leaderstats["Net Worth"]` as a witness** — it reads empty.

## Still pending

1. Whether walking into `CollectButton` normally releases the 713.7K offline fish. Only the user
   can test this (walk in, no script, watch `Offline Fish`). If it does not release, the button is
   gated and the whole collect-money line should be dropped.
2. `KitchenEvent` / `BoatEvent` argument shapes (both exist, neither confirmed).
3. The server's exact pickup radius. Measured floor: works inside ~10, fails at 33+. Somewhere
   between. `Radius` is a menu knob, and the Report prints `landed` vs `missed` so it can be tuned
   from data.

## Version history

- v1.1.0 — attribute-only tier read: `lootWithTier=0` while 34 parts existed. **Broken.**
- v1.2.0 — tier from name + touch + remote + prompt. **Worked** (user: "nice").
- v2.0.0 — added auto-sell with three guessed shapes. **Crashed the client.**
- v2.1.0 — sell deleted, one toggle, `busy` guard. Pickup only.
- v2.2.0 — nearest-first sort, radius gate, teleport-to-loot default. Teleport later removed.
- v2.3.0 — teleport removed; walks with `Humanoid:MoveTo` and fires only inside `Radius` (10).
  Pushed and read back (sha `46cb9c4a`, 13,770 bytes). **Not yet confirmed in game.**
