# Piano Rhythm Tiles

- **Script file:** `Piano Rhythm Tiles` in `bobashid/roblox-matcha` (current version: v2.0)
- **Place IDs:** 101999883467300 (source: `matcha_status`, 2026-10-26)
- **Hub entry:** PLACES yes (v2.4), SCRIPTS keys: pianorhythmtiles, pianotiles
- **Features:** auto play - one toggle plus one Fire Y slider; taps on the Y line, holds the lane while that tile persists, lane math fixed

## Confirmed facts
- Confirmed (2026-10-26): **the lane mapping was off by one to the left.** `laneOf` tested a range `laneX[i]-12 .. laneX[i]+laneW+12` per lane, and those ranges overlap (lane 1 ended at 804, lane 2 began at 780), so the loop always matched the LOWER lane: tile x 798 -> lane 1 (should be 2), 965 -> lane 2 (should be 3), 1133 -> lane 3 (should be 4). User report: "right clicking time but wrong key". Fix: iterate lanes 4..1 and take the highest lane whose left edge is at or below x - 12. Proof: user statement plus the observed tile X values.
- Confirmed (2026-10-26): **a tile's `AbsolutePosition.X` is its lane's left edge plus about 6.** Observed tile X values were exactly 630, 798, 965, 1133 against lane left edges 625, 792, 959, 1127. Proof: per-tile `AbsolutePosition` prints in `matcha_exec`.
- Confirmed (2026-10-26): **timing is correct with the v1.9 slider** (user: "right clicking time"). The Fire Y slider works and applies live.
- Confirmed (2026-10-26): **v1.8 landed two clicks** (user: "you got two clicks in thats good"). Proof: user statement after testing v1.8.
- Confirmed (2026-10-26): **v1.7 with `FIRE_Y = 950` works.** The single-threshold trigger plus `setrobloxinput(true)` plus the `isrbxactive()` gate lands hits. Proof: user statement after testing v1.7.
- Confirmed (2026-10-26): the board is `PlayerGui.PianoTiles.Board`; its containers are `Lanes`, `Tiles`, `FX`, `JudgeLine`. Tiles are Frames named `Tile` under `Board.Tiles`. Proof: `matcha_tree` on `PlayerGui.PianoTiles` depth 3.
- Confirmed (2026-10-26): 4 lanes, left edges at x = 625, 792, 959, 1127, width 167, `SepEnd` at 1292. Proof: `AbsolutePosition.X` of `Lanes` children (`Flash1..4`, `Sep1..4`).
- Confirmed (2026-10-26): tiles carry attributes `TileIndex` and `ChartVersion`, and `.Address` is a usable identity key. Proof: printed per-tile attributes.
- Confirmed (2026-10-26): `Client.Input` picks the lane for a mouse click or touch with `Board.laneAt(input.Position.X)` - **X only, Y is never used to choose a lane**. Keys are matched separately by `KeyCode`. Proof: `matcha_decompile` of `PlayerScripts.Client.Input`.
- Confirmed (2026-10-26): **tiles fall DOWNWARD and Y increases toward the bottom.** A live chart read as a clean 100 px ladder: idx 3 y=991, idx 4 y=891, idx 5 y=791, idx 6 y=690, idx 7 y=590, idx 8 y=490, idx 9 y=390, idx 10 y=290, idx 11 y=191, idx 12 y=91 - lower `TileIndex` nearer the bottom. Proof: per-tile `AbsolutePosition` print in `matcha_exec`.
- Confirmed (2026-10-26): **board geometry.** `Board.AbsolutePosition` is (625, 0) and the lanes end at 1292, so the board is **667 px wide**. `Shared.Config.BoardAspect = 0.62` gives a board height of 667 / 0.62 ~ **1076 px**, so the board occupies **y 0..1076**. Proof: `matcha_exec` print of `Board.AbsolutePosition` plus the `BoardAspect` value in `Shared.Config`.
- Confirmed (2026-10-26): in the idle/menu state a tile sits at y = 1104 and disappears at 1104. **1104 is BELOW the board bottom (1076), i.e. the park spot where a clipped-away tile waits - NOT the judged line.** Proof: vanish-tracking probe (`first=1104 last=1104 lo=1104 hi=1104`) compared against the computed board height.
- Confirmed (2026-10-26): tile Y is NOT frozen globally - the earlier "frozen at 364" reading was the idle board. During a chart Y takes a whole spread of values. Proof: the 100 px ladder above.
- Confirmed (2026-10-26): **`lp.Character` / `HumanoidRootPart` are nil in this game** (`hrp=nil` while board and lanes read fine; the client sets the camera to a scripted `CFrame.new(0, 40, 0)`). Consequence: the `firetouchinterest(hrp, frame, 0)` route can never work here. Proof: `matcha_exec` printing `hrp=nil`.
- Confirmed (2026-10-26): `ReplicatedStorage.Shared.Config` holds `Lanes = 4`, `ApproachTime = 1.5`, `LeadIn = 2.4`, `TileLen = 0.375`, `BoardAspect = 0.62`, `DefaultKeys = {'D','F','J','K'}`. Proof: `matcha_decompile`.
- Confirmed (2026-10-26): `mouse1press`, `mouse1release`, `mouse1click`, `mousemoveabs`, `firetouchinterest`, `fireclickdetector`, `keypress` all exist and `pcall` returns `ok=true, err=nil`; `Camera.ViewportSize` reads `1920x1080`. Proof: `matcha_exec` global sweep.
- Confirmed (2026-10-26): `getgc("laneAt")` returns 5 entries whose `value` is a **string**, not the Board module table. Proof: `matcha_exec` getgc probe printing `vtype=string press=nil`.
- Confirmed (2026-10-26): remotes are `ReplicatedStorage.Remotes.Request` (RemoteFunction), `.Push` (RemoteEvent), `.DuelSync`; the client wraps calls as `Request:InvokeServer(<name>, <payload>)`. Proof: `matcha_decompile` of `PlayerScripts.PianoClient`.
- Confirmed (2026-10-26): **a build firing at `y >= 1034` scored exactly one hit, and it was LATE.** Proof: user report after testing v1.5.

