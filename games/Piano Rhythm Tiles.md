# Piano Rhythm Tiles

- **Script file:** `Piano Rhythm Tiles` in `bobashid/roblox-matcha` (current version: v2.3)
- **Place IDs:** 101999883467300 (source: `matcha_status`, 2026-10-26)
- **Hub entry:** PLACES yes (v2.4), SCRIPTS keys: pianorhythmtiles, pianotiles
- **Features:** auto play - one toggle, one Fire Y slider, one output toggle; **clicks the lane column** so the game resolves the lane itself, keys (D F J K) as fallback

## Confirmed facts
- Confirmed (2026-10-26): **the single-character `TextLabel`s inside `PianoTiles` are the JUDGEMENT TALLY, not key labels.** Probe found five: `'0' x=640` sitting above `PERFECT x=641`, `'0' x=803` above `COOL x=803`, `'0' x=965` above `GOOD x=965`, `'1' x=1127` above `MISS x=1127`. Consequence: key order cannot be read from the GUI, so a label scan will always find 5 hits and must refuse to guess. Proof: two `matcha_exec` probes dumping single-letter labels and TextLabel names with X positions.
- Confirmed (2026-10-26): **live tiles read `965,511` and `1133,31`** - the lane columns 3 and 4 from the 630/798/965/1133 set, so tile X is stable and lane discovery from it works. Proof: `matcha_exec` printing `tiles 2: 965,511 1133,31`.
- Confirmed (2026-10-26): **lane columns are correct and the KEY was the remaining error.** User reported after v2.1: "still wrong please look into it there is 4 lanes 4 keys to press". Since the columns were verified from tile X, the wrong part can only be the column-to-key mapping, which the client builds from the saved profile's key preset (`Shared.Config.KeyPresets`: D F J K, A S D F, 1 2 3 4, or arrows).
- Confirmed (2026-10-26): `Client.Input` resolves a mouse click or touch with `Board.laneAt(input.Position.X)` - **the game itself decides the lane from the click X**, so clicking the column removes the key-order question entirely. Proof: `matcha_decompile` of `PlayerScripts.Client.Input`.
- Confirmed (2026-10-26): `Client.Input` maps `KeyCode -> lane` through a table built from the profile's key preset, so the key per lane is profile data, not a constant.
- Confirmed (2026-10-26): **v2.2's push read back CORRUPTED** - the tail came back as a literal placeholder line (`-- ==== menu / stop / registration identical pattern, omitted here ====`), so that file was not a valid script. Rewritten in full as v2.3. Proof: `get_file_contents` on sha `85384ca5...`.
- Confirmed (2026-10-26): **a tile's `AbsolutePosition.X` is its lane's left edge plus about 5.** Observed tile X values 630 / 798 / 965 / 1133 against lane left edges 625 / 792 / 959 / 1127. Proof: per-tile `AbsolutePosition` prints.
- Confirmed (2026-10-26): **timing is correct** (user: "right clicking time"). The Fire Y slider works and applies live.
- Confirmed (2026-10-26): **v1.8 landed two clicks** (user: "you got two clicks in thats good").
- Confirmed (2026-10-26): **v1.7 with `FIRE_Y = 950` works.**
- Confirmed (2026-10-26): the board is `PlayerGui.PianoTiles.Board`; containers `Lanes`, `Tiles`, `FX`, `JudgeLine`. Tiles are Frames named `Tile` under `Board.Tiles`.
- Confirmed (2026-10-26): 4 lanes, left edges x = 625, 792, 959, 1127, width 167, `SepEnd` 1292.
- Confirmed (2026-10-26): tiles carry `TileIndex` and `ChartVersion` attributes; `.Address` is a usable identity key.
- Confirmed (2026-10-26): **tiles fall DOWNWARD, Y increases toward the bottom.** Live chart is a 100 px ladder: idx 3 y=991 ... idx 12 y=91.
- Confirmed (2026-10-26): **board geometry:** board is 667 px wide (625..1292), `BoardAspect 0.62` -> height ~1076, so the board occupies y 0..1076 and **y = 1104 is the park spot below it, NOT the judge line**.
- Confirmed (2026-10-26): **`lp.Character` / `HumanoidRootPart` are nil in this game** - `firetouchinterest` can never work here.
- Confirmed (2026-10-26): `ReplicatedStorage.Shared.Config` holds `Lanes = 4`, `ApproachTime = 1.5`, `LeadIn = 2.4`, `TileLen = 0.375`, `BoardAspect = 0.62`, `DefaultKeys = {'D','F','J','K'}`.
- Confirmed (2026-10-26): `mouse1press`, `mouse1release`, `mouse1click`, `mousemoveabs`, `keypress`, `isrbxactive`, `setrobloxinput` all exist; `Camera.ViewportSize` = 1920x1080.
- Confirmed (2026-10-26): `getgc("laneAt")` returns strings, not the Board module table - cannot reach `Board.press` that way.
- Confirmed (2026-10-26): remotes are `ReplicatedStorage.Remotes.Request` (RemoteFunction), `.Push`, `.DuelSync`; client wraps `Request:InvokeServer(<name>, <payload>)`.
- Confirmed (2026-10-26): **firing at `y >= 1034` scored one hit and it was LATE.**

