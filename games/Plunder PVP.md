# Plunder [PVP] (place 85646026816836)

`game.Name` reports **`Ugc`** (the creator group). The title **Plunder [PVP]** was supplied by the
user on 2026-10-29; the earlier name `Pulse Fitness Club` came from in-game signage and was wrong
as a game title. Never name a file after `game.Name` here.

- Script: `bobashid/roblox-matcha/Plunder PVP`, current **v1.3**, 22919 bytes.
- Hub: pinned in `PLACES` as of Hub **v3.2** (no Hub edit needed when a file keeps its name).
- Features: player ESP (box + name + distance), item ESP (cleaned labels, coins skipped),
  container/safe ESP with lock state, Max Distance and Transparency sliders.

## Confirmed (2026-10-29, live probes)

- **Players are `Workspace` Models named exactly `R15Rig`, carrying an attribute `userid`.**
  The local player's id (2251313101) appears on one rig; a second carried 10194758356; a third
  carried **-1500000017**, i.e. NPC rigs use a negative id - a reliable player/NPC split.
- `Players.GetPlayerByUserId` is **nil** here; build `[uid] = p.Name` from `Players:GetPlayers()`.
- The character model is a plain `R15Rig` in `Workspace`, not parented under the player. Do not
  rely on `lp.Character` for a per-frame ESP.
- `Model:GetBoundingBox` and `Model:GetExtentsSize` are **nil** here. Build the box from `Head` +
  `HumanoidRootPart`. Measured: Head `4.16 x 3.18 x 3.18`, HRP `1.76 x 2.0 x 0.8`.
- Items: `Workspace.GameSystem.WorldItem`, 35-42 children, a live count that changes as loot spawns
  and despawns. Each child has a real `BasePart` (`Handle`, `Tape`, `tripo_node_...`).
- **Containers are at `Workspace.GameArea.SpaceCenter.Container`** - 160 children, 20 kinds:
  `SpaceSupplyCase` x46, `TrashCan1SC` x13, `AeroToolChest` x12, `ComputerTower` x11,
  `ComputerTowerSC` x9, `DrawerCabinetSC` x8, `MetalLocker` x7, `MetalLockerSC` x7, `Server` x7,
  `TallCabinetSC` x7, `RocketPartsRack` x6, `SmallSafe` x5, `WallMedicalBoxSC` x5,
  `ControlConsole` x4, `SuitLocker` x4, `TrashCan2SC` x4, `LargeSafeSC` x2, `WallMedicalBox` x2,
  `TallCabinet` x1, `SampleFridge` x1.
- **That container folder is intermittent.** One probe returned `PATH true` with 160 children and
  7 safes; another minutes later returned `PATH false` with the identical code. Treat a nil folder
  as "area not loaded / round transition", not as a wrong path. The script re-resolves it every
  second scan and simply draws nothing while it is absent - do NOT "fix" it by hardcoding a new
  path in response to a nil read.
- Containers carry `SafeLocked`, `ContainerOpened`, `ContainerCid`, `ContainerLayout` attributes.
  `SafeLocked == true` is the lock state; the ESP shows it red with a `LOCKED` suffix. Confirmed on
  the safes themselves: `SmallSafe locked=true opened=false`, `LargeSafeSC locked=true
  opened=false`, one `SmallSafe locked=false`. Not every kind has it (`ComputerTower` had none), so
  the read is tri-state: true / false / nil-means-unknown.
- **World item names are not replicated.** `WorldItem` children have no `BillboardGui`,
  `SurfaceGui`, `TextLabel` or `StringValue` anywhere in their tree (walked to depth 8), and no
  attributes. Only mesh/part names exist (`Tape`, `SIG556`, `tripo_node_...`), so a label has to be
  built from those, skipping junk names.
- The 120 `ItemName` TextLabels belong to `Slot_<Name>` frames inside `Backpack` / `Pocket` /
  `SafeCase` **inventory GUIs**, not to world items. Reading them cannot label a floor item.
- `MarketplaceService:GetProductInfo` is **missing**, so the title must come from the user or the
  Roblox page.
- **v1.1 bug, fixed in v1.2:** `scanContainers` looked up `Workspace.GameScene.Container`, which
  does not exist, so the container toggle drew nothing for a whole version. The path above is
  correct; the function also falls back to a name search for any `Folder` named `Container` with
  more than 20 children.
- **User-stated (2026-10-29): coins must not be drawn by the item ESP.** v1.3 skips a coin by its
  **resolved label** (`SKIP_LABEL` plus a `coin` substring catch), applied in `scanItems` before the
  entry is added to the list. Do not implement this by adding `coin` to `JUNK`: `JUNK` only rejects
  a part name and falls through to the next part, so a coin would render as `Loot` instead of
  disappearing.

## UNVERIFIED

- Whether the coin's label actually reads `coin` in a live session. Coins never appeared in the
  `WorldItem` probes (`WorldItem_11`, `WorldItem_13`, `Tape`, `SIG556`), so the filter is a label
  match, not an observed name. If a coin still draws, read its on-screen label and add it.
- Whether the server validates range for the container prompts. Irrelevant to the ESP; matters if
  an auto-loot is added - use `fireproximityprompt`, then read state back.
- Whether the container folder is absent only between rounds or also while inside a match. Two
  samples only, one each way.

## Open questions

- Real item names would need the game's item registry. `ReplicatedStorage.ItemValue` is a
  `ModuleScript` and `require` fails in Matcha, so the only remaining route is decompiling that
  module and mapping model names to display names by hand.
