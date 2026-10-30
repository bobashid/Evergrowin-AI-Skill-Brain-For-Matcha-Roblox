# The Strongest Battlegrounds

- Script file: none in `bobashid/roblox-matcha`
- Place IDs: NOT KNOWN — the Discord post gives no URL and no id. Do not invent one.
- Source: Matcha Discord, public player-made-scripts channel, post by `Ban [឵឵]`, 2026-06-02 to 2026-06-11, titled "The Strongest Battlegrounds | AutoBlock". Script at `https://raw.githubusercontent.com/Qxlk/Tsb/refs/heads/main/AutoBlock`.
- Status: **third-party, never run or probed by us.** Everything below is what the script asserts about the game, plus what it demonstrates about the Matcha API. It is a lead, not a fact, until a probe confirms it.

## What the script asserts about the game
- Players/opponents are children of `workspace.Live` (a Folder). The script lists that folder and caches the child list for 0.5 s rather than per frame.
- An opponent's attack state is read from **child markers on the player model**: `M1ing` and `NoBlock`. If either exists, they are attacking or unable to block.
  - This is the same family as "read state from the model, not from a remote": no remote is fired, nothing is written. It also means the block must be raised while the marker exists and dropped when it goes away.
- Block key is `F` (0x46). Target toggle key is `C` (0x43). Range is 12 studs (`range = 12`, compared squared).
- Target selection: closest to the mouse cursor **in screen space**, using squared pixel distance with an early cutoff of 40000 (≈200 px).

## Matcha API observed in the dump
- **`WorldToScreen(worldPos)` returns two values: `(Vector2, boolean)`.** The script reads both: `local s_pos, on_screen = WorldToScreen(hrp.Position)` and `local sp, v = WorldToScreen(wpos)`.
  - It applies **no sign flip and no viewport offset** — the returned Vector2 is used directly as a Drawing `Position`/`From`/`To`. That is the "use it directly, get the signs right" case.
  - Status: UNVERIFIED (2026-10-30). Proof is a third-party script that its author reports as working, not our own probe. Test that would confirm it: one `matcha_exec` print of `WorldToScreen(hrp.Position)` next to the same part's known screen position.
- **`iskeypressed(vk)` polled at `task.wait(0.01)`** for a key toggle, with a `last_c` edge flag — a key press is detected without `InputBegan`, which matters because this VM drops many key events.
- **`keypress(vk)` / `keyrelease(vk)` used as a sustained state**: `setblock(true)` presses F and leaves it down while the condition holds; `setblock(false)` releases. Guarded by `if state == blocking then return end` so the same state is never re-sent.
  - This is the pattern Piano Rhythm Tiles needed for holds: hold the key while the condition is true, release when it goes false, and never repeat a state you are already in.

## Reusable patterns (general, not game-specific)
1. **World-space ring as screen-space line segments.** Sample N points on a circle around a world position, run each through `WorldToScreen`, and draw the chords as `Drawing.Line`s. Keep a per-point visibility flag and hide any segment whose endpoint is off-screen, so the part of the ring behind the camera is hidden instead of smearing across the screen. 16 segments was enough.
2. **Closest-to-cursor targeting.** Convert each candidate to screen space, take squared distance to `mouse.X/Y`, keep the best, with an early cutoff so far-away candidates cost nothing.
3. **Cache `GetChildren()` on a timer** (0.5 s here) instead of calling it every frame when the list is only used for membership tests.
4. **Idempotent state setter.** `setblock(state)` early-returns when the state already matches. Any loop that re-applies a state every frame must be built this way or it spams `keypress`.

## Do not copy this structure
- **No `stop()`, no `_G` registration, no `running` flag.** The script is two unbounded `while true do ... end` loops. It cannot be unloaded, and re-running it stacks a second copy on top of the first. That violates the script contract in `script-standard.md` — every pushed script registers `_G.<Name>` with a synchronous `stop()` that sets `running = false`, disconnects connections and removes Drawings.
- The outer loop polls at `task.wait(0.01)` (100 Hz). Fine for a key edge check; wasteful as a default. The brain's own note is that menus poll at 0.15 s and action loops at 0.25 s.
- `continue` is used freely. It is a Luau keyword and clearly works in the author's build; not separately confirmed in this VM, but low risk.

## Open questions (UNVERIFIED)
- UNVERIFIED (2026-10-30): the place id for this game — not given in the post.
- UNVERIFIED (2026-10-30): `WorldToScreen`'s exact return order and whether the Y axis needs flipping in this build. The dump says no flip; that is one third-party sample.
- UNVERIFIED (2026-10-30): whether `workspace.Live` and the `M1ing` / `NoBlock` child markers still exist — the post is from June 2026 and the game has almost certainly changed since.
