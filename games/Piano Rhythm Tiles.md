# Piano Rhythm Tiles

- **Script file:** `Piano Rhythm Tiles` in `bobashid/roblox-matcha` (current version: v1.1)
- **Place IDs:** 101999883467300 (source: `matcha_status`, 2026-10-26)
- **Hub entry:** PLACES yes (v2.4), SCRIPTS keys: pianorhythmtiles, pianotiles
- **Features:** auto play - mouse-clicks the lane (or presses D F J K), timed off the tile spawn clock; offset / approach / hold / click-Y sliders; focus indicator; test buttons; HUD

## Confirmed facts
- Confirmed (2026-10-26): the board is `PlayerGui.PianoTiles.Board`; its containers are `Lanes`, `Tiles`, `FX`, `JudgeLine`. Tiles are Frames named `Tile` under `Board.Tiles`. Proof: `matcha_tree` on `game.Players.<LP>.PlayerGui.PianoTiles` depth 3.
- Confirmed (2026-10-26): 4 lanes, left edges at x = 625, 792, 959, 1126, width 167, `SepEnd` at 1292. Proof: `AbsolutePosition.X` of `Lanes` children (`Flash1..4`, `Sep1..4`).
- Confirmed (2026-10-26): tiles carry attributes `TileIndex` and `ChartVersion`, and `.Address` is a usable identity key. Proof: printed per-tile attributes.
- Confirmed (2026-10-26): **`AbsolutePosition.Y` of a tile does not change in this VM** - 157 samples over 4 s all read y = 364 while the tile was on screen. Screen Y therefore cannot drive hit timing; only the spawn clock or the game's own chart time can. Proof: two sampling probes.
- Confirmed (2026-10-26): `Board.AbsoluteSize` and `JudgeLine.AbsoluteSize` read `(0,0)`, and frame `.Position` / `.Size` read `nil`. Only `AbsolutePosition` is readable, and `JudgeLine.AbsolutePosition` is `(625, 0)` (useless). Proof: `matcha_exec` print of those properties.
- Confirmed (2026-10-26): `ReplicatedStorage.Shared.Config` holds `Lanes = 4`, `ApproachTime = 1.5`, `LeadIn = 2.4`, `TileLen = 0.375`, `MissGrace = 0.05`, `DefaultSpeed = 'normal'`, `DefaultKeys = {'D','F','J','K'}`. Proof: `matcha_decompile` of that ModuleScript.
- Confirmed (2026-10-26): `Client.Input` picks the lane for a mouse click or touch with `Board.laneAt(input.Position.X)` - **X only, Y is never used to choose a lane**, so a click anywhere in the lane's column counts. Keys are matched separately by `KeyCode`. Proof: `matcha_decompile` of `PlayerScripts.Client.Input`.
- Confirmed (2026-10-26): remotes are `ReplicatedStorage.Remotes.Request` (RemoteFunction), `.Push` (RemoteEvent), `.DuelSync`. Proof: `matcha_decompile` of `PlayerScripts.PianoClient` header lines.

## Quirks and traps
- While no song is running, tiles exist but never move and then vanish with a measured life of 0.00 s, so no live-chart measurement (scroll speed, judge line, hold length) can be taken from the menu state. Start a song before probing timing.
- `AbsoluteSize` being `(0,0)` means lane width must come from the `Lanes` separators, not from `AbsoluteSize`.
- **Injected input is dead while the Roblox window is unfocused.** `keypress(0x44)` produced 0 `InputBegan` events when `isrbxactive()` was false, while `GetChildren()` on the board still read fine (14 tiles). DataModel reads work unfocused; only input is gated. Any autoplay must show focus state to the user and must be tested with the game window active. Proof: `matcha_exec` input probe, 2026-10-26.
- "The auto play does nothing" has twice been an infrastructure bug, not a timing bug: v1.0 connected `RunService.PreRender`, which is nil in this build, inside a `pcall`, so the loop never ran. Check the loop is actually ticking (HUD counter) before tuning timing.

## Open questions (UNVERIFIED)
- UNVERIFIED (2026-10-26): a tile is created exactly `ApproachTime` (1.5 s) before its hit moment, so press time = spawn + 1.5 s + offset. Test: run with the approach slider and read the in-game judgement text; if it is consistently EARLY or COOL the lead is not 1.5 s.
- UNVERIFIED (2026-10-26): hold tiles can be told apart from taps by a child frame (`Spine` / `Ring` / `Dot`) being visible. Test: print those children's `Visible` on a known hold tile.
- UNVERIFIED (2026-10-26): calling the game's own `Board.press(lane, os.clock(), pos)` reached through `getgc` would bypass window focus entirely, unlike simulated mouse/key input. Test: `getgc` for the Board module table, confirm it exposes `press`, call it once and read the judgement.
