# Piano Rhythm Tiles

- **Script file:** `Piano Rhythm Tiles` in `bobashid/roblox-matcha` (current version: v1.3)
- **Place IDs:** 101999883467300 (source: `matcha_status`, 2026-10-26)
- **Hub entry:** PLACES yes (v2.4), SCRIPTS keys: pianorhythmtiles, pianotiles
- **Features:** auto play - line-crossing trigger with a learned judge line, spawn-clock fallback, touch / mouse / key output, per-path counters, HUD

## Confirmed facts
- Confirmed (2026-10-26): the board is `PlayerGui.PianoTiles.Board`; its containers are `Lanes`, `Tiles`, `FX`, `JudgeLine`. Tiles are Frames named `Tile` under `Board.Tiles`. Proof: `matcha_tree` on `PlayerGui.PianoTiles` depth 3.
- Confirmed (2026-10-26): 4 lanes, left edges at x = 625, 792, 959, 1126, width 167, `SepEnd` at 1292. Proof: `AbsolutePosition.X` of `Lanes` children (`Flash1..4`, `Sep1..4`).
- Confirmed (2026-10-26): tiles carry attributes `TileIndex` and `ChartVersion`, and `.Address` is a usable identity key. Proof: printed per-tile attributes.
- Confirmed (2026-10-26): `Client.Input` picks the lane for a mouse click or touch with `Board.laneAt(input.Position.X)` - **X only, Y is never used to choose a lane**. Keys are matched separately by `KeyCode`. Proof: `matcha_decompile` of `PlayerScripts.Client.Input`.
- Confirmed (2026-10-26): **tiles fall DOWNWARD toward a judge line near the bottom of the screen.** A live snapshot of one chart read: idx 11 y=1004, idx 14 y=705, idx 17 y=404, idx 19 y=204, idx 20 y=104, idx 21 y=4, idx 22 y=-95, idx 23 y=-194 - later notes are higher up (negative Y) and earlier notes are near the bottom, so Y increases as a note approaches its hit. Judge line is therefore around y = 1100, not the middle of the screen. Proof: per-tile `AbsolutePosition` print in `matcha_exec`.
- Confirmed (2026-10-26): in the idle/menu state a tile sits at y = 1104 and disappears at 1104, i.e. parked on the judge line and never moving. Proof: vanish-tracking probe (`first=1104 last=1104 lo=1104 hi=1104`).
- Confirmed (2026-10-26): tile Y is NOT frozen globally - the earlier "frozen at 364" reading was the idle board. During a chart Y takes a whole spread of values. Proof: the 8-tile snapshot above.
- Confirmed (2026-10-26): `ReplicatedStorage.Shared.Config` holds `Lanes = 4`, `ApproachTime = 1.5`, `LeadIn = 2.4`, `TileLen = 0.375`, `DefaultKeys = {'D','F','J','K'}`. Proof: `matcha_decompile`.
- Confirmed (2026-10-26): `mouse1press`, `mouse1release`, `mouse1click`, `mousemoveabs`, `firetouchinterest`, `fireclickdetector`, `keypress` all exist and `pcall` returns `ok=true, err=nil`; `Camera.ViewportSize` reads `1920x1080`. Proof: `matcha_exec` global sweep.
- Confirmed (2026-10-26): `getgc("laneAt")` returns 5 entries whose `value` is a **string**, not the Board module table - so `getgc` cannot reach `Board.press` here. Do not build the "call the game's own press function" route on getgc key names. Proof: `matcha_exec` getgc probe printing `vtype=string press=nil`.

## Quirks and traps
- **Injected input is dead while the Roblox window is unfocused.** `keypress(0x44)` produced 0 `InputBegan` events when `isrbxactive()` was false, while board reads still worked. Any autoplay must show focus state and be tested with the game window active. Proof: `matcha_exec` input probe.
- **This VM caps a chunk at 200 local variables** - `Matcha: error: too many local variables (limit is 200, got 205)`. Keep probe and script bodies lean; prefer table fields over dozens of locals.
- `Board.AbsoluteSize` and `JudgeLine.AbsoluteSize` read `(0,0)`; frame `.Position` / `.Size` read nil. Only `AbsolutePosition` is readable and `JudgeLine.AbsolutePosition` is `(625, 0)` (useless) - the line must be learned or hardcoded, never read.
- "The auto play does nothing" was an infrastructure bug twice, not a timing bug: v1.0 hooked `RunService.PreRender`, which is nil in this build, inside a `pcall`, so the loop never ran. Always show a live counter so a dead loop is visible.

## Open questions (UNVERIFIED)
- UNVERIFIED (2026-10-26): the exact judged Y is around 1100-1104; the script learns it from tiles that vanish after moving. Test: read the in-game judgement text and trim the Line Y slider until every hit is PERFECT.
- UNVERIFIED (2026-10-26): hold tiles can be told apart from taps by a child frame (`Spine` / `Ring` / `Dot`) being visible. Test: print those children's `Visible` on a known hold tile.
- UNVERIFIED (2026-10-26): `firetouchinterest(hrp, laneFrame, 0)` registers as a lane press; the game's own handler reads `input.Position.X`, which a real touch carries and a synthetic one may not. Test: press Test touch lane 1 on a live tile and read the judgement.
