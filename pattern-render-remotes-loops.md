# Pattern library 2/2: rendering, game state, remotes, loops, persistence (brain part)

## RENDERING / ESP RECIPES

### R1. `WorldToScreen` contract
Returns `(Vector2, bool)`. ALWAYS check the bool (`local sp, on = WorldToScreen(p)`). It is reliable ONLY inside a `RenderStepped` callback. To use a screen position from a `task.spawn` loop (aim, mine, click), cache it during `RenderStepped` onto the target table (`t.sx`, `t.sy`, `t.sAt = tick()`) and have the other loop read the cache with a freshness check (`tick() - t.sAt < 0.5`). Behind-camera and far off-axis points return `(0,0), false`.

### R2. 3D box from 12 edges (best-looking ESP in the repo)
Get 8 corners via `cf:PointToWorldSpace(Vector3.new(+-hx, +-hy, +-hz))`, project each, and draw 12 `Drawing.new('Line')` pairs. Thickness scales with distance: `clamp(floor(150 / max(dist, 40)), 1, 5)`. CACHE the last thickness per pool entry so you do not rewrite 12 lines every frame.
```lua
local EDGES = {
    {1,2},{3,4},{5,6},{7,8},
    {1,3},{2,4},{5,7},{6,8},
    {1,5},{2,6},{3,7},{4,8},
}
local th = math.floor(150 / math.max(dist, 40))
if th < 1 then th = 1 elseif th > 5 then th = 5 end
if e.thick ~= th then
    for k = 1, 12 do e.lines[k].Thickness = th end
    e.thick = th
end
```
(Matcha already has a built-in ESP with 3D boxes; this recipe is for game-specific object ESP such as ore, eggs or items.)

### R3. Transparency = distance fade + user slider
Combine so the slider still works at close range: `tr = 0.1 + 0.65 * (dist / MAX_DIST)`, add 0.2 if out of reach, then `tr = tr + userTr * (1 - tr)`, clamp 0..1. Expose the slider as 0..100 in the UI and divide by 100.

### R4. Health-bar pool with smoothing and snap
Per bar: a background `Square` (Filled), a fill `Square` (Filled) and a `Text` label. Smooth per frame with `alpha = 1 - exp(-rate * dt)`, clamp `alpha` to `[0.35, 1]` so slow frames still track, and SNAP when the target jumps more than `SNAP_PX` (`if dx*dx+dy*dy > snapSq then e.x, e.y = tx, ty`). Reuse pool entries by a stable key, refresh `lastSeen`, and drop the key after `FORGET_AFTER` seconds unseen. Colour the fill by fraction (green above 50%, yellow above 25%, red below).

### R5. Pools are fixed and pre-allocated
Create every `Drawing` ONCE at load into a table (48-64 entries), then only flip `Visible` and write properties. NEVER call `Drawing.new` inside a loop or frame callback. Every pool needs a `clearPool(pool, cap)` used when its feature is toggled off, and `stop()` must `:Remove()` every Drawing.

### R6. Cull before projecting
Distance-cull in world space (`MAX_BAR_DIST`, `MAX_DIST`) BEFORE calling `WorldToScreen`, and cap the number drawn (`MAX_BOXES`). Projection is the expensive part.

### R7. Pool keys: use `inst.Address`
`tostring(inst.Address)` is a stable per-instance identity. Use it (or `UserId` for players) as the key for pools and lookup tables, never the model name (duplicates) and not the Instance itself in tables you later prune. (`==` between instances is supported per the docs, but Address keys are safest for pools.)

## GAME STATE: PREFER ATTRIBUTES
Read state with `:GetAttribute` instead of scanning descendants when the game already exposes it. Keys seen working in the user's games: `CarriedLogs`, `CarryRarity`, `Up_GearBackpack`, `HP`, `Gems`, `CollectedBy`, `Rarity`, `Cell`, `CombatantId`, `Zone` ('Dead' / 'Arena' / 'Lobby'), `Health`, `MaxHealth`, `Damage`, `MaxAmmo`, `Local_CurrentAmmo`, `OwnerUserId`, `Cycle`.
- Wrap reads: `local ok, v = pcall(function() return inst:GetAttribute(k) end)`.
- `tonumber(...)` every numeric attribute; it can arrive as a string or nil.
- `:SetAttribute` is a legitimate LOCAL refill for client-predicted values (`gun:SetAttribute('Local_CurrentAmmo', max)`).
- Identify the player's own plot by `OwnerUserId == LocalPlayer.UserId` (number to number).
- Detect a reset or round change through a changing attribute (`Cycle`, `Cell` generation) and clear your caches when it moves.
- `GetAttributes()` on the local player is a quick way to map a game's progression state.

