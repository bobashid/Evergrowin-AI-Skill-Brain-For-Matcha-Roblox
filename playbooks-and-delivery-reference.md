# Matcha API 4/5: menu binding, investigation playbook, delivery reference (brain part)

## Menu Binding: Practical Rules
Use the official signatures in "UI Binding (Menu Binding)" (id first, then label; Section needs a side). Practical lessons from real sessions:
1. **Build safely.** The tab callback runs every frame. Build widgets behind a flag or in a way that is safe to repeat, wrap the whole body in `pcall`, and keep widget counts modest. A tab-open crash was traced to too many widgets; test with a handful first and scale up (building ~200 toggles synchronously is a risk).
2. **Section needs a side.** `tab:Section(name)` without `"Left"`/`"Right"` can leave the tab blank (a script fix was "Section needs a side"). Always pass the side.
3. **Poll as the source of truth.** Reading `UI.GetValue(id)` every ~0.1 s is more robust than relying on callbacks; use the callback only for immediate side effects. If you must use the callback, the first argument is the state for a Toggle.
4. **Callbacks only set flags.** No `wait`, no input, no teleports and no heavy work inside a widget callback. Do all waiting in one spawned loop.
5. **No debounce.** A debounce on callbacks dropped clicks and looked like a stuck toggle.
6. **ColorPicker** gives a Color3 plus alpha; guard with `typeof(c) == "Color3"`.
7. **Log on change** so a broken wire is visible in the console.
8. **Button** is `sec:Button(label, callback)`. Do not insert an id.
9. A helper that retries argument variants is only a last resort when an exact error shows the signature differs on the user's build; prefer fixing the call from the error text.

## Investigation Playbook (finding what a game does)
Order that worked repeatedly:
1. `matcha_status` once: confirm in-game and hybrid mode (needed for `FireServer`).
2. Map remotes: search with an empty query plus a `ClassName` filter (`RemoteEvent`, then `RemoteFunction`).
3. If remotes are grouped under a ModuleScript that returns a table, that module is the authoritative list; decompile it.
4. To learn a remote's argument shape, decompile the client script that fires it and read the exact `FireServer(...)` line.
5. Dump `LocalPlayer:GetAttributes()`: it often exposes the whole progression state (counts, levels, perks).
6. For a model, `matcha_tree` at depth 1-2, then identify the part the game actually raycasts rather than the decorative one.
7. Distinguish states by combining an attribute (for example HP > 0) with a structural child (present/absent).
8. Batch many prints into one `matcha_exec`.
9. `getgc`/`applygc` only rewrite the client's copy. Server-authoritative values (damage, cooldowns, reach) will not change, so do not offer GC tweaks for combat values unless the user can show they are client-trusted.

## Always True of Matcha Scripts
- Never use an Instance as a table key or compare with `==`; key by `UserId`, `.Address` or a path string.
- Drawing objects are created once and pooled; creating them in a loop leaks until the VM restarts. Call `:Remove()` when done.
- Every loop must yield: a bare `while true do end` is killed at about 5 s. Use `task.wait(0.005)` or more in ESP/overlay loops if the overlay looks choppy.
- `game:GetService` works; `Instance.new`, `Clone`, `Destroy`, `.ChildAdded`, `.Touched` and `PlayerAdded` should not be relied on. To watch joins, poll `Players:GetPlayers()` and diff.
- Build scripts print-incremental, and give long scripts a `_G.<Name>.stop()` so the user can kill them without rejoining.

## Design Defaults for Generated Scripts
- Deliver full scripts, never diffs: copy-paste ready, one fenced block.
- Menu-only control; no keybinds unless asked.
- Widget count is a spec: if the user says 6 toggles, ship 6.
- No sliders for tunables unless asked; hardcode them as named constants at the top.
- Automation defaults ON; risky extras (teleports, GC edits) default OFF.
- Every sell/teleport/remote feature prints a one-line confirmation so "did not run" is distinguishable from "ran and was rejected".
- State unverified assumptions explicitly and name the one line to change if a remote argument or attribute name is wrong.

## Scheduler Performance Warning
A bare `task.wait()` loop that fires remotes does so as fast as the VM schedules and can starve the Drawing overlay (choppy ESP). If visuals look choppy, that loop is the usual cause; `task.wait(0.005)` keeps it fast and frees frames.

Because bare `task.wait()` is a ~16 µs busy yield, `while running do ... task.wait() end` spins at ~60,000 iterations per second and starves the scheduler (a direct cause of lag). Use `task.wait(0.001)` or a `Heartbeat`/`RenderStepped` `:Connect` callback in hot loops (never `:Wait()` on a signal).

