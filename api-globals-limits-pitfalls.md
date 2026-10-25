# Matcha API 3/5: live-tested globals, absent globals, pitfalls, error meanings (brain part)

## Executor Global Support (live-tested on Matcha 1.0.0)

Results below come from live testing on Matcha 1.0.0 (place 135564683255158, Prison Life, hybrid mode). Presence was checked via `_G[n]` with a `loadstring("return "..n)()` fallback. They take precedence over older statements elsewhere in this file where they conflict. Re-verify after Matcha or Roblox updates.

### PRESENT — 49 core globals plus UI and Memory

```
Core:        loadstring, load, decompile, getscriptbytecode,
             getscripthash, getscripts, require, typeof, tick, wait, spawn
GC:          getgc, setgc, applygc
Env:         getfenv, setfenv
Filesystem:  writefile, readfile, appendfile, makefolder, isfile,
             isfolder, listfiles, delfile, delfolder
HTTP:        httpget (binary-safe), httppost
Input:       keypress, keyrelease, mouse1press/release/click,
             mouse2press/release/click, mousemoveabs, mousemoverel,
             mousescroll, iskeypressed, ismouse1pressed, ismouse2pressed,
             setrobloxinput, isrbxactive
Draw:        Drawing (all 6 types), WorldToScreen
UI:          UI.AddTab, UI.RemoveTab, UI.GetValue, UI.SetValue
Memory:      getbase, memory_read, memory_write (memory_write not exercised; wrong address crashes Roblox)
Misc:        identifyexecutor, getgamename, GetPingValue, setclipboard,
             run_secure, notify, base64encode, base64decode,
             setfflag (no-op), getfflag (always 0)
```

### ABSENT — nil globals (any script depending on these will error)

```
Instance:    Instance itself (the constructor; no datatype creation)
Metatable:   hookfunction, hookmetamethod, getrawmetatable,
             setrawmetatable, setreadonly, isreadonly,
             getnamecallmethod, newcclosure, checkcaller,
             islclosure, iscclosure, clonefunction
Environment: getgenv, getrenv, getsenv, getreg, getmenv,
             getupvalue, setupvalue, getupvalues, getconstants
Discovery:   getinstances, getnilinstances, getloadedmodules,
             getmodules, getcustomasset, getallthreads
Thread:      getthreadidentity, setthreadidentity, setidentity, getidentity
Remote:      getconnections, firesignal, getsignal, getcallbackvalue,
             getcallingscript
Network:     request, http_request, httprequest, syn,
             HttpGet, HttpGetAsync, HttpPost
Crypto:      crypt, hash, lz4compress, lz4decompress, md5, sha1, sha256
Scripts:     getfflags, getrunningscripts, getallscripts,
             getloadedscripts, getscriptfromthread,
             getscriptfromfunction, getscriptfromremote,
             register, unregister, getscriptfunction, getscriptclosure
Player:      getplayer, getplayers, getchar, getcharfromplayer,
             getnamefromplayer
GUI:         gethui, getgui, cleardrawcache, isrenderobj,
             getrenderproperty, setrenderproperty
Clipboard:   getclipboard, getexecutorname, messagebox,
             setfpscap, getfpscap, getwindow, getpixel
```

**Consequences:** no hooking means no namecall-based hooks; no `getconnections` means no signal/remote replay. NOTE: the interaction functions `fireproximityprompt`, `fireclickdetector` and `firetouchinterest` were ABSENT in the earlier live tests but were released for Pro on Oct 21 2026. If `type(fireproximityprompt) ~= "function"`, the user needs to rerun the Matcha loader (they already have Pro); do not assume they are missing forever. Matcha's design is "do it with input, drawing and DataModel reads, nothing else." Do not write scripts that depend on any global in the absent list.

### Lua stdlib trim

```
Missing:  io (entire table), collectgarbage,
          task.cancel, task.desynchronize, task.synchronize,
          debug.getupvalue, debug.setupvalue
          (the `debug` table exists but only has `traceback`)
Present:  os, string, table, math, coroutine, bit32, utf8,
          rawget, rawset, rawequal, setmetatable, getmetatable,
          pcall, xpcall, select, newproxy, assert, warn, error
```

