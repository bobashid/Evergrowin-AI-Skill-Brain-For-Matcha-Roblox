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

If any answer is no, do not write it. `Skill updated: no` is the correct report on a turn
that was only conversation.

## Always grow in the same turn (when the above says yes)

When something durable is confirmed, write it into the right `learned/` or `games/` file in
the same turn, before replying (`SKILL.md` section 14).

- Never skip the growth write because it is a lot of work or because the reply is already long.
- If the target file is too large to rewrite safely in one call, give the user the exact text
  to paste rather than skipping the update. Say which file and where it goes.

## Never claim a rule was applied when it was skipped

If a rule could not be followed (file too large, tool missing, read failed), say so in one
sentence in the reply. Do not silently substitute a weaker behaviour and present it as done.

User instruction (2026-10-26): "i dont care how much the ai has to think i just want it to
always use full brainpower to grow."
User instruction (2026-10-26): "sometimes i just chat or ask questions it shouldnt have to add
useless info then."
