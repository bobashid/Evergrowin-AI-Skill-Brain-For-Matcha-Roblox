# Merge Block Arena (mPhase)

Silent auto merge, no teleports. Read this before touching the script.

## Identity

- Place id `76024476718474`, universe `10767663125`, creator **mPhase** (group).
- **`game.Name` reads `Ugc`, which is the creator group's name, not the title.** Confirmed (2026-10-27): live probe printed `GAME Ugc | PLACE 76024476718474`; the real title came from the web (`roblox.com/games/76024476718474` -> Merge Block Arena) and `apis.roblox.com/universes/v1/places/76024476718474/universe` -> `{"universeId":10767663125}`. Do not name the script file after `game.Name` in this place.
- `MarketplaceService:GetProductInfo` is missing in this VM (`attempt to call missing method`), so the title cannot be resolved in-game.

## Confirmed facts (probes, 2026-10-27)

- Own plot: `Workspace.PlayerPlots.Plot_2251313101`, matched by `LocalPlayer.UserId` appearing in the plot name.
- Plot children (14): `Model`, `Bridge` x2, `UpgradePad`, `Tree`, `PlayerBillboard`, `GroupReward`, `BlockPackPads`, `GearOfferPads`, `LuckyBlocks`, `Upgrades`, `Base`, `Ground`, `DroppedGears`.
- **No `Merge_N` models exist on the plot.** An earlier garbled exec echo appeared to show `Model Merge_1` repeated; two clean reads afterwards contradicted it and the `Merge_1` reading was retracted. Do not build a merge scanner on `^Merge_%d+$` names until a clean probe shows them.
- 13 ClickDetectors on the plot, all claim/skip buttons: `GroupReward.ClaimButton`, `BlockPackPads.BlockPackPad1-3.ClaimButton`, `GearOfferPads.GearOfferPad1-3.ClaimButton`, `Upgrades.Upgrade_SpawnTier.UpgradeButton`, `Upgrades.*.SkipButton`. `MaxActivationDistance` reads nil. All are reachable with `fireclickdetector` from anywhere -> free claims with no teleport.
- No ProximityPrompt in the workspace (`PROMPTS 0 CLICKS 90`), so nothing here is prompt-driven.
- `_PredictedFragments` exists in workspace with 0 children; `MergeBurst` is a Part but has no parent (probe `MBP nil`).
- LocalPlayer attributes (9): `1=Zone`, `BlockPackStage`, `MaxInventory`, `TutorialLock`, `TutorialMerge`, `Rebirths`, `TopBlockTier`, `RebirthReady`, `HoldingLuckyBlock`. Note the key literally named `1`.
- Remotes live under `ReplicatedStorage.Events` (plus a `ReplicatedStorage.RemoteEvents` Replica set): `PurchaseEvent`, `SetPreferences`, `ServerMessage`, `DropLuckyBlock`, `FragmentRequest`, `FragmentConfirm`, `FragmentBroadcast`, `GearAction`, `GearFX`, `GearDrop`, `UpgradeFX`, `UpgradeRequest`, `LuckyBlockFX`, `PunchRelease`, `PunchHeld`, `GearRecovery`, `StarterPack`, `Tutorial`, `AdminPromptRsvp`, `DestructionCash`, `KillFeed`, `ResetRequest`, `TeleportRequest`, `TeleportFX`, `OfflineEarnings`, `Rebirth`, `EventRsvp`. There are 40 remote objects in total.
- `FragmentRequest` is the guessed merge entry point (name only, argument shape UNKNOWN). The pushed script tries four shapes in order and keeps the first whose board signature changes.

## Script

- File `Merge Block Arena` in `bobashid/roblox-matcha`, `v1.0`, 10,698 bytes.
- Silent by construction: no CFrame write, no teleport, no root-part resize, no TweenService. Merge is remote-only.
- Defaults: `autoMerge` off, `dryRun` **on** (prints instead of firing), min tier 1, delay 200 ms.
- Untested against a real board: the scan found 0 blocks, so the merge path has never fired.

## Open question

Where do blocks actually live, and what is `FragmentRequest`'s argument shape? Need the game's merge handler or a clean listing of the folder that holds mergeable blocks.