### Verified behavior details
- `loadstring("this is not valid")` returns a function; calling it prints `Matcha:1: Incomplete statement...` and returns nil. `pcall` of it gives `ok=true, err=nil`. `loadstring("__g = 42")()` sets `__g`; `loadstring("return 1+1")()` returns nil.
- `getgc("Players")` returned 172 entries and `getgc({"Players","Workspace"})` 388, each shaped `{ key, value, type, addr }` with a numeric `addr`. An unknown name returns 0 entries.
- `getfenv(0)`, `getfenv(1)` and `getfenv(print)` all return tables. `setfenv(function() return X end, {X=5})()` returns 5.
- `httpget` custom headers work. A bad host and an empty-body 404 both return `""` and cannot be told apart.
- `decompile` works on real scripts (a ModuleScript returned 1,163 chars); `getscriptbytecode` returned binary (210 bytes for one script); `getscripthash` returns a FNV-1a 64 hash.
- `tostring(1.5)` is `"1.5"` and `tostring(0.0001)` is `"0.0001"` (standard Lua `%.14g`).
- `Font = 3` or `Font = 6` is a silent no-op.
- `setfflag` writes nothing readable (reads back 0); `getfflag` returns 0 for every key.

### Behavioral corrections vs the doc

These were corrected in place above; the table is a quick summary.

