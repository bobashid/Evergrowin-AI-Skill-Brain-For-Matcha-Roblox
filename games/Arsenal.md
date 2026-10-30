# Arsenal

- Script file: none in `bobashid/roblox-matcha`
- Place IDs: NOT KNOWN — the Discord post gives no URL and no id. Do not invent one.
- Source: Matcha Discord, public player-made-scripts channel, post by `OpenSky [MEOW]`, 2025-11-19, titled "Arsenal Melee Changer". Toggle menu with F1. Author notes "Some may not equip".
- Status: **third-party, never run or probed by us**, and the post is from November 2025 — the game has almost certainly changed since. Everything below is what the script asserts, plus what it demonstrates about the Matcha API. A lead, not a fact.

## What the script asserts about the game
- **`player.Data.Melee` is a StringValue holding the equipped melee name**, and writing to it changes the equipped melee. No remote is fired. That means the value is client-authoritative in this game — the same family as any "server trusts the client" fault, but here the trust is in a replicated Value rather than a remote argument.
- **`ReplicatedStorage.Melees` is a Folder whose children are the melee catalogue** — `GetChildren()` names, sorted, is the whole list. That is the clean way to enumerate: the game's own data, not a hardcoded table.
- **The game clears or poisons the value when you write a melee it rejects.** The script's entire "crash protection" block exists because `meleeValue.Value` can come back as a non-string or an empty string. Validation used: `type(v) == "string" and string.find(v, "%a")` — "a string containing at least one letter".
- **The HUD names the held weapon at `player.PlayerGui.GUI_Interface.Vitals.Ammo.Gun.Text`.** For the standard knife it reads `"Knife"` even when a skin is equipped; for the golden knife it reads `"Golden Knife"`.
- **A custom melee skin breaks hit registration.** The script's "SwingFix" forces the default dagger for ~0.42 s so the swing lands, then restores the chosen skin. This is the single most interesting claim in the dump: the game keys swing logic off the melee name, so cosmetic and functional are the same value.

## Matcha API observed in the dump
- `Drawing.new` for the whole menu — Square, Text, Line — pooled into one `UIObjects` table so `setMenuVisible` can walk them all. No `UI.AddTab` anywhere.
- `iskeypressed(vk)` polled per frame for the F1 toggle, with a `LastF1` edge flag; same shape for the whole typed-search keyboard (`0x41`-`0x5A`, `0x30`-`0x39`, `0x08` backspace, `0x20` space, `0x10` shift).
- `ismouse1pressed()` with a `LastMouseDown` edge flag for click-vs-hold, which is what makes dragging, scrollbar scrubbing and row selection coexist.
- `setrobloxinput(not State.MenuOpen)` — see the hazard below.

## Hazards — do not copy these
- **`setrobloxinput()` is called every iteration of a `while true do ... wait(0.001) end` loop.** That is up to 1000 calls a second. `failure-protocols.md` F9 says `setrobloxinput` takes ONE boolean and must never be spammed; this script does exactly the thing F9 forbids. Call it only when the value *changes*.
- **`wait(0.001)` at the end of the main loop** — 1000 Hz. The brain's own guidance is 0.15 s for menu polling and 0.25 s for action loops. This will burn CPU for no benefit.
- **`spawn()` and `wait()`** — the legacy globals. `task.spawn` / `task.delay` are the forms used everywhere else in the brain.
- **No `stop()`, no `_G` registration, no `running` flag.** The script is one unbounded loop and cannot be unloaded; running it twice stacks a second copy that fights the first over the same Value.
- **A hand-rolled Drawing menu instead of `UI.AddTab`.** Not forbidden, but it is ~400 lines of layout that Matcha's own menu does for free, and it is why the script is a monolith.

## What is genuinely worth taking
1. **Enumerate the game's own catalogue** rather than hardcoding melee names — `ReplicatedStorage.Melees:GetChildren()`.
2. **Restore-last-known-good** on a Value the game validates (see `learned/patterns.md`).
3. **Virtual scrolling list with a fixed Drawing pool** (see `learned/patterns.md`).
4. **Separate cosmetic from functional** when the game conflates them (see `learned/patterns.md`).
5. **Edge-flag every polled input** — `iskeypressed` returns a level, not an event, so a toggle needs `if now and not last then`.

## Open questions (UNVERIFIED)
- UNVERIFIED (2026-10-30): the place id — not given in the post.
- UNVERIFIED (2026-10-30): whether `player.Data.Melee` still exists and is still writable. If the game has since moved the melee onto the server, every version of this script becomes a no-op with no error.
- UNVERIFIED (2026-10-30): whether a custom skin still breaks hit registration, and whether 0.42 s is the real swing window. Both are the author's claims; the constant especially must be re-measured, not copied.
- UNVERIFIED (2026-10-30): whether `GUI_Interface.Vitals.Ammo.Gun.Text` is still the HUD path, and whether `"Knife"` / `"Golden Knife"` are still the exact strings.
