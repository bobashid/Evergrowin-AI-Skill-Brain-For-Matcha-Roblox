# Failure protocols (brain part, ALWAYS read)

(lessons from a real 19-version session; these override your instincts)

A real session shipped about 19 versions of one script, froze the user's client so badly that Process Explorer was needed, reported pushes that never landed, and shipped repeated syntax errors. Follow these rules to avoid repeating that.

## F1. A tool result you did not see is a failure
- If a push, read-back, probe or any tool returns "unknown", empty or an error, then it did NOT succeed. Say "not confirmed", never describe the file as pushed, and never write a "Verified:" line for it.
- Never invent or reuse a SHA, byte size, commit id or "read-back confirmed" claim. Quote only values that appeared in a tool result this turn.
- If a push fails with a SHA mismatch, re-read the SHA and push again before replying. Do not reply as if it landed.
- Before you say a version is live, the read-back must show the new version stamp. If it shows the old version, you have not delivered.

## F2. Syntax errors are your most common bug: lint before every push
The session shipped missing closing parentheses (a `print(...)` with no `)`) and mismatched `end`s several times, plus garbled identifiers such as `for , rar in ipairs(...)` and `G[NAME]` where `_G[NAME]` was meant.
- Never use a bare `_` as a loop variable; use `i`, `k`, `idx`. Never leave a global written as `G[...]`; always write `_G[...]`.
- If bash and a Lua checker (`luac -p`, `luau`, `lua5.1`) are available, run it on the file before pushing. If not, do a manual pass: count `(`/`)`, count blocks vs `end`, confirm every `function`, `if`, `for`, `while` closes, and confirm the last statement of the file is complete.
- After the read-back, search the returned text for `for ,`, ` G[`, and an unterminated final line. Fix and re-push if any exist.
- Do NOT claim "compiles" from `loadstring` inside Matcha: `loadstring` returns a function even for broken source and `pcall` cannot catch the error. The only evidence is the Matcha console printing a compile error when the chunk is run.

## F3. If the user reports a freeze, crash, hang or "all my inputs stopped": STOP experimenting
- A freeze that needs Process Explorer or killing the process is a severity-1 problem. Do not push another feature version.
- Immediately push (or tell the user to load) the last known-good version, or a minimal safe stub with no input, no teleport and no heavy UI. Say plainly that nothing risky will run.
- Do not run input, teleport or CFrame probes in the user's live game after a freeze report unless they explicitly agree. Probing is what froze this user's game repeatedly.
- Find the cause by bisection, one change at a time, not by guessing: load with only the loader and a print, then add the menu, then add one input call, then the loop. Ask the user which step first freezes and for the last console line before the hang.
- Do not state a root cause as fact without evidence. Say "most likely" and what test would confirm it. Declaring a new "found it" cause in every message while the freeze continues destroys trust.
- Known freeze candidates to isolate (unconfirmed, from that session): repeated `setrobloxinput(false, ...)` calls (it is a one-argument on/off switch for injected input, see F9), a thread released only by `task.cancel`, hundreds of menu widgets built at load, repeated CFrame writes on a simulating character. Test them separately.

## F4. Cap the version churn
- One hypothesis per push, with one test the user can run. If two consecutive pushes fail to fix the same symptom, stop pushing and report what is known and unknown, and ask for the specific log line you need.
- Do not push multiple versions in one reply. One reply, one loader, one stop instruction.
- Do not remove, rename or rewrite working parts (menu, buy list, helper functions) while chasing an unrelated bug.

## F5. Deliver what was asked; do not silently downgrade it
- The user asked for auto roll plus auto buy, with toggles for every possible outcome, two sections (one per roller), and later a single toggle that rolls both rollers, teleports between them and waits out cooldowns. The session repeatedly shrank this to a read-only watcher, a stub or "not achievable". Do not downgrade the deliverable without the user's explicit agreement; if something truly cannot work, say what you tried, what the evidence is, and offer options, then wait.
- Keep the requested structure (toggles, sections, per-outcome buy list) even while debugging the action layer.
- Do not add guards that stop an action based on an assumption you have not verified (for example "skip if an item is pending"). Log the condition and proceed; guards that never fire the action produce scripts that silently do nothing.

## F6. Matcha is controlled from the menu, not from typing
The user cannot type commands in Matcha. Never rely on `_G.X.test()`, `_G.X.stop()` or other console commands for the user's workflow. Every control (enable, stop, test, buy list, release keys) must be a menu toggle or button. Console commands may exist as extras but are never the instructions you give.

## F7. Menus: build from the game's real data, and keep them small
- Per-outcome toggles must come from the game's actual outcomes for that roller (read the roller's possible results or item list from the game), not a guessed list. Put each roller in its own section.
- Verify the menu actually renders items. A section built from an empty saved list shows a blank menu.
- Many widgets built synchronously at load can hang the client. Cap widgets per section, build them inside the tab callback once, and test with a handful before scaling up. If a large list is required, build it in chunks with a yield between chunks.