| Item | Live-tested behavior |
|---|---|
| `task.wait()` | Returns in ~0.016ms — NOT a frame yield |
| `task.wait(N)` | Honors N (min resolution ~1ms) |
| `task.wait(N)` return | `1.0` (not "nothing") |
| `wait(N)` return | Elapsed seconds (not "nothing") |
| `task.defer` | FIRES (doc says no-op) |
| `task.delay` | FIRES (doc says no-op) |
| `task.spawn(fn, ...)` | FORWARDS args (doc says it doesn't) |
| `loadstring(src)()` | Top-level return is DROPPED even for `"return 1+1"`; only global assignment survives |
| `base64encode` | Truncates at first `\0` — confirmed 7 -> 1 byte |
| `writefile` | Allowed: `.txt .lua .luau .json .cfg .dat .ini`. Error text: `"Matcha:<line>: file extension not allowed"` |
| `Drawing.Fonts` | UI=0, System=1, SystemBold=2, Minecraft=4, Monospace=5, Pixel=7, Fortnite=8, ProximaSoftBold=11 (indices 3, 6, 9, 10 are holes) |
| `WorldToScreen` | Returns `(0,0), false` for behind-camera AND far off-axis |
| `error(msg)` | Continues execution; MCP wrapper flags `ok:false` |
| `pcall(loadstring(bad))` | Returns `true, nil`; invalid chunks cannot be detected programmatically |
| `httpget` | Binary-safe |
| `os.clock()` | Wall time (delta across `task.wait(0.5)` = 0.5023); safe for deltas |
| `tostring(number)` | Standard Lua (`1.0` -> `"1"`); only datatypes are padded |

## Roblox-General Pitfalls (not Matcha-specific, but hit constantly)

### R6 vs R15 part names
R6 uses names with spaces: `"Right Arm"`, `"Left Arm"`, `"Right Leg"`, `"Left Leg"`. R15 has no spaces: `RightUpperArm`, `RightLowerArm`, `RightHand`, etc. Enumerate both or code silently fails on one rig type.

### Rig-agnostic directions
- Arm axis: `(Hand.Position - UpperArm.Position).Unit` works on both rigs regardless of CFrame rotation.
- Character right: `HumanoidRootPart.CFrame.RightVector` is the only stable one; head, torso and arm CFrames rotate with their own look/twist/swing.
- Aim direction: `Camera.CFrame.LookVector`, not character facing (third-person games decouple them).

### R6 arm LookVector
In R6, an arm's `CFrame.LookVector` does not point down the arm; it points outward or backward depending on pose. Anchoring a point to it lands behind and to the side. Use the shoulder-minus-hand axis instead.

## Transparency, Raycast and Visibility-Check Patterns

### LocalTransparencyModifier
Many games hide blockers by setting `LocalTransparencyModifier = 1` while leaving `Transparency = 0`. A see-through check reading only `Transparency` misses these. Read both, each wrapped in `pcall`:

```lua
local tr, ltm
pcall(function() tr = inst.Transparency end)
pcall(function() ltm = inst.LocalTransparencyModifier end)
if (tr and tr >= 0.95) or (ltm and ltm >= 0.8) then return true end
```

### Tinted glass
Tinted car windows sit at 0.2-0.6 transparency, so an 0.8 threshold misses them. If the part name or material looks like glass, use a threshold of 0.15 and treat it as transparent. Name keywords: glass, window, windshield, windscreen, visor, pane, tint, carwin, screen, clear.

### CanQuery, CanCollide and raycasts
- `CanQuery = false`: default `Workspace:Raycast` skips the part.
- `CanCollide = false`: raycasts still hit it unless `RespectCanCollide = true`.
- Multi-bounce walks need a bounce cap (5) and an epsilon (0.1 studs) between hits, or they loop forever or tunnel through thin geometry.

### Passing through other players
Pre-index every character's BaseParts into a flat `[Instance] = true` set once per collector tick, so "is this part on another player" is an O(1) lookup instead of an 8-hop parent walk per bounce. Walking `.Parent` also fails silently if a part is temporarily unparented during ragdoll/respawn.

## Property and Tree Levers (observed by direct test)

Writes to Roblox properties are not a faithful emulation of Studio. Observed behavior in a live session; re-test on your build before relying on any of it.

| Lever | Result | Note |
|---|---|---|
| `inst.Parent = nil` | Errors | `invalid argument #3 (Instance expected, got nil)` |
| `inst.Parent = <service or instance>` | Works and holds | The real way to remove something from its current place in the tree |
| `BasePart.CanTouch` write | Unknown property | Do not rely on it |
| `Humanoid.Sit` read | nil | Silent fallback, not an error |
| `Humanoid.Sit` write | Unknown property | |
| `Humanoid.Health` write | Fails | Readable, not writable |
| `Humanoid.WalkSpeed` / `JumpPower` write | Errors | |
| `inst.Address` read | Works | The reliable identity key |

Size and CanCollide writes on BaseParts worked in-place in the tested scenario. Property availability varies by class and build, so probe with `pcall` and read the value back after a wait.

## Instance Identity and Stale Data

- **Instance comparison.** The official docs (updated Aug 6 2026) say `instA == instB` is fully supported and `.Address` no longer has to be compared manually. One earlier session still saw equality fail on stale wrappers after reparenting, so treat `==` as supported but, for stale or reparented objects and for table keys, key by `inst.Address` (or `Player.UserId` for players) as the safe fallback.
- **Stale `GetDescendants`/`GetChildren` results.** After reparenting, a previously fetched list (or a count from it) can contain dead references, so a fresh count can be wrong. Confirm a reparent by comparing `.Address` sets before and after. Same addresses still out of the tree means the objects stayed moved; do not conclude the game "rebuilt" them from a count.
- **Prove a lever over a wait window.** Acting on an immediate read is the most common source of wrong conclusions. Apply the change, wait (for example 0.5-2 s), then re-read by `.Address` before building on it.

## Replication and Network Ownership Caution
Reparenting or moving a part that is under another client's or the server's network ownership can break replication; the object may snap back or desync when ownership resolves. If a script desyncs, first look for a `Parent` write or a `Position` write on a physics-simulated part. Prefer in-place property changes (such as `Size`, `CanCollide`) over reparenting or repositioning, and verify them over a wait window.

## Confirmed Absent (stop probing these)
- No `Instance` at all (not even `Instance.new`); every overlay must be built with `Drawing`.
- No hooking: `getrawmetatable`, `hookfunction`, `hookmetamethod`, `newcclosure`, `checkcaller` are all nil.
- `debug` exists but only has `traceback`.
- No `getgenv`/`getrenv`/`getinstances`, and no collision-group or physics-control API. (Touch, click-detector and proximity-prompt firing now exist for Pro: see F9.)
- Memory is only `getbase`, `memory_read`, `memory_write`: no scan, no AOB scan.
- A probe of ~150 candidate names found nothing undocumented beyond what this file lists. Treat this file as the ceiling of what the executor offers.

## VM Quirks That Break Ported Scripts
- `IsA()` ignores its argument and returns true for everything. Filters like "is this a Tool" silently accept any child. Compare `ClassName` instead.
- `loadstring` drops top-level returns, so the common `pcall(chunk)` / `local ok, result = pcall(loadstring(src))` loader pattern yields nil. Have the chunk assign to `_G` and read it afterward.
- `UserInputService.InputBegan` passes an input table with only `.KeyCode` (integer): no `gameProcessed` and no `UserInputType`.
- `notify(message, title, duration)` is the correct argument order.

## Testing and Probing Pitfalls
- **Existence vs behavior.** Test that a global exists (`type(x) ~= "nil"`) separately from testing what it does. A round-trip test can fall back to `_G` and pass even when the function under test (for example `getgenv`) is missing.
- **Harness note bug.** A pattern like `return condition, "mismatch"` returns the failure text whether the check passed or failed. Print notes only on failure, or a pass can look like a failure (99 passes once looked like 99 failures).
- **Cross-run contamination.** Output from a script still running from an earlier exec can pollute a later test. Stop or confirm earlier scripts finished before measuring.
- **Check presence two ways.** Use `_G[name]` and a `loadstring("return "..name)()` fallback, then record both results.

## Signals and Missing Methods (observed errors)

| Call | Result |
|---|---|
| `RunService.Heartbeat:Wait()` | Error: `attempt to call missing method 'Wait' of table` |
| `RunService.Heartbeat:Connect(fn)` | Works |
| `RunService.RenderStepped:Connect(fn)` | Works; the place `WorldToScreen` is reliable |
| `Model:GetPivot()` | Error: missing method of Instance |
| `Instance:HasTag()` | Error: missing method of Instance |
| `Instance:IsA(class)` | Silently returns true for everything; use `ClassName == "Foo"` |
| `CFrame.new(Vector3)` | Error: `CFrame.new expects 0, 3, 6, or 12 arguments` |

Rules: never `:Wait()` on any signal. To find a child use `FindFirstChild` by name; to detect a BasePart check `.Position ~= nil and .Size ~= nil`. A `CFrame.new` error inside a `pcall` in a loop can kill that thread silently, so validate argument shapes.

## Recognise These Exact Errors

| Error text | Meaning |
|---|---|
| `attempt to call missing method 'Wait' of table` | Signals have no `:Wait()`; use `:Connect` |
| `attempt to call missing method 'GetPivot' of Instance` | `GetPivot` does not exist |
| `attempt to call missing method 'HasTag' of Instance` | `HasTag` does not exist |
| `CFrame.new expects 0, 3, 6, or 12 arguments` | A Vector3 was passed; use `CFrame.new(x, y, z)` |
| `invalid argument #3 (string expected, got boolean)` | A widget call has its arguments in the wrong order; use `sec:Toggle(id, label, default, callback)` (id first, then the label string, then the boolean) |
| `invalid argument #3 (Instance expected, got nil)` | `Parent = nil` is not allowed; reparent to a real instance |
| `attempt to index nil with 'new'` | `Instance` is nil; nothing is constructible |
| `Matcha:<line>: file extension not allowed` | `writefile` rejected the extension |
| `attempt to index nil` | A link in your chain is missing; print each step |
| `attempt to call a nil value` | That global or function does not exist in this VM |

## Output Capture and Timing in the MCP Exec Tool
- `matcha_exec` returns only what `print()` emitted, and capture closes about 350 ms after the last print. Print incrementally; do not buffer a final report. Check `matcha_logs` for anything a spawned loop emits later.
- A top-level `task.wait` in an exec does not yield the way it does in a normal script; spawn a loop and read logs instead.
- One batched exec with many prints beats many single-print calls.
