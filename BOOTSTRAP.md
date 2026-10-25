# BOOTSTRAP: the note loaded at the start of every Matcha chat

This file is the **canonical** text of the bootstrap note. The copy in the user's skill
settings is a paste of this file. If they differ, edit this one; the settings copy is
downstream. That is deliberate: it means a chat can fix the bootstrap without the user
touching their skill settings by hand.

Design rule: **this note lists no files.** It says "read the whole repo". Adding,
splitting, renaming or moving a brain file therefore never requires editing this note.
Only a repo rename, a branch change, or the repo growing past the point where a full
read is practical would force an edit.

---

# Matcha bootstrap

## Repos

| Repo | Holds |
|---|---|
| `bobashid/Evergrowin-AI-Skill-Brain-For-Matcha-Roblox` | The skill. Read all of it. |
| `bobashid/roblox-matcha` | Game scripts and `Hub` only. Never skill notes. |

Never mix them. Nothing from the brain gets a loader or a Hub entry; nothing in the
script repo is ever treated as a skill file.

## Step 1: read the whole brain before anything else

1. `get_file_contents` with owner `bobashid`, the brain repo, path `""` (empty) to list the root.
2. Read every file returned, in full. If `SKILL.md` is among them, read it first: its locked
   rules (L1 to L7) govern everything else.
3. For every folder returned (`learned/`, `parts/`, `games/`, or any other), list it and read
   every file inside. Repeat for nested folders.
4. If a result comes back cut off or in parts, request the remaining ranges until complete.
   Do not skim. Do not answer from habits of other Roblox executors.
5. Precedence when two files disagree: `SKILL.md` locked rules (L1 to L7) win over everything;
   then `learned/corrections.md`; then `learned/*` facts; then the reference parts.
   `INDEX.md` wins only on *where things are*, not on what the rules are.
6. If the repo is unreachable (tool error, empty result, no GitHub tool): say so in one
   sentence, follow Step 2, ask before guessing, and never claim you read it.

## Step 2: rules that hold even if the read fails

- Scripts go to `bobashid/roblox-matcha`, never into the chat. Over 40 lines is pushed; the
  reply is the status block with the loader line. Paste code only on an override word
  (`inline`, `paste it here`, `give me the code`, `just the script`, `no github`, `in chat`).
- No GitHub tools in the chat: reply "GitHub tools aren't enabled in this chat. Enable the
  GitHub connector, or say 'inline' and I'll paste the script here." and stop.
- A tool result you did not see is a failure. No claimed push, sha, size, version or
  "verified" without it in a tool result this turn. Lint before every push. After a freeze
  or crash report: stop adding features, restore a known-good version, bisect one change at
  a time.
- Matcha is not Roblox: `Instance.new` is nil; menus use `UI.AddTab`; use
  `fireproximityprompt` / `fireclickdetector` / `firetouchinterest` rather than keys or
  teleports; do not rebuild Matcha's built-in ESP, aimbot, silent aim or triggerbot.

## Step 3: grow the brain, in the same turn, without asking

When something durable is confirmed (a probe or log proves an API fact or an error meaning, a
bug cause is isolated by a test, the user corrects you, a pattern works in a real game, a game
fact is learned), write it into the right brain file in the same turn, following `SKILL.md`
section 14: read the file and its sha, make the smallest edit, write, read it back, update
`INDEX.md` if a file was added or moved.

Label every entry `Confirmed (YYYY-MM-DD)` with its proof, or `UNVERIFIED (YYYY-MM-DD)` with
the test that would confirm it. Never store a guess as a fact. Never weaken L1 to L7.
