# Sell a Million Sushi

Place id **125866884667103**. `game.Name` reports **`Ugc`**. Title supplied by the user as the
place URL slug **Sell-a-Million-Sushi**. Script: none pushed yet — see "Still pending" below.

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
- `LocalPlayer:GetAttribute("CurrentCount")` is a **number** (read 77, later 642).
- `LocalPlayer.leaderstats["Net Worth"].Value` is a **STRING** like `"$1.3K"` — it must be
  parsed (`K`/`M`/`B`) before any delta is meaningful. In the touch tests it printed **empty**,
  so the node may be a value-less holder or the label reads elsewhere.
- `CollectionService:GetTagged` / `GetTags` / `AddTag` are **nil** in this VM. Any target
  discovery must walk descendants.
- `firetouchinterest` is a **function**; `fireproximityprompt` and `fireclickdetector` exist.
- Loot counts at probe time: Plot_1=34, Plot_3=23, Plot_4=69, Plot_5=5, Plot_2=0, Plot_6=0.

## THE PICKUP REMOTE (decompiled `PlayerScripts.LootPickupScript`)

    RemoteEvents.LootPickup:FireServer(<array of LootTier numbers>, <isDeckHand boolean>)
    RemoteEvents.ChestPickup:FireServer("Chest" .. <ChestTier number>)

The game's own client builds that array from `workspace:GetPartBoundsInBox(CFrame.new(HRP.Position),
Vector3.new(5,15,5), params)` with `CollisionGroup = "Loot"` (and `"Chests"`), reads
`part:GetAttribute("LootTier")` off each hit, **destroys the part client-side**, then fires ONE
call with the whole array. `FireServer({3}, false)` is a one-element array, tier 3.

The `EMP_DeckHand` branch adds a second box around the plot's `TroughPart` and sets the boolean
true — that is what the second argument means.

**Consequence:** the client destroys loot locally inside ~5 studs, so **a part vanishing is not
evidence the server accepted anything.** The only honest witnesses are `CurrentCount` and parsed
`Net Worth`.

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

**Conclusion:** the `CollectButton` server script does not act on `firetouchinterest`. Note the
caveat: whether the CFrame teleport actually *held* is unproven — a server-authoritative game can
snap the character back before the touch lands, so test 2 is "touch did nothing", not proof the
character was genuinely standing on the button.

`type(firetouchinterest) == 'function'` only proves the global exists. Exactly the pattern already
recorded for `workspace:Raycast`: listed, callable, inert. A `fire*` call returning without error
is not evidence it delivered a `Touched` event.

## These scripts CANNOT be decompiled - do not retry

`DepositButtonScript`, `CollectButtonScript`, `InstantSellButtonScript`, `UpgradeButtonScript` and
`Plot_1.OfflineIncome.CollectOfflineIncome` all return **`-- Decompiler: Invalid bytecodes`**.
They are protected server Scripts. Reading them is a dead end.

## Offline income is a large uncollected number

`Plot_1.OfflineIncome` is a Model containing `Script:CollectOfflineIncome`, a `BillboardGui`,
`Part1`, `Part2`, `Highlight`. The billboard reads:

    TextLabel:OfflineIncomeTitle = "Offline Fish"
    TextLabel:AmountText        = "713.7K"

Unchanged across every touch attempt. `CollectButton` is worth touching first once the mechanism
is found — there is real value parked behind it.

## Deposit is tier-matched through the trough

`Plot_1.BoatLine.Boat_3` is a **Configuration** with attributes
`Owner, S0, STarget, Slot, T0, Tier, Docked`.
`ReplicatedStorage.FishermanModels` holds `Unit1`..`Unit24`, each with `Tier` and `FishName`.
`TroughPart` is a single `Part` with one `Attachment` and nothing interactive — a **volume
target**, fed by overlap, not clicked. Deposit therefore goes through the trough, not per-fish.

## ReplicatedStorage layout

Modules: `HelperModule`, `GameAnalytics`, `MusicModuleScript`, `UserInputModuleScript`,
`BigNumberModule`, `FishData`, `SushiStyle`, `KitchenConfig`, `BoatConfig`, `ChefAnimator`,
`FishNameStyle`, `EmployeeAnimator`.
Folders: `RemoteEvents`(29), `Audio`(21), `UI`(0), `Music`(1), `ChestImages`(6),
`SushiNPCs`(22), `BoatModels`(6), `LootModels`(24), `LeaderboardData`(3), `FishermanModels`(24).
Loose: `Model SushiChef`, `RemoteEvent PostieSent`, `PostieReceived`, `GameAnalyticsRemoteConfigs`,
`GameAnalyticsError`, `BindableEvent OnPlayerReadyEvent`.

### The 29 remotes

`ChangeSongLocal, UpdateCurrencyEvent, UpdateLootEvent, UpdateChestUnlockCost, ToggleMusicMuted,
StopSongLocal, PlaySellSound, PromptBuyCurrency, ShowHelpTextLocal, showGuideArrowLocal,
MessageFeedEvent, UpdatePaidMoneyButtons, LootPickup, UpdateCollection, UpdateTotalCount,
UpdateMoneyMultInfo, PromptBuyMoneyMult, CantAffordEffect, ChestPickup, TeleportHome, DiscardChest,
UpdateChestGui, ChestOpenAnimation, UnlockChest, InstantSellButton, EmployeeToggle, TraderThreshold,
KitchenEvent, BoatEvent`

Note what is **absent**: no `DepositFish`, no `CollectMoney`, no `Sell` remote. Most of these are
server→client updates (`Update*`, `PlaySellSound`, `CantAffordEffect`). `InstantSellButton` exists
but its argument shape was never confirmed.

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
3. **`CollectionService:GetTagged`** — nil in this VM, so tag-based targeting never works.
4. **Reading `LootTier` as an attribute** — it does not exist; the tier is in the name.
5. **Trusting a loot part disappearing as success** — the game's own client destroys it locally.
6. **Touching another player's buttons** — owner-validated, and reads as griefing.
7. **`firetouchinterest` on `CollectButton.Button`**, both with the character away and moved onto
   the button — zero change on `CurrentCount` and on the 713.7K offline billboard, twice, with the
   call sequence confirmed to run. Synthetic touch is not accepted by the server.

## Still pending (the honest open items)

1. **Does walking into `CollectButton` normally collect the 713.7K?** This is the decisive test and
   only the user can run it: walk into the button with no script, watch `Offline Fish`. If it
   collects -> the server wants genuine physics contact and the only remaining lever is real
   engine movement (`Humanoid:MoveTo` / `WalkToPoint`), not a synthetic touch. If it does not
   collect -> the button is disabled or gated (e.g. only after a true offline period, or a
   gamepass) and the whole path should be abandoned.
2. Whether `OfflineIncome` collection is capped per session or per hour — the script is protected,
   so only a live before/after answers it.
3. `KitchenEvent` / `BoatEvent` argument shapes (both exist, neither confirmed).

## Version history of builds shown to the user (inline, never pushed)

- v1.1.0 — attribute-only tier read: `lootWithTier=0` while 34 parts existed. **Broken.**
- v1.2.0 — tier from name + touch + remote + prompt. **Worked** (user: "nice").
- v2.0.0 — added auto-sell with three guessed shapes. **Crashed the client.**
- v2.1.0 — sell deleted (not disabled), one toggle, `busy` guard, cooldown measured from the end
  of a pass. Pickup only.
