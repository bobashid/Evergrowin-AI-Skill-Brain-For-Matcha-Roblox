# Learned: the dump protocol (how the brain learns from game code)

A **dump** is anything real the user pastes from the game: a decompiled handler, a prompt or remote name, an explorer path, an attribute list, a log line, or a whole script.

**A dump beats everything else.** It beats my memory of the API, any pattern from another game, and any guess based on a script that "looks similar". If a dump exists for the thing I am about to touch, use it and stop reasoning about what the API probably does.

## When to ask for a dump

Ask when one of these is true:
1. I need a name I cannot discover cheaply: a remote, a ProximityPrompt, an attribute key, a folder path, a UI element id.
2. Two sources disagree about the same name, path or signature.
3. The dump I have is truncated, or contains a merge conflict, and the missing part is the part I need.
4. The change is destructive or unverifiable: deleting a file whose bytes I cannot reproduce, overwriting a working script, a bulk rename.
5. The request maps to two different behaviours and picking wrong wastes the user's time ("auto sell" = sell from inventory, or sell by touching the sell pad?).
6. A script silently did nothing and I cannot tell which branch it failed on.

## When NOT to ask

- Names I can probe with a one-line print script the user runs in ten seconds.
- Anything I can read back from the repo myself.
- Anything the dump I already have answers.
- UI argument order when a retry ladder already works: record the answer, do not ask again for the same call.
- Obfuscated payloads. They teach nothing and copying them risks a corrupted blob.

## How to ask

One message, one question. Never a list of five things.

    To do <specific thing> I need <specific name/path/function>.
    Run this and paste the output:

    ```lua
    <a short, read-only probe that prints what I need>
    ```

    Or, if you have it, paste the game's own handler for <thing> - just that
    function, not the whole script.

Probe rules:
- Read-only: prints, `GetDescendants`, `GetChildren`, `GetAttribute`, `FindFirstChild`.
- Never fires a prompt, never writes a property, never teleports.
- Labelled line per result, so a partial paste is still readable.
- Under about 15 lines. If it needs more, print less, not more.
- Runs once and exits. No loops, no `while true`.

If the answer lives in the game's own source, ask for the handler instead of the probe. A 20-line real handler is worth more than any probe I can write.

## How to receive a dump

1. Read it fully before writing code. Do not start editing halfway through.
2. Check for truncation and merge markers: `<<<<<<<`, `=======`, `>>>>>>>`, a duplicated copy, or a paste that ends mid-token. All of those mean partial.
3. If partial, say exactly which line it stops at and what I need next. One question.
4. Extract only what the task needs. Do not rewrite the script around the dump.
5. Record what was learned in the right `learned/` file before replying.

## What to dump, best value first

1. **The game's own handler for the thing** (the roll, the buy, the sell, the stamina grant). One function, not the whole script. This is the highest-value dump there is.
2. **Verified UI signatures** - `Section`, `Toggle`, `SliderInt`, `ColorPicker`, `GetValue`, `SetValue`: which argument order actually worked and what each returns. Right now every menu carries a three-way retry ladder because the order is not confirmed.
3. **Any game where a script silently did nothing** - the print log plus the code path it should have hit. Silent no-ops are the hardest thing to fix blind.

Format for each: game name + place id + one line on what it should do + the dump + what actually happened. The "what happened" is often worth more than the code.

Leave out: obfuscated blobs (like `Command An Army`, ~76 KB of escapes), and anything containing a token or key.

## Where a dump's facts go

| What the dump proves | File |
|---|---|
| An API fact, signature, or a name that works | `learned/api-facts.md` |
| An error text and its meaning | `learned/errors.md` |
| A reusable pattern | `learned/patterns.md` |
| Something about one game | `games/<Game Name>.md` from `_TEMPLATE.md` |
| A rule the brain had wrong | fix it in place, then one line in `learned/corrections.md` |
| A hunch the dump suggests but does not prove | `learned/unverified.md`, labelled UNVERIFIED |
