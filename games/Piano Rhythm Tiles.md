# Piano Rhythm Tiles

- **Script file:** `Piano Rhythm Tiles` in `bobashid/roblox-matcha` (current version: v2.11)
- **Place IDs:** 101999883467300 (source: `matcha_status`, 2026-10-26)
- **Hub entry:** PLACES yes (v2.4), SCRIPTS keys: pianorhythmtiles, pianotiles
- **Features:** auto play - ONE toggle + ONE Fire Y slider. Keys only. Micro-jittered timing, hold notes sustained by tile lifetime.

## Confirmed facts
- User-stated (2026-10-26): **the perfect window opens around Y = 625.** Firing anywhere from there down to the working value scores, which is why a small jitter on the fire point cannot cause a miss. Proof: user statement ("625 is the beggining of where perfect starts").
- User-stated (2026-10-26): **v2.10 works well** ("it works good") but **holding does not work**, and the timing reads **too robotic**. Proof: user statement.
- Confirmed (2026-10-26): **a hold tile stops moving at the judge position while its body drains**, so requiring `Y > FIRE_Y + 20` for the sustain (v2.10) never engaged. v2.11 detects a hold purely by tile lifetime: a tile still alive `HOLD_GRACE` (90 ms) after its press is a hold, and the key stays down until the tile disappears. Proof: user report that holds did nothing, plus the fact that the Y-gated condition could not fire.
- Confirmed (2026-10-26): **v2.9 never compiled** - `Matcha:129: Expected identifier when parsing expression, got 'until'`. `until` is a Lua reserved keyword and cannot be a bare table key. Fixed in v2.10 by renaming the field to `due`. Proof: user-pasted console error, then a clean v2.10 read-back.
- User-stated (2026-10-26): **the game has hold notes** ("just know there are hold ones too").
- Confirmed (2026-10-26): **the script worked for the first few notes and then degraded**; the cause was a 120 px arming window that a dense passage could skip between frames. v2.10 onward arms any tile first seen above `FIRE_Y`. Proof: user statement, then working v2.10.
- Confirmed (2026-10-26): **mouse movement does not work in this game.** A HUD read `cur 1919,1079` - the bottom-right corner of a 1920x1080 screen - and the cursor never moved. `mousemoveabs` and `mousemoverel` both return ok and change nothing. Mouse output removed from v2.7 onward. Proof: HUD screenshot (`v2.6 | KEYS | focus true | taps 2 | fireY 772 | off +0 | lanes 0/630/798/965 | tile x798 => click x958 | cur 1919,1079 | chart 44`).
- Confirmed (2026-10-26): **a tile can report `AbsolutePosition.X = 0`**, which polluted the learned lane ladder (lane width computed as 321 instead of 167). Tile X is only accepted inside 560..1400. Proof: same HUD screenshot.
- Confirmed (2026-10-26): **there is NO hit/press/release remote.** Only telemetry (`play_requested`, `offer`, `frames`), `clientErrors` and `songVote` exist. **Hits are scored locally in `Client.Engine`**, so the remote route is closed permanently. Proof: user-pasted remote calls plus `matcha_decompile` of `PlayerScripts.PianoClient`.
- Confirmed (2026-10-26): **keypress reaches the game** - v1.7 landed hits with `keypress`, and the v2.6 HUD showed `KEYS` mode with taps climbing. Keys are the working route and the only route that can hold several lanes at once.
- Confirmed (2026-10-26): the single-character `TextLabel`s inside `PianoTiles` are the **judgement tally**, not key labels: `'0' x=640` above `PERFECT x=641`, `'0' x=803` above `COOL x=803`, `'0' x=965` above `GOOD x=965`, `'1' x=1127` above `MISS x=1127`. A key-label scan finds 5, not 4.
- Confirmed (2026-10-26): live tiles read `965,511` and `1133,31`; the four lane columns are 630 / 798 / 965 / 1133 (left to right).
- Confirmed (2026-10-26): `Client.Input` resolves a click or touch with `Board.laneAt(input.Position.X)` - X only - and maps `KeyCode -> lane` from the profile's key preset (`Shared.Config.KeyPresets`: D F J K, A S D F, 1 2 3 4, arrows).
- Confirmed (2026-10-26): `Shared.Config.DefaultKeys = {'D','F','J','K'}`; `Lanes = 4`, `ApproachTime = 1.5`, `TileLen = 0.375`, `BoardAspect = 0.62`, `LeadIn = 2.4`.
- Confirmed (2026-10-26): board is x 625..1292 (667 wide), `BoardAspect 0.62` -> height ~1076, so y = 1104 is the park spot below the board, not the judge line.
- Confirmed (2026-10-26): tiles fall DOWNWARD, Y increases toward the bottom; a live chart is a 100 px ladder (idx 3 y=991 ... idx 12 y=91).
- Confirmed (2026-10-26): **`lp.Character` / `HumanoidRootPart` are nil in this game**, so `firetouchinterest` can never work here.
- Confirmed (2026-10-26): `Drawing` Line / Circle / Text all construct and render; a pooled overlay works in this game.

