# Matcha API 1/5: overview, MCP setup, function reference (brain part)

# Matcha LuaVM - The Complete Documentation

## Overview
Matcha is a Roblox LuaVM script executor that emulates Roblox API functions externally rather than hooking into internal engine routines. This design makes it 100% undetected by server-side and client-side anti-cheat mechanisms, because it operates without memory hook signatures on Roblox engine routines. 

The trade-off is that Matcha has no direct access to internal Roblox API functions—it cannot use `Instance.new()` (the `Instance` global is nil: no Roblox instance can be constructed from the executor, everything must be read from the live DataModel or built from primitives like `Vector3.new`/`CFrame.new`/`Color3.new`), `game:HttpGet()`, or `game:HttpPost()` directly. Instead, it provides native global replacements and a rich set of emulated functions.

**Live Status (docs updated Oct 21 2026):** The Pro Hybrid interaction functions `firetouchinterest`, `fireclickdetector` and `fireproximityprompt` are now released (Oct 21 2026; Pro, run the loader to update). `RemoteFunction` support in Remote Spy was fixed the same day. Newer Matcha builds also fixed the decompiler (Oct 3) and lighting/speed/jump features (Sep 23).
---

## MCP Server Integration (Pro)
Matcha Pro includes native Model Context Protocol (MCP) support for seamless integration with AI coding assistants.

### Server URL
http://127.0.0.1:8765/mcp

text
The server binds to localhost only. The AI client must run on the same machine as Matcha.

### Configuration by Client

