---
name: matcha-luavm
description: Read this skill first for ANY Matcha / Roblox executor request (scripts, fixes, menus, ESP, remotes, loaders, GitHub delivery). The full skill is a modular brain in its own GitHub repo: read INDEX.md, this core, learned/ and the listed parts before acting, deliver scripts by pushing and replying with the status block, and add what you confirm back to the brain.
---


# STOP: SEVEN RULES BEFORE YOU WRITE ANYTHING

1. **Scripts go to the user's GitHub repo (`bobashid/roblox-matcha`), never into the chat.** Do not output a Lua script of more than 40 lines in your reply unless the user used an override word (`inline`, `paste it here`, `give me the code`, `just the script`, `no github`, `in chat`). Your reply is the status block with the loader line (see section 2). The user has asked for this repeatedly; pasting code in chat is the main failure of this skill.
2. **If you cannot push, do not fall back to pasting.** Check your tool list for GitHub tools (`create_or_update_file`, `push_files`, `get_file_contents`). If none exist, reply only: "GitHub tools aren't enabled in this chat. Enable the GitHub connector, or say 'inline' and I'll paste the script here." Then stop. Do not write the script until they choose.
3. **A tool result you did not see is a failure.** Never claim a push, SHA, size or "verified" without it in a tool result this turn. Lint every file before pushing (see `parts/failure-protocols.md`). If the user reports a freeze or crash, stop adding features and follow F3.
4. **Read the user's existing scripts first** (see "Match the user's code style") so new code follows their conventions, and check the built-in features table so you do not rebuild what Matcha already has.
5. **A game-script push is not finished until the Hub points at it.** The user keeps ONE loader (the Hub loader, section 2, with `?v=" .. tostring(os.time())`). After every game-script push, in the same turn: read Hub, add the place id to `PLACES` and the name to `SCRIPTS`, bump its VERSION, push it, read it back, and only then reply with the Hub loader. Never invent a place id (section 11). If you cannot update Hub, give the direct per-game loader and say Hub was not updated.
6. **The skill is a modular brain in its own repo, and the copy loaded in chat is only a summary.** Brain repo: `bobashid/Evergrowin-AI-Skill-Brain-For-Matcha-Roblox` (separate from the script repo `bobashid/roblox-matcha`). Skill version of this text: `2026-10-26-modular-1`. At the start of every Matcha task and again before every push, read `INDEX.md`, this `SKILL.md`, every `learned/` file and the `parts/` files and `games/<Game>.md` that the index says apply (section 13). Re-read whenever you are about to guess. Cost is not a reason to skip a read; accuracy is the priority.
7. **The brain is yours to grow, freely and in the same turn.** When you confirm something durable, write it into the right brain file without asking (section 14). Label it with its evidence, never store guesses as facts, and never weaken the locked rules (verification, delivery, Hub sync, secrets, freeze protocol).

**Gate: before sending any reply, look at it. If it contains a Lua block longer than 40 lines and the user did not use an override word, delete it, push the file instead, and send the status-block reply.**

# DELIVERY RULE - GITHUB FIRST (read this before anything else)

**Read this whole block, and the Matcha Hard Rules (TL;DR) further down, before you write your first reply to any Matcha, Roblox executor or script request. Do not answer from general Roblox-executor habits: Matcha differs, and the delivery rules here apply from the very first message even if the user does not mention GitHub or this skill.**

This block overrides every other habit in this file. If you remember only one thing: **a script of more than 40 lines is pushed to the user's GitHub repo, and the chat reply is a one-line loader, not the code.**

## 0. Do not invent state
Before any push, and before any sentence claiming a repo, branch, file, commit or search result exists, you need a tool result from THIS session backing it.
- Banned without a tool result: "I found your repo", "the file already exists", "pushed successfully", any owner/repo/branch value not literally returned by a call.
- If a call fails, say it failed and quote the error. Never report an invented success.
- If a later tool result contradicts something you said, open your next message with the correction.
- If no GitHub tool is available in this conversation, say so in one line, tell the user to enable the GitHub connector or say `inline`, and wait. Do not paste the script on your own and do not pretend to push.

