# Piano Rhythm Tiles

- **Script file:** `Piano Rhythm Tiles` in `bobashid/roblox-matcha` (current version: v2.12)
- **Place IDs:** 101999883467300 (source: `matcha_status`, 2026-10-26)
- **Hub entry:** PLACES yes (v2.4), SCRIPTS keys: pianorhythmtiles, pianotiles
- **Features:** auto play - ONE toggle only. Fire Y fixed at 625. Keys only. Holds detected by a head+tail tile pair. Micro jitter, humanized taps.

## Confirmed facts
- Confirmed (2026-10-26): **every `Tile` carries the children `Corner`(UICorner), `Grad`(UIGradient), `Edge`(UIStroke), `Spine`(Frame), `Dot`(Frame), `Ring`(Frame), `Shine`(Frame)** - and all four Frames read `Visible = false` on live tap tiles. Visibility is therefore NOT a hold signal. Proof: read-only `matcha_exec` probe dumping each tile's children with class, Visible, AbsolutePosition and AbsoluteSize.
- Confirmed (2026-10-26): **two distinct tiles can occupy the identical position.** The same probe read `idx=31 x=798 y=311` and `idx=32 x=798 y=311` - two different `TileIndex` values, same X and same Y. Sequential taps cannot do that (notes are separated in time, so they sit at different Y), which makes a shared position a head+tail pair, i.e. a hold. Proof: that probe output.
- User-stated (2026-10-26): **the perfect window opens around Y = 625** ("625 is the beggining of where perfect starts"). Firing from there downward scores, so a small positive jitter cannot cause a miss. Fire Y is now a **constant at 625**, not a slider (user request).
- User-stated (2026-10-26): **v2.10 works well** ("it works good") but holding does not work and the timing reads too robotic.
- Confirmed (2026-10-26): **holding failed twice for two different reasons.** v2.10 gated the sustain on `Y > FIRE_Y + 20`, which never becomes true if a hold tile stops moving at the judge position. v2.11 switched to tile lifetime, which also fails because the tile disappears at the press. v2.12 detects a hold structurally, by a partner tile sharing the lane and position. Proof: user reports "holding still doesnt work" after both versions.
- Confirmed (2026-10-26): **v2.9 never compiled** - `Matcha:129: Expected identifier when parsing expression, got 'until'`. `until` is a Lua reserved keyword and cannot be a bare table key. Fixed in v2.10 by renaming the field to `due`. Proof: user-pasted console error, then a clean read-back.
- Confirmed (2026-10-26): **the script worked for the first few notes and then degraded**; the cause was a 120 px arming window that a dense passage could skip between frames. v2.10 onward arms any tile first seen above `FIRE_Y`. Proof: user statement, then working v2.10.
- Confirmed (2026-10-26): **mouse movement does not work in this game.** A HUD read `cur 1919,1079` - the bottom-right corner of a 1920x1080 screen - and the cursor never moved. `mousemoveabs` and `mousemoverel` both return ok and change nothing. Mouse output removed from v2.7 onward. Proof: HUD screenshot (`v2.6 | KEYS | focus true | taps 2 | fireY 772 | off +0 | lanes 0/630/798/965 | tile x798 => click x958 | cur 1919,1079 | chart 44`).
- Confirmed (2026-10-26): **a tile can report `AbsolutePosition.X = 0`**, which polluted the learned lane ladder (lane width computed as 321 instead of 167). Tile X is only accepted inside 560..1400. Proof: same HUD screenshot.
- Confirmed (2026-10-26): **there is NO hit/press/release remote.** Only telemetry (`play_requested`, `offer`, `frames`), `clientErrors` and `songVote` exist. **Hits are scored locally in `Client.Engine`**, so the remote route is closed permanently. Proof: user-pasted remote calls plus `matcha_decompile` of `PlayerScripts.PianoClient`.
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
- **Do not use tile `Visible`, tile Y movement, or tile lifetime to detect a hold.** All three failed: the Spine/Dot/Ring/Shine Frames read `Visible=false`, a hold tile can stop moving at the judge position, and the tile disappears at the press. The working signal is structural: **another live tile in the same lane within ~26 px of the same Y** = head+tail = hold.
- **Fire Y is fixed at 625, not a slider.** 625 is the earliest Y that still scores PERFECT, so firing there cannot be late. Jitter is **positive only** (0..18 px) so a jittered note can never fire before the window opens.
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
- **Both halves of the input path are required:** `setrobloxinput(true)` at load *and* an `isrbxactive()` gate.
- **`isrbxactive` is nil inside the `matcha_exec` sandbox**, so probes cannot measure whether input lands.
- **This VM caps a chunk at 200 local variables.**
- `Board.AbsoluteSize` / `JudgeLine.AbsoluteSize` read `(0,0)`; `.Position` / `.Size` read nil. The line must be derived, never read.

## Hold notes (v2.12)
- On press of tile A in lane L, the script scans the same frame's tile list for another live tile in lane L within `HOLD_TOL` = 26 px of A's Y. If one exists, the note is a hold: `HOLD_BASE` 0.35 s is held, and each frame the partner is still alive pushes `due` forward by `HOLD_STEP` 0.05 s, capped at `MAX_HOLD` 6 s.
- A tap has no partner, so it gets a plain 35-55 ms key press.
- Four independent keys can hold several lanes at once - the reason holds are keys-only.

## Humanization (v2.12)
- Each tile gets one `math.random(0, 18)` px jitter on its fire point (positive only, so it can never fire before 625).
- Key-down time for taps is randomised 35-55 ms.
- No menu control for either; both are constants so the menu stays a single toggle.

## Overlays
- Yellow `Line` at `FIRE_Y` (625) with a `FIRE 625` label.
- Grey `Line` at y = 1076 (board bottom) for reference.
- A `Circle` dot on every live note at `tileX + 83, tileY`: grey = fired/written off, yellow = within 60 px of the line, white = armed.

## Open questions (UNVERIFIED)
- UNVERIFIED (2026-10-26): **does the head+tail pair really mark a hold?** This is the third hold attempt and the first with a structural signal, so it needs a real test. Play a song with long notes and read the HUD `holds` counter: climbing means the pair detection works and the sustain is running; stuck at 0 means no two tiles ever share a position and holds need another route.
- UNVERIFIED (2026-10-26): **does the held key survive for the whole note?** Even if detection is right, the extension only pushes `due` forward while the partner is alive. If long notes still break with `holds` climbing, the partner vanishes early and the sustain needs a fixed duration instead.
- UNVERIFIED (2026-10-26): **is 18 px of positive jitter enough to stop it reading robotic** while every hit still scores? Watch the judgement text; widen `MICRO` only while hits stay clean.
- UNVERIFIED (2026-10-26): **is the key order left-to-right D F J K on this account?** Assumed from `Shared.Config.DefaultKeys`, but the client builds the map from the saved profile's preset. If presses land in the wrong column, change `KEYS`.
