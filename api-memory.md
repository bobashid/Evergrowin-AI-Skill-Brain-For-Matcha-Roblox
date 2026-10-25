# Matcha API 2/5: memory functions (brain part)

## Memory

Requires unsafe LuaU execution to be allowed.

| Function | Description |
|----------|-------------|
| getbase(): int | Returns base address of RobloxPlayerBeta.exe module. Functional (verified). |
| memory_write(memoryType, address, value) | Writes value to address as type (int, float, double, byte, string, uintptr_t). Wrong address can crash Roblox. |
| memory_read(memoryType, address): Any | Reads value at address. Functional (verified: `memory_read("int", getbase())` returned 9460301 = 0x00905A4D, the DOS `MZ` header magic). Reading unmapped address safely returns 0. String type reads until first non-printable byte. |

Memory Types: int → int, float → number, double → number, byte → int, string → string, uintptr_t → int.

Garbage Collector
Function	Description
getgc(name: string): { table } / getgc(names: { string }): { table }	Returns every GC value stored under name. Result entries: { key, value, type, addr }.
setgc(key: string, value: Any): int / setgc(values: { [string]: Any }): int	Sets every GC value stored under key. Scans on every call—do not run in loops.
applygc(cache: table, key: string, value: Any): int / applygc(cache: table, values: { [string]: Any }): int	Same as setgc but applies to a cache from getgc. Fast—safe for loops.
GC Modding Guide Summary:

Method 1 (Recommended): getgc once → cache → applygc in loops. Perfect for values the game resets (ammo, recoil).

Method 2 (One-offs): setgc for single modifications. Do not run in loops.

**GC edits only change the CLIENT copy.** `applygc`/`setgc` write into Luau tables in this client. Values the server validates (damage, cooldown, reach, fire rate, movement speed) will not change in effect, because the server applies its own number when the remote fires. Do not offer GC tweaks for combat or other server-validated values unless the user can show the value is client-trusted, and say so up front. `applygc` is broad (it writes the same value to every table carrying the key, not just the one you care about) and there is no automatic restore; respawn or rejoin to reset.

Practical example: For games that rebuild configuration values on respawn, the source recommends refreshing the `getgc` cache and then using `applygc`. The exact key and value depend on the target table and are not specified here.

**Example — cache first, then apply a known key/value:**

```lua
local cache = getgc("KNOWN_KEY") -- replace with a documented target key
-- Inspect/confirm the target before changing anything.
applygc(cache, "KNOWN_KEY", desiredValue)
```

`KNOWN_KEY` and `desiredValue` are placeholders, not literal API values. Memory/GC modification can destabilize the game; use only when the target is understood.

File System
Paths are relative to the Matcha workspace (e.g., C:\matcha\workspace\). / and \ are interchangeable; paths are case-insensitive. Allowed extensions: .txt .lua .luau .json .cfg .dat .ini. Others (.bin, .png, .exe) are rejected with the exact error text `Matcha:<line>: file extension not allowed`—pattern-match on that to handle failures.

Function	Description
writefile(path, content)	Creates/overwrites. Missing parent folders created automatically. Content is binary-safe.
readfile(path): string	Returns contents (binary-safe). Errors if file doesn't exist—guard with isfile or pcall.
appendfile(path, content)	Appends to end. No separator inserted.
makefolder(path)	Creates folder including parents. No-op if exists. Errors if file occupies path.
isfile(path): boolean	True only for existing files.
isfolder(path): boolean	True only for existing folders.
listfiles(path): { string }	Immediate children (not recursive), sorted alphabetically. listfiles("") lists workspace root.
delfile(path): boolean	Deletes file or empty folder. Errors for non-empty folders—use delfolder.
delfolder(path): boolean	Recursively deletes folder and contents. Always returns true.
HTTP
httpget and httppost are synchronous—they block until response. Responses are binary-safe (a real PNG was fetched intact: `89 50 4E 47` magic, 8090 bytes), so they are the safe transport for binary payloads.  Never raise errors: HTTP error status returns server response body; connection failure returns empty string.

Function	Signature
httpget	(url: string, headers?: table): string
httppost	(url: string, body: string, contentType?: string, headers?: table): string — contentType defaults to "application/json".
Full Lua 5.1 Base Library Reference
Matcha runs on a Luau-based VM. Standard Lua(U) globals are available per Lua 5.1 reference.

Math Library
math.abs, math.acos, math.asin, math.atan, math.atan2, math.ceil, math.cos, math.cosh, math.deg, math.exp, math.floor, math.fmod, math.frexp, math.huge, math.ldexp, math.log, math.log10, math.max, math.min, math.modf, math.pi, math.pow, math.rad, math.random, math.randomseed, math.sin, math.sinh, math.sqrt, math.tan, math.tanh.

