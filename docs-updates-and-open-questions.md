# Matcha API 5/5: extra documented API, docs-site updates, open and unverified items (brain part)

## Additional Documented API (not covered by the live tests above)

Taken from a second copy of the official Matcha docs. These appear in the docs but were not part of the live global sweep, so confirm with a quick probe before depending on them.

**Globals**
- `getrbxversion(): string` returns the Roblox client version (for example `"0.635.0.6350567"`).
- `gethwid(): string` returns the machine's hardware id. Never write it to a file, log, repo or chat output.
- `rawlen(t: table | string): number` standard raw length (bypasses metamethods).
- `gcinfo(): number` returns the VM's memory use in kilobytes.
- `setmetatable` on a table whose metatable is protected raises `cannot change a protected metatable`; `newproxy(true)` returns a userdata with a settable metatable (so `__index` and `__tostring` work).
- `tick()` is a monotonic clock, not a Unix timestamp; use it for elapsed time. `getscripthash` returns the FNV-1a 64-bit hash as a decimal string. `getgamename()` is the place title, distinct from `game.Name`.

**Extra classes**
- `ValueBase`: `Value` (any). Covers `ObjectValue`, `Color3Value`, `NumberValue`, `IntValue`, `FloatValue`, `BoolValue`, `StringValue`.
- `GuiObject`: `AbsoluteSize` (Vector2), `AbsolutePosition` (Vector2).
- `TextLabel`: `Text` (string).
- `MeshPart`: `TextureId` (string), `MeshId` (string).

**Extra datatypes**
- `UDim.new(scale, offset)` with `.Scale`, `.Offset`.
- `UDim2.new(xScale, xOffset, yScale, yOffset)` or `UDim2.new(x: UDim, y: UDim)`, plus `UDim2.fromOffset(x, y)`, `UDim2.fromScale(x, y)`, with `.X` and `.Y` (UDims).

**Fix notes from the docs (Aug 6 2026):** `decompile` stability and output improved; `GetAttribute`/`SetAttribute` now work; `GuiObject.AbsolutePosition`/`AbsoluteSize` return accurate values; `TextLabel.Text` reads and writes work. `RemoteFunction:InvokeServer` was fixed Sep 11 (Pro). The docs' compatibility table also lists `Stepped` as partial (it receives one argument, `deltaTime`, instead of two). If live behavior disagrees with these notes, trust the live result and record it.

## Docs Site Review (Oct 21 2026): Other New or Corrected Facts
Source: https://huoadf.github.io/matcha-docs/ (agents can also read `llms.txt` / `llms-full.txt` there, or add the docs MCP at `https://huoadf.github.io/matcha-docs/mcp`). When unsure about an API, fetch the docs page instead of guessing.
- **HTTP:** `httpget`/`httppost` never raise. An HTTP error status (for example 404) returns the server's response body (such as "404: Not Found" text), not an empty string; only a connection failure returns `""`. Check for both an empty string and error text. The docs call them the global equivalents of `game:HttpGet`/`game:HttpPost`, and `DataModel:HttpGet(url, headers?)` is listed in the Classes page; the VM-limitations page still says `game:HttpGet` is unsupported, so if it errors use `httpget`.
- **Remote Spy / hybrid (Pro):** `RemoteEvent:FireServer(...)`, `RemoteFunction:InvokeServer(...)` and `:InvokeFunction(...)` are supported in Hybrid mode; `RemoteFunction` logging in Remote Spy was fixed Oct 21 2026.
- **Memory:** `memory_read`/`memory_write` require "Allow Unsafe LuaU" to be enabled; types are `int`, `float`, `double`, `byte`, `string`, `uintptr_t`; reading an unmapped address returns `0`; `string` reads until the first non-printable byte.
- **GC:** `setgc` rescans on every call and must not run in a loop (it lags the client); use `getgc` once and `applygc` for repeated changes, and rerun `getgc` after respawn or weapon change. Both accept a table of keys. `getgc` entries have `key`, `value`, `type`, `addr`.
- **`wait` / `task.wait`:** the docs say they return nothing, `task.wait()` yields until the next frame, and `task.defer`/`task.delay` never run; earlier live tests in this file contradicted these (bare `task.wait()` returned in ~16 us, `defer`/`delay` fired, `wait(N)` returned elapsed time). Trust live results over the docs and re-test.
- **`require(path)`** loads a `.lua`/`.luau` module by path with the extension required, from its own module root (separate from the `writefile` workspace), so you cannot `writefile` a module and `require` it.
- **Classes/properties listed:** `Instance:FindFirstChildOfClass`, `FindFirstChildWhichIsA`, `GetFullName`, `IsDescendantOf`, `GetAttribute`, `GetAttributes`, `SetAttribute`, `WaitForChild`; `Workspace:Raycast(origin, direction, params?)`; `Camera.CFrame`, `FieldOfView`, `ViewportSize`; `Lighting.Ambient`, `ClockTime`, `ExposureCompensation`; `Mouse.X/Y` via `Player:GetMouse()`; `Players.PlayerAdded/PlayerRemoving`; `Player.UserId`, `Team`, `Character`; `Humanoid.MaxHealth/Health`; `Model.PrimaryPart`. `UserInputService.InputBegan/InputEnded` are listed with `(input, gameProcessed)` but one session observed an input table with only `.KeyCode`.
- **Datatypes:** `Ray.new(origin, direction)`; `UDim`/`UDim2`; `Vector3` methods Abs, Angle, Ceil, Cross, Dot, Floor, FuzzyEq, Lerp, Max, Min, Sign; `CFrame.fromOrientation`, `Angles`, `GetComponents`, `Inverse`, `Lerp`, `PointToObjectSpace/WorldSpace`, `VectorToObjectSpace/WorldSpace`, `ToEulerAnglesXYZ`, `ToOrientation`; `CFrame == CFrame` uses reference equality; `Color3.fromHex`.
- **Drawing:** six types (Square, Line, Circle, Text, Triangle, Image); Image has `Data` (raw bytes, for example from `httpget`), `Size`, `Rounding`; Square also has `Corner`.
- **Updating Matcha:** features are delivered by reopening the Matcha loader; Roblox must be on the forced current version. Pro adds Hybrid mode and the MCP server (`http://127.0.0.1:8765/mcp`).

