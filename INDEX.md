# INDEX: map of the Matcha brain

Read this first, then `SKILL.md`, then everything in `learned/`, then the parts that apply. Keep this file accurate: update it in the same turn whenever a file is added, split, renamed or removed (section 14 of `SKILL.md`).

## Two repos, never mixed

| Repo | Holds | Never holds |
|---|---|---|
| `bobashid/Evergrowin-AI-Skill-Brain-For-Matcha-Roblox` (this repo) | The skill: `INDEX.md`, `SKILL.md`, `BOOTSTRAP.md`, `learned/`, the reference parts | Game scripts, `Hub`, loaders |
| `bobashid/roblox-matcha` | Game scripts and `Hub` — the files the user loads | Skill notes, brain files |

Scripts are pushed to `roblox-matcha`. What gets learned is written here. Nothing here ever gets a loader or a `Hub` entry, and nothing there is ever treated as a skill file. If a task says "read the skill", read this repo; if it says "push the script", push to `roblox-matcha`.

## Layout right now

- `BOOTSTRAP.md` is the canonical bootstrap. The Atomic Chat skill settings hold only a **pointer** to it (raw URL + `get_file_contents` + a four-line safety net), so the settings text never needs editing again. It deliberately lists no files, only the read procedure.
- `learned/` is a real folder and the growth target: `api-facts.md`, `errors.md`, `patterns.md`, `dump-protocol.md`, `corrections.md`, `unverified.md`. New facts go here first.
- The reference parts sit flat at the repo root under their bare names (`failure-protocols.md`, `hub-sync.md`, `api-memory.md`, ...). `SKILL.md` calls them `parts/api-*`; the actual path is the bare root name. Moving them into `parts/` must be done with GitHub's web rename and `INDEX.md` updated in the same commit; re-uploading them risks silent corruption.
- `_TEMPLATE.md` is the game-note template. Copy it to `games/<Game Name>.md` when a game gets worked on.

## Always read
| File | What it holds |
|---|---|
| `BOOTSTRAP.md` | Canonical bootstrap: the two repos, the read procedure, precedence, the rules that survive a failed read, the growth rule |
| `SKILL.md` | Core: stop rules, GitHub delivery and status-block reply, push procedure, naming, commit messages, Hard Rules, brain layout (13), free-editing rules (14) |
| `failure-protocols.md` | F1 to F11: no invented results, lint, freezes, version churn, interaction layer (`fire*`), probing |
| `learned/*.md` | Newest proven facts, errors, patterns, the dump protocol, corrections, unverified leads |

## Read by task
| Task | Read |
|---|---|
| Any game-script push | `hub-sync.md`, `script-standard.md`, `games/<Game>.md` if it exists |
| The user is about to paste game code, or I am missing a name | `learned/dump-protocol.md` |
| Building or fixing a menu, keybinds, input | `pattern-ui-input.md`, `playbooks-and-delivery-reference.md` |
| ESP, drawing, remotes, loops, saving config, anti-AFK | `pattern-render-remotes-loops.md` |
| Deciding whether Matcha already has a feature; style; intake questions | `builtins-and-style.md` |
| Using any global or class you have not used this session | `api-globals-limits-pitfalls.md`, then `api-overview-mcp-functions.md` |
| Memory functions, `getgc`/`applygc`, datatypes, classes | `api-memory.md` |
| Newer docs facts, open questions, unverified items | `docs-updates-and-open-questions.md` |
| Stale version, "no script for place", wrong game loaded | `hub-sync.md` |
| A freeze or crash report | `failure-protocols.md` (F3) and the bisect recipe in `pattern-render-remotes-loops.md` |
| A push that read back wrong, or two files with the same sha | `learned/errors.md` |
| Fixing or changing the bootstrap note itself | `BOOTSTRAP.md` |

## Folders
- `learned/`: grows every session (API facts, errors, patterns, dump protocol, corrections, unverified).
- `games/`: create on demand from `_TEMPLATE.md`, one note per game.
- Reference parts: flat at the repo root for now (see Layout above).

## Not in this repo
Game scripts and `Hub` live in `bobashid/roblox-matcha`. Nothing here ever gets a loader or a Hub entry.