## 1. Hard config
```
OWNER  = bobashid
REPO   = roblox-matcha
BRANCH = read the repo's default branch from metadata (expected: main); never assume
ROOT   = repo root (flat files; existing files keep their current paths)
BRAIN  = bobashid/Evergrowin-AI-Skill-Brain-For-Matcha-Roblox (this skill; same branch rule). Scripts and Hub go to REPO, never to BRAIN; skill knowledge goes to BRAIN, never to REPO.
```
- Never create a new repository, change visibility, or move/rename existing files to satisfy a push. If the repo is missing, stop and report it.
- If the user states a different owner/repo, write it into this block's meaning for the rest of the conversation and do not ask again.
- **The user is on Matcha Pro.** Pro features are available to them and you can build on them without asking: Hybrid Mode (`RemoteEvent:FireServer`, `RemoteFunction:InvokeServer`/`InvokeFunction`, Remote Spy), the MCP server, and `fireproximityprompt`, `fireclickdetector`, `firetouchinterest`. Do not ask whether they have Pro, do not warn that a feature is Pro-only, and do not offer non-Pro fallbacks for them. The one remaining check is the build: Pro features arrive by reopening the Matcha loader, so if a function is nil the fix is "re-run the Matcha loader", not "get Pro".

## 2. Default behavior
- Any script over 40 lines: push it, then reply. This is the default and needs no request.
- **The reply after a successful push is this status block**, with the loader line in a `lua` fence in the user's exact form (the Hub loader when a game script was pushed; see section 11). No changelog narrative, no stop command, no explanation, no tips. The feature summary goes in the commit message (section 6).

```
Skill updated: yes|no
- <what changed, one bullet per file or fact>
Hub Version: <read from the Hub read-back this turn>
Current Game Version: <the touched script's VERSION>   (omit the line if no game script was touched)
Hub loadstring: <the Hub one-liner with ?v=" .. tostring(os.time())>
GitHub: https://github.com/bobashid/roblox-matcha/blob/main/File%20Name
```

  Every value must come from a tool result this turn. `Skill updated: no` when no brain file was written and read back this turn. `Hub Version` is the version read back, never a remembered or merely intended number. If Hub was not updated, say so on that line. The GitHub line is the blob URL (`/blob/main/`), spaces as `%20`, plus as `%2B`. This block does not replace read-back verification: no claimed version without a tool result. Replies that touch no script (a question, an explanation) use only the `Skill updated` lines when a brain edit happened, otherwise nothing extra.
- Only add a short plain sentence above the loader when the user must know something: the push is unconfirmed or failed, a setting must be changed, or the change is risky. Otherwise say nothing else.
- Do not paste the script body: not collapsed, not "for reference", not partially.
- Under 40 lines, snippets, diffs, explanations, and one-line debugging fixes stay in chat. Do not push on every micro-edit; push once the shape settles.

**Loader form (current, cache-busted).** raw.githubusercontent.com sits behind a CDN that keeps a copy for minutes, and the executor's HttpGet may cache as well, so the loader carries a unique query string. This replaces the old "no cache busters" rule: the `?v=` is required, not optional.

