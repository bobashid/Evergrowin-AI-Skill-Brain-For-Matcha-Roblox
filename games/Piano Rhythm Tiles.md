# Piano Rhythm Tiles

- **Script file:** `Piano Rhythm Tiles` in `bobashid/roblox-matcha` (current version: v2.7)
- **Place IDs:** 101999883467300 (source: `matcha_status`, 2026-10-26)
- **Hub entry:** PLACES yes (v2.4), SCRIPTS keys: pianorhythmtiles, pianotiles
- **Features:** auto play - ONE toggle and ONE slider (Fire Y). Keys only. Fire line, board-bottom line, per-note dot overlay.

## Confirmed facts
- Confirmed (2026-10-26): **mouse movement does not work in this game.** A user HUD read `cur 1919,1079` - the bottom-right corner of a 1920x1080 screen - and the cursor never moved from there, so every click fired into the corner outside the board. `mousemoveabs` and `mousemoverel` both return ok and change nothing. Mouse output was removed in v2.7. Proof: HUD screenshot from the user (`v2.6 | KEYS | focus true | taps 2 | fireY 772 | off +0 | lanes 0/630/798/965 | tile x798 => click x958 | cur 1919,1079 | chart 44`).
- Confirmed (2026-10-26): **a tile can report `AbsolutePosition.X = 0`**, which polluted the learned lane ladder: the HUD showed `lanes 0/630/798/965`, so lane width computed as (965-0)/3 = 321 instead of 167 and every target was wrong. Tile X is now only accepted inside 560..1400. Proof: same HUD screenshot.
- Confirmed (2026-10-26): **there is NO hit/press/release remote.** The only remote traffic is telemetry: `Request:InvokeServer("telemetry", {event="play_requested", mode="classic", input="keyboard"})`, `{event="offer", key="practiceMode", outcome="shown"}`, `{event="frames", taps=0, long=0, seconds=8.73, orient="landscape"}` - plus `clientErrors` and `songVote` from the decompile of `PianoClient`. **Hits are scored locally in `Client.Engine`**, so the remote route is closed permanently and hold notes can never be submitted via a remote. Proof: user-pasted remote calls plus `matcha_decompile` of `PlayerScripts.PianoClient`.
- Confirmed (2026-10-26): **keypress reaches the game** - v1.7 landed hits with `keypress` at FIRE_Y 950, and the user's own HUD in the failing run shows `KEYS` mode with `taps 2`, i.e. keys were being sent. Keys are the working route.
- Confirmed (2026-10-26): **timing is right, the lane was wrong** across three user reports ("right clicking time but wrong lane", "still wrong ... 4 lanes 4 keys", "clicking the wrong key please fix"). Timing is settled.
- Confirmed (2026-10-26): the single-character `TextLabel`s inside `PianoTiles` are the **judgement tally**, not key labels: `'0' x=640` above `PERFECT x=641`, `'0' x=803` above `COOL x=803`, `'0' x=965` above `GOOD x=965`, `'1' x=1127` above `MISS x=1127`.
- Confirmed (2026-10-26): live tiles read `965,511` and `1133,31`; the four lane columns are 630 / 798 / 965 / 1133 (left to right).
- Confirmed (2026-10-26): `Client.Input` resolves a mouse click or touch with `Board.laneAt(input.Position.X)` - X only - and maps `KeyCode -> lane` from a table built from the profile's key preset (`Shared.Config.KeyPresets`: D F J K, A S D F, 1 2 3 4, arrows).
- Confirmed (2026-10-26): `Shared.Config.DefaultKeys = {'D','F','J','K'}`; `Lanes = 4`, `ApproachTime = 1.5`, `TileLen = 0.375`, `BoardAspect = 0.62`.
- Confirmed (2026-10-26): **v2.2's push read back CORRUPTED** - tail replaced by a literal placeholder line. Always compare the read-back tail against what was sent.
- Confirmed (2026-10-26): v1.8 landed two clicks; v1.7 with FIRE_Y 950 works; firing at y >= 1034 was LATE.
- Confirmed (2026-10-26): board is x 625..1292 (667 wide), `BoardAspect 0.62` -> height ~1076, so y = 1104 is the park spot below the board, not the judge line.
- Confirmed (2026-10-26): tiles fall DOWNWARD, Y increases toward the bottom; a live chart is a 100 px ladder (idx 3 y=991 ... idx 12 y=91).
- Confirmed (2026-10-26): **`lp.Character` / `HumanoidRootPart` are nil in this game**, so `firetouchinterest` can never work here.
- Confirmed (2026-10-26): `Drawing` Line / Circle / Text all construct and render; a pooled overlay works in this game.

## Quirks and traps
- **Mouse input is dead here - do not rebuild it.** Both `mousemoveabs` and `mousemoverel` report success and leave the cursor pinned at the screen corner (1919,1079). Any version that clicks will fire outside the board and silently fall back to keys. Keys only.
- **Reject impossible tile X values.** A parked or hidden tile can report `X = 0`, which poisons the learned ladder and skews lane width. v2.7 only accepts 560..1400 and falls back to the known 630/798/965/1133 ladder.
- **Do not ask for the press remote - it does not exist.** Remotes are telemetry/errors/votes only; scoring is local.
- **Never map a lane with overlapping per-lane ranges.** Lanes are 167 px wide; padded ranges overlap and `for i = 1, 4` returns the lower lane every time. v2.7 uses nearest-column with a 120 px cap.
- **The working trigger is ONE plain threshold.** v1.6's speed prediction came from the same lagging `AbsolutePosition` readout and fired nothing. Do not re-add it.
- **`FIRE_Y` direction:** tiles fall downward, so **LOWER fires EARLIER**, HIGHER fires LATER.
- **Both halves of the input path are required:** `setrobloxinput(true)` at load *and* an `isrbxactive()` gate.
- **`isrbxactive` is nil inside the `matcha_exec` sandbox**, so probes cannot measure whether input lands.
- **This VM caps a chunk at 200 local variables.**
- `Board.AbsoluteSize` / `JudgeLine.AbsoluteSize` read `(0,0)`; `.Position` / `.Size` read nil. The line must be derived, never read.

## Overlays
- Yellow `Line` at `FIRE_Y`, follows the slider live; label reads `FIRE <n>`.
- Grey `Line` at y = 1076 (board bottom) for reference.
- A `Circle` dot on every live note at `tileX + 83, tileY`: cyan = far up, white = armed, yellow = within 60 px of the line, grey = already fired.

## Open questions (UNVERIFIED)
- UNVERIFIED (2026-10-26): **is the key order left-to-right D F J K on this account?** v2.7 assumes it from `Shared.Config.DefaultKeys`, but the client builds the map from the saved profile's preset. If presses still land in the wrong column, the order - not the geometry - is what to change next (`KEYS` in the script, and `LANE_FALLBACK` order stays fixed).
- UNVERIFIED (2026-10-26): the exact `FIRE_Y` the user settled on - timing confirmed right, number never reported. The slider exists so it can be reported.
- UNVERIFIED (2026-10-26): **hold notes.** v2.7 taps (press + 45 ms). Four independent keys could hold chords and long notes, which the mouse never could - that is the next feature if holds are wanted.
