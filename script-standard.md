# Script standard for the script repo (brain part)

Read before writing or updating any script in `bobashid/roblox-matcha`. It makes every game file look the same, so a future chat (or the user) can find the facts at a glance. Match the style of the user's existing scripts first; this standard fills the gaps and sets the header.

## Repo layout (roblox-matcha)
- Flat files named after the game (`Chop A Forest`), plus `Hub`, plus `lib/` helpers a script really needs. No changelog, README or notes files there; knowledge goes to the brain (`games/<Game Name>.md`).
- File names, commit messages, loaders and Hub sync follow SKILL.md sections 2 to 6 and 11.

## Header block (first lines of every game script)
```lua
--[[
  Game:       <Game Name>            (same as the file name)
  Place IDs:  <id>, <id>             (only ids from the user or matcha_status; "unknown" if none)
  Version:    vX.Y                   (matches the VERSION constant below)
  Features:   <the same list as the commit message>
  Controls:   menu tab "<Game Name>"; every control is a toggle or button (no typed commands)
  Needs:      Matcha Pro / Hybrid Mode / fire* globals / Allow Unsafe LuaU  (list only what it really uses)
  Confirmed:  <what was proven in game, one line each>
  UNVERIFIED: <what is assumed, one line each>
]]
```
Keep Confirmed and UNVERIFIED current: when a guess is proven, move it. Copy the same facts into `games/<Game Name>.md` in the brain.

## Layout order inside the script
1. Header block, `NAME`, `VERSION = 'vX.Y'`, kill-previous-instance line, version print.
2. `Config` table (all tunables and defaults in one place) and saved-config load.
3. Helpers (`mkSection`, `mkToggle`, `mkButton`, scanners, pools), copied from the pattern library, not re-invented.
4. Game lookups resolved once (remotes, folders), with a `warn` if missing.
5. Collectors and feature loops, one `task.spawn` each, one wait per iteration.
6. Renderer (`RenderStepped`) reading caches only.
7. Menu build (built once, inside `pcall`).
8. `stop()`, then `_G[NAME] = { stop = stop, version = VERSION }` only after init, then the loaded notice.

## Versioning
One scheme: `vX.Y`. Bump Y for every pushed change, X for a rewrite. The same string is printed on load, stored in `_G[NAME].version`, and read back after every push. Do not put versions in file names or commit messages.

## Game notes (brain side)
For each game the brain keeps `games/<Game Name>.md` from `games/_TEMPLATE.md`: place ids, script file and current version, feature list, confirmed remotes with argument shapes, attributes, instance paths, quirks, open questions. Update it in the same turn whenever a script push teaches you something about the game.
