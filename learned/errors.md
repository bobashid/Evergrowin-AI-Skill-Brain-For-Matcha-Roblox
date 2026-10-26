# Learned: error messages and what they mean

Format: `Error: "<exact text>" -> <meaning and fix>. Confirmed (YYYY-MM-DD). Proof: <source>.`

- Confirmed (2026-10-26): **two write results returning the same sha and size for two different paths** means the writes did not land as intended. Treat both as corrupt. Fix: re-read each path, compare the returned text against what was sent (spelling, duplicated sections, missing words), rewrite in full, read back again. Proof: a Ride A Dino brain push returned sha `f2a8c41d...` / 2,814 B for both `learned/patterns.md` and `learned/api-facts.md`; the read-back showed `CurriedDino` where `CarriedDino` was sent, one section duplicated, and words missing.
- Confirmed (2026-10-26): a read-back that returns only a heading, or returns another file's heading, is corruption or ordering trouble - not a successful read. Proof: same push as above, where the `api-facts.md` read returned `# Learned: patterns that worked`.
- Confirmed (2026-10-26): a stale `sha` in `create_or_update_file` fails visibly (409 / mismatch), so it is safe to write with the last known sha and retry with a fresh read if it errors. It never silently overwrites. Proof: repo writes in this session that succeeded only after a re-read sha.