## Quirks and traps
- **Do not try to read the key order from the GUI.** The only single-character TextLabels are the judgement counts, which sit directly above PERFECT/COOL/GOOD/MISS. Any scan will find 5, not 4.
- **The column -> key mapping is profile data**, from `Shared.Config.KeyPresets`. Hardcoding D F J K is a guess that can be wrong per account. Prefer clicking the column, where `Board.laneAt(X)` decides.
- **Never map a lane with overlapping per-lane ranges.** Lanes are 167 px wide; padded ranges overlap and `for i = 1, 4` returns the lower lane every time. Take the highest lane whose left edge is at or below the tile X.
- **The working trigger is ONE plain threshold, not a prediction.** v1.6's speed prediction was derived from the same lagging `AbsolutePosition` readout and fired nothing. Do not re-add it.
- **`FIRE_Y` direction:** tiles fall downward, so **LOWER fires EARLIER**, HIGHER fires LATER.
- **Both halves of the input path are required:** `setrobloxinput(true)` at load *and* an `isrbxactive()` gate.
- **`isrbxactive` is nil inside the `matcha_exec` sandbox**, so probes cannot measure whether input lands.
- **This VM caps a chunk at 200 local variables.**
- `Board.AbsoluteSize` and `JudgeLine.AbsoluteSize` read `(0,0)`; `.Position` / `.Size` read nil. `JudgeLine.AbsolutePosition` is `(625, 0)` - useless. The line must be derived.
- A push can read back with a placeholder tail. Always compare the read-back tail against what was sent; if it differs, rewrite the whole file.

## Open questions (UNVERIFIED)
- UNVERIFIED (2026-10-26): **whether an injected `mouse1press` reaches `UserInputService` with a usable `Position`.** v2.3 depends on this. Test: run v2.3 with Click lanes ON and watch whether the HUD's `last click lane N xNNN` produces a hit in the matching column. If clicks do nothing, turn Click lanes OFF and report which column each key hits (via `Test key order`), then set `CFG.keys` to match.
- UNVERIFIED (2026-10-26): **the exact `FIRE_Y` the user settled on.** Timing confirmed right, number never reported. Ask once, then bake it in.
- UNVERIFIED (2026-10-26): **hold support.** The press is held while the triggering tile stays in `Board.Tiles`, released when it disappears (3 s cap). No hold tile has been observed. Test: watch HUD `holds` on a song with long notes; `holds` staying 0 while long notes drop means holds need the remote route.
- UNVERIFIED (2026-10-26): the exact remote name and payload for hit and release - the only exact route for holds. Test: decompile `PlayerScripts.Client.Engine`'s press handler for its `Request:InvokeServer(...)` line.