## Open / Unverified
- `UI.GetValue` wiring (see UI Binding section): unverified until tested in-game.
- `WorldToScreen` for a point just outside the viewport rectangle but inside the FOV frustum: only tested with a large (500 stud) offset, which returned `(0,0), false`. Not isolated precisely.

Roles & Versions
Tier	Features
Buyer	Standard access: Matcha executor, drawing API, memory tools, base Lua library.
Pro	Early-access features: Hybrid Mode (Remote Spy, FireServer, InvokeFunction) + native AI Agent / MCP server integration.
FAQ (Quick Answers)
What is Matcha? A Roblox LuaVM script executor that emulates Roblox API functions externally without hooking into internal engine functions.

Is Matcha detectable? No. It operates externally without memory hook signatures on Roblox engine routines.

How do I update Matcha? Reopen the Matcha loader—it automatically downloads and applies the latest version.

Why are features broken in-game? Ensure your Roblox game client is updated to the latest forced version. If Roblox updated recently, force-update or reopen the loader.

Why is loadstring dropping returns? Matcha's loadstring drops top-level return statements. Pass values out via globals (e.g., _G.MyResult = 5).

How do I connect Matcha to Antigravity or Codex? See the MCP Integration section above. Antigravity uses serverUrl in mcp_config.json; Codex uses codex mcp add matcha --url http://127.0.0.1:8765/mcp.

Why is task.defer not working? It is, in live testing: task.defer and task.delay fire, and task.spawn forwards arguments. Older notes calling them no-ops are outdated.

How do I raise a catchable error? Use assert(false, "message"). The error() and errorl() functions only print to the console; they do not halt execution or raise catchable errors.

Changelog (Comprehensive)
Date	Version / Event	Notes
Oct 7, 2026	cec3ad5889b447cf	Updated to latest version; Vault Dev: "I will back at features update in week or sooner"
Oct 4, 2026	—	Fixed Auth. Run loader to update
Oct 3, 2026	—	Decompiler fixed (no restart needed)
Sep 23, 2026	—	Fixed Speed, Inf Jump, Jump, Float
Sep 22, 2026	—	Pro: Fixed RemoteFunction in Remote Spy
Sep 18, 2026	—	Fixed Decompiler
Sep 14, 2026	—	Fixed Lightings
Sep 13, 2026	—	Announced Hybrid features: fireclickdetector, fireproximityprompt, firetouchinterest
Oct 21, 2026	—	Released (Pro): firetouchinterest, fireclickdetector, fireproximityprompt; fixed RemoteFunction in Remote Spy
Sep 11, 2026	—	Pro: Fixed Bloxstrike Support, Fixed InvokeServer, Fixed Unsafe Lua Prompt & Hybrid Prompt
Aug 6, 2026	—	Decompiler stability fixed; Instance comparison (==) fully supported; GetAttribute/SetAttribute fixed
Resources
Matcha Docs: https://huoadf.github.io/matcha-docs

Lua 5.1 Reference: https://www.lua.org/manual/5.1/

MatchaScripts Repository: https://github.com/cconstellation/MatchaScripts

Windows Virtual-Key Codes: https://learn.microsoft.com/en-us/windows/win32/inputdev/virtual-key-codes
