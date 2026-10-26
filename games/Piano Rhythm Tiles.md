# Piano Rhythm Tiles

- **Script file:** `Piano Rhythm Tiles` in `bobashid/roblox-matcha` (current version: v2.14)
- **Place IDs:** 101999883467300 (source: `matcha_status`, 2026-10-26)
- **Hub entry:** PLACES yes (v2.4), SCRIPTS keys: pianorhythmtiles, pianotiles
- **Features:** auto play - ONE toggle. Fire Y fixed at 625. Keys only. Every note held 2.0 s so hold notes (0.6-2.4 s in this game) can complete.

## Confirmed facts
- Confirmed (2026-10-26): **`Shared.Config` gives the real hold lengths: `HoldMinLen = 0.6`, `HoldMaxLen = 2.4`, `HoldTickRate = 8`, `HoldKeepFrac = 0.82`, `HoldMinGap = 0.55`, `HoldGapRatio = 1.55`, `HoldMaxRun = 2`, `HoldTailPad = 0.16`, `HoldMinLenMade = 0.45`, `SameLaneMinDt = 0.17`.** Proof: `matcha_decompile` of `ReplicatedStorage.Shared.Config`.
- Confirmed (2026-10-26): **v2.13 held every note for only 0.50 s, which is BELOW `HoldMinLen = 0.6`.** A hold could therefore never complete: the key was released before the shortest possible hold ended. That is why "still not holding" happened even though the press path was correct. v2.14 holds 2.0 s, inside the game's range. Proof: the Config values above plus the user report.
- Confirmed (2026-10-26): **a hold note is ONE line, not two tiles** (user statement). The v2.12 head+tail pair theory is WRONG; the `idx=31`/`idx=32` pair at `798,311` was two sequential notes, not a head and tail.
- Confirmed (2026-10-26): **the tile is destroyed at the press.** v2.10 gated the sustain on `Y > FIRE_Y + 20` (failed) and v2.11 on the tile still existing 90 ms later (failed); both fail only if the tile is gone at the moment it is pressed. So no tile property can measure a hold's length - the duration must be assumed from Config.
- Confirmed (2026-10-26): **every `Tile` carries `Corner`(UICorner), `Grad`(UIGradient), `Edge`(UIStroke), `Spine`(Frame), `Dot`(Frame), `Ring`(Frame), `Shine`(Frame)**, and all four Frames read `Visible = false` on live tiles. Visibility is NOT a hold signal. Proof: read-only `matcha_exec` probe dumping each tile's children with class, Visible, AbsolutePosition and AbsoluteSize.
- User-stated (2026-10-26): **the perfect window opens around Y = 625** ("625 is the beggining of where perfect starts"). Fire Y is now a constant at 625, not a slider (user request).
- User-stated (2026-10-26): **v2.10 works well** ("it works good") but holding does not work and the timing reads too robotic.
- Confirmed (2026-10-26): **v2.9 never compiled** - `Matcha:129: Expected identifier when parsing expression, got 'until'`. `until` is a Lua reserved keyword. Fixed in v2.10 by renaming the field to `due`.
- Confirmed (2026-10-26): **the script worked for the first few notes and then degraded**; the cause was a 120 px arming window a dense passage could skip between frames. v2.10 onward arms any tile first seen above `FIRE_Y`.
- Confirmed (2026-10-26): **mouse movement does not work in this game.** A HUD read `cur 1919,1079` (bottom-right of 1920x1080) and never moved; `mousemoveabs` and `mousemoverel` both return ok and change nothing. Mouse output removed from v2.7 onward. Proof: HUD screenshot.
- Confirmed (2026-10-26): **a tile can report `AbsolutePosition.X = 0`**, which polluted the learned lane ladder (lane width computed 321 instead of 167). Tile X is only accepted inside 560..1400. Proof: same HUD screenshot (`lanes 0/630/798/965`).
- Confirmed (2026-10-26): **there is NO hit/press/release remote.** Only telemetry (`play_requested`, `offer`, `frames`), `clientErrors` and `songVote` exist. Hits are scored locally in `Client.Engine`, so the remote route is closed permanently. Proof: user-pasted remote calls plus `matcha_decompile` of `PlayerScripts.PianoClient`.
- Confirmed (2026-10-26): **keypress reaches the game** - v1.7 landed hits with `keypress`, and the v2.6 HUD showed `KEYS` mode with taps climbing. Keys are the working route and the only route that can hold several lanes at once.
- Confirmed (2026-10-26): the single-character `TextLabel`s inside `PianoTiles` are the **judgement tally**, not key labels: `'0' x=640` above `PERFECT x=641`, `'0' x=803` above `COOL x=803`, `'0' x=965` above `GOOD x=965`, `'1' x=1127` above `MISS x=1127`. A key-label scan finds 5, not 4.
- Confirmed (2026-10-26): live tiles read `965,511` and `1133,31`; the four lane columns are 630 / 798 / 965 / 1133 (left to right).
- Confirmed (2026-10-26): `Client.Input` resolves a click or touch with `Board.laneAt(input.Position.X)` - X only - and maps `KeyCode -> lane` from a table built from the profile's key preset (`Shared.Config.KeyPresets`: D F J K, A S D F, 1 2 3 4, arrows).
- Confirmed (2026-10-26): `Shared.Config.DefaultKeys = {'D','F','J','K'}`; `Lanes = 4`, `ApproachTime = 1.5`, `TileLen = 0.375`, `BoardAspect = 0.62`, `LeadIn = 2.4`.
- Confirmed (2026-10-26): board is x 625..1292 (667 wide), `BoardAspect 0.62` -> height ~1076, so y = 1104 is the park spot below the board, not the judge line.
- Confirmed (2026-10-26): tiles fall DOWNWARD, Y increases toward the bottom; a live chart is a 100 px ladder (idx 3 y=991 ... idx 12 y=91).
- Confirmed (2026-10-26): **`lp.Character` / `HumanoidRootPart` are nil in this game**, so `firetouchinterest` can never work here.
- Confirmed (2026-10-26): `Drawing` Line / Circle / Text all construct and render; a pooled overlay works in this game.