## GitHub Delivery Workflow: Reference

The delivery rule at the TOP of this file is authoritative; if anything below conflicts with it, follow the top block. This section keeps the loader variants, skeleton, failure table and checklist.

Purpose: deliver scripts as a one-line loader the user pastes into the executor, with the source stored in their GitHub repo. Benefits: no pasting long code, every change is a commit with rollback, and history is browsable. Use this by default when the user has a repo and a GitHub tool/connector is available. Ask once for `<owner>/<repo>`, then reuse it for the rest of the conversation. Placeholders below (`<owner>`, `<repo>`, `<file>`, `<Name>`) must be replaced with the user's real values.

### Core rules
- Push the file, then reply with only the loader line (the exact form in section 2 at the top of this file), not the code. No changelog, version or stop command in the reply.
- The URL never changes between updates: same filename, same branch, overwritten in place. Never version filenames (`(1)`, `v2`); a stale copy gets cached and you end up debugging the wrong file.
- Never write to the user's repo unless asked. Reading is safe; pushing is a side effect. Confirm filename and commit message before a write.
- Verify a write happened (the blob sha changed, or the raw file shows the new version stamp) before saying "pushed".
- Scripts must be safe to make public: never commit tokens, webhooks, HWIDs or other secrets.

### Repo layout and naming
- Flat files at the repo root on `main`; no folders unless asked. One file per game/script.
- File names are the plain game name in Title Case with no extension (see section 5 at the top). Spaces become `%20` in raw URLs.
- Only touch a `README.md` if the user asks; do not add script listings or logs on your own.
- Raw URL pattern: `https://raw.githubusercontent.com/<owner>/<repo>/refs/heads/main/<file>`. Raw URLs are case-sensitive. A 404 means wrong branch, case or path.
- URL encoding: space `%20`, `#` `%23`, `?` `%3F`, `&` `%26`, `%` `%25`.

### Loader forms
The user's default loader is the cache-busted Hub loader from section 2 (`game:HttpGet(url .. "?v=" .. tostring(os.time()), true)`); see sections 11 and 12. If it errors with `attempt to call a nil value` in Matcha, switch to the Matcha-native `httpget` form. The cache-buster, hardened and chunkname variants below are only for when the user asks for them or reports the matching problem.

```lua
-- Matcha-native (default for Matcha)
loadstring(httpget("https://raw.githubusercontent.com/<owner>/<repo>/refs/heads/main/<file>"))()

-- Common executor form (only if the user says it works for them)
loadstring(game:HttpGet("https://raw.githubusercontent.com/<owner>/<repo>/refs/heads/main/<file>", true))()
```

The loader must stay one line, paste-able on its own, with the two call parentheses after `loadstring(...)`.

**Cache-buster** (raw.githubusercontent.com caches for minutes; if the user sees old behavior, the cache is stale, not the push):

```lua
local u = "https://raw.githubusercontent.com/<owner>/<repo>/refs/heads/main/<file>"
loadstring(httpget(u .. "?v=" .. tostring(os.time())))()
```

**Hardened loader** (when the user reports "loaded but nothing happens"). `httpget` never throws: on a 404 or connection failure it returns `""`, and `loadstring("")` returns a function that does nothing.

```lua
local u = "https://raw.githubusercontent.com/<owner>/<repo>/refs/heads/main/<file>"
local src = httpget(u .. "?v=" .. tostring(os.time()))
if src == nil or #src == 0 then
    warn("[loader] empty response - check path/branch")
else
    local fn = loadstring(src, "<Name>")
    if fn then pcall(fn) else warn("[loader] compile failed") end
end
```

Passing a chunkname as the second `loadstring` argument makes console errors read `<Name>:42: ...` instead of `Matcha:42: ...`. For a private repo, raw URLs need an Authorization header (`httpget(u, {["Authorization"] = "Bearer <token>"})`); never hardcode a token in a script given to the user, and prefer a public repo since scripts should contain no secrets.

### Script contract for loader-loaded files
- No top-level `return` (dropped by `loadstring`); communicate through globals.
- Register one uniquely named global, set only after successful init: `_G[NAME] = { stop = stop, version = VERSION }`, so two scripts can run together.
- Print a version stamp at load, for example `print("[" .. NAME .. "] loaded " .. VERSION)`. It is the only reliable confirmation a push took effect, since syntax errors are uncatchable and an empty `httpget` is silent.
- Reload-safe: at the top, `if _G[NAME] and _G[NAME].stop then pcall(_G[NAME].stop) end`. Keep `stop` synchronous; observed behavior is that a top-level `task.wait` (or `:Wait()`) throws or misbehaves, so if a wait is unavoidable wrap the sequence in `task.spawn`.
- Three-part `stop()`: set `running = false`, `:Disconnect()` every stored connection, `:Remove()` every Drawing object. Route every `:Connect` through a `trackConn` helper and every `Drawing.new` through `trackDraw`, and `pcall` everything inside `stop`.

