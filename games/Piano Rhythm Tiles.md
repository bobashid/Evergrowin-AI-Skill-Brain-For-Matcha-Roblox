# Piano Rhythm Tiles

- **Script file:** `Piano Rhythm Tiles` in `bobashid/roblox-matcha` (current version: v1.7)
- **Place IDs:** 101999883467300 (source: `matcha_status`, 2026-10-26)
- **Hub entry:** PLACES yes (v2.4), SCRIPTS keys: pianorhythmtiles, pianotiles
- **Features:** auto play - ONE toggle; taps the lane when a tile crosses a single Y line. **WORKING as of v1.7 with FIRE_Y = 950.**

## Confirmed facts
- Confirmed (2026-10-26): **v1.7 with `FIRE_Y = 950` works.** User report: "its now working". The single-threshold trigger plus `setrobloxinput(true)` plus the `isrbxactive()` gate lands hits. Proof: user statement after testing v1.7.
- Confirmed (2026-10-26): the board is `PlayerGui.PianoTiles.Board`; its containers are `Lanes`, `Tiles`, `FX`, `JudgeLine`. Tiles are Frames named `Tile` under `Board.Tiles`. Proof: `matcha_tree` on `PlayerGui.PianoTiles` depth 3.
- Confirmed (2026-10-26): 4 lanes, left edges at x = 625, 792, 959, 1126, width 167, `SepEnd` at 1292. Proof: `AbsolutePosition.X` of `Lanes` children (`Flash1..4`, `Sep1..4`).
- Confirmed (2026-10-26): tiles carry attributes `TileIndex` and `ChartVersion`, and `.Address` is a usable identity key. Proof: printed per-tile attributes.
- Confirmed (2026-10-26): `Client.Input` picks the lane for a mouse click or touch with `Board.laneAt(input.Position.X)` - **X only, Y is never used to choose a lane**. Keys are matched separately by `KeyCode`. Proof: `matcha_decompile` of `PlayerScripts.Client.Input`.
- Confirmed (2026-10-26): **tiles fall DOWNWARD and Y increases toward the bottom.** A live chart read as a clean 100 px ladder: idx 3 y=991, idx 4 y=891, idx 5 y=791, idx 6 y=690, idx 7 y=590, idx 8 y=490, idx 9 y=390, idx 10 y=290, idx 11 y=191, idx 12 y=91 - lower `TileIndex` nearer the bottom. Proof: per-tile `AbsolutePosition` print in `matcha_exec`.
- Confirmed (2026-10-26): **board geometry.** `Board.AbsolutePosition` is (625, 0) and the lanes end at 1292, so the board is **667 px wide**. `Shared.Config.BoardAspect = 0.62` gives a board height of 667 / 0.62 ~ **1076 px**, so the board occupies **y 0..1076**. Proof: `matcha_exec` print of `Board.AbsolutePosition` plus the `BoardAspect` value in `Shared.Config`.
- Confirmed (2026-10-26): in the idle/menu state a tile sits at y = 1104 and disappears at 1104. **1104 is BELOW the board bottom (1076), i.e. the park spot where a clipped-away tile waits - NOT the judged line.** Proof: vanish-tracking probe (`first=1104 last=1104 lo=1104 hi=1104`) compared against the computed board height.
- Confirmed (2026-10-26): tile Y is NOT frozen globally - the earlier "frozen at 364" reading was the idle board. During a chart Y takes a whole spread of values. Proof: the 100 px ladder above.
- Confirmed (2026-10-26): **`lp.Character` / `HumanoidRootPart` are nil in this game** (`hrp=nil` while board and lanes read fine; the client sets the camera to a scripted `CFrame.new(0, 40, 0)`). Consequence: the `firetouchinterest(hrp, frame, 0)` route can never work here - there is no part to touch with. Use mouse clicks or the game's remote only. Proof: `matcha_exec` printing `hrp=nil`.
- Confirmed (2026-10-26): `ReplicatedStorage.Shared.Config` holds `Lanes = 4`, `ApproachTime = 1.5`, `LeadIn = 2.4`, `TileLen = 0.375`, `BoardAspect = 0.62`, `DefaultKeys = {'D','F','J','K'}`. Proof: `matcha_decompile`.
- Confirmed (2026-10-26): `mouse1press`, `mouse1release`, `mouse1click`, `mousemoveabs`, `firetouchinterest`, `fireclickdetector`, `keypress` all exist and `pcall` returns `ok=true, err=nil`; `Camera.ViewportSize` reads `1920x1080`. Proof: `matcha_exec` global sweep.
- Confirmed (2026-10-26): `getgc("laneAt")` returns 5 entries whose `value` is a **string**, not the Board module table - so `getgc` cannot reach `Board.press` here. Proof: `matcha_exec` getgc probe printing `vtype=string press=nil`.
- Confirmed (2026-10-26): remotes are `ReplicatedStorage.Remotes.Request` (RemoteFunction), `.Push` (RemoteEvent), `.DuelSync`; the client wraps calls as `Request:InvokeServer(<name>, <payload>)`. Proof: `matcha_decompile` of `PlayerScripts.PianoClient`.
- Confirmed (2026-10-26): **a build firing at `y >= 1034` scored exactly one hit, and it was LATE.** So the tap must fire earlier in readout-space than 1034. Proof: user report after testing v1.5 ("i did get one but very late").

