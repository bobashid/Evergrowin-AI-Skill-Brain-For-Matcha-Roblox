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
- `LocalPlayer:GetAttribute("CurrentCount")` is a **number** (read 77).
- `LocalPlayer.leaderstats["Net Worth"].Value` is a **STRING** like `"$1.3K"` — it must be
  parsed (`K`/`M`/`B`) before any delta is meaningful.
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

**This is why the remote viewer shows nothing for deposit/sell/collect:** the client never fires a
named remote for them. `firetouchinterest(hrp, buttonPart, 0)` then `(hrp, buttonPart, 1)` is the
whole mechanism, and it needs no argument shape at all.

## These scripts CANNOT be decompiled - do not retry

`DepositButtonScript`, `CollectButtonScript`, `InstantSellButtonScript`, `UpgradeButtonScript` and
`Plot_1.OfflineIncome.CollectOfflineIncome` all return **`-- Decompiler: Invalid bytecodes`**.
They are protected server Scripts. Reading them is a dead end; the touch mechanism is the way in.

## Offline income is a large uncollected number

`Plot_1.OfflineIncome` is a Model containing `Script:CollectOfflineIncome`, a `BillboardGui`,
`Part1`, `Part2`, `Highlight`. At probe time the billboard read:

    TextLabel:OfflineIncomeTitle = "Offline Fish"
    TextLabel:AmountText        = "713.7K"

So `CollectButton` is worth touching first — there is real value parked behind it.

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
   2 s with ~1.3 s of sleeps inside the loop. **It crashed the user's client.** The touch buttons
   make the remote unnecessary; do not go back to it.
2. **Decompiling the four button scripts or `CollectOfflineIncome`** — `Invalid bytecodes`.
3. **`CollectionService:GetTagged`** — nil in this VM, so tag-based targeting never works.
4. **Reading `LootTier` as an attribute** — it does not exist; the tier is in the name.
5. **Trusting a loot part disappearing as success** — the game's own client destroys it locally.
6. **Touching another player's buttons.** `CollectButton` on someone else's plot is almost
   certainly owner-validated and reads as griefing. Keep to your own plot.

## Still pending (the honest open items)

1. **Does `firetouchinterest` on `CollectButton.Button` actually pay out?** One-shot test written
   and handed to the user; it prints `BEFORE/AFTER offline=… worth=… count=…`. If nothing moves,
   the server wants the character physically present and the fix is teleport-next-to-button
   (the same approach that worked for `Attack` in the insect game), not more touch interest.
2. Whether `OfflineIncome` collection is capped per session or per hour — the script is protected,
   so only a live before/after answers it.
3. `KitchenEvent` / `BoatEvent` argument shapes (both exist, neither confirmed).

## Version history of builds shown to the user (inline, never pushed)

- v1.1.0 — attribute-only tier read: `lootWithTier=0` while 34 parts existed. **Broken.**
- v1.2.0 — tier from name + touch + remote + prompt. **Worked** (user: "nice").
- v2.0.0 — added auto-sell with three guessed shapes. **Crashed the client.**
- v2.1.0 — sell deleted (not disabled), one toggle, `busy` guard, cooldown measured from the end
  of a pass. Pickup only.
