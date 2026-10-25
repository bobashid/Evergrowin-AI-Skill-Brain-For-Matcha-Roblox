# Built-ins, task intake and code style (brain part)

Read before writing any script: what Matcha already does, what to confirm with the user, how to match their style.

## Task Intake: Ask or Confirm These Before Writing

Most wasted round trips come from guessing. For any non-trivial script, get or confirm these five things first (ask once, concisely; if the user gave them, do not re-ask):
1. **Exact target and action.** Not "auto X" but "fire the remote the game uses for X at the part it raycasts." If the remote path is known, use it; if not, say so and decompile to find it.
2. **Live verification or one-shot.** If Matcha is running and connected via MCP, probe first; if the user only wants code, skip probing and state assumptions.
3. **Game-state context.** Relevant inventory/capacity limits, equipped tool, upgrades or levels owned. Guessing these produces wrong thresholds.
4. **UI shape as a count.** For example "6 toggles, 3 automation and 3 ESP", or "no sliders, no keybinds". Treat the count as a spec.
5. **Off-limits list.** Anything the user does not want touched (for example no teleport, no GC edits, no stat changes) so it is not built and then removed.

## Use Built-ins First: Do Not Reimplement What Matcha Already Has

Before writing any helper, library or "system" (menu, ESP, input, file saving, projection, notifications), check this table and the global lists below. If Matcha already provides it, call the built-in. Writing your own version wastes effort, adds bugs, and often breaks on this VM. Only build your own when the thing is on the ABSENT list or a probe shows the built-in does not work.

| Need | Use the built-in | Do NOT write |
|---|---|---|
| On-screen overlays, ESP boxes, lines, text, circles | `Drawing` (Square, Line, Circle, Text, Triangle, Image) | A custom renderer or GUI via `Instance.new` (it is nil anyway) |
| 3D to 2D projection | `WorldToScreen(pos)` inside a `RenderStepped` callback | Your own camera/FOV matrix math |
| Menu, toggles, sliders, color pickers | `UI.AddTab` / `tab:Section` / widgets, read with `UI.GetValue` | A hand-built Drawing-based menu or keybind UI (unless the user asks) |
| Notifications | `notify(message, title, duration)` | Custom toast/popup drawings |
| Clipboard | `setclipboard(str)` | Workarounds or fake clipboard files |
| Keyboard and mouse | `keypress`, `keyrelease`, `mouse1click`, `mousemoveabs`, `mousemoverel`, `mousescroll`, `iskeypressed`, `ismouse1pressed` | Your own input emulation |
| Saving config / data | `writefile`, `readfile`, `appendfile`, `isfile`, `makefolder`, `listfiles`, `delfile` | An `io`-based saver (`io` is absent) |
| HTTP | `httpget`, `httppost` | `request`, `syn.request`, `HttpService` wrappers, `game:HttpGet` |
| Base64 | `base64encode` / `base64decode` for text (not binary with `\0`) | A custom base64 for plain text |
| Reading game scripts / remotes | `decompile`, `getscripts`, `getscriptbytecode`, `getscripthash`, `getgc` | Re-implementing a script dumper |
| Finding values in client memory | `getgc` / `applygc` (client copy only) | Custom memory scanners (there is no scan API; only `getbase`, `memory_read`, `memory_write`) |
| Game, ping, version info | `getgamename()`, `GetPingValue()`, `identifyexecutor()`, `getrbxversion()` | Guessing or scraping these |
| Timing | `tick()` (monotonic), `os.time()`, `os.clock()` (wall time) | A custom clock |
| Bit operations, UTF-8 | `bit32`, `utf8` (present) | Hand-rolled bit tricks |
| Safe loading of strings | `loadstring` (remember: uncatchable errors, dropped returns) | A custom Lua interpreter or parser |
| Firing remotes | `RemoteEvent:FireServer` / `RemoteFunction:InvokeServer` (Pro, hybrid mode) | Hand-rolling input simulation for prompts/click detectors instead of using `fireproximityprompt` / `fireclickdetector` / `firetouchinterest` (Pro) |

