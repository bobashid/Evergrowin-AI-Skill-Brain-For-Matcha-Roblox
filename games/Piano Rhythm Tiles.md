# Piano Rhythm Tiles

- **Script file:** `Piano Rhythm Tiles` in `bobashid/roblox-matcha` (current version: v1.4)
- **Place IDs:** 101999883467300 (source: `matcha_status`, 2026-10-26)
- **Hub entry:** PLACES yes (v2.4), SCRIPTS keys: pianorhythmtiles, pianotiles
- **Features:** auto play - ONE toggle; clicks the lane when a tile reaches the judge line (line learned at runtime), spawn-clock fallback, HUD

## Confirmed facts
- Confirmed (2026-10-26): the board is `PlayerGui.PianoTiles.Board`; its containers are `Lanes`, `Tiles`, `FX`, `JudgeLine`. Tiles are Frames named `Tile` under `Board.Tiles`. Proof: `matcha_tree` on `PlayerGui.PianoTiles` depth 3.
- Confirmed (2026-10-26): 4 lanes, left edges at x = 625, 792, 959, 1126, width 167, `SepEnd` at 1292. Proof: `AbsolutePosition.X` of `Lanes` children (`Flash1..4`, `Sep1..4`).
- Confirmed (2026-10-26): tiles carry attributes `TileIndex` and `ChartVersion`, and `.Address` is a usable identity key. Proof: printed per-tile attributes.
- Confirmed (2026-10-26): `Client.Input` picks the lane for a mouse click or touch with `Board.laneAt(input.Position.X)` - **X only, Y is never used to choose a lane**. Keys are matched separately by `KeyCode`. Proof: `matcha_decompile` of `PlayerScripts.Client.Input`.
- Confirmed (2026-10-26): **tiles march DOWNWARD toward a judge line near the bottom (y ~ 1104), one note exactly 100 px above the next.** A 13-tile snapshot read idx 3 at y=991, idx 4 at 891, idx 5 at 791, idx 6 at 690, idx 7 at 590, idx 8 at 490, idx 9 at 390, idx 10 at 290, idx 11 at 191, idx 12 at 91 - a clean 100 px ladder with lower `TileIndex` nearer the line. So Y **increases** as a note approaches its hit and later notes sit higher up. Proof: per-tile `AbsolutePosition` print in `matcha_exec`.
- Confirmed (2026-10-26): in the idle/menu state a tile sits at y = 1104 and disappears at 1104, i.e. parked on the judge line and never moving. Proof: vanish-tracking probe (`first=1104 last=1104 lo=1104 hi=1104`).
- Confirmed (2026-10-26): tile Y is NOT frozen globally - the earlier "frozen at 364" reading was the idle board. During a chart Y takes a whole spread of values. Proof: the 10-tile 100 px ladder above.
- Confirmed (2026-10-26): **`lp.Character` / `HumanoidRootPart` are nil in this game** (`hrp=nil` while board and lanes read fine; the client sets the camera to a scripted `CFrame.new(0, 40, 0)`). Consequence: the `firetouchinterest(hrp, frame, 0)` route can never work here - there is no part to touch with. Use mouse clicks or the game's remote only. Proof: `matcha_exec` printing `hrp=nil`.
- Confirmed (2026-10-26): `ReplicatedStorage.Shared.Config` holds `Lanes = 4`, `ApproachTime = 1.5`, `LeadIn = 2.4`, `TileLen = 0.375`, `DefaultKeys = {'D','F','J','K'}`. Proof: `matcha_decompile`.
- Confirmed (2026-10-26): `mouse1press`, `mouse1release`, `mouse1click`, `mousemoveabs`, `firetouchinterest`, `fireclickdetector`, `keypress` all exist and `pcall` returns `ok=true, err=nil`; `Camera.ViewportSize` reads `1920x1080`. Proof: `matcha_exec` global sweep.
- Confirmed (2026-10-26): `getgc("laneAt")` returns 5 entries whose `value` is a **string**, not the Board module table - so `getgc` cannot reach `Board.press` here. Do not build the "call the game's own press function" route on getgc key names. Proof: `matcha_exec` getgc probe printing `vtype=string press=nil`.
- Confirmed (2026-10-26): remotes are `ReplicatedStorage.Remotes.Request` (RemoteFunction), `.Push` (RemoteEvent), `.DuelSync`; the client wraps calls as `Request:InvokeServer(<name>, <payload>)`. Proof: `matcha_decompile` of `PlayerScripts.PianoClient`.

## Quirks and traps
- **`isrbxactive` is nil inside the `matcha_exec` sandbox**, so a probe can never measure whether injected input lands. Only a script pushed through the loader can check focus. Proof: runtime hint `matcha_exec runs in a sandbox where isrbxactive is nil`.
- **Injected input is dead while the Roblox window is unfocused.** `keypress(0x44)` produced 0 `InputBegan` events when `isrbxactive()` was false. Testing therefore requires the game window active - and it reads `focus=false` whenever the user is typing in another window.
- **This VM caps a chunk at 200 local variables** - `Matcha: error: too many local variables (limit is 200, got 205)`. Keep probe and script bodies lean; prefer table fields over dozens of locals.
- `Board.AbsoluteSize` and `JudgeLine.AbsoluteSize` read `(0,0)`; frame `.Position` / `.Size` read nil. Only `AbsolutePosition` is readable and `JudgeLine.AbsolutePosition` is `(625, 0)` (useless) - the line must be learned or hardcoded, never read.
- "The auto play does nothing" was an infrastructure bug twice, not a timing bug: v1.0 hooked `RunService.PreRender`, which is nil in this build, inside a `pcall`, so the loop never ran. Always show a live counter so a dead loop is visible.

## Open questions (UNVERIFIED)
- UNVERIFIED (2026-10-26): the judged Y is ~1104 (idle park position and the vanish point); v1.4 learns it from tiles that vanish after moving. Test: read the in-game judgement text and confirm PERFECT.
- UNVERIFIED (2026-10-26): hold tiles can be told apart from taps by a child frame (`Spine` / `Ring` / `Dot`) being visible. Test: print those children's `Visible` on a known hold tile.
- UNVERIFIED (2026-10-26): the exact remote name and payload the client uses to report a hit - the next step if mouse clicks do not land. Test: decompile `PlayerScripts.Client.Engine`'s press handler for its `Request:InvokeServer(name, payload)` line.
