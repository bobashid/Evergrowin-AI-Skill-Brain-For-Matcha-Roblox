# Pet Simulator 99 (place 8737899170)

Script: `Pet Simulator 99` in `bobashid/roblox-matcha`, v1.0. One toggle: Auto Tap.

## What the script does

- One menu toggle, `Auto Tap`, in the `PS99` tab. No other widgets, no keybinds, no stop key.
- Every `CONFIG.tick` (0.05 s) it walks `workspace` up to three levels deep, and for any part
  whose name contains `coin`, `gem` or `diamond` within `CONFIG.range` (80 studs) it fires
  `firetouchinterest(hrp, part, 0)` then `(hrp, part, 1)`. No movement, no teleport.
- Child lists larger than `maxKids` (250) are skipped so one tick cannot stall a frame.

## UNVERIFIED - this game has NOT been probed

Everything below is a guess. Do not promote any of it to Confirmed without a probe.

- UNVERIFIED (2026-11-05): pickup names contain `coin` / `gem` / `diamond`. Test: run the
  script and read the console - if taps stay at 0 with pickups visibly nearby, the names are
  different. Fix is one line: `CONFIG.words`.
- UNVERIFIED (2026-11-05): the server accepts a `firetouchinterest` fired from a distance.
  Test: with the toggle on, see whether the pickup disappears and the counter climbs.
- UNVERIFIED (2026-11-05): pickups are collected by touch at all. PS99 may use a
  ProximityPrompt or a collect RemoteEvent instead, in which case `fireproximityprompt` or the
  remote is the correct route and the touch path will silently do nothing.

## Next probe (first tool-enabled turn in this game)

1. `matcha_exec`: list `workspace` children and the children of anything that looks like a
   pickup folder; print names and `ClassName` of the first few.
2. Print `ReplicatedStorage` children looking for a collect / pickup RemoteEvent or
   RemoteFunction.
3. Only then decide: touch (current), prompt, or remote.

## Known-good patterns reused here

- Section built with the side ladder (`Left`, then `Right`, then bare name), toggle built with
  the three-form ladder (`id,label`, `label,id`, `label,default`), `UI.GetValue` polled and
  authoritative once it returns a boolean. All from `pattern-ui-input.md`.
- No global stop key, per the standing rule in `learned/working-style.md`.