## REMOTES: PAYLOADS, VERIFYING, RATE LIMITING

### RM1. Resolve remotes once at the top, warn if missing
```lua
local RS = game:GetService('ReplicatedStorage')
local remotes = RS:FindFirstChild('Remotes')
local Attack  = remotes and remotes:FindFirstChild('Attack')
if not Attack then warn('[NAME] Remotes.Attack not found - script idle') end
```
`RemoteEvent` uses `:FireServer(...)`; `RemoteFunction` uses `:InvokeServer(...)`. Always `pcall` the call itself. Remote firing needs Hybrid mode, which the user has (Pro).

### RM2. Copy the payload from the client, do not guess
Decompile the client script that fires it and copy the exact argument list. Shapes confirmed working in the user's games: `Chop:FireServer(tree, hitPos, 1, counter)`, `PickUpLog:FireServer(log)`, `DepositLogs:FireServer()`, `ShootEvent:FireServer({{origin, targetPos, targetPart}})`, `Attack:FireServer(model, {crit, sprint, charge, vt = os.time(), dir, eye, t = os.clock()})`, `EggGrab:InvokeServer(eggId)`, `ArrestPlayer:InvokeServer(plr)`, `InteractWithItem:InvokeServer(model)`. When a shape is a guess, mark it inline: `-- UNVERIFIED: arg shape`.

### RM3. Verify by reading state back, never by assuming
After `EggGrab`, check `CarryRarity` is now non-empty. After `DepositLogs`, check `CarriedLogs` dropped. After a swing, check the target's `HP` fell. Only then count it as success, and use that to open a "hurt window" so you do not re-hit a target the server is ignoring.

### RM4. Rate-limit and fail-track invokes
Every invoke needs a cooldown (`GRAB_GAP`, `SELL_COOLDOWN`, `rateMs`), a per-target ban list (`banUntil[id] = os.clock() + BAN_TIME`), a consecutive-failure counter, and a pause after N misses (`pausedUntil = os.clock() + PAUSE_TIME`). Hammering a bad id or a rejected remote gets you kicked or frozen. Respect the game's own cooldowns (for example roll cooldowns): wait them out in place rather than moving between targets.

### RM5. Distance is server-checked
The server may validate range even when you fire a remote. For actions that require proximity, move near first or use TP-and-return. The `fire*` interaction globals bypass client-side range and hold time, but whether the server checks range is game-specific; verify by reading state back.

## TP-AND-RETURN (named recipe)
Use only when an action truly needs you at a spot (a remote or prompt that validates distance on the server) and no `fire*` global or remote route avoids it. For actions that need you at a spot for one or two frames (sell, grab, arrest):
1. Save `local home = hrp.CFrame`.
2. Jump: `hrp.CFrame = CFrame.new(target.X, target.Y + hoverY, target.Z)`.
3. `task.wait(0.18)` (let the server see the new position).
4. Fire the action, twice 0.18 s apart in case the first is dropped.
5. `task.wait(0.12)`.
6. `hrp.CFrame = home`.
Notes: it is visible to other players for about half a second; never do it inside a `RenderStepped` callback, only from a `task.spawn` loop with a cooldown; do one teleport pair per pass (not one per press) and wait out cooldowns in place; if the character respawns mid-sequence the restore fails silently, so `pcall` it. Repeated CFrame writes on a simulating character can desync it (see "Replication and Network Ownership Caution").

## `getgc` / `applygc`: SCHEDULE IT, NEVER LOOP IT
`getgc(keyArray)` plus `applygc(cache, key, value)` cost 10-20 seconds on the user's build (the docs also warn `setgc` lags if looped). Rules:
- Call `getgc` once in a background `task.spawn` at load, and again ONLY from a manual "Rescan" button.
- Never inside a `while` loop, never twice in one frame.
- Reuse the returned `cache` in a cheap apply loop (`applygc` per key, about 4 Hz).
- If a key class returns 0 hits, `notify` the user to press Rescan while the value is actually changing (stamina draining, gun equipped).
- This is the one documented exception to the "probes under 1.5 s" rule (F8); label it as such so it is not "fixed" later.
- GC edits only change the client's copy; server-validated values will not change in effect.

