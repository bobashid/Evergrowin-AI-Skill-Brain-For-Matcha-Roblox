# Learned: patterns that worked

Short, reusable, proven in a real game. Format: name, when to use, 3 to 10 lines of code or steps, proof and date. A pattern used everywhere moves into the matching reference file.

## Fire the game's own interaction instead of writing properties
Use when a game grants something through a ProximityPrompt, ClickDetector or touch. The game's own handler does the work, so it survives updates and does not desync state the way attribute writes do.
1. Find the prompt: scan `GetDescendants()` for `ClassName == 'ProximityPrompt'` plus the expected name.
2. `pcall(fireproximityprompt, prompt)`.
3. Read state back to confirm (a label changes, an item disappears, a bar starts draining).

User-stated (2026-10-26): Feed A Mosquito and Ride A Dino both work this way. Firing `MountDinosaur` starts the stamina tick with no range check. Proof: the user's Ride A Dino dump, header line "Checked this session (place 93523763376122)".

## Guard before re-applying
Every auto-loop needs a state check before the action, or the re-apply becomes the bug: it dismounts you, double-buys, or cancels what you just did.
- Ride A Dino: never fire `MountDinosaur` while already mounted, or it dismounts.

User-stated (2026-10-26), Ride A Dino dump.

## Find "yours" by attribute, not by name
`plot:GetAttribute('Owner') == LocalPlayer.UserId` beats assuming `Plot1`, `House1`, `Slot1`. Names change between servers and players; the owner attribute does not.

User-stated (2026-10-26): confirmed in Ride A Dino as `Workspace.Pens.<Plot>` with `Owner` equal to UserId.

## Multi-signal state detection when properties read nil
Matcha properties often read nil, so a single check silently reports false forever. OR several signals together.
- Mounted in Ride A Dino: a `CarriedDino` child under the character, a seat/weld named seat/mount/saddle, or `Humanoid.Sit == true`. `Humanoid.Sit` reads nil there, so the other two carry it.

User-stated (2026-10-26), Ride A Dino dump.

## Toggle state: poll the getter, do not trust the callback
Register every boolean widget in a list, poll `UI.GetValue(id)` then `UI.GetValue(name)`, and treat the getter as authoritative for a key only after it has returned a boolean (`getterWorks[key] = true`). Until then the callback is the only writer; after that the callback early-returns so the two cannot fight.

User-stated (2026-10-26): the working pattern in the user's Ride A Dino script.

## pcall every property read that can be nil
Nil-returning properties are normal in this VM, not an error case. Wrap each read in `pcall` and treat nil as "unknown", not as false.

User-stated (2026-10-26), Ride A Dino dump.

## stop() cleans up and changes nothing else
Disconnect every connection, remove the tab, clear the global. No dismount, no teleport, no restore writes unless the user explicitly asked for them.

User-stated (2026-10-26), Ride A Dino dump.

## Read the game's own Config before choosing a duration or threshold
Before picking any timing constant (hold length, cooldown, approach time), decompile the game's Config/Shared module and read its limits. A constant outside the game's own range cannot work no matter how correct the surrounding logic is.
- Piano Rhythm Tiles: `Shared.Config` had `HoldMinLen = 0.6`, `HoldMaxLen = 2.4`, `HoldTickRate = 8`, `HoldKeepFrac = 0.82`, `SameLaneMinDt = 0.17`. A build holding every note for 0.50 s was below the 0.6 s minimum, so every hold was guaranteed to fail while taps looked fine.

Confirmed (2026-10-26): three different hold-detection schemes failed before anyone checked the Config; the constant was the bug, not the detection. Proof: user reports plus a decompile of `Shared.Config`.

