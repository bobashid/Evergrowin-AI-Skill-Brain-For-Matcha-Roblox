# Pattern library 1/2: menus, input, keybinds (brain part)

Read before building any menu, keybind or input code.

## Pattern library (proven in the user's own repo scripts; copy these patterns, do not re-derive them)

These patterns come from reading the scripts in `bobashid/roblox-matcha` (Chop A Forest, Tower Incremental, Prison Life HvH, Anti Afk, Dogs, Auto Roll, +1 Upgrade Per Kill, Clean Everything, Hike For Eggs). Where a snippet uses `[NAME]` or `NAME`, substitute the script's own name. Where this library conflicts with the official docs, the docs win on API signatures and this library wins on what actually broke in the user's games. If a pattern here conflicts with an older rule elsewhere in this file, this library is the newer, repo-proven rule.

## UI / MENU WIDGETS (read this before building any tab)

Every script builds its menu through `UI.AddTab`. Several failure modes are SILENT: the call returns without error and the widget never appears or never fires. The official signature is `sec:Toggle(id, label, default, callback)` (see "UI Binding"), but builds differ, so the user's scripts use the retry ladders below and they are proven to work. Use them; do not improvise a shorter version.

### U1. Detect UI once, at the top
```lua
local HAS_UI = type(UI) == 'table' and type(UI.AddTab) == 'function'
```
If `HAS_UI` is false: warn once, run with defaults, and (only if the script has a genuinely useful keyboard mode) install a key fallback. A missing UI must never abort the script; the automation must still run.