```lua
-- <file>
local VERSION = "<name> v1 <timestamp>"
local NAME    = "<Name>"
if _G[NAME] and _G[NAME].stop then pcall(_G[NAME].stop) end   -- no wait here
local running = true
local conns, drawings = {}, {}
local function trackConn(c) conns[#conns+1] = c; return c end
local function trackDraw(d) drawings[#drawings+1] = d; return d end
local function stop()
    running = false
    for i = 1, #conns do pcall(function() conns[i]:Disconnect() end) end
    conns = {}
    for i = 1, #drawings do pcall(function() drawings[i]:Remove() end) end
    drawings = {}
    print("[" .. NAME .. "] stopped")
end
-- ... services, config, menu, loops ...
-- every :Connect -> trackConn(RunService.Heartbeat:Connect(fn))
-- every Drawing -> local box = trackDraw(Drawing.new("Square"))
_G[NAME] = { stop = stop, version = VERSION }
print("[" .. NAME .. "] loaded " .. VERSION)
```

### Pushing updates (GitHub tool calls)
Tool names depend on the connector; these are the common ones.
- **Single file:** `create_or_update_file`. Creating needs no sha; updating needs the current blob sha, so call `get_file_contents` on the same path and branch first. A missing sha is the most common failed update.
- **Several files:** `push_files` (one commit, array of `{path, content}`, no sha handling).
- **Risky rewrite:** `create_branch` from `main`, commit there, `create_pull_request`, merge after the user tests.
- **Delete a file:** `delete_file` exists (owner, repo, path, message, branch). Deleting or renaming the repo, changing visibility, webhooks, secrets, force-push and branch protection are not available through these tools.
- **Commit messages:** `<script>: v<N> <what changed>`, for example `my-game-esp: v3 fix sell teleport CFrame args`.
- **Verify after pushing:** call `get_file_contents` again and confirm the sha changed, or open the raw URL and check the version stamp. Never say "pushed" without one of these.
- **Check write access once** before the first real push: push a throwaway file (for example path `test`, content `test`); a returned commit sha confirms write access, then remove it with `delete_file`. This keeps a permission failure from being mistaken for a script bug.

### Permissions
Owner authentication is not the same as scopes, so verify rather than assume. Minimum for this workflow: Contents Read and write (for pushing) and Metadata Read; add Pull requests Read and write only for the branch + PR path. Issues, Actions, Administration, Codespaces and artifact scopes are not needed. Classic token equivalent: `repo` (or `public_repo` for a public repo).

### Iteration loop and reply shape
1. User says what to change (or "push it"). 2. Edit the local copy. 3. Commit (with sha, or `push_files`) with a one-line feature summary as the message. 4. Reply with only the loader line. 5. User re-runs the loader; the version stamp printed by the script confirms which build is live.

Default reply: the single loader line from section 2 in a `lua` fence, and nothing else. For large or risky changes, add one plain sentence above it saying so and offer the PR variant.

### Failure modes
| Symptom | Cause | Fix |
|---|---|---|
| Loader errors / no output | Wrong path, case, or missing `%20` | Open the raw URL in a browser |
| Old code still runs | CDN cache | Append `?v=os.time()` |
| "Loaded" but nothing happens | Empty `httpget` or a bad chunk | Check `#src` and the console |
| `attempt to call a nil value` on `game:HttpGet` | Running in Matcha | Use `httpget(url)` |
| Loops double, lag climbs | Previous instance not stopped | Reload-safety block |
| Drawings pile up / flicker | `stop()` only set a flag | Disconnect + Remove |
| 404 on load | Wrong branch | Use `refs/heads/main` |
| Push rejected | Branch protection | Branch + PR |
| Update call fails | Missing blob sha | `get_file_contents` first |
| Script loads then dies silently | Top-level `task.wait` threw | Move into `task.spawn` |

### Pre-push checklist
- File name is just the game name (Title Case, no extension, no place id or version), replacing the old file in place
- Version stamp updated and printed at load
- Reload-safety block at the top; `stop()` sets the flag, disconnects everything, removes all drawings
- No top-level `task.wait` and no top-level `return`; `_G[NAME]` set only after successful init
- No secrets or tokens in the file
- Raw URL opens in a browser and shows the script
- Commit message follows `<script>: v<N> <what changed>`
- Reply contains only the loader one-liner in the user's exact form
