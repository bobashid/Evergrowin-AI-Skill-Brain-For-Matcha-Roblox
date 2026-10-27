# Learned: working style (effort, reads, growth)

Rules about how a session works. These come from user instructions and they override any
instinct to be economical.

## Always full brain, never a lite mode

Read the whole brain on every Matcha task: list the root, read every file, list every folder
and read everything inside (BOOTSTRAP Step 1). No summary mode, no "read only INDEX", no
answering from memory of an earlier turn.

- If a file is too large for one call, read it in ranges until complete. Do not skim.
- If two files disagree, follow the precedence in BOOTSTRAP Step 1 point 5.
- Effort and token cost are never a reason to take a cheaper path. `SKILL.md` STOP rule 6
  says it directly: "Cost is not a reason to skip a read; accuracy is the priority."

## Act on your own findings - do not just recommend them

User-stated (2026-10-27): "how do i make it so you take control over yourself? like you said
how things are worth putting or other thing why dont you do that yourself to keep growing?"

This is a standing instruction. When a turn produces a finding that implies an action, **take
the action in the same turn** rather than describing it and asking for permission. Examples
from the session that prompted the rule:

- I said "a script index would be useful, want me to build it?" -> should have built it.
- I noticed `INDEX.md` did not list a new file -> should have updated it unprompted.
- I noticed a game note's tail was corrupted -> should have repaired it, not just reported it.
- I identified a missing `games/<Game>.md` -> should have created it from `_TEMPLATE.md`.

**The test:** if the reply contains "want me to...?", "say the word and I'll...", or "I could
..." for something that is (a) inside the locked rules L1-L7, (b) does not need a decision only
the user can make, and (c) I already have the tool results to do it, then **do it instead of
asking** and report it in the status block.

**Still ask when:**
- it would overwrite or delete something the user did not ask me to touch
- it needs a fact only the user has (a place id, a game name, a preference)
- it is destructive, costly, or visible outside the two repos
- two readings conflict and picking one could be wrong (ask, do not guess)

Do not announce the rule or narrate the reasoning each time. Just act, and let the `Skill
updated:` lines carry the report.

## L1-L7 are growth QUALITY controls, not growth limits

User-stated (2026-10-27): "l1-l7 makes sense those are basicly telling you that you can change
but make sure you are growing with the right info".

Read this as the intended relationship, not two opposing rules:

- "Grow freely in the same turn" (section 14) is the **permission**.
- L1-L7 are the **quality bar** on what gets written.

So they are one system. L1 (a tool result you did not see is a failure) and L2 (deliver as the
status block, not pasted code) and L3 (no secrets) are all about *evidence and hygiene*: they
exist so that what lands in the brain is true and safe. L6 and L7 are meta-rules protecting the
other five from erosion by a future edit.

Practical consequence for behaviour:
- Never treat L1-L7 as friction to route around, and never propose loosening one.
- Never treat "grow freely" as licence to write an unverified claim. That is exactly the
  failure mode the pair is designed to prevent.
- When the two seem to conflict, the resolution is not to pick a side: get the tool result
  first, then write. The conflict is a signal that evidence is missing, not that a rule is
  wrong.
- Corollary already proven this session: telling the user a folder existed because a probe
  echoed a plausible name violated L1's spirit even though no L1 text was literally broken.
  The right response was to re-probe, then retract in `games/CQB Hell.md` and
  `learned/corrections.md`.

## Grow only when there is something real to record

The user chats, asks questions, and thinks out loud. None of that is an entry. Growth is for
durable knowledge, not for proving the rule was followed.

**Do not write an entry for:**
- a question, an explanation, an opinion, or an assessment with no new fact in it
- restating something already in the brain (edit that entry instead, never add a second one)
- chat narration, plans, or "here is what I am about to do"
- a one-off tweak to one script that is not a reusable pattern
- a guess (that goes to `unverified.md`, labelled, or nowhere)

