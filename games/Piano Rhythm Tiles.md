# Piano Rhythm Tiles

- **Script file:** `Piano Rhythm Tiles` in `bobashid/roblox-matcha` (current version: v2.4)
- **Place IDs:** 101999883467300 (source: `matcha_status`, 2026-10-26)
- **Hub entry:** PLACES yes (v2.4), SCRIPTS keys: pianorhythmtiles, pianotiles
- **Features:** auto play - one toggle, one Fire Y slider; clicks at the tile's own X (game resolves the lane), verifies the cursor moved before pressing, draws the fire line and a dot on every note

## Confirmed facts
- Confirmed (2026-10-26): **timing is correct, lane was wrong** (user: "right clicking time but wrong lane"). Since the click demonstrably reaches the game, the remaining fault is either the lane index or the cursor never actually moving. v2.4 removes the index (clicks at the tile's own X) and verifies the cursor.
- Confirmed (2026-10-26): **the single-character `TextLabel`s inside `PianoTiles` are the JUDGEMENT TALLY, not key labels.** Probe found `'0' x=640` above `PERFECT x=641`, `'0' x=803` above `COOL x=803`, `'0' x=965` above `GOOD x=965`, `'1' x=1127` above `MISS x=1127` - five hits, so a key-label scan must refuse to guess. Proof: two `matcha_exec` label dumps.
- Confirmed (2026-10-26): **live tiles read `965,511` and `1133,31`** - columns 3 and 4 of the 630/798/965/1133 set, so tile X is stable and usable. Proof: `matcha_exec` printing `tiles 2: 965,511 1133,31`.
- Confirmed (2026-10-26): `Client.Input` resolves a mouse click or touch with `Board.laneAt(input.Position.X)` - **the game itself decides the lane from the click X**, so clicking at a tile's own X cannot be mis-indexed. Proof: `matcha_decompile` of `PlayerScripts.Client.Input`.
- Confirmed (2026-10-26): `Client.Input` maps `KeyCode -> lane` from a table built out of the profile's key preset, so the key per lane is **profile data**, not a constant. `Shared.Config.KeyPresets` offers D F J K, A S D F, 1 2 3 4 and the arrows.
- Confirmed (2026-10-26): **v2.2's push read back CORRUPTED** - the tail came back as a literal placeholder line (`-- ==== menu / stop / registration identical pattern, omitted here ====`). Rewritten in full as v2.3, then v2.4. Proof: `get_file_contents` on sha `85384ca5...`.
- Confirmed (2026-10-26): a tile's `AbsolutePosition.X` is its lane's left edge + ~5 (630/798/965/1133 vs edges 625/792/959/1127).
- Confirmed (2026-10-26): **v1.8 landed two clicks** (user), **v1.7 with FIRE_Y 950 works** (user), **a build firing at y >= 1034 was LATE** (user).
- Confirmed (2026-10-26): board is x 625..1292 (667 wide), `BoardAspect 0.62` -> height ~1076, so the board occupies y 0..1076 and **y = 1104 is the park spot below it, not the judge line**.
- Confirmed (2026-10-26): tiles fall DOWNWARD, Y increases toward the bottom. Live chart is a 100 px ladder: idx 3 y=991 ... idx 12 y=91.
- Confirmed (2026-10-26): board containers are `Lanes`, `Tiles`, `FX`, `JudgeLine`; tiles are Frames named `Tile` under `Board.Tiles`, carrying `TileIndex` and `ChartVersion`, with `.Address` as a usable key.
- Confirmed (2026-10-26): **`lp.Character` / `HumanoidRootPart` are nil in this game**, so `firetouchinterest` can never work here.
- Confirmed (2026-10-26): `Shared.Config` holds `Lanes = 4`, `ApproachTime = 1.5`, `LeadIn = 2.4`, `TileLen = 0.375`, `BoardAspect = 0.62`, `DefaultKeys = {'D','F','J','K'}`.
- Confirmed (2026-10-26): `mouse1press`, `mouse1release`, `mouse1click`, `mousemoveabs`, `keypress`, `isrbxactive`, `setrobloxinput` all exist; `Camera.ViewportSize` = 1920x1080; `lp:GetMouse()` reads `.X` / `.Y`.
- Confirmed (2026-10-26): `getgc("laneAt")` returns strings, not the Board module table - cannot reach `Board.press` that way.
- Confirmed (2026-10-26): remotes are `ReplicatedStorage.Remotes.Request` (RemoteFunction), `.Push`, `.DuelSync`; client wraps `Request:InvokeServer(<name>, <payload>)`.
- Confirmed (2026-10-26): `Drawing` types Line, Circle and Text all construct and render; the overlay pool approach works in this game (HUD text and dots render).

## Quirks and traps
- **Click at the tile's own X, never at an index-derived X.** `Board.laneAt(tileX)` is the same call a human click goes through, so the lane is correct by construction. Any scheme that maps column -> index -> key introduces a mapping that can be wrong per account.
- **Verify the cursor moved before pressing.** `mousemoveabs` can silently do nothing, which makes the press land in the column the cursor was already in - indistinguishable from a lane-index bug. v2.4 reads `lp:GetMouse().X/.Y` back and nudges with `mousemoverel` up to 3 times.
- **Do not try to read the key order from the GUI.** The only single-character TextLabels are the judgement counts above PERFECT/COOL/GOOD/MISS.
- **Never map a lane with overlapping per-lane ranges.** Lanes are 167 px wide; padded ranges overlap and `for i = 1, 4` returns the lower lane every time.
- **The working trigger is ONE plain threshold.** v1.6's speed prediction was derived from the same lagging `AbsolutePosition` readout and fired nothing. Do not re-add it.
- **`FIRE_Y` direction:** tiles fall downward, so **LOWER fires EARLIER**, HIGHER fires LATER.
- **Both halves of the input path are required:** `setrobloxinput(true)` at load *and* an `isrbxactive()` gate.
- **`isrbxactive` is nil inside the `matcha_exec` sandbox**, so probes cannot measure whether input lands.
- **This VM caps a chunk at 200 local variables.**
- `Board.AbsoluteSize` / `JudgeLine.AbsoluteSize` read `(0,0)`; `.Position` / `.Size` read nil. The line must be derived, never read.
- A push can read back with a placeholder tail. Always compare the read-back tail against what was sent; if it differs, rewrite the whole file.

## Overlays (v2.4)
- Yellow `Line` at `FIRE_Y`, follows the slider live; label reads `FIRE <n>`.
- Grey `Line` at y = 1076 (board bottom) for reference.
- A `Circle` dot on every live note at `tileX + laneWidth/2, tileY`: cyan = far up, white = armed, yellow = within 60 px of the line, grey = already fired. If a dot is not in its note's column, `AbsolutePosition.X` is wrong and the dot shows it.

## Open questions (UNVERIFIED)
- UNVERIFIED (2026-10-26): **does the dot line up with its note?** That is the decisive test for whether `AbsolutePosition.X` is trustworthy in this game. Watch a song with the overlays on: if the dots sit in their notes' columns, the coordinates are good and the lane fault is elsewhere; if they are offset, the offset value is the correction to apply.
- UNVERIFIED (2026-10-26): **does `mousemoveabs` actually move the cursor while Roblox is focused?** v2.4 verifies and falls back to keys after 4 failures; if the HUD shows `last move fail cursor x,y want x,y`, the deltas are the evidence.
- UNVERIFIED (2026-10-26): **the exact `FIRE_Y` the user settled on** - timing confirmed right, number never reported.
- UNVERIFIED (2026-10-26): **hold notes.** v2.4 taps only (press + 35 ms), and one mouse button cannot sustain two lanes. A hold needs the remote route, which needs the `Request:InvokeServer(...)` line from `PlayerScripts.Client.Engine`'s press handler.
