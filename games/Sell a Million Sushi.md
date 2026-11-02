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
- `LocalPlayer:GetAttribute("CurrentCount")` is a **number** (read 77, 642, 1142, 1542).
- **MONEY lives at `LocalPlayer:GetAttribute("NetWorth")`, a STRING like `"$2.6M"`.**
  `leaderstats["Net Worth"].Value` reads **EMPTY** and must never be used as a witness.
  Parse `$`, commas, and `K`/`M`/`B`/`T` before comparing.
- `CollectionService:GetTagged` / `GetTags` / `AddTag` are **nil** in this VM. Target discovery
  must walk descendants.
- `Humanoid:MoveTo` returns successfully but **does not move the character** in this VM
  (37 calls, zero studs of movement). Never gate an action behind arrival.
- `firetouchinterest` exists as a function but has produced no effect on any button tested
  (dead ends #7 and #11).
- `TextButton` count in a plot is **0** and `ProximityPrompt` count is **0**. Every interactive
  is a 3D touch part. Any prompt- or GUI-button-based approach is dead on arrival.
- `Drawing.new` supports `Square`, `Line`, `Text`, `Circle`, `Quad`, `Triangle`, `Image` — all
  present and constructible (probe 2026-11-02).

## FISH VALUE IS READABLE AND OVERLAY-ABLE (v2.4.1, 2026-11-02)

`Workspace.Plots.<plot>.Restaurant.Interior.KitchenBoard.MarketBoard` carries a `SurfaceGui`
named `MarketGui` with these `TextLabel`s:

    Title = "Fish Market"
    Price = "Fish Value: 1.4x <garbage char>"     <- the live multiplier
    Axis  = "0.5x" / "1.0x" / "1.5x"              <- chart axis
    Next  = "Updating in 9s"                      <- countdown to the next change

Parse with `string.match(text, '(%d+%.?%d*)%s*[xX]')` -> `1.4`. Observed range matches the
user's 0.1x .. 2.0x expectation.

**The overlay needs NO `WorldToScreen`.** It is a fixed-position `Drawing.Text`, so camera
projection is irrelevant — the one thing in this game that can be drawn without any world→screen
math. Labels are created ONCE at load and only mutated afterwards; reading the MarketBoard subtree
is throttled to ~7 Hz because the number changes once every 9 s.

Colour coding in v2.4.1+: `>= DepositAt` green, `>= 1.0` amber, below 1.0 red, unreadable grey.

## MARKET-GATED AUTO DEPOSIT (v2.5.0, 2026-11-02) — NOT YET CONFIRMED

Behaviour: watch the multiplier on its own loop. When it reaches the slider value, teleport ONTO
`Sell.DepositButton.Button` **once**, `firetouchinterest` 0 then 1, return to the saved CFrame, then
read `CurrentCount` back. **Latch** after the attempt and re-arm only when the value drops below the
threshold — that is what makes it "once when it hits" rather than a spam loop.

- Slider `Deposit At (x)`: 1.0 .. 2.0, snapped to 0.1 steps by `clampRound`.
- `DepositGap = 3.0 s` minimum between attempts, so a flapping value cannot thrash the button.
- `VerifyWait = 0.45 s` before reading `CurrentCount` back — the acceptance test.
- `Deposit Now (ignore value)` and `Re-arm Deposit` buttons for manual testing.
- `lastDeposit` reports `took N fish @ X.XXx` when `CurrentCount` fell, else `no change`.

**Why it measures instead of claiming:** every button server script returns `Invalid bytecodes`,
so what the server checks cannot be read. `firetouchinterest` on `CollectButton` did nothing twice.
A deposit that the report did not see is not a deposit.

## THE BUTTON MAP (probed 2026-11-02)

Every interactive is a `Model` containing a `Part` named **`Button`** with a `TouchTransmitter`
named **`TouchInterest`**. Identify a button by its **parent model name**, never by the part name
(every part is called `Button`).

| Purpose | Path under `Workspace.Plots.<plot>` | Its script |
|---|---|---|
| Hire fisherman x1 | `Buttons.ButtonBuy1.Button` | `BuyScript` |
| Hire fisherman x5 | `Buttons.ButtonBuy5.Button` | `BuyScript` |
| Hire fisherman x25 | `Buttons.ButtonBuy25.Button` | `BuyScript` |
| Hire fisherman x100 | `Buttons.ButtonBuy100.Button` | `BuyScript` |
| Merge | `Buttons.ButtonMerge.Button` | `ButtonScript` |
| Upgrade chef | `Sell.UpgradeButton.Button` | `UpgradeButtonScript` |
| Collect money | `Sell.CollectButton.Button` | `CollectButtonScript` |
| Deposit fish | `Sell.DepositButton.Button` | `DepositButtonScript` |
| Collect group reward | `GroupReward.CollectButton.Button` | `CollectButtonScript` |

Instant sell is `Sell.InstantSellButton.Button` (`InstantSellButtonScript`).

v3.0.0 automated all of these and was rolled back. v2.5.0 uses **only** `DepositButton`.

## BUTTON PRICES ARE ON EACH MODEL'S BillboardGui

Each button model has a `BillboardGui` child whose descendants carry a `Text` like `$1.2K`.
Parse with `string.match(t, '%$([%d%.]+)%s*([KkMmBbTt]?)')` then scale by the suffix.

## CROSS-MAP SILENT PICKUP IS MEASURED IMPOSSIBLE (2026-11-02)

Two independent measurements, both on `CurrentCount`:

1. **One single far loot item, fired alone.** Furthest part in `Plot_1.LootSpawned`
   (`Loot1`, tier 1, **d=98**) fired as `LootPickup:FireServer({1}, false)` with nothing else.
   Result: `C0=1142` -> `single-far ok=true C=1142 delta=0`.
2. **The user's own 86-pass run** at `radius=32`, nearest loot `33m`:
   `passes=86 landed=0 missed=86 tiersSent=94`, `CurrentCount 1142 -> 1142`.

Test 1 removes every alternative explanation. The server distance-gates pickup and the gate is
short.

**The client's own `LootPickupScript` destroys loot inside ~5 studs and fires the pickup itself.**
So a script's scan almost only ever sees loot *outside* that box - precisely the loot the server
rejects. That is why every far-fire build looked like "works sometimes". It never worked far.

## THE v2.3.0 LESSON: NEVER GATE AN ACTION BEHIND AN UNREADABLE VALUE

v2.3.0 only fired `LootPickup` when `nearest <= Radius`. `Humanoid:MoveTo` is inert here, so the
distance never closed and it fired **zero times across 38 passes** - `tiersSent=0` on every report
while the user watched it "do nothing". Rule: an unreadable price, distance or market value must
fall through to "attempt anyway" for anything harmless, and only block for something that costs
the player (deposit).

## THE PICKUP REMOTE (decompiled `PlayerScripts.LootPickupScript`)

    RemoteEvents.LootPickup:FireServer(<array of LootTier numbers>, <isDeckHand boolean>)
    RemoteEvents.ChestPickup:FireServer("Chest" .. <ChestTier number>)

The client builds that array from `workspace:GetPartBoundsInBox(CFrame.new(HRP.Position),
Vector3.new(5,15,5), params)` with `CollisionGroup = "Loot"`, reads `LootTier` off each hit,
destroys the part client-side, then fires ONE call with the whole array.

## These scripts CANNOT be decompiled - do not retry

`BuyScript`, `ButtonScript`, `UpgradeButtonScript`, `CollectButtonScript`, `DepositButtonScript`,
`InstantSellButtonScript` and `Plot_1.OfflineIncome.CollectOfflineIncome` all return
**`-- Decompiler: Invalid bytecodes`**. Protected server Scripts.

## Offline income is a large uncollected number

`Plot_1.OfflineIncome` billboard reads `Offline Fish = 713.7K`, unchanged across every touch
attempt. No client-side mechanism found so far releases it.

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

No `DepositFish`, no `CollectMoney`, no `Sell`, no `Hire` remote. Most are server->client updates.

## LocalPlayer attributes (25)

`OnboardStep, assignedPlot, musicMuted, incomeMultsPurchased, incomeMult, KitchenBusy,
KitchenRemainingLog, KitchenRateLog, EMP_Cashier, EMP_DeckHand, EMP_Manager, EMP_FishTrader,
EMP_SousChef, EMP_Greeter, TraderThreshold, CurrentCount, NetWorth, NetWorthLog, Playtime,
GP_AutoMoney, GP_AutoLoot, GP_DoubleLoot, GP_FishTrader, GP_SousChef, GP_Greeter`

All `EMP_*` were **false** at probe time. `GP_AutoLoot` false.

## Dead ends (do not retry)

1. **Guessing `InstantSellButton` argument shapes.** v2.0.0 walked `()`, `(false)`, `(true)` every
   2 s with ~1.3 s of sleeps inside the loop. **It crashed the user's client.**
2. **Decompiling the button scripts or `CollectOfflineIncome`** — `Invalid bytecodes`.
3. **`CollectionService:GetTagged`** — nil in this VM.
4. **Reading `LootTier` as an attribute** — does not exist; the tier is in the name.
5. **Trusting a loot part disappearing as success** — the game destroys it client-side.
6. **Touching another player's buttons** — owner-validated, reads as griefing.
7. **`firetouchinterest` on `CollectButton.Button`** — zero change, twice, with the call sequence
   confirmed to run.
8. **Cross-map / long-range `LootPickup`.** Single far item at 98 studs `delta=0`; 86-pass run at
   33m+ `landed=0`. Measured, not assumed.
9. **`Humanoid:MoveTo` / `WalkToPoint` as a movement lever** — returns ok, moves nothing.
10. **`leaderstats["Net Worth"]`** — reads empty; use the `NetWorth` attribute.
11. **Gating any action behind an unreadable value** — v2.3.0 fired zero times in 38 passes.
12. **Any prompt- or TextButton-based approach** — a plot contains zero of both.
13. **v3.0.0 full button automation (chef/collect/merge/hire/deposit).** Rolled back at the user's
    request on 2026-11-02; do not re-offer it without being asked. v2.5.0 re-adds only deposit.

## The live build is v2.5.0

v2.4.0 pickup (fires `LootPickup` every pass at the nearest `PerPass` loot, no arrival gate, no
distance gate), plus the v2.4.1 fish-value overlay, plus market-gated auto deposit.

## Version history

- v1.1.0 — attribute-only tier read: `lootWithTier=0`. **Broken.**
- v1.2.0 — tier from name + touch + remote + prompt. **Worked** (user: "nice").
- v2.0.0 — auto-sell with three guessed shapes. **Crashed the client.**
- v2.1.0 — sell deleted, one toggle, `busy` guard. Pickup only.
- v2.2.0 — nearest-first sort, radius gate, teleport-to-loot. Teleport later removed.
- v2.3.0 — walk with `Humanoid:MoveTo`, fire only inside `Radius`. **Fired zero times** because
  `MoveTo` is inert and the arrival gate blocked every pass.
- v2.4.0 — arrival gate removed; fires every pass regardless of distance. **Pickup confirmed
  working by the user.**
- v2.4.1 — v2.4.0 pickup byte-identical, plus the fish-value multiplier overlay. **User: "nice".**
- v2.5.0 (2026-11-02) — adds market-gated auto deposit: teleport onto `DepositButton.Button` once
  at the slider threshold, latch, measure `CurrentCount`. Pushed and read back (sha `059c5015`,
  24,254 bytes). **Deposit not yet confirmed in game.**
- v3.0.0 — chef upgrade (priority), collect, merge, hire, market-gated deposit, slider.
  **Rolled back to v2.4.0 at the user's request on 2026-11-02.**
