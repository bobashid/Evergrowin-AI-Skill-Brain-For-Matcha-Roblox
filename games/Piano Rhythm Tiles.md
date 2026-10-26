# Piano Rhythm Tiles

- **Script file:** `Piano Rhythm Tiles` in `bobashid/roblox-matcha` (current version: v2.5)
- **Place IDs:** 101999883467300 (source: `matcha_status`, 2026-10-26)
- **Hub entry:** PLACES yes (v2.4), SCRIPTS keys: pianorhythmtiles, pianotiles
- **Features:** auto play - one toggle, one Fire Y slider, relative/absolute move toggle, mirror toggle; draws the fire line and a dot on every note

## Confirmed facts
- Confirmed (2026-10-26): **timing is right, the click lands in the wrong column** (user, three separate runs: "right clicking time but wrong lane", "still wrong please look into it there is 4 lanes 4 keys to press", "clicking the wrong key please fix"). Timing is therefore settled; only the lane is wrong.
- Confirmed (2026-10-26): **v2.4 clicked at the tile's own X and verified with `lp:GetMouse()`, and still landed wrong.** That eliminates "lane index arithmetic" as the cause, because v2.4 had no index in the click path. It leaves one assumption standing: that `mousemoveabs`, `lp:GetMouse()` and the game's `input.Position` share a coordinate space.
- Confirmed (2026-10-26): **the single-character `TextLabel`s inside `PianoTiles` are the JUDGEMENT TALLY, not key labels.** `'0' x=640` above `PERFECT x=641`, `'0' x=803` above `COOL x=803`, `'0' x=965` above `GOOD x=965`, `'1' x=1127` above `MISS x=1127` - five hits, so a key-label scan must refuse to guess.
- Confirmed (2026-10-26): live tiles read `965,511` and `1133,31` - columns 3 and 4 of the 630/798/965/1133 set.
- Confirmed (2026-10-26): `Client.Input` resolves a mouse click or touch with `Board.laneAt(input.Position.X)` - **X only**; the game itself decides the lane from the click X.
- Confirmed (2026-10-26): `Client.Input` maps `KeyCode -> lane` from a table built out of the profile's key preset, so the key per lane is **profile data** (`Shared.Config.KeyPresets`: D F J K, A S D F, 1 2 3 4, arrows).
- Confirmed (2026-10-26): a tile's `AbsolutePosition.X` is its lane's left edge + ~5 (630/798/965/1133 vs edges 625/792/959/1127).
- Confirmed (2026-10-26): **v2.2's push read back CORRUPTED** - tail replaced by a literal placeholder line. Rewritten in full. Always compare the read-back tail against what was sent.
- Confirmed (2026-10-26): v1.8 landed two clicks; v1.7 with FIRE_Y 950 works; firing at y >= 1034 was LATE.
- Confirmed (2026-10-26): board is x 625..1292 (667 wide), `BoardAspect 0.62` -> height ~1076, so y = 1104 is the park spot below the board, not the judge line.
- Confirmed (2026-10-26): tiles fall DOWNWARD, Y increases toward the bottom; a live chart is a 100 px ladder (idx 3 y=991 ... idx 12 y=91).
- Confirmed (2026-10-26): **`lp.Character` / `HumanoidRootPart` are nil in this game**, so `firetouchinterest` can never work here.
- Confirmed (2026-10-26): `mouse1press`, `mouse1release`, `mouse1click`, `mousemoveabs`, `mousemoverel`, `keypress`, `isrbxactive`, `setrobloxinput` all exist; `Camera.ViewportSize` = 1920x1080.
- Confirmed (2026-10-26): `Drawing` Line / Circle / Text all construct and render; a pooled overlay works in this game.

## Quirks and traps
- **Suspect the coordinate space before the lane maths.** Three versions in a row produced correct timing with the wrong column. v2.4 removed the lane index entirely and clicked at the tile's own X, verified by `lp:GetMouse()`, and was still wrong. When a verified move still misses, the verification and the consumer are reading different spaces - v2.5 therefore uses `mousemoverel` deltas, which are identical in every space.
- **Click at the tile's own X, never at an index-derived X.** `Board.laneAt(tileX)` is the call a human click goes through, so the lane is right by construction.
- **Do not try to read the key order from the GUI.** The only single-character TextLabels are the judgement counts above PERFECT/COOL/GOOD/MISS.
- **Never map a lane with overlapping per-lane ranges.** Lanes are 167 px wide; padded ranges overlap and `for i = 1, 4` returns the lower lane every time.
- **The working trigger is ONE plain threshold.** v1.6's speed prediction was derived from the same lagging `AbsolutePosition` readout and fired nothing. Do not re-add it.
- **`FIRE_Y` direction:** tiles fall downward, so **LOWER fires EARLIER**, HIGHER fires LATER.
- **Both halves of the input path are required:** `setrobloxinput(true)` at load *and* an `isrbxactive()` gate.
- **`isrbxactive` is nil inside the `matcha_exec` sandbox**, so probes cannot measure whether input lands.
- **This VM caps a chunk at 200 local variables.**
- `Board.AbsoluteSize` / `JudgeLine.AbsoluteSize` read `(0,0)`; `.Position` / `.Size` read nil. The line must be derived, never read.

## Overlays (v2.4 onward)
- Yellow `Line` at `FIRE_Y`, follows the slider live; label reads `FIRE <n>`.
- Grey `Line` at y = 1076 (board bottom) for reference.
- A `Circle` dot on every live note at `tileX + laneWidth/2, tileY`: cyan = far up, white = armed, yellow = within 60 px of the line, grey = already fired. If a dot is not in its note's column, `AbsolutePosition.X` is not trustworthy and the dot shows it.

## Open questions (UNVERIFIED)
- UNVERIFIED (2026-10-26): **do `mousemoveabs` and `input.Position` share a space?** This is the live hypothesis. Test: run v2.5 with the default REL mode and watch whether the click column matches the note column. Flip **Absolute mouse move** ON and compare - if REL is right and ABS is wrong, the spaces differ and REL is the fix for good.
- UNVERIFIED (2026-10-26): **is the board mirrored?** `Engine.setUpscroll` and line mode can flip the direction of travel. Test: if hits land in the mirror-image column (1 when it should be 4, 2 when it should be 3), turn **Mirror lanes** ON.
- UNVERIFIED (2026-10-26): the exact `FIRE_Y` the user settled on - timing confirmed right, number never reported.
- UNVERIFIED (2026-10-26): hold notes. v2.5 taps only (press + 35 ms) and one mouse button cannot sustain two lanes; holds need the remote route from `PlayerScripts.Client.Engine`'s press handler.
