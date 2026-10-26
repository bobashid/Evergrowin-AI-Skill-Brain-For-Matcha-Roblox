# Piano Rhythm Tiles

- **Script file:** `Piano Rhythm Tiles` in `bobashid/roblox-matcha` (current version: v2.9)
- **Place IDs:** 101999883467300 (source: `matcha_status`, 2026-10-26)
- **Hub entry:** PLACES yes (v2.4), SCRIPTS keys: pianorhythmtiles, pianotiles
- **Features:** auto play - ONE toggle + ONE Fire Y slider. Keys only. Hold notes sustained by keeping the lane key down while the tile persists. Fire line, board-bottom line, per-note dots.

## Confirmed facts
- User-stated (2026-10-26): **the game has hold notes** ("just know there are hold ones too"). Any autoplay that only taps will drop them. Proof: user statement.
- Confirmed (2026-10-26): **the script worked for the first few notes and then degraded** (user: "it was working fine for the first few notes after that it diddnt work well anymore"). The failure is mid-song, not at load, which points at state that accumulates or at a window only sparse opening bars satisfy. v2.9 addresses both candidates. Proof: user statement.
- Confirmed (2026-10-26): **mouse movement does not work in this game.** A user HUD read `cur 1919,1079` - the bottom-right corner of a 1920x1080 screen - and the cursor never moved, so every click fired outside the board. `mousemoveabs` and `mousemoverel` both return ok and change nothing. Mouse output removed from v2.7 onward. Proof: HUD screenshot (`v2.6 | KEYS | focus true | taps 2 | fireY 772 | off +0 | lanes 0/630/798/965 | tile x798 => click x958 | cur 1919,1079 | chart 44`).
- Confirmed (2026-10-26): **a tile can report `AbsolutePosition.X = 0`**, which polluted the learned lane ladder: HUD showed `lanes 0/630/798/965`, so lane width computed as (965-0)/3 = 321 instead of 167 and every target was wrong. Tile X is now only accepted inside 560..1400. Proof: same HUD screenshot.
- Confirmed (2026-10-26): **there is NO hit/press/release remote.** The only remote traffic is telemetry: `Request:InvokeServer("telemetry", {event="play_requested", mode="classic", input="keyboard"})`, `{event="offer", key="practiceMode", outcome="shown"}`, `{event="frames", taps=0, long=0, seconds=8.73, orient="landscape"}` - plus `clientErrors` and `songVote` from `PianoClient`. **Hits are scored locally in `Client.Engine`**, so the remote route is closed permanently. Proof: user-pasted remote calls plus `matcha_decompile` of `PlayerScripts.PianoClient`.
- Confirmed (2026-10-26): **keypress reaches the game** - v1.7 landed hits with `keypress` at FIRE_Y 950, and the failing v2.6 HUD showed `KEYS` mode with `taps 2`. Keys are the working route, and the only route that can hold several lanes at once.
- Confirmed (2026-10-26): **timing is right, the lane was wrong** across three earlier user reports ("right clicking time but wrong lane", "still wrong ... 4 lanes 4 keys", "clicking the wrong key please fix").
- Confirmed (2026-10-26): the single-character `TextLabel`s inside `PianoTiles` are the **judgement tally**, not key labels: `'0' x=640` above `PERFECT x=641`, `'0' x=803` above `COOL x=803`, `'0' x=965` above `GOOD x=965`, `'1' x=1127` above `MISS x=1127`. A key-label scan will find 5, not 4.
- Confirmed (2026-10-26): live tiles read `965,511` and `1133,31`; the four lane columns are 630 / 798 / 965 / 1133 (left to right).
- Confirmed (2026-10-26): `Client.Input` resolves a click or touch with `Board.laneAt(input.Position.X)` - X only - and maps `KeyCode -> lane` from a table built from the profile's key preset (`Shared.Config.KeyPresets`: D F J K, A S D F, 1 2 3 4, arrows).
- Confirmed (2026-10-26): `Shared.Config.DefaultKeys = {'D','F','J','K'}`; `Lanes = 4`, `ApproachTime = 1.5`, `TileLen = 0.375`, `BoardAspect = 0.62`, `LeadIn = 2.4`.
- Confirmed (2026-10-26): v1.8 landed two clicks; v1.7 with FIRE_Y 950 works; firing at y >= 1034 was LATE.
- Confirmed (2026-10-26): board is x 625..1292 (667 wide), `BoardAspect 0.62` -> height ~1076, so y = 1104 is the park spot below the board, not the judge line.
- Confirmed (2026-10-26): tiles fall DOWNWARD, Y increases toward the bottom; a live chart is a 100 px ladder (idx 3 y=991 ... idx 12 y=91).
- Confirmed (2026-10-26): **`lp.Character` / `HumanoidRootPart` are nil in this game**, so `firetouchinterest` can never work here.
- Confirmed (2026-10-26): `Drawing` Line / Circle / Text all construct and render; a pooled overlay works in this game.