String Library
string.byte, string.char, string.dump, string.find, string.format, string.gmatch, string.gsub, string.len, string.lower, string.match, string.rep, string.reverse, string.sub, string.upper.

Table Library
table.concat, table.insert, table.maxn, table.remove, table.sort.

OS Library
os.clock, os.date, os.difftime, os.execute, os.exit, os.getenv, os.remove, os.rename, os.setlocale, os.time, os.tmpname.

IO Library
The entire `io` table is absent (nil) in Matcha 1.0.0 despite being part of Lua 5.1. Use `readfile`/`writefile`/`appendfile` instead. `collectgarbage` is also missing.

Key Matcha-specific differences from Standard Lua
Function	Difference
type	Roblox datatypes and Instances report as "userdata"—use typeof for specific name.
tostring	Only Roblox datatypes get fixed 4-decimal formatting: `tostring(Vector3.new(1,2,3))` returns `"Vector3(1.0000, 2.0000, 3.0000)"`. Plain numbers follow standard Lua: `tostring(1.0)` returns `"1"`, `tostring(1/3)` returns `"0.3333333333333333"`.
tonumber	Trims whitespace, parses 0x hex, accepts base (2–36).
pairs	Does not honor Luau __iter metamethod—always iterates raw table.
unpack	Exposed as global (mainline Luau only provides table.unpack).
assert	Raises catchable error (Matcha: or assertion failed!)—the way to raise a catchable error in Matcha.
pcall	Returns false, "Matcha: ..." on runtime fault.
newproxy	Lua 5.1 holdover, fully supported.
Drawing API
Function	Description
Drawing.new(drawingType: string): DrawingObject	Types: Square, Line, Circle, Text, Triangle, Image.
DrawingObject:Remove()	Removes the drawing.
Shared properties: Color (Color3), Transparency (number), Visible (bool), Position (Vector2), ZIndex (int).
Square: Size (Vector2), Filled (bool).
Line: From (Vector2), To (Vector2), Thickness (int).
Circle: Radius (float), NumSides (int), Thickness (int).
Text: Text (string), Outline (bool), Center (bool), Font, FontSize (int).
Triangle: PointA, PointB, PointC (Vector2).
Image: Data (string, raw binary), Size (Vector2, default 50x50), Rounding (number, default 0).

Fonts (index map; use `Drawing.Fonts.<Name>` or the number): UI=0 (ProggyClean), System=1 (San Francisco), SystemBold=2, Minecraft=4, Monospace=5 (JetBrains), Pixel=7, Fortnite=8, ProximaSoftBold=11. Indices 3, 6, 9, 10 do not exist (count = 8); `Font = 3` fails silently.

**Example — draw a red line:**

```lua
local line = Drawing.new("Line")
line.From = Vector2.new(100, 100)
line.To = Vector2.new(200, 200)
line.Color = Color3.fromRGB(255, 0, 0)
line.Visible = true
```

UI Binding (Menu Binding)
Create custom menu tabs, widgets, sections and hotkey keybinds directly from Lua. Source: MatchaScripts Repository (github.com/cconstellation/MatchaScripts). Signatures below are from the current official docs (checked Oct 21 2026). **Argument order: `id` first, then the label.**

```lua
UI.AddTab("My Script", function(tab)
    local sec = tab:Section("Settings", "Left")
    sec:Toggle("enabled", "Enabled")
    sec:Keybind("enabled_kb", 0x46, "hold")
    sec:SliderInt("range", "Range", 1, 5000, 2000)
end)
```

Global functions:
- `UI.AddTab(name: string, fn: function)` creates a custom tab. `fn(tab)` runs every frame to render and update the tab.
- `UI.RemoveTab(name: string)` removes a tab.
- `UI.GetValue(id: string): any` reads a widget's current value from anywhere in the script.
- `UI.SetValue(id: string, value: any)` sets a widget's value programmatically.

Sections: `tab:Section(name, side)` where `side` is `"Left"` or `"Right"` and is REQUIRED (a section without a side leaves the tab empty). Pass an array of page names as a third argument for a tabbed section (check `sec.page` to render widgets per page: `0`, `1`, ...); pass `max_height` as a fourth argument for a scrollable section. A section closes automatically when a new section begins.

```lua
local sec = tab:Section("Aimbot", "Left", {"Main", "Advanced"}, 400)
if sec.page == 0 then
    -- Main page widgets
elseif sec.page == 1 then
    -- Advanced page widgets
end
```

Every widget method returns an object: `w.value`, `w:GetValue()`, `w:SetValue(v)`. All widgets accept an optional `callback` as the last argument.