## PERSISTENCE (`writefile` / `readfile`)
Flat `key=value` lines, one per line, written on every change and read once at load. Clamp numbers on load and never trust the file. `readfile` errors when the file does not exist yet; that is normal, so `pcall` it and use defaults. Allowed extensions are `.txt .lua .luau .json .cfg .dat .ini`.
```lua
local CONFIG_FILE = 'gamename.cfg'
local function saveCfg(cfg)
    local lines = {
        'keyname=' .. tostring(cfg.keyName),
        'trans='   .. tostring(cfg.trans),
        'silver='  .. tostring(cfg.silver),
    }
    pcall(function() writefile(CONFIG_FILE, table.concat(lines, '\n')) end)
end
local function loadCfg(cfg)
    local ok, data = pcall(function() return readfile(CONFIG_FILE) end)
    if not ok or not data or data == '' then return end
    for line in string.gmatch(data, '[^\n]+') do
        local k, v = string.match(line, '^(%w+)=(.*)$')
        if k == 'trans' then
            local n = tonumber(v)
            if n then cfg.trans = math.clamp(n, 0, 1) end
        elseif k == 'silver' then
            cfg.silver = (v == 'true')
        end
    end
end
```

## ANTI-AFK AND THE UNFOCUSED CLIENT
Only three things keep an unfocused client alive (from the user's Anti Afk script):
1. `RobloxReplicatedStorage.SetUserActive:FireServer(true)`, the client's own activity remote. Find it by name (`SetUserActive`, or scan for `active`/`idle`/`presence` in `RemoteEvent` names).
2. A camera micro-drift: `cam.CFrame = cam.CFrame * CFrame.Angles(0, math.rad(0.3), 0)` a few times per pass. Invisible to others and does not move the character.
3. `isrbxactive()` to know whether the key path is even worth trying. `keypress`, `mouse1click` and `setrobloxinput` do not reach an unfocused window.
Jitter the schedule (`interval +- random(-j, j)`, floor 10 s) so the pattern is not machine-perfect.

## LOOP ARCHITECTURE AND PERFORMANCE
- **L1. One `task.spawn` per subsystem.** Collector(s), renderer, fire loop, sell loop and menu poller are SEPARATE `while running do ... task.wait(x) end` loops. Never one mega-loop; a slow subsystem must not stall the renderer. (Every loop needs exactly one wait per iteration; a loop with no wait is killed at about 5 s.)
- **L2. Tick rates.** Collector 0.1-0.4 s; health refresh 0.1 s; menu poll 0.1-0.15 s; sell check 0.1 s; fire loop per-frame only when proven safe, otherwise 0.05 s or more; render in `RenderStepped`. Expensive world enumeration (`GetDescendants`) every 0.75-2 s and inside `pcall`. Position caches at 1 Hz. A bare `task.wait()` spins about 60 kHz in Matcha, so use `task.wait(0.005)` or longer in hot loops.
- **L3. Sort by squared distance and cap the list.** `d2 = dx*dx + dy*dy + dz*dz`, `table.sort(out, function(a, b) return a.d2 < b.d2 end)`, then truncate to `MAX_TARGETS` before firing. `Magnitude` only for display strings.
- **L4. Cache expensive enumeration.** Tree positions rebuilt at 1 Hz, egg positions keyed per `Cycle` attribute, enemy lists rebuilt every 0.75 s. The renderer reads CACHES, never walks `Workspace` itself.
- **L5. Clock choice.** `os.clock()` for cooldowns and deltas, `tick()` for timestamps and freshness checks, `os.time()` inside remote payloads that expect a wall clock.
- **L6. Character guard.** Every loop starts by getting `lp.Character` and `HumanoidRootPart`; if missing, `task.wait(0.5)` and continue. Never assume the character exists.

## CRASH / FREEZE BISECT (concrete recipe for F3)
When the user reports a freeze or crash, do NOT add features and do NOT probe their live game without permission. Bisect, using Hike For Eggs as the model ("Removed EVERYTHING that writes to the character. Only the remote call remains."):
1. Push a version that does ONE thing: the single remote invoke, from wherever the character stands. Remove ALL CFrame writes, tweens, teleports, `WalkToPoint` and input simulation.
2. If it still crashes, movement was never the cause; the remote, the rate or the payload is next. Slow the rate and add the fail-counter pause.
3. If it stops crashing, add movement back ONE piece at a time (walk, then `WalkToPoint`, then a single CFrame write, then teleport).
4. Keep the fail-counter pause (`MAX_FAILS` then `pausedUntil`) permanently, so a bad id or rejected remote can never be hammered in a tight loop.
Say "most likely" until a log line proves a cause. Two failed pushes on the same theory means stop and ask for the log.

## NAMING OBJECTS WHEN PROPERTIES ARE NIL
`ProximityPrompt.ObjectText` and `.ActionText` read as `nil` in Matcha even though they exist in Roblox. Resolve a human-readable name like this:
1. Walk up about 6 parents; at each level look for a `BillboardGui` containing a `TextLabel` and take its `Text` (the floating item label).
2. If a `Model` name is found on the way up, use it.
3. Fall back to the instance's own `Name`.
4. Filter everything through a `GENERIC` blacklist (`Prompt`, `Prompts`, `Interact`, `Item`, `Items`, `Part`, `Model`, `MeshPart`, `Handle`, `Folder`, `Spawn`, `ClickDetector`, ...) and drop all-numeric names.
Keep each prompt as its OWN entry (do not merge by base name) when the user wants one button per discovered item. Cap the button count (about 30) and offer a "Rescan" button.

## `fire*` GLOBALS: GUARD AND WARN ONCE
Check existence inside the function that fires (not only at load), because the Pro build may be updated while the script runs. Warn ONCE per session per global with a flag; never once per frame. A nil here means the user must re-run the Matcha loader (Pro).
```lua
local warnedPrompt = false
local function firePrompt(prompt)
    if type(fireproximityprompt) ~= 'function' then
        if not warnedPrompt then
            warnedPrompt = true
            print('[NAME] fireproximityprompt missing - re-run the Matcha loader')
        end
        return false
    end
    return pcall(fireproximityprompt, prompt)
end
-- click detector, exact args: fireclickdetector(cd, 0, 'MouseClick')
```
Find the prompt instance with `GetDescendants()` and `ClassName == 'ProximityPrompt'` (`IsA` always returns true). Rate-limit with at least `task.wait(0.05)` and respect the game's cooldowns.

## STYLE CONVENTIONS (match these in every new script)
- Header banner comment: what the script does, what was confirmed in game, what is unverified.
- `local NAME`, `local VERSION = 'vX.Y'` (one scheme: `vX.Y`, printed as `[NAME] vX.Y`; the user's scripts drift between 3.0.0, v22 and v1.4, so standardize), kill the previous `_G[NAME]` at the top with no wait, print the version immediately.
- `local running = true`, `local conns = {}`, `local function track(c)` for every connection; `stop()` disconnects all, `:Remove()`s all Drawings, calls `UI.RemoveTab`, then sets `_G[NAME] = nil`.
- Register `_G[NAME] = { stop = stop, version = VERSION }` ONLY after init (not early).
- `setBool(key, v)` / `setNum(key, v)` logging setters instead of raw assignment; the log is the only debugging channel in Matcha.
- `-- ---- section ----` dividers; every print prefixed `[Name]`.
- `pcall` around EVERY game call, including `:FireServer`, `notify` and `UI.*`.
- Record probe results inline as comments: `-- Confirmed in game: ...` and `-- UNVERIFIED: ...`. The user explicitly wants these lines kept.
- Finish with `pcall(notify, 'Name', 'vX.Y loaded', 4)` and a print. (`notify` is documented as `(message, title, duration)`; the order has been inconsistent, so `pcall` it and treat it as cosmetic.)

## SMALL API FACTS CONFIRMED BY THE USER'S SCRIPTS (append to the "Matcha is not Roblox" list)
`CFrame.lookAt` exists; `AssemblyLinearVelocity` read AND write works; `Humanoid.WalkToPoint` works; `workspace:Raycast` + `RaycastParams.new()` + `Enum.RaycastFilterType.Exclude` work; `IsDescendantOf` works; `FindFirstChildOfClass` works; `math.clamp` exists; use `typeof(v) == 'Vector3'` guards; `workspace` and `game:GetService('Workspace')` both work; `string.find(n, x, 1, true)` (plain find) is used; `table.unpack` works; `notify` and `warn` work; Drawing types `Line`, `Square`, `Text` with `.Filled`, `.Thickness`, `.Center`, `.Outline`, `.Transparency`, `.From`/`.To`; `getgc`/`applygc` exist but are slow (10-20 s); `writefile`/`readfile`; `isrbxactive`; `iskeypressed`; the mouse functions and `lp:GetMouse()`; `cam.ViewportSize`; `inst.Address`.

## CONTRADICTIONS SETTLED (do not re-litigate)
1. Section handle: test `sec ~= nil`, never `type(sec) == 'table'`.
2. `UI.GetValue` is not authoritative alone: use the registry plus poller, and trust the getter for a key only once it has answered with a boolean.
3. `_G[NAME]` registers ONLY after init.
4. Button order is `(label, cb)` first, always.
5. `matcha_status` may be absent from the toolset (it errored in one chat): fall back to the place id already in the file's header comment, or ask the user once. Do not invent a place id.
