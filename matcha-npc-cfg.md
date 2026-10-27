# Matcha NPC / bot config (JSON) - schema, defaults, validation

Matcha exports its NPC list as a JSON object (a `.cfg` export is the same data in INI form).
When the user asks for enemy/bot entries, **generate or edit this JSON** rather than prose.
User-stated (2026-10-27): "this should be added to your skill to be able to make npc/bot cfg".

## Top level

```json
{
  "npc_dir_order": [],
  "npc_map": { "<key>": { ...entry... } },
  "npc_order": [ "<key>", ... ],
  "selected_map_type": 1,
  "selected_npc_key": "<key>"
}
```

- `npc_map` keys are the entry names. `npc_order` is the display order.
- **Invariant: `npc_order` must list exactly the `npc_map` keys.** A key present in the map but
  missing from the order is invisible in the UI; an order entry with no map key is a dangling
  reference. Both were true of one config (a stray `""` entry).
- `npc_dir_order` was empty in every observed config; leave it `[]`.

## Entry fields (complete set, 27 keys)

```json
{
  "allow_multiple": true,
  "custom_color_a": 1.0,
  "custom_color_b": 0.3,
  "custom_color_g": 0.3,
  "custom_color_r": 1.0,
  "enable_box": false,
  "enable_chams": false,
  "enable_distance": false,
  "enable_esp": true,
  "enable_healthbar": true,
  "enable_name": true,
  "enable_skeleton": true,
  "enable_tracers": false,
  "head": "Head",
  "ignore_players": true,
  "is_hostile": false,
  "match_mode": 2,
  "match_string": "AKM",
  "max_health": 100.0,
  "name": "AKM",
  "npc_name_type": 0,
  "path": "AKM",
  "rig_type": 0,
  "root_part": "HumanoidRootPart",
  "team_name": "NPC",
  "torso": "Torso",
  "use_custom_color": false
}
```

That block is the working template. In the user's real config all 58 good entries used exactly
those values, with `name`, `path` and `match_string` all equal to the enemy name.

`custom_color_*` floats may arrive as `0.30000001192092896` (float32 artifact); `0.3` is
equivalent and safe to write.

## COST: every entry is a scan - keep the list small

User-stated (2026-10-27): "im also noticing that its more delayed the more people i add".

Each `npc_map` entry is a separate match predicate evaluated against the model tree, so cost
scales roughly linearly with entry count: a 65-entry list is ~65 passes where a 12-entry list
is 12. The delay is cumulative and grows with every batch of names added.

**This overturns the earlier line in this file: "a name that never spawns costs nothing."**
That was wrong. A name that never matches still costs a full scan on every update; it is pure
overhead. Retracted 2026-10-27 on the user's report above.

Rules that follow:

- **Budget 10-25 entries per profile, not 60+.** A typical map spawns only 6-14 unique NPC
  names (three `GunModels` reads on one game gave 14 / 6 / 8 unique), so most of a large list
  can never match anything on the map that is loaded.
- **One profile per map, not one union of every map.** `selected_map_type` exists for this.
  Unioning all maps is what inflates a list past usability.
- **Prefer long, specific `match_string` values.** A short fragment such as `AR` or `M9` costs
  more and risks matching unintended names. Never shorten a name to "cover more" - add the
  full name instead.
- **Add on sight, remove on absence.** Rebuild from a live probe at round start
  (`workspace:GetDescendants()` + `Humanoid` + strip `_Corpse`/`GP`) and drop entries the round
  never shows. Do not carry speculative names "just in case".
- **Do not duplicate case variants speculatively.** `DB` + `Db` is two entries for one enemy.
  Add the second spelling only after a probe shows the first failing to match.
- **Check the entry count before blaming the script.** If aim or ESP gets laggier after a
  config edit, count `npc_map` keys first; a weapon-only or menu-only script is not the cause.
- **Symptom to recognise:** input feels progressively delayed after each batch of entries,
  and removing the newest batch restores responsiveness.

## `match_mode` - the most important field

| Value | Meaning | Evidence |
|---|---|---|
| `2` | **Name Contains** - the working value | 56 of 59 entries used it, and those are the ones that hit enemies |
| `1` | unknown (one entry: `VAL`, with an empty `match_string`) | that entry matched nothing |
| `0` | unknown (the stray `""` entry) | never validated |

Treat `2` as the only confirmed mode. Set every generated entry to `2`.

## Validation checklist - run before handing a config back

1. No empty `match_string` with `match_mode` 2. An empty string under a Contains match is the
   worst case: it can match **every** model, including the local player's `Viewmodel`.
2. `match_string` must equal `name`. A mismatch (seen: `name: "Tec9"` with
   `match_string: "RPK"`) silently duplicates another entry and never covers its own name.
3. No `_Corpse` or `GP` in `match_string` or `path`. Contains already covers `AKM`,
   `AKMGP` and `AKM_Corpse`; narrowing to `p226_Corpse` matches only dead bodies.
4. No wall props as enemies: `Medkit Dispenser`, `Armor Dispenser`, `Grenade Dispenser`,
   `Random Dispencer`, `Ammo Dispencer`.
5. No `Viewmodel`, `Dummy`, `DummyTwo` - all carry a `Humanoid`, and `Viewmodel` is the
   local player's own arms.
6. `npc_order` matches `npc_map` keys exactly.
7. Visual flags consistent across entries unless the user asked otherwise. A lone entry with
   `enable_box: true, enable_distance: true, enable_skeleton: false` among 58 identical ones
   is a leftover, not a preference - call it out.
8. `ignore_players` stays `true` for games where player characters share the world with NPCs.
9. **Entry count is within budget (10-25).** If it is not, propose the trim before delivering.

## Name case and spelling

Matcha may match case-sensitively. Before delivering, compare each `match_string` against the
names actually seen in game. Real examples of drift caught this way: config said `DB` where
live models were `Db`, and `DW-15` where a live probe showed `GV-15`.

Add a second casing only when a probe confirms the first does not match - each variant is a
full extra scan (see the COST section), so a speculative pair is a self-inflicted slowdown.

## Delivery

Give the whole JSON back in one fence, ready to paste, with each entry on one line so the diff
is readable. Do not hand back prose describing the changes and no file - the user asked for a
paste. List the changes above or below the block, never inside it (JSON has no comments).