| Widget | Signature | Callback args |
|---|---|---|
| Toggle | `sec:Toggle(id, label [, default: boolean] [, callback])` | `state` |
| Keybind | `sec:Keybind(id [, key: VK number or Enum.KeyCode] [, type: "toggle"/"hold"/"always"/"click"])` must be placed directly after a Toggle | none |
| SliderInt | `sec:SliderInt(id, label, min, max [, default] [, callback])` | `value` |
| SliderFloat | `sec:SliderFloat(id, label, min, max [, default] [, format: "%.1f"] [, callback])` | `value` |
| Combo (dropdown) | `sec:Combo(id, label, items: table [, default: 0-based index] [, callback])` | `index, text` |
| Button | `sec:Button(label [, callback])` or `sec:Button(label, width, height [, callback])` | none |
| ColorPicker | `sec:ColorPicker(id [, r, g, b, a] [, callback])` must follow a Toggle; values 0-1 | `Color3, alpha` |
| ColorPicker2 | `sec:ColorPicker2(id1, {r,g,b,a}, id2, {r,g,b,a} [, callback])` must follow a Toggle | `c1, a1, c2, a2` |
| InputText | `sec:InputText(id, label [, default] [, callback])` | `text` (fires when the field loses focus) |
| Text / Tip / Spacing | `sec:Text("label")`, `sec:Tip("tooltip for previous widget")`, `sec:Spacing()` | n/a |

Keybind object: `.value`/`:IsEnabled()` (active now), `:GetKey()`, `:SetKey(vk)`, `:GetKeyName()`, `:GetType()`, `:SetType(typeStr)`, `:AddToHotkey(label, toggle_id)` (shows in the on-screen hotkey overlay while the toggle is on), `:RemoveFromHotkey()`. Key codes: VK `0x01` LMB, `0x02` RMB, `0x04` MMB, `0x41`-`0x5A` A-Z; or `Enum.KeyCode.MouseButton1`/`A`-`Z`. Combo object: `c:Add(item)`, `c:Remove(item)`, `c:Clear()`, `c:GetItems()`, `c:GetText()`, `c:SetValue(index)`.

Reading values in a loop (the common pattern):
```lua
while true do
    if UI.GetValue("aim_on") then
        local fov = UI.GetValue("aim_fov")
        -- logic here
    end
    wait()
end
```

Example with callbacks:
```lua
UI.AddTab("Demo", function(tab)
    local sec = tab:Section("Objects", "Left")
    local t = sec:Toggle("demo_t", "Toggle", false, function(state) print("toggle", state) end)
    local s = sec:SliderInt("demo_s", "Slider", 0, 100, 50)
    local c = sec:Combo("demo_c", "Combo", {"A", "B", "C"}, 0)
    sec:Button("Reset", function()
        UI.SetValue("demo_t", false)
        UI.SetValue("demo_s", 50)
    end)
end)
```

**Matcha UI rules that prevent crashes and blank menus:** the tab callback runs every frame, so create widgets in a way that is safe to repeat (build once behind a flag, or rely on the widget ids), wrap the body in `pcall`, and keep widget counts reasonable. See "Menu Binding: Practical Rules" below. A previous AI session observed a different argument order (extra string before the boolean) and callbacks that passed unexpected arguments; the official docs above are authoritative, so if a call errors, read the exact error text and fix the call rather than guessing variants.

Roblox Classes Reference
Matcha emulates a subset of Roblox classes. These are accessed via game:GetService() or through Workspace, Players, etc.

Class	Properties / Methods
BasePart	Size (Vector3), Position (Vector3), Transparency (number), Color (Color3), Velocity (Vector3), AssemblyLinearVelocity (Vector3), CanCollide (bool), CFrame (CFrame), Anchored (bool), Material (Enum).
Camera	lookAt(at: Vector3, lookAt: Vector3), ViewportSize (Vector2), FieldOfView (number), Position (Vector3), CFrame (CFrame), Focus (CFrame).
DataModel	PlaceId, GameId, JobId, HttpGet, HttpPost, GetService(name), IsLoaded(): boolean, FindService(name).
HttpService	JSONEncode(tbl), JSONDecode(str), GenerateGUID().
Humanoid	MaxHealth (number), Health (number), WalkSpeed (number), JumpPower (number), MoveTo(position: Vector3, part?: BasePart), TakeDamage(amount: number). Observed: `Health` is readable but writing it fails; writing `WalkSpeed`/`JumpPower` errors; `Sit` reads nil and writing it errors with Unknown property. Treat humanoid state as read-only unless a test proves otherwise.
Model	PrimaryPart (BasePart), GetChildren(), FindFirstChild(name), WaitForChild(name).
Workspace	CurrentCamera (Camera), Raycast(origin, direction, params?): RaycastResult?, Gravity (number), FallenPartsDestroyHeight (number).
Lighting	Ambient (Color3), ClockTime (number), Brightness (number), GlobalShadows (bool), GetSunDirection().
Players	LocalPlayer (Player), GetPlayers(), GetPlayerFromCharacter(character).
RunService	IsClient(), IsServer(), IsStudio(), Heartbeat, RenderStepped, Stepped.
UserInputService	IsKeyDown(keyCode), GetMouseLocation(), MouseBehavior.
TweenService	Create(instance, info, propertyTable).
TeleportService	Teleport(placeId, player, teleportData).
MarketplaceService	PromptPurchase(player, assetId).
Datatypes Reference
Vector3
Constructors: Vector3.new(x,y,z), zero, one, xAxis, yAxis, zAxis.

