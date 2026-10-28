# Restaurant Tycoon 3

Place id **119048529960596**. `game.Name` reports **`Ugc`** (the creator group's name), so
name matching in Hub cannot find this place - it must be pinned in `PLACES`. Done in Hub v3.0.

Script: `Restaurant Tycoon 3` in `bobashid/roblox-matcha`.

## Confirmed (2026-10-27, live probes via matcha_exec)

- **Ownership is an ObjectValue, not an attribute.** `workspace.Tycoons` holds 8 children all
  named `Folder "Tycoon"`; `GetAttribute('Owner')` returns **nil** on every one. Each has a
  child `ObjectValue` named `Player`, so the owner test is `plot.Player.Value == LocalPlayer`.
  Proof: 8 plots probed, all `owner=nil`, then `MINE game.Workspace.Tycoons.Tycoon` resolved
  via the ObjectValue.
- **Currency:** `LocalPlayer.leaderstats` -> `IntValue` named `💸 Cash` (emoji + space in the
  name). Read 124, then 285 a few minutes later with no collecting done by hand. Do not match
  on the plain string `Cash`.
- **Cash rises on its own.** 124 -> 285 between two probes while the player stood still, so the
  tycoon has passive income. Any "auto collect money" script has to be judged against that:
  a flat balance is not proof the script did nothing, and a rising balance is not proof it
  worked. Measure the *piles* disappearing, not the balance.
- **Tycoon layout:** `<Tycoon>` -> `Locations` (6 CFrameValues: Base, FrontBase, DefaultEntrance,
  DefaultWalkPoint, StartOfRoad, RatingsBoard), `Player` (ObjectValue), `Objects` (Folder:
  `Food`, `Advertising`, `CookingResources`), `GreyBox` (Part), `Default` (Model), `Items`.
  `Items` has 7 children: BeingPlaced, Furniture (19), Surface (10), Walls (42), Floor (9),
  Config (2), Stairs (0).
- **Cash piles are not client-interactable.**
  `Tycoons.Tycoon.Items.Furniture.K32.Cash` holds 16 child Models (`Cash1`..`Cash16`), each with
  2 `MeshPart`s. Counted inside that subtree: **0 `TouchTransmitter`, 0 `ProximityPrompt`,
  0 `ClickDetector`, 0 `Attachment`**. So `firetouchinterest` / `fireproximityprompt` /
  `fireclickdetector` have nothing to fire at.
- **No dish or money words in prompt names.** All 350 `ProximityPrompt`/`ClickDetector` in
  `workspace` were listed. Real names: `RestaurantBoardPrompt`, `SeatPrompt`, `DoorPrompt`,
  `BuyPrompt` (cosmetics), `IngredientShopPrompt`, `GrowAllPrompt`, `Dialog`, `ChefLuigi`,
  `CreditsPrompt`, `EventHistoryMachinePrompt`, plain `ProximityPrompt`. A keyword scan for
  dish/money/trash terms matched **0** of them.
- **Almost no dish objects in the world.** Whole-workspace name scan: `plate` -> 7 hits, all
  `TestNewPlate` union operations under `Model 3`; `dirty` -> **0** hits. Plates/trays/food on
  tables exist only as `Attachment`s (`PlateHeight`, `Tray1`, `Tray2`, `Food1`..`Food6`) under
  `Base` parts, not as interactable instances.

## Remotes (Confirmed 2026-10-27, name scan of ReplicatedStorage)

26 RemoteEvent/RemoteFunction names matched collect/cash/tip/food/clean style terms. The ones
that matter, with their parents:

| Remote | Class | Parent |
|---|---|---|
| `GrabFood` | RemoteFunction | `Restaurant` |
| `TipsCollected` | RemoteEvent | `Restaurant` |
| `PlayerCollectedDrops` | UnreliableRemoteEvent | `Currency` |
| `CollectOfflineEarningsRequested` | RemoteEvent | `Upgrades` |
| `FoodConsumed` | RemoteEvent | `Food` |
| `FoodInspected` | RemoteEvent | `Food` |
| `FoodInventoryUpdated` | RemoteEvent | `Food` |
| `FoodOrdered` | RemoteEvent | `Food` |
| `ItemCollected` | RemoteEvent | `ScavengerHunt` |

`GrabFood`, `TipsCollected` and `PlayerCollectedDrops` are the three candidates for the money
path. None has been fired yet - **argument shapes are unknown**, so firing them blind would be
the same guess the script already wasted a version on.

## Where the client logic lives (Confirmed 2026-10-27)

The game ships an organised client tree under
`Players.<LocalPlayer>.PlayerScripts.Source` - `Systems.InterfaceController`,
`Modules.Player`, `Interfaces`, etc. (module search for `Tycoon` returned
`Source.Systems.FocusedTycoon`, `Source.Interfaces.TycoonStartMenu`,
`Source.Systems.InterfaceController.TycoonStartMenuController`). The cash/dish handler is
reachable by decompiling the module that fires `GrabFood` / `TipsCollected`, **not** by
scanning workspace names. That is the next probe.

## Consequence for the script

v1.0.0 of `Restaurant Tycoon 3` discovers prompts by name keyword and fires them. On this game
that finds **nothing**, so it is a no-op. It is not a bug in the code path - the game does not
expose these actions as prompts at all.

## What is needed next

1. Decompile the module that fires `GrabFood` / `TipsCollected` and read the argument shape.
2. Or: do it once by hand in game and say what you did (walked over the pile? clicked it? it
   vanished on its own after a timer?). That single sentence decides whether this is a touch,
   a remote call, or fully automatic and needs no script at all.

Do not ship a third keyword-guessing version. Two discovery attempts on this game have already
returned zero (prompt keywords, world name keywords).
