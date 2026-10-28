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
  name). Read 124 during the probe. Do not match on the plain string `Cash`.
- **Tycoon layout:** `<Tycoon>` -> `Locations` (6 CFrameValues: Base, FrontBase, DefaultEntrance,
  DefaultWalkPoint, StartOfRoad, RatingsBoard), `Player` (ObjectValue), `Objects` (Folder:
  `Food`, `Advertising`, `CookingResources`), `GreyBox` (Part), `Default` (Model), and `Items`.
- **Cash piles are not client-interactable.**
  `Tycoons.Tycoon.Items.Furniture.K32.Cash` holds 16 child Models (`Cash1`..`Cash16`), each with
  2 `MeshPart`s. Counted inside that subtree: **0 `TouchTransmitter`, 0 `ProximityPrompt`,
  0 `ClickDetector`**. So `firetouchinterest` / `fireproximityprompt` / `fireclickdetector`
  have nothing to fire at.
- **No dish or money words in prompt names.** All 350 `ProximityPrompt`/`ClickDetector` in
  `workspace` were listed. Real names: `RestaurantBoardPrompt`, `SeatPrompt`, `DoorPrompt`,
  `BuyPrompt` (cosmetics), `IngredientShopPrompt`, `GrowAllPrompt`, `Dialog`, `ChefLuigi`,
  `CreditsPrompt`, `EventHistoryMachinePrompt`, plain `ProximityPrompt`. A keyword scan for
  dish/money/trash terms matched **0** of them.
- **Almost no dish objects in the world.** Whole-workspace name scan: `plate` -> 7 hits, all
  `TestNewPlate` union operations under `Model 3`; `dirty` -> **0** hits. Plates/trays/food on
  tables exist only as `Attachment`s (`PlateHeight`, `Tray1`, `Tray2`, `Food1`..`Food6`) under
  `Base` parts, not as interactable instances.
- **No obvious collect remote.** Scan of `ReplicatedStorage` descendants for
  RemoteEvent/RemoteFunction names containing collect/cash/tip/clean/dish/plate/trash/money/
  wash/dirty/food/serve returned **0** hits. Treat as a lead, not proof: one scan, name-based
  only, and the same scan also echoed stale lines.

## Consequence for the script

v1.0.0 of `Restaurant Tycoon 3` discovers prompts by name keyword and fires them. On this game
that finds **nothing**, so it is a no-op. It is not a bug in the code path - the game does not
expose these actions as prompts at all.

## What is needed next (one dump, one question)

What actually collects the cash and clears the dishes is **not yet known**. The cheapest
answers, best first:

1. The game's own client handler for cash pickup (the function that runs when you collect a
   pile) - one function, not a whole script.
2. Or: do it once by hand in game and say what you did (walked over the pile? clicked it? it
   vanished on its own after a timer?). That single sentence decides whether this is a touch,
   a raycast-on-server, or fully automatic and needs no script at all.

Do not ship a third keyword-guessing version. Two discovery attempts on this game have already
returned zero (prompt keywords, world name keywords).
