# Script index: game -> place id -> file -> brain note

A recovery map for the script repo. `Hub` is what actually loads scripts; **this file is the
survivable record of which place maps to which file**, so a corrupted or overwritten `Hub` can
be rebuilt instead of guessed at. Hub was corrupted twice on 2026-10-27 (stray `}` after
`SCRIPTS`, then an unterminated `print(` in `stop()`), which is exactly the failure this covers.

**Maintenance: when a game script is pushed and Hub is synced, add or update its row here in
the same turn.** That is part of the push procedure, not an extra step.

## The table

| Game (file name) | Place ID | Bytes | Brain note |
|---|---|---|---|
| `CQB Hell` | 98689921952741 | ~21k | `games/CQB Hell.md` |
| `Piano Rhythm Tiles` | 101999883467300 | 15147 | `games/Piano Rhythm Tiles.md` |
| `Realistic Home Defense` | 105527962898452 | 14208 | `games/Realistic Home Defense.md` |
| `Chop A Forest` | 103363457515322 | 21785 | - |
| `Clean Everything` | 140576663170542 | 17762 | - |
| `Command An Army` | 101770480176177 | 75920 | - |
| `Dogs` | 94009908234110 | 8624 | - |
| `Feed A Mosquito` | 125310131519813 | 72413 | - |
| `Hike For Eggs` | 110432290690507 | 10634 | - |
| `PRISON LIFE TRIGGERBOT` | 155615604 | 65731 | - |
| `Prison Life HvH` | 155615604 (alt) | 15971 | - |
| `Tower Incremental` | 75766778174856 | 29169 | - |
| `What If Disasters` | 109440485612336 | 40823 | - |
| `+1 Upgrade Per Kill` | 113273588963037 | 27161 | - |
| `Build And Kill Zombies` | 105011592530400 | 8355 | - |
| `Hub` | n/a (loader) | 5836 | `hub-sync.md` |

## Notes on the table

- **Prison Life has two scripts on one place id.** `PLACES` can hold one, so it points at
  `PRISON LIFE TRIGGERBOT`; the HvH build is reachable with
  `_G.Hub.load('Prison Life HvH')`. Both stay in `SCRIPTS` so manual load keeps working.
- **Place ids come from `Hub`'s `PLACES` table**, read 2026-10-27. If one is wrong, `PLACES` is
  wrong and both need fixing together.
- **Byte sizes are from the repo listing at commit `98da4858`**, which predates `CQB Hell`. Any
  size here is a rough indicator only; read the file for the real number before quoting one.
- A note in the `Brain note` column means a `games/<Game>.md` exists with confirmed facts
  (remotes, attributes, quirks). No note means the next session starts from zero on that game.
- Scripts with no brain note are the cheapest candidates for a future note: `Chop A Forest`
  (21785 bytes) and `Tower Incremental` (29169) already have confirmed remotes inside them.

## Rebuilding Hub from this table

If `Hub` is lost, `PLACES` can be regenerated directly from the rows above, and `SCRIPTS` from
the file names. That is the whole point of keeping this file small and flat: it survives even
when the loader does not.

## What deliberately does NOT live here

No script bodies. The brain is read in full every Matcha task (`BOOTSTRAP.md` Step 1), so a
copy of `Command An Army` (76 KB) or `Feed A Mosquito` (72 KB) inside it would push the read
past the point where a session can honestly claim to have read the brain. Scripts live in
`bobashid/roblox-matcha`; knowledge about them lives here.