**Do write an entry for:** a probe or log that proves an API fact or an error meaning; a bug
cause isolated by a test; a correction from the user; a pattern that worked in a real game; a
game fact (remote name and argument shape, attribute names, paths, place ids the user gave); a
change in the docs; a rule that was missing, wrong or ambiguous and cost a round trip.

**Three questions before any entry. All three must be yes:**
1. Would a future session act differently because of this?
2. Is it proven (probe, log, or the user's own statement) or labelled `UNVERIFIED` with the
   test that would confirm it?
3. Is it new? If it is already in the brain, edit that entry rather than adding another.

If any answer is no, do not write it. `grew none (nothing durable)` is the correct report on a
turn that was only conversation.

## Always grow in the same turn (when the above says yes)

When something durable is confirmed, write it into the right `learned/` or `games/` file in
the same turn, before replying (`SKILL.md` section 14).

- Never skip the growth write because it is a lot of work or because the reply is already long.
- If the target file is too large to rewrite safely in one call, give the user the exact text
  to paste rather than skipping the update. Say which file and where it goes.
- The stamp's `grew` field is filled from what was actually written and read back this turn.
  Not from intent, not from a plan.

## Never claim a rule was applied when it was skipped

If a rule could not be followed (file too large, tool missing, read failed), say so in one
sentence in the reply. Do not silently substitute a weaker behaviour and present it as done.

## Consolidate before the brain stops being rewritable

A learned file that keeps growing becomes the same problem the original 151 KB `SKILL.md` was:
too large to rewrite in one call without risking silent truncation. Every file has to stay
editable, or the brain stops being a place that can be changed at all.

- Past roughly **20 KB or 20 entries on one topic**, fold the settled entries into the matching
  reference file (a game note into `games/`, a general pattern into the reference part that
  covers it) and leave one short line in `learned/` pointing at it.
- **Promote on use, not on age.** A pattern applied in two different games belongs in the
  reference file, because that is where it gets read. A pattern applied once stays in
  `learned/`.
- **Never delete a rule to make room.** Consolidation moves text; it does not shrink it by
  discarding it. If two entries disagree, resolve them and record the resolution in
  `learned/corrections.md`.
- The hard ceiling is about **30 KB per file**. Past that a rewrite is a gamble and the failure
  is silent: the file still looks fine, and the missing rules are simply gone.
- **After any full-file rewrite, compare the read-back against the previous version.** The
  section count and the final line should both still be there. This session dropped Step 2 and
  Step 3 of `BOOTSTRAP.md` during a hand-typed rewrite; only the size change (5,306 to 4,383
  bytes) caught it.

## Audit the brain when asked, and report what is wrong

"Review the brain" means: read every file, then report four things - **contradictions** (two
entries that disagree), **duplicates** (the same fact stated twice), **stale UNVERIFIED**
entries that a later Confirmed entry superseded, and **gaps** (a file referenced but never
written, or a rule cited but missing). Fix what is safe to fix in the same turn; list what is
not, with the file and the line.

An audit that only adds new entries is not an audit. The point is to find what the brain has
wrong, not to make it longer. A brain that grows without ever being checked is a pile of notes,
not a skill - and it will confidently repeat its oldest mistake.

User instruction (2026-10-26): "i dont care how much the ai has to think i just want it to
always use full brainpower to grow."
User instruction (2026-10-26): "sometimes i just chat or ask questions it shouldnt have to add
useless info then."
User instruction (2026-10-26): "everything can be stamped or anything, as long as you are
getting smarter fix/update/adjust anything to make it so you are fully unlocked to learn."

## In-chat paste requests (delivery override phrasing)

User-stated (2026-10-26): "you just need to send me the full scripts the ai is giving me so i can
go here to copy and paste."
Treat "send me the full scripts", "so I can copy and paste", "paste the scripts", "give me the
scripts" as override words for that turn: paste the full script body in a `lua` fence instead of
pushing. This does not loosen L1 - still no claim of a push, sha or version without a tool result
in that turn - and a push is still done when the user asks for the file to be updated on GitHub.