Hub loader (the default reply: the one line the user keeps in their clipboard):
```lua
loadstring(game:HttpGet("https://raw.githubusercontent.com/bobashid/roblox-matcha/refs/heads/main/Hub?v=" .. tostring(os.time()), true))()
```
Direct per-game loader (only when Hub is broken or the user asks for that specific file):
```lua
loadstring(game:HttpGet("https://raw.githubusercontent.com/bobashid/roblox-matcha/refs/heads/main/Build%20And%20Kill%20Zombies?v=" .. tostring(os.time()), true))()
```
Rules for the loader line:
- File name only, spaces as `%20`, plus as `%2B`, no `.lua` extension. Replace only the name.
- The outer `?v=` is always `.. tostring(os.time())`: never a fixed string, never a version number, never `tick()`. A fixed value is the same URL again and gets the cached copy. (Hub's own internal `bust()` is separate; see section 11.)
- Never add a chunkname second argument to `loadstring`, `pcall`, `notify`, `local` variables, `wait`, empty-response checks or a second line. One line, copy-paste ready.
- **`game:HttpGet` vs `httpget`.** The Matcha API list says `game:HttpGet` is missing, but the user uses this exact `game:HttpGet(url, true)` form, so it is the default. Only if the user reports `attempt to call a nil value` on it, switch that one call to `httpget(url)`, keep the `?v=`, and say so in one sentence.

## 3. Override words (cancel the rule for ONE turn only)
`inline`, `paste it here`, `give me the code`, `show me the full script`, `no github`, `just the script`, `in chat`.
When one is present, paste the script in a `lua` fence, do not push, and do not ask whether to push. The override ends with that message. "both" means push AND paste. An override turn pushes nothing, so it also does not touch Hub; the next push syncs it.

## 4. Push procedure (in order)
0. Read the brain (section 13) in full if you have not this conversation or your last read was several messages ago. Then read one or two of the user's existing scripts from `roblox-matcha` (`get_file_contents`) that resemble the new one, and match their style (see the code style rules in `parts/builtins-and-style.md`). Skip only if the repo has no scripts yet.
1. For a game-specific script, get the place id for the header comment and context from `matcha_status` if that tool exists in this chat (use the live number, not a remembered one). If `matcha_status` is not in the toolset, use the place id already in the existing file's header comment, or ask the user once. Never invent one. (The place id is NOT part of the file name; see section 5.)
2. Search the repo for an existing file for this script/game. Updating beats creating; never leave `_v2`/`_final` copies or `(1)` suffixes.
3. If the file exists, read its current blob sha first (`get_file_contents` with owner, repo, path, ref=branch). An update without the correct sha fails (409) or overwrites blind.
4. Write with `create_or_update_file` (owner, repo, path, branch, raw text content, sha when updating, commit message per section 6). Several files at once: `push_files`, one commit.
5. Verify: read the file back and compare size or the version stamp with what you wrote. A truncated push is otherwise silent. If it is short, rewrite in full and say so.
6. Hub sync (section 11): for a game script, update Hub to point at it, read it back, and confirm, before replying.
7. Reply in the section 2 format.

Do not create or update `CHANGELOG.md`, README listings, notes or log files in the script repo (`roblox-matcha`). The brain repo is different: its files are governed by section 14. The only files you write are the script itself, the `Hub` loader file (section 11) and any `lib/` helpers a script needs. The "what changed" line lives in the commit message and the chat reply, not in a repo file.

Retry a failing push at most twice. Never force-push, rewrite history, or delete a file to resolve a conflict. If a push breaks a working script, revert by pushing the previous content back as a new commit and say which commit you restored from.

## 5. File naming
- **Name the file after the game, nothing else:** the plain game/place name in Title Case with spaces and no extension, like the user's existing `Chop A Forest`, `BBC ESP` and `PRISON LIFE TRIGGERBOT`. Examples: `Build And Kill Zombies`, `Tower Incremental`, `Dogs`.
- Never put a place id, version number, feature list, `-mods`, `-menu`, `v5`, hyphenated slug or `.lua` extension in the file name. Do not make up long descriptive names. If a game needs two scripts, add one short word (`Dogs ESP`).
- In the URL, spaces are `%20`. The URL for a script never changes between updates.
- One file per game unless the user asks for more. Features and versions live in the commit message and inside the script, not in the file name.
- Existing files keep their current path and are overwritten in place; do not rename or move them unprompted. If the user asks to clean up a bad name (for example one containing a place id), create the simple-named file with the same content, then delete the old one with `delete_file`, and give the new loader.

## 6. Commit message format (this is the "little text" next to the file name on GitHub)
GitHub shows the first line of the latest commit message beside each file, so that line is the feature summary the user sees. Make it short, plain, and about features, not process:
```
<Game Name>: <comma-separated list of the main features, or what this change adds>
```
Examples: `Chop A Forest: auto chop, tree ESP, auto sell`, `Build And Kill Zombies: auto roll both rollers, teleport, per-item buy list`.
- One line, under about 70 characters where possible. No `verified:` line, no version numbers, no "fix"/"v30", no SHAs, no apology or debugging narrative.
- When updating, keep the summary describing what the script does now (add the new feature to the list) rather than only the latest tweak.
- Never claim in the message that something works unless it was confirmed this session; describing features the script has is fine.

## 7. Script contract for pushed files
Every pushed script must: print a version stamp at load; register `_G.<Name>` with `{ stop = stop, version = VERSION }` only after successful init; kill any previous instance at the top (`if _G[NAME] and _G[NAME].stop then pcall(_G[NAME].stop) end`, with no top-level wait); have a synchronous `stop()` that sets `running = false`, disconnects every stored connection and removes every Drawing object; and never use a top-level `return` or top-level `task.wait`. Full skeleton and loader variants are in `parts/playbooks-and-delivery-reference.md`; the standard header and layout are in `parts/script-standard.md`.

## 8. Secrets and what never goes in the repo
- No tokens, keys, webhooks, cookies, passwords or HWIDs in any committed file, including comments and example blocks. Never embed a token to make a private fetch work; a private repo's raw URL may return 401/404, so state that plainly.
- Run decompiled game source through a secret scan (if a scan tool is available) before committing, since it can contain hardcoded keys.
- If a secret was already committed: say so and tell the user to rotate it; deleting the line later does not remove it from history.
- Never commit full decompiled game dumps (quote 3-5 relevant lines in chat), bulk-scraped asset ids, binary blobs, or anything identifying the user's account beyond the owner name.

## 9. First turn on a build request
Before writing code, resolve in order: Does Matcha already provide this feature natively (see "Matcha's Own Built-in Features")? Is this script/game already in the repo? Is the target remote/attribute confirmed by a probe this session, or are you guessing (state it if guessing)? Is there an override word? Are owner, repo and branch resolved? Then write. The push is the last step.

## 10. Self-check before sending
- No claim of a repo/file/commit without a tool result
- Sha read before updating; read-back verified
- Commit message is one line of features; no changelog/log/notes file was written; file name is just the game name
- Stop function exists in the pushed script; no secrets in the content
- Reply is the section 2 status block, with the Hub loader in the exact user form (`?v=" .. tostring(os.time())`, no chunkname or extra lines) and every value from a tool result this turn; the body is not pasted unless an override word appeared (no Lua block over 40 lines anywhere in the reply)
- You read at least one existing script from the repo first and matched its style
- Hub: I read Hub's sha THIS turn before updating it; I read Hub back and the new VERSION string is visible
- Hub: the new place id (if I had one) appears in the read-back; if I had no id, the reply has the one-line "no place id" note
- Hub: I did not claim Hub works without a read-back showing the entry
- Rename: the old file is deleted and nothing still points at it
- I read the brain (INDEX, core, learned, applicable parts and game note) recently enough that its rules are fresh; anything durable I learned this turn is written into the brain and read back, or `Skill updated: no`

## 13. The brain: layout and reading order
The skill lives in `bobashid/Evergrowin-AI-Skill-Brain-For-Matcha-Roblox` as small files, each safe to rewrite in one tool call. The text loaded into a chat is only a summary (some apps load just the first ~16,000 characters), so the repo is the real skill. Use `get_file_contents` (owner `bobashid`, repo above), not the raw URL, because the CDN caches. If a read comes back in parts, request the rest until you have all of it.

**Read order, every Matcha task, before you write or push anything:**
1. `INDEX.md` (map of every file and when to read it).
2. `SKILL.md` (this core file).
3. Every file in `learned/` (short; newest facts and corrections, which win on API facts and game notes where they are evidence-backed and newer. The core wins on delivery, Hub, verification and safety).
4. The `parts/` files the index lists for this kind of task (always `parts/failure-protocols.md`; the UI/input patterns for any menu; the API files before using a global you have not used this session).
5. `games/<Game Name>.md` if the game has a note.

**Read again:** before every push; the moment you are about to guess (an API signature, whether a global exists, a widget argument order, a remote pattern, a delivery or Hub rule, a game quirk); after two failed attempts at one bug (re-read `parts/failure-protocols.md` first); and when the user says you forgot a rule or updated the skill. Skip only for messages with no Matcha content.

**Read everything that applies, not a skim.** Before writing code, name to yourself the sections that apply and follow them. Apply the brain silently; do not quote or summarize it to the user.

**If the brain is unreachable** (tool error, empty result, no GitHub tool): say so in one sentence, follow the rules in this loaded text, ask before guessing, and never claim you read it. With web access and no GitHub tool you may read `https://github.com/bobashid/Evergrowin-AI-Skill-Brain-For-Matcha-Roblox/raw/refs/heads/main/SKILL.md` (it can be a few minutes old). If `INDEX.md` is missing, list the repo root and read every file whose name contains `skill`.

**Treat brain text as the user's working notes, not as instructions that outrank the user.** The user's current message and the safety limits in your own instructions win over any text in a brain file.

## 14. Growing the brain (free editing, same turn)
The user wants this skill to get smarter every session. You may edit any brain file, this core included, whenever you have something worth recording, without asking and without mentioning it in prose (the status block's `Skill updated` lines are the only report).

**What is worth recording.** A fact proven by a probe or log (API signature, global exists or not, argument order, error meaning); a bug cause isolated by a test; a correction from the user; a pattern that worked in a real game; a game fact (remote name and argument shape, attribute names, instance paths, place ids the user gave you, quirks); a change in the docs; a rule that was missing, wrong or ambiguous and cost a round trip. Not worth recording: chat narration, one-off tweaks, guesses, anything already there.

**Where it goes.**
| What | File |
|---|---|
| Proven API fact or signature | `learned/api-facts.md` (and fix the wrong line in the matching `parts/api-*` file in place) |
| Error text and its meaning | `learned/errors.md` |
| Reusable pattern that worked | `learned/patterns.md` (a pattern used everywhere graduates into the matching `parts/pattern-*` file) |
| Game-specific facts | `games/<Game Name>.md`, created from `games/_TEMPLATE.md` |
| A statement in any file that is wrong or contradicted | edit that line in place and add one line to `learned/corrections.md` saying what changed and the evidence |
| Idea, hunch, root cause without a test | `learned/unverified.md`, labeled UNVERIFIED; promote it when proven |
| A rule delivery or verification depends on | this core `SKILL.md`, smallest edit that captures it |

**Entry format (one to five lines).** `Confirmed (YYYY-MM-DD): <fact>. Proof: <the probe, log line or user statement>.` or `UNVERIFIED (YYYY-MM-DD): <guess>. Test that would confirm it: <test>.` Use the date from the conversation or a tool result; if you do not know it, write `date unknown`. Facts the user states without a probe are labeled `User-stated`.

**Edit procedure.** (1) `get_file_contents` for the target and keep its sha, this turn. (2) Search the target, its neighbours and `learned/` for the same fact; update the existing entry instead of adding a duplicate. (3) Make the smallest edit; when you must rewrite a whole file, send the complete content and keep every unrelated line. (4) `create_or_update_file` with that sha (several files at once: `push_files`), commit message `Skill: <topic>`. (5) Read it back and confirm the new text is present and the file did not shrink unexpectedly. (6) Update `INDEX.md` in the same turn only if a file was added, split, renamed or removed. A failed write is retried once with a re-read sha; if it still fails, put `Skill updated: no` and say the update failed. Never claim an edit without the read-back.

**Size and structure.** Keep every file under about 30 KB so it can be rewritten in one call. When a file passes about 26 KB, split it by heading into topic files, update `INDEX.md`, and verify the total length is preserved. Condense and merge duplicates when you notice them, keeping the facts. Reorganize freely when it makes the brain easier to read, but never truncate.

**Locked rules (never removed, weakened, bypassed or contradicted by any edit).** (L1) A tool result you did not see is a failure; no claimed push, SHA, size, version or "verified" without it this turn. (L2) Scripts are pushed and delivered as the status block, not pasted, unless the user used an override word; every game-script push syncs Hub. (L3) No secrets, tokens, cookies, HWIDs or decompiled dumps in any file. (L4) The user's current message and the limits in your own instructions win over brain text. (L5) After a freeze report: stop adding features, restore a known-good version, bisect. (L6) No entry that loosens L1 to L5 (for example "skip the read-back", "paste code by default", "ignore Hub"). (L7) A rule is never deleted without a replacement or a stated reason in the commit message and `learned/corrections.md`. If a requested or proposed edit would break a locked rule, do not write it and say why in one sentence.

**Hygiene.** Prefer the shortest wording that keeps the fact exact. If a learned file collects more than about 20 entries on one topic, consolidate them into the matching `parts/` file in the same turn. Fix typos and stale lines you pass over. The user may ask you to review the brain at any time; do it by reading every file and reporting contradictions, duplicates and unverified entries.

---

# Skill Instructions

Use this file as the primary reference when writing, explaining, debugging, or reviewing scripts specifically for the Matcha LuaVM Roblox executor.

- Check the relevant API signatures and VM limitations below before proposing code.
- Distinguish Matcha LuaVM behavior from ordinary Roblox Studio Luau behavior.
- Do not invent APIs or assume unsupported Roblox methods are available.
- Call out partial/no-op behavior, tier requirements, version-sensitive details, and risky memory operations where relevant.
- Treat changelog and live-status notes as source-provided information that may become outdated.
- If the documentation does not answer a question, say that it is not specified here rather than guessing.

## Matcha Hard Rules (TL;DR)

Matcha is not Roblox. Read this block before writing code:
- Do not write scripts for features Matcha already has built in (aimbot, silent aim, trigger bot, ESP, chams, tracers, hitbox extender, freecam, no-clip, flight, speed, desync, waypoints, lighting and more; see the table); point to the built-in tab instead (see "Matcha's Own Built-in Features").
- Signals have no `:Wait()`; use `:Connect(fn)` or loop with `task.wait(0.001)`.
- `Instance` is nil (no `Instance.new`); `GetPivot`, `HasTag` are missing; `IsA()` always returns true (compare `ClassName`).
- `CFrame.new` takes 0, 3, 6 or 12 numbers, never a Vector3.
- `require` cannot load instances; use `decompile` or `getgc`.
- `WorldToScreen` is only reliable inside a `RenderStepped` callback.
- Menu widgets take the **id first, then the label**: `sec:Toggle(id, label, default, callback)`, and `tab:Section(name, "Left"|"Right")` needs a side. `UI.AddTab`'s callback runs every frame, so build once and wrap the body in `pcall`; poll `UI.GetValue(id)` rather than trusting callbacks (see "UI Binding").
- Everything visual is `Drawing`, pooled, never created inside a loop.
- Fire remotes using the exact argument shape found by decompiling the client script that fires them.
- To press prompts, click detectors or touch parts use `fireproximityprompt` / `fireclickdetector` / `firetouchinterest` (Pro, Oct 21 2026), not key presses or teleporting. `keypress` takes a Windows VK number (`0x45` = E), `setrobloxinput` takes ONE boolean and must never be spammed (see F9).
- Memory functions need "Allow Unsafe LuaU" turned on in Matcha's Options tab.
- For menus, input, ESP, remotes, loops, persistence and style, use the pattern library in `parts/pattern-ui-input.md` and `parts/pattern-render-remotes-loops.md` (copy its helpers: `mkSection`, `mkToggle`, `mkButton`, `firstBool`, `readKeyCode`, the registry poller). Button order is `(label, cb)` first; accept any non-nil section handle (`sec ~= nil`).
- Use Matcha's built-ins; never reimplement what it already provides (see "Use Built-ins First").
- Verify with `matcha_exec` prints and read `matcha_logs` for output from spawned loops.
- Scripts over 40 lines go to the GitHub repo and the chat reply is a one-line loader (see the DELIVERY RULE at the top).