## Quirks and traps
- **Read the game's Config before choosing a hold duration.** `HoldMinLen = 0.6` is the shortest hold the game can produce, so any assumed hold shorter than that guarantees failure. This cost a whole version (v2.13 held 0.50 s).
- **Do not try to detect a hold from the tile.** `Visible` on Spine/Dot/Ring/Shine is always false, a hold tile can stop moving at the judge line, and the tile is destroyed at the press. Detect nothing - hold every note, and make the duration long enough for the game's own `HoldMaxLen`.
- **Holding every note is safe because the press path force-releases the lane first.** `SameLaneMinDt = 0.17` s, and a tap is judged on key-down, so a long hold does not lose the tap or swallow the next note in that lane.
- **Fire Y is fixed at 625, not a slider.** 625 is the earliest Y that still scores PERFECT, so firing there cannot be late. Jitter is positive only (0..18 px) so a note can never fire before the window opens.
- **Never name a table key after a Lua reserved word.** `until` broke the whole chunk at compile time, so nothing ran - not even the version print.
- **A narrow arming window silently drops notes in dense sections.** Arm any tile first seen above `FIRE_Y`.
- **Never schedule a key release with `task.delay`.** A missed delayed release leaves the key down, and Windows sends no new keydown for a held key, so that lane is dead for the rest of the song.
- **A write can report a new sha while a later read returns the previous content.** Observed twice this session. The read-back's version stamp is authoritative.
- **A read-back can carry stray markup or duplicated lines.** Always read the tail.
- **Mouse input is dead here - do not rebuild it.** Keys only.
- **Reject impossible tile X values.** Accept only 560..1400, fall back to 630/798/965/1133.
- **Do not ask for the press remote - it does not exist.**
- **Never map a lane with overlapping per-lane ranges.** Use nearest-column with a 120 px cap.
- **The working trigger is ONE plain threshold.** v1.6's speed prediction came from the same lagging `AbsolutePosition` readout and fired nothing. Do not re-add it.
- **Both halves of the input path are required:** `setrobloxinput(true)` at load *and* an `isrbxactive()` gate.
- **`isrbxactive` is nil inside the `matcha_exec` sandbox**, so probes cannot measure whether input lands.
- **This VM caps a chunk at 200 local variables.**
- `Board.AbsoluteSize` / `JudgeLine.AbsoluteSize` read `(0,0)`; `.Position` / `.Size` read nil. The line must be derived, never read.

## Hold notes (v2.14)
- Every note is held for `HOLD_ALL` = **2.0 s** plus 0-80 ms jitter, which covers the game's 0.6-2.4 s hold range for almost its whole body. `MAX_HOLD` = 3.0 s is the absolute cap.
- No detection is attempted: the tile is destroyed at the press, so there is nothing to observe. The duration comes from `Shared.Config`.
- `HOLD_ALL` is the one number to change: raise toward 2.4 if holds still break, lower toward 1.0 if taps mis-score or notes get skipped.
- Four independent keys can hold several lanes at once - the reason holds are keys-only.

## Humanization
- Each tile gets one `math.random(0, 18)` px jitter on its fire point (positive only, so it can never fire before 625).
- No menu control; it is a constant so the menu stays a single toggle.

## Overlays
- Yellow `Line` at `FIRE_Y` (625) with a `FIRE 625` label.
- Grey `Line` at y = 1076 (board bottom) for reference.
- A `Circle` dot on every live note at `tileX + 83, tileY`: grey = fired/written off, yellow = within 60 px of the line, white = armed.

## Open questions (UNVERIFIED)
- UNVERIFIED (2026-10-26): **does holding 2.0 s let holds complete without costing taps?** This is the first version whose hold duration is inside the game's own range, so it needs a real test. Play a song with long notes: holds scoring means the Config-derived duration was the missing piece; taps mis-scoring or notes skipped means 2.0 s is too long for the chart density and `HOLD_ALL` should come down toward 1.0.
- UNVERIFIED (2026-10-26): **is the key order left-to-right D F J K on this account?** Assumed from `Shared.Config.DefaultKeys`, but the client builds the map from the saved profile's preset. If presses land in the wrong column, change `KEYS`.
