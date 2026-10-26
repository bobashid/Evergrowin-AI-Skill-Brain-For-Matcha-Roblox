# BOOTSTRAP: the note loaded at the start of every Matcha chat

## Loader for a loader

Atomic Chat's skill settings hold a **pointer**, not the note. The note lives here and is
edited here. That is the whole design: everything that changes lives in this file, so the
settings text never needs to be touched again — not for a new rule, not for a new file, not
for a rename inside the brain.

**Recommended settings paste** (pointer plus a four-line safety net, so a GitHub outage
does not leave the chat with no rules at all):

```
The Matcha bootstrap is at
https://raw.githubusercontent.com/bobashid/Evergrowin-AI-Skill-Brain-For-Matcha-Roblox/refs/heads/main/BOOTSTRAP.md
Read it with get_file_contents (owner bobashid, repo Evergrowin-AI-Skill-Brain-For-Matcha-Roblox,
path BOOTSTRAP.md, ref main) and follow it exactly. If the raw URL is the only route, append a
cache-buster: ?v=<any unique value>.

If it cannot be read: say so in one sentence, and until it can be read follow only these —
scripts go to bobashid/roblox-matcha and never into the chat; over 40 lines is pushed and the
reply is the status block with the loader line; a tool result you did not see is a failure, so
never claim a push, sha, size, version or "verified" without it this turn; ask before guessing.
```

**Pure one-liner** (smallest possible, but a GitHub outage leaves the chat with no rules):

```
Read https://raw.githubusercontent.com/bobashid/Evergrowin-AI-Skill-Brain-For-Matcha-Roblox/refs/heads/main/BOOTSTRAP.md
and follow it exactly as the Matcha bootstrap for this chat.
```

**Why `get_file_contents` is preferred over the raw URL.** raw.githubusercontent.com sits
behind a CDN that can serve a copy for a few minutes after a commit, so a note edited this
session may read back stale. The GitHub tool reads the commit directly. A `?v=` query string
on the raw URL is the fallback cache-buster, same idea as the script loaders.

**Design rule: this note lists no files.** It says "read the whole repo", so adding,
splitting, renaming or moving a brain file never requires editing it. Only three things
force an edit — a repo rename, a branch change, or the repo outgrowing a full read — and even
then the edit happens here, not in the Atomic Chat settings.

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
7. **Print a dated brain stamp before your first sentence of substance.** One line:

       brain: <YYYY-MM-DD HH:MM UTC> · SKILL <version> · learned <N> files · <sha8> · grew <file|none> · reread <K>

   Use the date and sha from the read you just did. The `grew` field is filled from the growth
   write Step 3 requires — that write happens before the reply, so the stamp can carry it. If
   nothing durable was confirmed, write `grew none (nothing durable)`. `reread K` is the read
   clock from Step 4.

   If the repo could not be read, print `brain: NOT READ` and follow Step 2 only.

   Never omit this line. Without it, "read the brain and followed it" and "ignored the brain
   and improvised from habit" look identical in the reply — which is exactly how a session can
   paste ten script versions while believing it is following the rules. It dates the brain, so
   pasting output from another chat back here shows whether that chat was current or stale. And
   `grew` makes a turn that learned nothing visible as exactly that, instead of
   indistinguishable from a turn that learned a lot.

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

**Account for growth every turn, not only when it happens.** Before replying, ask whether the
turn produced anything durable. If yes, it is written and read back, and the stamp's `grew`
field names the file. If no, the stamp says `grew none (nothing durable)`. A turn that silently
skips the question is the failure this whole system exists to prevent: the brain stays the same
size while feeling like it is growing.

A correction that stays in the chat is narration, not learning. If a turn contained a
self-correction, the read-back should show a new or edited entry; if it does not, the lesson
dies with the window and the next session repeats it.

When a `learned/` file approaches its rewrite ceiling, consolidate rather than append.
`learned/working-style.md` holds the thresholds, the promotion rule, and the self-audit
procedure.

## Step 4: stay in the loop without being told to reread

A full read at message one is not enough. Conversations run long, and the rules that were
sharp at the start blur by the middle of one. The user should never have to type "reread" to
bring them back. **The triggers below are the reread instruction — treat them as if the user
had sent it.**

The stamp carries `reread K`: the number of replies since the last full read of the brain.
Reset it to 0 after a full read, add one per reply otherwise, and compute it from the actual
reads in this conversation. A fabricated low K is the same lie as a fabricated sha.

**Re-read the whole brain (Step 1) before replying when any of these is true, whatever K is:**

1. About to push, edit or delete any file.
2. About to state a version, a sha, a byte size, or the word "verified".
3. The work moved to a different game, a different script, or a different kind of task.
4. The user reports something failing, or pastes output — a HUD line, a console error, a
   screenshot, another chat's reply.
5. About to ask the user for a dump or a probe result.
6. About to say a rule does not apply, or that the skill is wrong about something.
7. `K` has reached 5.

**Between full reads, re-read these three before every reply.** They are small, and they are
where drift shows first: `learned/corrections.md`, `learned/working-style.md`, and the note
for the game currently being worked on (if `games/<Game Name>.md` exists).

If the user says "reread", that overrides everything above and is trigger zero.

Why this exists: a skill can be invoked once and then quietly stop being followed, with no
error anywhere — the reply still looks confident. Triggers tied to observable events (a push,
a failure report, a pasted log) fire exactly when drift is most expensive, and they fire without
the user having to notice anything.
