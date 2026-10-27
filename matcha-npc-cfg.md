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

## Name case and spelling

Matcha may match case-sensitively. Before delivering, compare each `match_string` against the
names actually seen in game. Real examples of drift caught this way: config said `DB` where
live models were `Db`, and `DW-15` where a live probe showed `GV-15`. When unsure, add BOTH
spellings as separate entries - a name that never spawns costs nothing, a missing name loses
kills.

## Delivery

Give the whole JSON back in one fence, ready to paste, with each entry on one line so the diff
is readable. Do not hand back prose describing the changes and no file - the user asked for a
paste. List the changes above or below the block, never inside it (JSON has no comments).