## Quirks and traps
- **Never map a lane with overlapping per-lane ranges.** Lanes are 167 px wide with a 12 px tolerance on each side, so adjacent ranges overlap by 24 px and `for i = 1, 4` returns the lower lane every time. Take the highest lane whose left edge is at or below the tile X instead (iterate 4 down to 1), which is what v2.0 does.
- **The working trigger is ONE plain threshold, not a prediction.** v1.5 (1034) was late; v1.6 added speed prediction derived from the same lagging `AbsolutePosition` readout and fired nothing; v1.7 dropped to `pos.Y >= FIRE_Y` with `FIRE_Y = 950` and works. Do not re-add prediction, line learning or a spawn clock to this game.
- **`FIRE_Y` direction:** tiles fall downward, so **a LOWER `FIRE_Y` fires EARLIER** and a HIGHER one fires LATER.
- **`FIRE_Y` is a slider on purpose** (user request). The arming point follows it automatically (`armY = FIRE_Y - 100`), so there is no second control to keep in sync and nothing to reset between songs.
- **Both halves of the input path are required:** `setrobloxinput(true)` once at load *and* an `isrbxactive()` gate. `keypress(0x44)` produced 0 `InputBegan` events when `isrbxactive()` was false, so input is dropped on an unfocused window while DataModel reads keep working.
- **`isrbxactive` is nil inside the `matcha_exec` sandbox**, so a probe can never measure whether injected input lands. Proof: runtime hint `matcha_exec runs in a sandbox where isrbxactive is nil`.
- **This VM caps a chunk at 200 local variables** - `Matcha: error: too many local variables (limit is 200, got 205)`.
- `Board.AbsoluteSize` and `JudgeLine.AbsoluteSize` read `(0,0)`; frame `.Position` / `.Size` read nil. Only `AbsolutePosition` is readable and `JudgeLine.AbsolutePosition` is `(625, 0)` (useless) - the line must be derived, never read.
- "The auto play does nothing" was an infrastructure bug twice, not a timing bug: v1.0 hooked `RunService.PreRender`, which is nil in this build, inside a `pcall`, so the loop never ran. Always show a live counter so a dead loop is visible.

## Open questions (UNVERIFIED)
- UNVERIFIED (2026-10-26): **the exact `FIRE_Y` value the user settled on.** Timing is confirmed right but the number was not reported. Ask once, then bake it in as the default.
- UNVERIFIED (2026-10-26): **the key order is assumed left to right = D F J K**, from `Shared.Config.DefaultKeys`. If the fixed lane math still gives a wrong lane, the lane-to-key order is what to change next (`CFG.keys`), not the lane mapping. The HUD prints `lane N @ x NNN` on every press so the two can be compared directly.
- UNVERIFIED (2026-10-26): **hold support (v1.8 onward).** The lane key stays down while the tile that triggered it is still in `Board.Tiles`, released when that tile disappears (3 s cap). Nothing about a hold tile has been observed yet. Test: watch the HUD `holds` counter on a song with long notes; `holds` staying 0 while long notes drop means the tile does not persist and holds need the remote route.
- UNVERIFIED (2026-10-26): hold tiles may be distinguishable by a child frame (`Spine` / `Ring` / `Dot`) being visible. Test: print those children's `Visible` on a known hold tile.
- UNVERIFIED (2026-10-26): the exact remote name and payload the client uses to report a hit and a release - the fallback if key input is ever rejected, and the only exact route for holds. Test: decompile `PlayerScripts.Client.Engine`'s press handler for its `Request:InvokeServer(name, payload)` line.