## Quirks and traps
- **The working trigger is ONE plain threshold, not a prediction.** v1.5 (1034) was late; v1.6 added speed prediction derived from the same lagging `AbsolutePosition` readout and fired nothing; v1.7 dropped to `pos.Y >= FIRE_Y` with `FIRE_Y = 950` and works. Do not re-add prediction, line learning or a spawn clock to this game.
- **`FIRE_Y` direction, get this right:** tiles fall downward, so **a LOWER `FIRE_Y` fires EARLIER** and a HIGHER one fires LATER. A draft of v1.7 had this backwards in a comment.
- **Both halves of the input path are required:** `setrobloxinput(true)` once at load *and* an `isrbxactive()` gate. `keypress(0x44)` produced 0 `InputBegan` events when `isrbxactive()` was false, so input is dropped entirely on an unfocused window while DataModel reads keep working.
- **`isrbxactive` is nil inside the `matcha_exec` sandbox**, so a probe can never measure whether injected input lands. Only a script pushed through the loader can check focus. Proof: runtime hint `matcha_exec runs in a sandbox where isrbxactive is nil`.
- **This VM caps a chunk at 200 local variables** - `Matcha: error: too many local variables (limit is 200, got 205)`. Keep probe and script bodies lean.
- `Board.AbsoluteSize` and `JudgeLine.AbsoluteSize` read `(0,0)`; frame `.Position` / `.Size` read nil. Only `AbsolutePosition` is readable and `JudgeLine.AbsolutePosition` is `(625, 0)` (useless) - the line must be derived from board geometry, never read.
- "The auto play does nothing" was an infrastructure bug twice, not a timing bug: v1.0 hooked `RunService.PreRender`, which is nil in this build, inside a `pcall`, so the loop never ran. Always show a live counter so a dead loop is visible.

## Open questions (UNVERIFIED)
- UNVERIFIED (2026-10-26): the exact judged Y is bracketed by **950 (works)** and **1034 (late)**, so it sits between them. Firing at 950 lands hits; nudging up to ~980 may score tighter (PERFECT instead of GREAT/GOOD) and nudging past ~1000 is expected to go late again. Test: raise `FIRE_Y` to 980, play a song, and read the judgement text; revert to 950 if LATE returns.
- UNVERIFIED (2026-10-26): hold tiles can be told apart from taps by a child frame (`Spine` / `Ring` / `Dot`) being visible. The current script taps once per tile and holds nothing, so long notes may still break. Test: print those children's `Visible` on a known hold tile.
- UNVERIFIED (2026-10-26): the exact remote name and payload the client uses to report a hit - the fallback if key input is ever rejected. Test: decompile `PlayerScripts.Client.Engine`'s press handler for its `Request:InvokeServer(name, payload)` line.