Rules:
1. **Check first, build second.** Search this file for the capability. If it is listed as present or documented, use it directly and only add the glue code.
2. **Do not recreate Roblox executor globals that are absent.** If a global is on the ABSENT list (hooking, `getgenv`, `request`, crypto, `getconnections`, etc.), do not write a fake version that pretends to work. Tell the user it is not available and use another approach (input, Drawing, DataModel reads).
3. **Do not duplicate within the user's own project.** If the repo already has a script or `lib/` helper for the same thing, update it instead of creating a second one (the push procedure already requires searching first).
4. **Keep scripts small.** Menu, ESP and loop boilerplate should be a thin layer over built-ins, not a framework. Do not add libraries, wrappers or abstractions the user did not ask for.
5. **If unsure whether something exists, probe it** with a one-line `matcha_exec` (`print(type(name))`) before writing a replacement. Test existence separately from behavior.

## Match the user's code style (read their repo first)
The pattern library (`parts/pattern-ui-input.md` and `parts/pattern-render-remotes-loops.md`) records the conventions already proven in the repo; apply it first, then read the closest existing script to match details.

The user's own scripts in `bobashid/roblox-matcha` are the reference for how code should be written. The repo has included scripts such as BBC ESP, Chop A Forest and PRISON LIFE TRIGGERBOT; read the ones closest to the new request.

Before writing a new script, and before editing an existing one:
1. List the repo root and open the 1-2 most similar scripts (same game first, otherwise the most similar feature). Read them in full, not just the top.
2. Copy their conventions rather than inventing your own: how they structure the file, naming of variables and config tables, how the menu is built (widget order, labels, defaults, toggle/poll pattern), how loops are timed, how they print status lines and the version stamp, how `stop()` is written, comment density and formatting.
3. Reuse working patterns from them (remote firing, attribute reads, menu helpers) instead of rewriting them differently.
4. When the user's existing style conflicts with a rule in this skill, prefer the skill's Matcha-safety rules (for example no top-level `task.wait`, `_G` registration) but keep the user's formatting and naming.
5. If the repo is unreachable or has nothing similar, say so in one line and follow this skill's defaults.
6. Do not copy anything secret from existing scripts, and never claim a style convention you did not actually see in a file you read this session.

## Matcha's Own Built-in Features: Do Not Rebuild Them

Besides API functions, Matcha itself ships ready-made features in its own menu. Do not write a script that duplicates a built-in feature; tell the user to enable the built-in one instead.

Source: screenshots of Matcha's menu (Pro build dated Oct 4 2026) supplied by the user. The menu has eight tabs: Combat, Visuals, World, Character, Options, Configs, NPC, Teams. Everything below already exists; **do not write a script that recreates it**. Tell the user which tab and setting to use instead. The list may be incomplete (hidden sub-tabs, newer builds), so if the user names a feature that is not here, add it.