### U2. Section: accept any non-nil handle
`tab:Section(name)` without a side returns something that is not a usable handle on at least one build, so the widgets vanish. `tab:Section(name, 'Left')` works. Try sides first, bare name last, and accept ANY non-nil return. Do not test `type(sec) == 'table'` (the user's scripts disagree; `sec ~= nil` is the settled rule).
```lua
local function mkSection(tab, name)
    local sides = { 'Left', 'Right' }
    for i = 1, #sides do
        local ok, sec = pcall(function() return tab:Section(name, sides[i]) end)
        if ok and sec ~= nil then return sec end
    end
    local ok, sec = pcall(function() return tab:Section(name) end)
    if ok and sec ~= nil then return sec end
    warn('[NAME] section ' .. name .. ' failed')
    return nil
end
```

### U3. Button argument order: `(label, cb)` MUST be first
`sec:Button(label, id, cb)` returns successfully and NEVER wires the callback. It is a silent no-op that has cost several debugging sessions. Order: `(label, cb)`, then `(label, label, cb)`, then warn. Never try `(label, id, cb)` first.
```lua
local function mkButton(sec, label, cb)
    if pcall(function() sec:Button(label, cb) end) then return true end
    if pcall(function() sec:Button(label, label .. '_btn', cb) end) then return true end
    warn('[NAME] button failed: ' .. label)
    return false
end
```

### U4. Toggle and SliderInt: argument-order ladder
The official order is `(id, label, ...)`; some builds accept `(label, id, ...)` or `(label, default, cb)`. Try each inside `pcall` and stop at the first that does not error. Always pass callbacks through the vararg scanners in U5.
```lua
local function mkToggle(sec, label, id, default, apply)
    local cb = function(...)
        local v = firstBool(...)
        if v ~= nil then apply(v) end
    end
    local tries = {
        function() sec:Toggle(id, label, default, cb) end,
        function() sec:Toggle(label, id, default, cb) end,
        function() sec:Toggle(label, default, cb) end,
    }
    for i = 1, #tries do
        if pcall(tries[i]) then return true end
    end
    warn('[NAME] toggle failed: ' .. label)
    return false
end
```
```lua
local function mkSlider(sec, label, id, min, max, default, apply)
    local cb = function(...)
        local v = firstNumber(...)
        if v ~= nil then apply(v) end
    end
    local tries = {
        function() sec:SliderInt(id, label, min, max, default, cb) end,
        function() sec:SliderInt(label, id, min, max, default, cb) end,
        function() sec:SliderInt(label, min, max, default, cb) end,
    }
    for i = 1, #tries do
        if pcall(tries[i]) then return true end
    end
    warn('[NAME] slider failed: ' .. label)
    return false
end
```

### U5. Callback arguments are unpredictable: always scan varargs
Widget callbacks arrive as `(value)`, `(self, value)`, or with other values in front. Never index `...` positionally. These two scanners are required helpers in every script that has a menu:
```lua
local function firstBool(...)
    for i = 1, select('#', ...) do
        local v = select(i, ...)
        if type(v) == 'boolean' then return v end
    end
    return nil
end
local function firstNumber(...)
    for i = 1, select('#', ...) do
        local v = select(i, ...)
        if type(v) == 'number' then return v end
    end
    return nil
end
```

### U6. Two sources of truth: callback AND poller, never one alone
`UI.GetValue(id)` works in some scripts and returned `nil` on every probe in Prison Life HvH, where only callbacks worked. Callbacks are skipped when a widget keeps its default and is never touched. Rule: register every boolean widget in a `TOGGLES` list, poll `UI.GetValue(id)` then `UI.GetValue(name)`, and treat the getter as authoritative for a key only after it has returned a boolean (`getterWorks[key] = true`). Until then the callback is the only writer; after that, the callback early-returns so the two cannot fight.
```lua
local TOGGLES, getterWorks = {}, {}
local function registerToggle(key, id, name)
    TOGGLES[#TOGGLES + 1] = { key = key, id = id, name = name }
end
local function applyBool(key, Config)
    return function(...)
        if getterWorks[key] then return end
        for i = 1, select('#', ...) do
            local v = select(i, ...)
            if type(v) == 'boolean' then Config[key] = v return end
        end
    end
end
```
```lua
task.spawn(function()
    while running do
        for i = 1, #TOGGLES do
            local t = TOGGLES[i]
            local val = nil
            local ok1, v1 = pcall(function() return UI.GetValue(t.id) end)
            if ok1 and type(v1) == 'boolean' then
                val = v1
            else
                local ok2, v2 = pcall(function() return UI.GetValue(t.name) end)
                if ok2 and type(v2) == 'boolean' then val = v2 end
            end
            if val ~= nil then
                getterWorks[t.key] = true
                if Config[t.key] ~= val then
                    Config[t.key] = val
                    print('[NAME] ' .. t.key .. ' = ' .. tostring(val))
                end
            end
        end
        task.wait(0.1)
    end
end)
```

### U7. The AddTab callback fires EVERY FRAME
`UI.AddTab(name, function(tab) ... end)` calls your builder repeatedly. Set a `built` flag BEFORE building, wrap the body in `pcall`, and log `'tab opened - building widgets'` so the console proves the callback fired at all.
```lua
local built = false
local function buildTab(tab)
    if built then return end
    built = true
    local ok, err = pcall(function()
        local sec = mkSection(tab, 'Automation')
        if not sec then return end
        -- widgets here
    end)
    if not ok then warn('[NAME] build error: ' .. tostring(err)) end
    print('[NAME] tab widgets built')
end
if HAS_UI then
    local ok, err = pcall(function()
        UI.AddTab('Script Name', function(tab)
            print('[NAME] tab opened - building widgets')
            pcall(buildTab, tab)
        end)
    end)
    if not ok then
        warn('[NAME] UI.AddTab failed: ' .. tostring(err))
        HAS_UI = false
    end
end
```

### U8. stop() must remove the tab
`UI.RemoveTab(name)` exists on at least some builds. Call it behind a `pcall` in `stop()`, guarded by `HAS_UI`. Leaving the tab behind after a reload is how you end up with several dead sections.

### U9. Menus are built from the GAME's data, not hardcoded
If the game has rollers, ore types, item names or rarities, enumerate them at build time (`discoverItemNames()` in Auto Roll, `RARITY_ORDER` in Hike For Eggs, ore kinds in Tower Incremental) and create one toggle or button per entry. Cap the widget count (about 30); never create 200 widgets.

## INPUT, KEYBINDS AND MOUSE (three separate APIs)

### I1. `InputBegan` KeyCode is an INTEGER, not an EnumItem
The input object is a table whose `KeyCode` is a number. `input.KeyCode == Enum.KeyCode.F` is always false. Match the integer, the EnumItem and the name:
```lua
local function readKeyCode(input)
    if input == nil then return nil end
    if type(input) == 'table' then return input.KeyCode end
    local ok, v = pcall(function() return input.KeyCode end)
    if ok and v ~= nil then return v end
    return input
end
local function codeValue(code)
    if code == nil then return nil end
    local ok, v = pcall(function() return code.Value end)
    if ok and type(v) == 'number' then return v end
    return nil
end
local function keyMatches(k, cfg)
    if k == nil then return false end
    if cfg.keyCode ~= nil and k == cfg.keyCode then return true end
    local num = (type(k) == 'number') and k or codeValue(k)
    if num ~= nil and cfg.keyVal ~= nil and num == cfg.keyVal then return true end
    return false
end
```

### I2. Held keys: poll `iskeypressed` with BOTH forms
`iskeypressed(Enum.KeyCode.F)` and `iskeypressed(enumValue)` differ across builds; try both inside `pcall`. Edge-detect in the poll loop (`wasDown`) so one press does not toggle 20 times. `iskeypressed` officially takes a Windows virtual-key code.

### I3. Input only lands while the window is FOCUSED
Probed in the user's Anti Afk work: `keypress`, `mouse1click` and `setrobloxinput` reach an unfocused client zero times. Gate key injection behind `isrbxactive()`. If a script must enable injection, call `setrobloxinput(true)` once before a tap and `setrobloxinput(false)` once after it, ONE boolean argument, per tap only, never in a loop or watchdog (see F9). When the client is unfocused, use remotes and camera drift instead (see ANTI-AFK).

### I4. Keybinds must be rebindable and persisted
Never hardcode a key the user cannot change (they cannot type in Matcha). Provide a "Set Key (F)" button that sets a `listening` flag; the next `InputBegan` binds it, saves the config file, and logs the new name. `keypress(vk)` / `keyrelease(vk)` take VIRTUAL-KEY CODES (for example `0x13` for Pause, `0x45` for E), not key names and not Enum items.

### I5. Mouse API that works
`mouse1press()`, `mouse1release()`, `mouse1click()`, `mousemoveabs(x, y)`, `mousemoverel(dx, dy)`, `ismouse1pressed()`, `lp:GetMouse().X` / `.Y`. After a non-first-person click, restore the cursor: save `m.X, m.Y` before `mousemoveabs` and move back afterward.

### I6. First-person aiming recipe
Loop `mousemoverel(dx * 0.85, dy * 0.85)` toward the viewport centre with a pixel tolerance (`AIM_TOL = 16`) for a few tries; if it does not converge (cursor locked), fall back to `cam.CFrame = CFrame.lookAt(cam.CFrame.Position, targetPos)`. `CFrame.lookAt` EXISTS in Matcha even though the datatype docs list only `CFrame.new`, `Angles` and `fromOrientation`. Keep re-nudging every `RenderStepped` while the button is held so the target stays under the crosshair. (Matcha already has built-in aimbot and silent aim; only build custom aiming when the user explicitly asks for a custom version.)