Properties: X, Y, Z, Magnitude, Unit.

Operations: +, -, *, / (vector-vector and vector-number), unary minus, ==.

Methods: Abs, Angle, Ceil, Cross, Dot, Floor, FuzzyEq, Lerp, Max, Min, Sign.

Vector2
Constructors: Vector2.new(x,y), zero, one.

Properties: X, Y, Magnitude, Unit.

Operations: +, -, *, / (vector-vector and vector-number), unary minus, ==.

Methods: Abs, Angle, Ceil, Cross, Dot, Floor, FuzzyEq, Lerp, Max, Min, Sign.

CFrame
Constructors: CFrame.new(), CFrame.new(x,y,z), CFrame.Angles(rx,ry,rz), CFrame.fromEulerAnglesXYZ(rx,ry,rz), CFrame.lookAt(at, lookAt, up?).

NOTE: `CFrame.new(Vector3)` is NOT supported here (it errors with `CFrame.new expects 0, 3, 6, or 12 arguments`). Always pass three numbers: `CFrame.new(x, y, z)`.

Properties: X, Y, Z, LookVector, RightVector, UpVector, Position.

Operations: * (CFrame * CFrame, CFrame * Vector3), + (CFrame + Vector3), - (CFrame - Vector3), ==.

Methods: Inverse, Lerp, ToWorldSpace, ToObjectSpace, ToEulerAnglesXYZ, ToOrientation.

Color3
Constructors: Color3.new(r,g,b), Color3.fromRGB(r,g,b), Color3.fromHSV(h,s,v), Color3.fromHex(hex).

Properties: R, G, B.

Methods: Lerp, ToHSV, ToHex.

Instance
Properties: Name, Parent, ClassName, Archivable, Address (readable identity; see Instance Identity below). Setting `Parent = nil` errors (`invalid argument #3 (Instance expected, got nil)`); reparenting to a real service/instance works and holds.

Methods: FindFirstChild(name), WaitForChild(name), GetChildren(), GetDescendants(), IsA(className), Destroy(), Clone(). Observed missing: `Model:GetPivot()` and `Instance:HasTag()` error with `attempt to call missing method`; `Clone`/`Destroy` and instance events such as `.Touched` were reported as non-functional in practice, so do not build on them. Observed: `IsA()` ignores its argument and returns true for everything, so use `inst.ClassName == "Tool"` (etc.) instead. `Address` (readable) is the reliable identity key.

Events: Changed, ChildAdded, ChildRemoved, DescendantAdded, DescendantRemoving.

Drawing
See Drawing API section above.

VM Limitations & API Differences
Matcha is an external emulator—it does not inject into or hook Roblox engine memory functions.

Feature / Pattern	Supported?	Replacement / Workaround
Instance.new()	❌ No	`Instance` is nil (`Instance.new("Part")` errors with `attempt to index nil with 'new'`). Read from the live DataModel instead. Vector3/CFrame/Color3 constructors still exist
game:HttpGet() / game:HttpPost()	❌ No	Use httpget(url) / httppost(url, data, contentType)
loadstring()	⚠️ Partial	Top-level returns drop; syntax errors are uncatchable by pcall and print only on call. Pass results via globals
RunService.Stepped	⚠️ Partial	Receives 1 arg (deltaTime) instead of 2 (time, deltaTime)
base64encode()	⚠️ Partial	Truncates at first \0 byte—destroys binary. Use raw httpget bytes
inst1 == inst2	✅ Yes	Direct comparison supported (fixed Aug 6, 2026)
GetAttribute()	✅ Yes	Supported (fixed Aug 6)
SetAttribute()	✅ Yes	Supported (fixed Aug 6)
RemoteEvent:FireServer	✅ Yes	Supported in Pro Hybrid mode
RemoteFunction:InvokeServer / InvokeFunction	✅ Yes	Supported in Pro Hybrid mode (fixed Sep 11, 2026)
