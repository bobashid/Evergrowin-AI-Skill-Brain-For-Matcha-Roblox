# Hub sync and stale reads (brain part)

Read this before every game-script push and whenever the user reports an old version or "no script for place".

## 11. Hub sync: mandatory on every game-script push
The user keeps ONE loader in their clipboard: the Hub. Hub reads `game.PlaceId`, finds the matching file name, fetches that file with a cache-busted URL, and runs it. If a new game script is pushed without Hub knowing about it, the user loads the Hub in that game and gets "no script for place ...". That is a failed delivery. So a game-script push is not finished until Hub points at it, in the same turn, before replying.

Brain files (the other repo) and any other reference or notes file are not game scripts: they never go in `PLACES` or `SCRIPTS`.

### 11.1 The two edits Hub needs
- `PLACES`: the authoritative map, `[placeId] = 'File Name'`. It wins over name matching and has no ambiguity. Use it whenever an id is known. A game with several places (lobby, match, sub-areas) has one PlaceId per place; add each id the user gives you.
- `SCRIPTS`: `{ file = 'File Name', keys = { 'normalised', 'aliases' } }`. Fallback for places with no id. Keys are lowercase letters and digits only (Hub's `norm()` strips everything else), so Chop A Forest becomes `chopaforest`. Add the short form the user would recognise (`chopforest`), not every permutation.

### 11.2 Order of operations, every time
1. Push the game script. Read it back. Confirm the version stamp matches.
2. Read Hub with `get_file_contents` (owner, repo, path `Hub`, ref = branch) and keep the returned sha. Never reuse a sha from an earlier turn.
3. Update Hub with `create_or_update_file`, passing that sha, with the new `PLACES`/`SCRIPTS` entries and a bumped VERSION string. Commit message: `Hub: add <Game>`.
4. Read Hub back. Confirm the new version string and the new place id appear in the returned text.
5. Only then reply.

If step 3 fails: retry once with a re-read sha. If it still fails, say plainly "Hub update failed, here is the direct loader for this game instead" and give the per-game loader. Do not present the Hub loader as working when Hub does not contain the entry.

If `Hub` does not exist in the repo yet, create it first from the contract in 11.6 (with the games already in the repo that you can map), tell the user in one sentence that Hub was created, then continue with the steps above.

### 11.3 Place ids: never invent one
- An id must come from a tool result or from the user. The user usually pastes a `roblox.com/games/<id>/<name>` URL: the number after `/games/` is the place id.
- If you do not have an id for the game, do NOT add a `PLACES` entry, do NOT guess, do NOT copy the id from a similar game. Add only the `SCRIPTS` entry (name matching will usually catch it) and put ONE line in the reply: `no place id for <Game> yet, send the URL and I'll pin it.`
- `matcha_status` returns the place id of wherever the user currently is. It is only valid for that one game; never reuse it for a different title.
- The place id goes in `PLACES` and in the script's header comment. It never goes in the file name.

### 11.4 One place, two scripts
If a place has more than one file (for example Prison Life has both `Prison Life HvH` and `PRISON LIFE TRIGGERBOT`), `PLACES` can only hold one. Pick the one the user last asked for, or the more general one, and say in one line which is the default and how to reach the other: `_G.Hub.load('PRISON LIFE TRIGGERBOT')`. Do not silently drop the second file from `SCRIPTS`; keep both entries so the manual load keeps working.

### 11.5 Renames and deletions
When a file is renamed to match its game:
1. Create the new file with the same content (message `<Game>: <features>`).
2. Point `PLACES` and `SCRIPTS` at the new name in the same turn.
3. Delete the old file with `delete_file`.
Never leave both. Never leave `_v2`, `_final`, `(1)`, or a place id in a name. When the source is obfuscated and cannot be rewritten (for example BBC ESP), create a thin loader named after the game that fetches the original file with a cache-busted URL and forwards `stop`. Rename it in Hub and keep the original file.

### 11.6 What Hub must contain (contract)
- `PLACES`, `SCRIPTS`, `norm()`, `bust()`, `load()`, `pick()`.
- `load(file)` encodes `+` as `%2B` and space as `%20`, then appends `bust()`.
- `bust()` returns `'?v=' .. math.floor(tick()*1000) .. '_' .. math.random(100000,999999)` (the inner fetch may use `tick()`; only the outer loader line is fixed to `os.time()`).
- `_G.Hub = { stop, load, places, version }`.
- Prints the version at load, and prints place id and game name at the end of every run, matched or not, so the user can always read off an id.
- On no match: prints `no script for place <id> (<Game Name>)` and notifies. It must never error and never load the wrong file.
- No menu, no keybinds, no loops, no input. It runs and exits.

### 11.7 Reply format when Hub was updated
The reply is the section 2 status block, plus the one-line "no place id for <Game>" note and the one-line conflict note if a place has two scripts. No changelog narrative, no list of files. Features live in the commit message.

### 11.8 Worked turn: new game script, id known
User: "make a script for Tower Incremental, here's the URL roblox.com/games/75766778174856/Tower-Incremental"
1. `get_file_contents` path `Tower Incremental` (exists? read it, get the sha). The file name is the plain game name in Title Case, not the URL slug.
2. `create_or_update_file` path `Tower Incremental`, sha = the one read, message `Tower Incremental: auto upgrade, auto buy, ESP`.
3. `get_file_contents` path `Tower Incremental`, confirm the version stamp.
4. `get_file_contents` path `Hub`, keep sha H.
5. `create_or_update_file` path `Hub`, sha H, VERSION bumped, `PLACES` gains `[75766778174856] = 'Tower Incremental'`, `SCRIPTS` entry added or updated.
6. `get_file_contents` path `Hub`, confirm the new version and the id are present.
7. Reply: the section 2 status block with the Hub loader (`?v=` + `os.time()`).

## 12. Stale reads: why the old version came back
Two URLs can serve a cached copy:

| URL | Who fetches it | Fixed by |
|---|---|---|
| Outer: the loader the user pastes | the user's executor | `?v=" .. tostring(os.time())` on the loader |
| Inner: Hub fetching the game file | Hub | `bust()` inside Hub |

Both need a unique query string. A unique query string is a different URL to the CDN, so it cannot return the previous commit's body. A stale read can also mean the push never landed, so a read-back (section 4, step 5) is still required.

Rules:
- Never diagnose "it's stale" as a game bug, a loadstring bug, or a wrong path until the `?v=` is present on the loader being tested.
- Never tell the user to "wait five minutes" as the fix. Offer the busted loader; mention the wait only as the fallback if they refuse the query string.
- If a read-back this session shows the new version but the user still runs the old one, the cause is the outer URL (or the Hub copy they have is old). Ask them to paste the exact line they ran.
- Do not put a version number in the query string (`?v=1.3`): it is stable between pushes of the same version and therefore cacheable.

Symptoms and the correct response:
- "I ran it and got the old menu": outer URL cached, or the push never landed. Read the file back, then give the `?v=` loader.
- "Hub says no script for place X": Hub is old or `PLACES` lacks the id. Read Hub back, confirm, add the entry, bump VERSION, push, give the `?v=` Hub loader.
- "It loaded the wrong game's script": name-matching collision. Get the place id, add it to `PLACES`, and remove the over-broad key from `SCRIPTS`.
- "It says compile failed": the file on GitHub is truncated or broken. Read it back, compare size, rewrite in full, then say it was rewritten.
