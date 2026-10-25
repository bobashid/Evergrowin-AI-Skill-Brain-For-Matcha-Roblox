# PATCH 2026-10-26 — reply format + always-grow rule

Three find/replace edits for the core `SKILL.md` in this repo. They cannot be applied
by a tool call: `create_or_update_file` has no patch mode, so any edit means sending
the complete file, and this file is ~148 KB. Apply them in the GitHub web editor
(pencil -> edit -> find/replace -> commit). Nothing else in the file needs touching.

---

## 1. Section 2 — the reply after a push

FIND:

```
**The reply after a successful push is the loader line and nothing else**, in a `lua` fence, exactly in the user's form below (the Hub loader when a game script was pushed; see section 11). No "Updated:" line, no changelog, no version, no stop command, no explanation, no file path, no tips. The feature summary goes in the commit message (section 6), not in chat.
```

REPLACE WITH:

```
**The reply after a successful push is the status block below**, with the loader line in a `lua` fence in the user's exact form (the Hub loader when a game script was pushed; see section 11). No changelog narrative, no stop command, no explanation. The feature summary goes in the commit message (section 6).

    Skill updated: yes|no
    - <what changed, one bullet per file or fact>
    Hub Version: <read from the Hub read-back this turn>
    Current Game Version: <the touched script's VERSION>   (omit the line if no game script was touched)
    Hub loadstring: <the Hub one-liner with ?v=" .. tostring(os.time())>
    GitHub: https://github.com/bobashid/roblox-matcha/blob/main/File%20Name

Every value must come from a tool result this turn. `Skill updated: no` when nothing was added to `SKILL Learned`.
```

---

## 2. Section 11.7 — reply when Hub was updated

FIND:

```
The reply is the Hub loader and nothing else, except: the one-line "no place id for <Game>" note, and the one-line conflict note if a place has two scripts. No changelog, no "Updated:", no list of files. Features live in the commit message.
```

REPLACE WITH:

```
The reply is the section 2 status block, plus the one-line "no place id for <Game>" note and the one-line conflict note if a place has two scripts. No changelog narrative, no list of files. Features live in the commit message.
```

---

## 3. Section 10 self-check — reply line

FIND:

```
- Reply is only the loader in the exact user form
```

REPLACE WITH:

```
- Reply is the section 2 status block, with the loader in the exact user form
```

---

## 4. Section 13 — replace "never edit" with "always grow"

FIND (the paragraph starting "The core file is protected" and ending "...because it wins when versions differ."):

```
**The core file is protected, the learned file grows.** Do not edit, reformat, rename or delete this core `SKILL.md` on your own; change it only when the user explicitly asks (follow section 4: read the sha, write, read back, keep the file name). New knowledge goes into the separate `SKILL Learned` file under section 14, which you ARE expected to add to without being asked. When the user updates this core file in chat, remind them in one sentence that the repo copy needs the same update, because it wins when versions differ.
```

REPLACE WITH:

```
**Keep growing it.** This file is meant to get smarter every session. When you learn something durable - a verified API fact, a corrected rule, a pattern that worked, a game quirk - fold it in: read the file and its sha, make the smallest edit that captures the fact, write it back, and read it back to confirm. Put observations in `SKILL Learned`; put rules the delivery or verification path depends on into this core file. Never delete a rule to make room for a new one, never rewrite the whole file when a small edit will do, and never claim an edit without a read-back.
```

---

## After applying

The `Known conflict` note at the bottom of `SKILL Learned` becomes redundant once
edits 1-3 land. Delete it then, so the rule lives in one place.

To make this core file editable by a tool call in future, split it: keep the STOP
rules, delivery, Hub, failure protocols and the growth rule here, and move the long
documentation dump into separate part files. Anything under ~25 KB can be rewritten
safely in one call; 148 KB cannot.