| Tab | Built-in features and settings |
|---|---|
| **Combat: Aimbot** | Enabled + keybind; Team Check, Visible Check, Health Check, Sticky Aim; Distance; Hit Part; Aim Type (for example Memory); Rage Method; Type (for example Camera Teleport); sub-tabs Prediction, Smoothness, FOV; Misc: Resolver |
| **Combat: Silent Aim** | Enabled + keybind; Team/Visible/Health Check, Sticky Aim; Distance; Hit Part; Method (for example Forced Magic-Bullet); sub-tabs Prediction, FOV |
| **Combat: Trigger Bot** | Enabled + keybind; Visible Check, Team Check; Hitbox Mul; Delay (ms); Release (ms) |
| **Visuals: ESP** | Enabled; Team Check, Visible Check, Team Based Color, Text Gradient, Text Background, Outline, Glow, Self ESP; Sizing Type; Render Distance; sub-tabs Crosshair, Misc, Flags |
| **Visuals: Box / Name** | Box (Enabled, Fill Box, Box Type 2D); Name (Enabled, Type) |
| **Visuals: Indicators** | Distance, Equipped Item, Skeleton, Head Dot, Head Dot Glow, Profile Picture; sub-tabs OOF Arrow, Radar |
| **Visuals: Health / Chams / Tracer** | Health (Health Bar, Health Based, Health Text, Text Pos); Chams (Mode, Enabled, Filled, Rendering Type); Tracer (Enabled, Origin) |
| **World** | Camera field of view; Freecam (Enabled, Teleport on click, Lock Character Position, Speed); Waypoints (create, remove, goto, tween goto and speed, visualize); World Lighting (Ambience, Custom Fog, Glow, Custom Exposure, Custom Brightness, Custom Time) |
| **Character: Hitbox Extender** | Enabled; Visualize Hitbox, Team Check, Health Check; Hitbox Size, Type, Hitbox Part |
| **Character: Target Hovering** | Enabled; Display Circle; Target Method (for example Closest To Mouse); Radius; Speed |
| **Character: Desync** | Enabled; Remove Walk Animation, Display Server Position, Reset on Desync Off, Use Tick; Desync Interval |
| **Character: Movement** | No-Clip, Inf Jump, Click TP, Speed (method + amount), Flight (method + amount), Jump (power), Float (height) |
| **Character: Exploit** | Timer Manipulation (method + multiplier) |
| **Options** | Theme/menu key/accent color; Streamproof, Vertical Sync, Auto Rescan Game, InGame Input Detection, Set Roblox FPS, Raycast Enable (and update rate), Allow Unsafe LuaU, Hybrid Mode (and Usage level); Session buttons (Rejoin, Server Hop, Rescan, Eject, Eject Bind); UI elements (Watermark, View Explorer, Keybind List, Notification Sound, Music Overlay, Keystroke, Lag Notifier, Array List); interface style (Dark Background, Snow Effect, Blur, Glass) |
| **Configs** | Scripts: Open Folder, script list, Load Script, Set Auto Execute. Configs: Open Folder, config list, Load, Save by name |
| **NPC** | Add NPC by Model or Directory; per-NPC General and Visuals settings; its own config load/save/folder |
| **Teams** | Use Custom Teams; Auto Detect Allied Team; per-team Ally/Enemy status; Quick Actions (All Enemy, All Ally, Reset to Default) |

Rules:
1. **Check before building.** On any request for a general-purpose feature (player ESP, aim assistance, speed/fly-style movement, generic teleports, anything that sounds like a standard menu feature), first ask yourself "is this likely built in?" If it is on the table above, stop and say so in one line. If it is not on the table but is a very common feature, ask the user once whether Matcha already has it before writing it.
2. **Do not recreate checks the menu already has.** Team check, visible check, health check, FOV/distance limits, hit part selection and team lists are handled by the Combat, Visuals and Teams tabs. A script should not carry its own copy of them unless the user asks; if it needs team data, tell the user to set it in the Teams tab.
3. **Only build what the game needs.** Write custom code for game-specific logic (remotes, attributes, game mechanics, auto-farming, sell/collect loops, custom overlays for objects that are not players). If the request is a generic version of a built-in, do not rebuild it just because the user's wording sounds different ("see players through walls" = player ESP).
4. **Extending is allowed, duplicating is not.** If the user explicitly wants custom behavior the built-in lacks (for example ESP for a specific game object, or a game-specific targeting rule), build only that missing part as a thin script and say what it adds beyond the built-in.
5. **Explicit override.** If the user says "write my own", "custom version", or "don't use the built-in", build it.
6. **When unsure whether a feature exists, ask or probe; never assume.** Do not claim a built-in feature exists that the user has not named; ask them to name it or check Matcha's menu, and add it to the table once confirmed.