## Quirks and traps
- **A narrow arming window silently drops notes in dense sections.** v2.7 armed a tile only if seen at `Y < FIRE_Y - 120`; one dropped frame in a fast passage can carry a tile past that window, so it never armed and was discarded with no error. v2.9 arms any tile first seen above `FIRE_Y`.
- **Never schedule a key release with `task.delay`.** A missed delayed release leaves the key down; Windows then delivers no new keydown for a held key, so Roblox raises no `InputBegan` and that lane is dead for the rest of the song. Release on a per-frame deadline and force a release immediately before each press.
- **A write can report a new sha while a later read returns the previous content.** Observed this session: a push reported a v2.8 sha and size, but two subsequent reads returned the v2.7 body and sha. Treat the read-back's version stamp as authoritative and re-push against the sha the file actually has, before telling the user a version is live.
- **Mouse input is dead here - do not rebuild it.** Both `mousemoveabs` and `mousemoverel` report success and leave the cursor pinned at the screen corner (1919,1079). Keys only.
- **Reject impossible tile X values.** A parked or hidden tile can report `X = 0`, which poisons the learned ladder and skews lane width. Accept only 560..1400, and fall back to the known 630/798/965/1133 ladder.
- **Do not ask for the press remote - it does not exist.** Remotes are telemetry/errors/votes only; scoring is local.
- **Never map a lane with overlapping per-lane ranges.** Lanes are 167 px wide; padded ranges overlap and `for i = 1, 4` returns the lower lane every time. Use nearest-column with a distance cap (120 px works).
- **The working trigger is ONE plain threshold.** v1.6's speed prediction came from the same lagging `AbsolutePosition` readout and fired nothing. Do not re-add it.
- **`FIRE_Y` direction:** tiles fall downward, so **LOWER fires EARLIER**, HIGHER fires LATER.
- **Both halves of the input path are required:** `setrobloxinput(true)` at load *and* an `isrbxactive()` gate.
- **`isrbxactive` is nil inside the `matcha_exec` sandbox**, so probes cannot measure whether input lands.
- **This VM caps a chunk at 200 local variables.**
- `Board.AbsoluteSize` / `JudgeLine.AbsoluteSize` read `(0,0)`; `.Position` / `.Size` read nil. The line must be derived, never read.

## Hold notes (v2.9)
- A hold is sustained by keeping that lane's key down while its tile is still live **and** has travelled past the line (`Y > FIRE_Y + 20`, up to `FIRE_Y + HOLD_TAIL` = 700). The release deadline is pushed forward every frame it qualifies, and the lane is released `TAP_MS` (45 ms) after the tile disappears, with `MAX_HOLD = 4 s` as a safety cap.
- A tap tile does not persist past the line, so it still gets a plain tap.
- Four independent keys can hold several lanes at once - the reason holds are keys-only and cannot be done with a mouse.

## Overlays
- Yellow `Line` at `FIRE_Y`, follows the slider live; label reads `FIRE <n>`.
- Grey `Line` at y = 1076 (board bottom) for reference.
- A `Circle` dot on every live note at `tileX + 83, tileY`: grey = already fired/written off, yellow = within 60 px of the line, white = armed.

## Open questions (UNVERIFIED)
- UNVERIFIED (2026-10-26): **does a hold tile actually persist past the line?** The whole hold mechanism depends on it. Test: play a song with long notes and read the HUD - `holds` climbing means tiles do persist and the sustain is firing; `holds` stuck at 0 while long notes still drop means the tile is destroyed at the line and holds need a different signal (a child frame such as `Spine`/`Ring`/`Dot`, or the chart clock).
- UNVERIFIED (2026-10-26): **does v2.9 hold up past the opening bars?** Test: play a full song with a dense section. `skipped` near 0 with `taps` climbing throughout means the arming fix is right; `skipped` climbing mid-song means reads are too slow and the trigger needs to move off a Y threshold.
- UNVERIFIED (2026-10-26): **is the key order left-to-right D F J K on this account?** Assumed from `Shared.Config.DefaultKeys`, but the client builds the map from the saved profile's preset. If presses land in the wrong column, the order - not the geometry - is what to change (`KEYS` in the script; `FALLBACK` order stays fixed).
- UNVERIFIED (2026-10-26): the exact `FIRE_Y` the user settles on - timing confirmed right, number never reported. The slider exists so it can be reported.