## Force-release a key immediately before pressing it
Windows delivers no new keydown for a key that is already held, so a key left down swallows every later press in that lane with no error anywhere. Call `keyrelease(vk)` (pcall'd) at the top of the press function, then `keypress(vk)`.
- This is what makes an over-long hold safe: even if the previous hold has not expired, the next note in the same lane still registers.

Confirmed (2026-10-26): reasoned from a HUD that showed keys being sent with no hits landing. Proof: Piano Rhythm Tiles v2.8 onward.

## Arm on first sight anywhere past the threshold, not inside a narrow window
A trigger that requires an object to be seen inside a band (e.g. `Y < fireY - 120`) silently drops objects when a frame is missed in a dense passage: the object jumps over the band and is never armed. Arm on first sight anywhere on the approach side of the threshold.
- Symptom to recognise: "works for the first few, then stops" - sparse openings always satisfy a narrow window, dense sections do not.

Confirmed (2026-10-26): user report "working fine for the first few notes after that it didn't work well anymore" on Piano Rhythm Tiles. Proof: user statement, fixed by widening the arming test.

## After two failed attempts at one feature, suspect the constant before the detection
Three separate detection schemes were written for one symptom (hold notes not registering). The actual fault was a single numeric constant outside the game's documented range. Before writing a fourth detection mechanism, re-read the constants involved and check them against the game's own data.

Confirmed (2026-10-26): v2.10 (Y-gated), v2.11 (tile lifetime) and v2.12 (head+tail pair) all failed; v2.14 fixed it by changing one duration constant. Proof: version history plus user reports.

## Reflection is not learning unless it is written down
A session can correct itself many times in one conversation and still leave the brain unchanged, so the next session repeats the same mistake with better commentary. Every self-correction - "that was my bug", "the real cause was X", "the constant was outside the range" - has to land in a brain file in the same turn, or it is lost when the window closes.
- Symptom: a long session full of admissions of error, but every entry in `learned/` predates it.
- Test at the end of any turn that contained a correction: the read-back should show a new or edited entry. If it does not, the correction was narration, not learning.
- Corollary: reading the brain and then improvising is the same failure with a better excuse. A dated brain stamp in the reply is the only cheap proof that the read happened.

Confirmed (2026-10-26): the user's own diagnosis of a parallel Piano Tiles chat - it found real bugs (stale read-backs, a cursor pinned at 1919,1079, mouse input dead, an out-of-range hold duration) and wrote none of them down, so a later session had to rediscover every one. Proof: user observation plus the transcript.

## Idempotent key-state setter: hold a key while a condition is true
When a game wants a key *held* rather than tapped (block, sprint, crouch, a hold note), drive it from a single setter that early-returns when the state already matches:
```lua
local state = false
local function set(vk, want)
    if want == state then return end
    state = want
    if want then keypress(vk) else keyrelease(vk) end
end
```
- The guard is the whole point. A loop that re-applies every frame without it spams `keypress`/`keyrelease` dozens of times a second and the game sees a flicker instead of a hold.
- Call the setter from one loop with the condition; do not press in one place and release in another.
- Clear it on every early-exit path (no character, target gone, toggle off) or the key stays down.

Third-party sample (2026-10-30): The Strongest Battlegrounds AutoBlock holds `F` while the opponent has an `M1ing`/`NoBlock` marker and releases when it goes, with exactly this guard (`if state == blocking then return end`). The same shape is what a hold note needs: key down while the note body is alive, release when it dies. UNVERIFIED as a Matcha-tested result - the sample is someone else's script, and the Piano Tiles hold was fixed by a duration constant, not by this. The reasoning is sound and cheap to apply; it is not yet a measured win in our own scripts.

## Screen-space ring from world-space points
To draw a range circle or any world-space outline with `Drawing.Line`:
1. Sample N points around the world position (16 was enough), `p_i = center + (cos a, 0, sin a) * radius`.
2. Run each through `WorldToScreen`, keeping both returns: position and on-screen flag.
3. Draw chord `i-1 -> i` only when *both* endpoints are flagged visible; otherwise hide that line.

- The per-segment visibility test is what stops the half of the ring behind the camera from smearing across the screen as one long line.
- Pool the Line objects once at startup and only mutate `From`/`To`/`Color`/`Visible` per frame; creating Drawings in a loop is the freeze.

Third-party sample (2026-10-30): The Strongest Battlegrounds AutoBlock, 16 segments, colour switched by whether a target is locked. Same UNVERIFIED caveat as above - seen in someone else's working script, not yet run in ours.
