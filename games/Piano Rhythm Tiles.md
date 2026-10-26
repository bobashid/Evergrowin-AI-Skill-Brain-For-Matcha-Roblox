# Piano Rhythm Tiles

- **Script file:** not in `bobashid/roblox-matcha` yet (delivered inline at the user's request).
- **Place IDs:** 101999883467300 (source: `matcha_status`, 2026-10-26)
- **Hub entry:** no (no file to point at yet)
- **Features (target):** auto play - press the lane key at the judged moment; HUD diagnostics.

## Confirmed facts
- Confirmed (2026-10-26): the board is `PlayerGui.PianoTiles.Board`; its containers are `Lanes`, `Tiles`, `FX`, `JudgeLine`. Tiles are Frames named `Tile` under `Board.Tiles`. Proof: `matcha_tree` on `game.Players.<LP>.PlayerGui.PianoTiles` depth 3.
- Confirmed (2026-10-26): 4 lanes, left edges at x = 625, 792, 959, 1126, width 167, `SepEnd` at 1292. Proof: `AbsolutePosition.X` of `Lanes` children (`Flash1..4`, `Sep1..4`).
- Confirmed (2026-10-26): tiles carry attributes `TileIndex` and `ChartVersion`, and `.Address` is a usable identity key. Proof: printed per-tile attributes.
- Confirmed (2026-10-26): **`AbsolutePosition.Y` of a tile does not change in this VM** - 157 samples over 4 s all read y = 364 while the tile was on screen. Screen Y therefore cannot drive hit timing; only the spawn clock or the game's own chart time can. Proof: two sampling probes.
- Confirmed (2026-10-26): `Board.AbsoluteSize` and `JudgeLine.AbsoluteSize` read `(0,0)`, and frame `.Position` / `.Size` read `nil`. Only `AbsolutePosition` is readable, and `JudgeLine.AbsolutePosition` is `(625, 0)` (useless). Proof: `matcha_exec` print of those properties.
- Confirmed (2026-10-26): `ReplicatedStorage.Shared.Config` holds `Lanes = 4`, `ApproachTime = 1.5`, `LeadIn = 2.4`, `TileLen = 0.375`, `MissGrace = 0.05`, `DefaultSpeed = 'normal'`, `DefaultKeys = {'D','F','J','K'}`. Proof: `matcha_decompile` of that ModuleScript.
- Confirmed (2026-10-26): `Client.Input` maps `KeyCode` to a lane and calls `board.press(lane, os.clock(), pos)`; `UserInputType.Touch` and `MouseButton1` also press, using `Board.laneAt(position.X)`. Proof: `matcha_decompile` of `PlayerScripts.Client.Input`.
- Confirmed (2026-10-26): remotes are `ReplicatedStorage.Remotes.Request` (RemoteFunction), `.Push` (RemoteEvent), `.DuelSync`. Proof: `matcha_decompile` of `PlayerScripts.PianoClient` header lines.

## Quirks and traps
- While no song is running, tiles exist but never move and then vanish with a measured life of 0.00 s, so no live-chart measurement (scroll speed, judge line, hold length) can be taken from the menu state. Start a song before probing timing.
- `AbsoluteSize` being `(0,0)` means lane width must come from the `Lanes` separators, not from `AbsoluteSize`.

## Open questions (UNVERIFIED)
- UNVERIFIED (2026-10-26): a tile is created exactly `ApproachTime` (1.5 s) before its hit moment, so press time = spawn + 1.5 s + offset. Test: run with the approach slider and read the in-game judgement text; if it is consistently EARLY or COOL the lead is not 1.5 s.
- UNVERIFIED (2026-10-26): hold tiles can be told apart from taps by a child frame (`Spine` / `Ring` / `Dot`) being visible. Test: print those children's `Visible` on a known hold tile.