**Antigravity:**
1. Navigate to `C:\Users\<YourUser>\.gemini\antigravity\` and open `mcp_config.json`.
2. Add:
```json
{
  "mcpServers": {
    "matcha": {
      "serverUrl": "http://127.0.0.1:8765/mcp"
    }
  }
}
```

Antigravity requires the field name `serverUrl` according to this source document.

**Codex:**

```bash
codex mcp add matcha --url http://127.0.0.1:8765/mcp
```
Cline:

Open the Cline panel → MCP Servers icon.

Open the Remote Servers tab.

Click Add Server and enter:

Name: Matcha

Server URL: http://127.0.0.1:8765/mcp

Transport: Streamable HTTP

Click Add Server.

ChatGPT:

Open Settings → Apps & Connectors.

Enable Developer mode under Advanced settings.

Click Create, name it (e.g., Matcha), and paste http://127.0.0.1:8765/mcp.

Choose No authentication and check "I trust this application".

Start a new chat and enable it via the + menu.

Claude Desktop / Claude Code:
Add to `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "matcha": {
      "url": "http://127.0.0.1:8765/mcp"
    }
  }
}
```

**Generic MCP client:**

```json
{
  "mcpServers": {
    "matcha": {
      "url": "http://127.0.0.1:8765/mcp"
    }
  }
}
```

**Health check:**

```bash
curl http://127.0.0.1:8765/health
```
MCP Troubleshooting

| Problem | Solution |
|---|---|
| Client cannot connect | Verify Matcha is running and listening on port 8765. Restart the Matcha loader. |
| Features broken in-game | Force-update the Roblox game client. Matcha may break after Roblox updates. |
| `loadstring` drops returns | Assign results to globals, for example `_G.MyResult = 5`. |
| Connection lost after restart | Restart both Matcha and the AI client to clear stale duplicate connections. |
| `serverUrl` vs `url` mismatch | This source says Antigravity requires `serverUrl`; Claude Desktop/Cursor/Codex use `url`; Windsurf accepts both aliases. Client requirements can change, so verify the client’s current schema. |

**Configuration troubleshooting note:** If the client rejects the MCP configuration, check its current configuration schema and try the key it expects, such as `url` or `serverUrl`. The original source claims Antigravity requires `serverUrl` and most other clients use `url`; treat this as client/version-specific rather than universal.
Function Reference
Globals
Function	Signature	Description
loadstring	(chunk: string, chunkname?: string): function	Compiles chunk. Syntax errors still return a function, and the error is NOT catchable: `pcall(f)` on an invalid chunk returns `true, nil` and only the console shows the error, so there is no programmatic way to detect a bad chunk. Top-level returns are dropped—pass results via globals.
load	(chunk: string, chunkname?: string): function	Equivalent to loadstring for string chunks.
decompile	(script: Instance): string	Returns decompiled source of a LocalScript, ModuleScript, or Script. Fixed Aug 6, 2026.
WorldToScreen	(position: Vector3): (Vector2, boolean)	Projects 3D world position to 2D screen space. `(0,0), false` is the universal off-screen sentinel: returned for behind-camera AND far off-axis (e.g. 500 studs off-axis) positions alike. On-axis returns correct viewport coords with `true`. Call it inside a `RenderStepped` callback for ESP: outside a render callback it can return `(0,0), false`, a silent wrong answer rather than an error.
notify	(message: string, title: string, duration: number)	Shows a Matcha notification. This order was confirmed in one session, but another review reported inconsistent argument order across sources; wrap in `pcall` and treat it as cosmetic.
identifyexecutor	(): (string, string)	Returns ("Matcha", version).
getscripthash	(script: Instance): string	FNV-1a 64-bit hash of bytecode.
getgamename	(): string	Returns place title (e.g., "Baseplate").
getscripts	(): { Instance }	Returns all loaded script instances.
getscriptbytecode	(script: Instance): string	Raw compiled bytecode as binary string.
base64encode	(data: string): string	Encodes to Base64. Truncates at first null byte (\0)—destroys binary: `base64encode("a\0b\0c")` returns `"YQ=="` (base64 of `"a"` alone); a 7-byte input with nulls round-trips as 1 byte. Do not pass binary through base64; use raw `httpget` bytes.
base64decode	(data: string): string	Decodes Base64. base64decode("") returns "".
GetPingValue	(): number	Current ping in milliseconds.
tick	(): number	Monotonic high-precision time (not Unix timestamp).
typeof	(value: any): string	Returns specific datatype name (Vector3, CFrame, Instance, Drawing, …) instead of "userdata".
getfenv	(target: function | number): table	Returns environment table. No arg / 0 / 1 returns global environment (_G).
setfenv	(f: function, env: table): function	Sets the environment of function f.
Console & Input
Function	Description
print(...) / printl(...)	Prints to console, space-separated.
warn(...)	Yellow warning-styled line.
error(...) / errorl(...)	Red error-styled line. Does NOT halt execution—only prints. Use assert(false, message) for a catchable error. Via the MCP `matcha_exec` wrapper, any `error()` call flags the whole call `ok:false` even if the script ran to completion.
setrobloxinput(state: boolean)	Toggles whether Matcha's injected input reaches the game.
isrbxactive(): boolean	True while Roblox window is foreground.
setclipboard(value: string)	Copies value to system clipboard.
keypress(keycode: int)	Sends key-down (Windows VK code, e.g., 0x41 = A).
keyrelease(keycode: int)	Sends key-up.
iskeypressed(keycode: int): boolean	True while key is held.
ismouse1pressed() / ismouse2pressed()	Left/right mouse button held.
mouse1press() / mouse1release() / mouse1click()	Left mouse control.
mouse2press() / mouse2release() / mouse2click()	Right mouse control.
mousemoveabs(x, y)	Moves cursor to absolute pixel position in Roblox viewport space.
mousemoverel(x, y)	Moves cursor by relative offset.
mousescroll(amount: int)	Scrolls mouse wheel; sign sets direction.
Miscellaneous & Scheduler
Function	Description
run_secure(text: string)	Runs base64-encoded protected payload. Not usable with ordinary scripts—protection tooling is not public.
setfflag(FFlag, value)	Intended to set fast flag. Incomplete—no effect that getfflag can read.
getfflag(FFlag): number	Stub—returns 0 for every flag.
require(path: string): table (NOT Roblox `require`: `require(<Instance>)` does not return the module; to read a game module use `decompile(instance)` or `getgc`)	Loads .lua/.luau module by path. Extension is required and validated. Resolves from its own module root, separate from writefile/readfile workspace.
wait(time: number)	Yields current thread. Returns elapsed seconds like Roblox's wait (e.g. `wait(0.05)` returns ~0.0505).
spawn(target: function)	Runs target on a new thread immediately.
task.spawn(target: function, ...)	Runs target immediately (synchronously). Forwards extra arguments (verified: `task.spawn(fn, 1, 2)` receives 1, 2).
task.defer(target: function, ...)	Fires (verified). Not a no-op.
task.delay(time, target)	Fires after time (verified with `task.delay(0.1, ...)`). Not a no-op.
task.wait(time?: number)	**Not a frame yield.** With no argument it returns almost immediately (~16 µs per call, a busy yield at ~60 kHz, not synced to the render loop). With N it honors N (min resolution ~1 ms: `task.wait(0.001)` ≈ 1.06 ms, `task.wait(0.2)` ≈ 200 ms). Returns `1.0` regardless of N—NOT elapsed time, so `local dt = task.wait(N)` gives nonsense. For a real ~1 ms yield use `task.wait(0.001)`; for true frame sync use `RunService.Heartbeat:Connect(fn)`.