## Quirks and traps
- **Do not gate a hold on tile Y.** A hold tile can stop moving at the judge position, so a `Y > FIRE_Y + n` condition never becomes true and the sustain silently never runs. Detect a hold by *lifetime instead*: still alive ~90 ms after its press.
- **Humanize in the safe direction only.** The perfect window opens around Y = 625, so jitter can be applied above that floor freely; clamp the jittered fire point to `PERFECT_MIN` so a jittered note can never fire before the window opens and turn a hit into a miss.
- **Never name a table key after a Lua reserved word.** `until` broke the whole chunk at compile time, so nothing ran - not even the version print.
- **A narrow arming window silently drops notes in dense sections.** Arm any tile first seen above `FIRE_Y`.
- **Never schedule a key release with `task.delay`.** A missed delayed release leaves the key down, and Windows sends no new keydown for a held key, so that lane is dead for the rest of the song. Release on a per-frame deadline and force a release before each press.
- **A write can report a new sha while a later read returns the previous content.** Observed twice this session. The read-back's version stamp is authoritative.
- **A read-back can carry stray markup or duplicated lines.** Always read the tail, not just the head.
- **Mouse input is dead here - do not rebuild it.** Keys only.
- **Reject impossible tile X values.** Accept only 560..1400, and fall back to 630/798/965/1133.
- **Do not ask for the press remote - it does not exist.**
- **Never map a lane with overlapping per-lane ranges.** Use nearest-column with a 120 px cap.
- **The working trigger is ONE plain threshold.** v1.6's speed prediction came from the same lagging `AbsolutePosition` readout and fired nothing. Do not re-add it.
- **`FIRE_Y` direction:** tiles fall downward, so **LOWER fires EARLIER**, HIGHER fires LATER.
- **Both halves of the input path are required:** `setrobloxinput(true)` at load *and* an `isrbxactive()` gate.
- **`isrbxactive` is nil inside the `matcha_exec` sandbox**, so probes cannot measure whether input lands.
- **This VM caps a chunk at 200 local variables.**
- `Board.AbsoluteSize` / `JudgeLine.AbsoluteSize` read `(0,0)`; `.Position` / `.Size` read nil. The line must be derived, never read.

## Hold notes (v2.11)
- Detection is **by lifetime only**: after a press, if that same tile address is still in `Board.Tiles` more than `HOLD_GRACE` (90 ms) later, it is a hold and the lane key is kept down until the tile disappears, capped at `MAX_HOLD` = 6 s. Tap tiles are gone within a frame or two and still just tap.
- No Y condition is involved - that was the v2.10 bug.
- Four independent keys can hold several lanes at once.

## Humanization (v2.11)
- Each tile gets one `math.random(-18, 18)` px jitter on its fire point, chosen when it is armed and stable for that note, clamped to `PERFECT_MIN` = 625.
- Key-down time is randomised 35-55 ms per press.
- No menu control for either; both are constants so the menu stays one toggle + one slider.

## Overlays
- Yellow `Line` at `FIRE_Y`, follows the slider live; label reads `FIRE <n>`.
- Grey `Line` at y = 1076 (board bottom) for reference.
- A `Circle` dot on every live note at `tileX + 83, tileY`: grey = fired/written off, yellow = within 60 px of the line, white = armed.

## Open questions (UNVERIFIED)
- UNVERIFIED (2026-10-26): **does the hold sustain now fire?** Test: play a song with long notes and read the HUD `holds` counter. It climbing means hold tiles do persist past their press. Stuck at 0 while long notes still drop means the tile is destroyed at the press, and holds then need another signal (the `Spine`/`Ring`/`Dot` children, or the chart clock).
- UNVERIFIED (2026-10-26): **is 18 px of jitter the right amount?** Too little still reads robotic; too much starts showing EARLY/LATE judgements. Test: watch the judgement text across a verse; widen `MICRO` only while every hit still reads PERFECT.
- UNVERIFIED (2026-10-26): **is the key order left-to-right D F J K on this account?** Assumed from `Shared.Config.DefaultKeys`, but the client builds the map from the saved profile's preset. If presses land in the wrong column, change `KEYS`.
- UNVERIFIED (2026-10-26): the exact `FIRE_Y` the user settles on - the slider exists so it can be reported.
