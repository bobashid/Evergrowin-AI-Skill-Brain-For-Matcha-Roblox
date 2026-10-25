# INDEX: map of the Matcha brain

Read this first, then `SKILL.md`, then everything in `learned/`, then the parts that apply. Keep this file accurate: update it in the same turn whenever a file is added, split, renamed or removed (section 14 of `SKILL.md`).

## Always read
| File | What it holds |
|---|---|
| `SKILL.md` | Core: stop rules, GitHub delivery and status-block reply, push procedure, naming, commit messages, Hard Rules, brain layout (13), free-editing rules (14) |
| `parts/failure-protocols.md` | F1 to F11: no invented results, lint, freezes, version churn, interaction layer (`fire*`), probing |
| `learned/*.md` | Newest proven facts, errors, patterns, corrections, unverified leads |

## Read by task
| Task | Read |
|---|---|
| Any game-script push | `parts/hub-sync.md`, `parts/script-standard.md`, `games/<Game>.md` if it exists |
| Building or fixing a menu, keybinds, input | `parts/pattern-ui-input.md`, `parts/playbooks-and-delivery-reference.md` |
| ESP, drawing, remotes, loops, saving config, anti-AFK | `parts/pattern-render-remotes-loops.md` |
| Deciding whether Matcha already has a feature; style; intake questions | `parts/builtins-and-style.md` |
| Using any global or class you have not used this session | `parts/api-globals-limits-pitfalls.md`, then `parts/api-overview-mcp-functions.md` |
| Memory functions, `getgc`/`applygc`, datatypes, classes | `parts/api-memory.md` |
| Newer docs facts, open questions, unverified items | `parts/docs-updates-and-open-questions.md` |
| Stale version, "no script for place", wrong game loaded | `parts/hub-sync.md` |
| A freeze or crash report | `parts/failure-protocols.md` (F3) and the bisect recipe in `parts/pattern-render-remotes-loops.md` |

## Folders
- `parts/`: the long reference, split by topic (each file under about 30 KB).
- `learned/`: grows every session (API facts, errors, patterns, corrections, unverified).
- `games/`: one note per game (`_TEMPLATE.md` to start one).

## Not in this repo
Game scripts and `Hub` live in `bobashid/roblox-matcha`. Nothing here ever gets a loader or a Hub entry.
