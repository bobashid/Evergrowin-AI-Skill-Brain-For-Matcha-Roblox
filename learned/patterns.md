# Learned: patterns that worked

Short, reusable, proven in a real game. Format: name, when to use, 3 to 10 lines of code or steps, proof and date. A pattern used everywhere moves into the matching reference file.

## Fire the game's own interaction instead of writing properties
Use when a game grants something through a ProximityPrompt, ClickDetector or touch. The game's own handler does the work, so it survives updates and does not desync state the way attribute writes do.
1. Find the prompt: scan `GetDescendants()` for `ClassName == 'ProximityPrompt'` plus the expected name.
2. `pcall(fireproximityprompt, prompt)`.
3. Read state back to confirm (a label changes, an item disappears, a bar starts draining).

User-stated (2026-10-26): Feed A Mosquito and Ride A Dino both work this way. Firing `MountDinosaur` starts the stamina tick with no range check. Proof: the user's Ride A Dino dump, header line "Checked this session (place 93523763376122)".

## Guard before re-applying
Every auto-loop needs a state check before the action, or the re-apply becomes the bug: it dismounts you, double-buys, or cancels what you just did.
- Ride A Dino: never fire `MountDinosaur` while already mounted, or it dismounts.

User-stated (2026-10-26), Ride A Dino dump.

## Find "yours" by attribute, not by name
`plot:GetAttribute('Owner') == LocalPlayer.UserId` beats assuming `Plot1`, `House1`, `Slot1`. Names change between servers and players; the owner attribute does not.

User-stated (2026-10-26): confirmed in Ride A Dino as `Workspace.Pens.<Plot>` with `Owner` equal to UserId.

## Multi-signal state detection when properties read nil
Matcha properties often read nil, so a single check silently reports false forever. OR several signals together.
- Mounted in Ride A Dino: a `CarriedDino` child under the character, a seat/weld named seat/mount/saddle, or `Humanoid.Sit == true`. `Humanoid.Sit` reads nil there, so the other two carry it.

User-stated (2026-10-26), Ride A Dino dump.

## Toggle state: poll the getter, do not trust the callback
Register every boolean widget in a list, poll `UI.GetValue(id)` then `UI.GetValue(name)`, and treat the getter as authoritative for a key only after it has returned a boolean (`getterWorks[key] = true`). Until then the callback is the only writer; after that the callback early-returns so the two cannot fight.

User-stated (2026-10-26): the working pattern in the user's Ride A Dino script.

## pcall every property read that can be nil
Nil-returning properties are normal in this VM, not an error case. Wrap each read in `pcall` and treat nil as "unknown", not as false.

User-stated (2026-10-26), Ride A Dino dump.

## stop() cleans up and changes nothing else
Disconnect every connection, remove the tab, clear the global. No dismount, no teleport, no restore writes unless the user explicitly asked for them.

User-stated (2026-10-26), Ride A Dino dump.