## F8. Probing rules for matcha_exec
- Output capture closes about 350 ms after the last print, and anything printed from a `task.spawn` that outlives that is lost. Keep probes under about 1.5 s, print each result as soon as you have it, and read `matcha_logs` for late output. Do not re-run the same truncated probe repeatedly; narrow it.
- Probes that move the character, press keys or click the mouse affect the user's live game. Ask first if the user has reported instability.
- Exception to the short-probe rule: `getgc`/`applygc` take 10-20 s on the user's build. Run them once in a background `task.spawn`, never in a loop, and read the result from `matcha_logs` (see the Pattern Library).

## F9. Interaction layer (ProximityPrompts, click detectors, touch, keys): the current truth
Checked against the official docs on Oct 21 2026. An earlier session wasted ~19 versions because it used the wrong tools; do not repeat that.

**Use the dedicated interaction functions (added Oct 21 2026; the user has Pro). They replace every key/mouse/teleport hack:**
- `fireproximityprompt(prompt)` instantly triggers and completes a ProximityPrompt. No hold time and no distance requirement, so no teleporting and no key simulation. Find the prompt with `roller.Roll:FindFirstChildWhichIsA("ProximityPrompt")` or by scanning `GetDescendants()` for `ClassName == "ProximityPrompt"`. This is the right way to press the prompts on a roller, a buy slot, a door, a lever.
- `fireclickdetector(clickDetector, distance?, event?)` events: `"MouseClick"` (default), `"RightMouseClick"`, `"MouseHoverEnter"`, `"MouseHoverLeave"`; `distance` defaults to `0.0`, which bypasses `MaxActivationDistance`.
- `firetouchinterest(origin_part, target_part, state)` where `0` = Touch and `1` = Untouch (for example `firetouchinterest(hrp, coin, 0)` then `1`).
- If `type(fireproximityprompt) ~= "function"`, the user's build is out of date: tell them to re-run the Matcha loader (the update is delivered by reopening it) in one sentence rather than falling back to key presses or teleports. They already have Pro; do not mention it.
- Verify each call by reading the result (a label changing, an item disappearing), and never loop faster than the game's own cooldown.
- Check existence inside the firing function with `type(fireproximityprompt) == "function"`; if false, warn ONCE (flag) that the user must re-run the Matcha loader and fall back to a remote route. `attempt to call a nil value` on one of them means the same thing: do not rewrite the script. On first use in a game, probe one prompt, print the result, and read `matcha_logs` before building a loop around it. Pass the real ProximityPrompt instance (find it by `ClassName == "ProximityPrompt"`; `IsA` always returns true). Some games still check range on the server, so if a `fire*` call changes nothing, read state back and only then consider moving the character near the target.

**Raw key and mouse input is the last resort, and its signatures matter:**
- `keypress(keycode)` / `keyrelease(keycode)` take a **Windows virtual-key code number** (`0x45` = E, `0x57` = W), NOT `Enum.KeyCode.E`. Passing an Enum value is wrong and likely explains "keypress did nothing" in the earlier session.
- `setrobloxinput(state: boolean)` takes ONE boolean: it toggles whether Matcha's injected input is sent to the game at all. It is not a key press function. Calling `setrobloxinput(false, 69)` repeatedly (as the earlier session did, including from a watchdog thread several times per second) turns injected input off, which is the most likely cause of the "all my inputs froze" reports. Never call it in a loop; call `setrobloxinput(true)` once if you must enable injected input, and prefer not touching it.
- `isrbxactive()` is true only while the Roblox window is the foreground window; injected input is only meaningful then. `iskeypressed(vk)` reads live keyboard state.
- `mousemoveabs(x, y)` uses the same pixel space as `WorldToScreen`/`Camera.ViewportSize`; `mouse1click()` is press + release.
- ProximityPrompt properties (`Enabled`, `HoldDuration`, `MaxActivationDistance`) read nil in Matcha, so do not branch on them; `fireproximityprompt` does not need them.

**Before writing any interaction code:** read the user's existing repo scripts for how they already trigger prompts/remotes and copy a pattern that works; prefer a direct `FireServer` (decompile the firing client script for the exact arguments) or a `fire*` function over input simulation; never teleport the character just to be "in range" when a `fire*` function bypasses range.

## F10. Game-specific facts to treat carefully
- Identify the user's plot by the `OwnerUserId` attribute equalling `LocalPlayer.UserId`, not by billboard text. This was the reliable key in the observed game.
- Display frames named like `Normal`, `Golden`, `Rainbow` inside an item slot are decorations, not items. Do not treat them as pending purchases.
- Roll cooldowns exist: do not bounce between rollers during a cooldown. Do one teleport pair per pass, then wait out the cooldown in place.

## F11. If a tool call shows up as text, it did not run
If your own reply contains raw tool-call markup (tags like `invoke`, `parameter`, or a JSON blob of arguments) instead of an executed call, nothing ran and there is no result. Do not describe or assume results. Re-issue the call as a proper tool call. Keep each probe small: one tool call per step, a short script that prints as it goes, and no huge multi-purpose scripts or several tool calls bundled together, which makes malformed calls more likely.
