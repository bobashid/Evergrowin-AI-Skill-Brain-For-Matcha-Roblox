# Violence District

Place id **unknown** (the premade script does not carry one; get it with `matcha_status` while in game and
pin it in `Hub` `PLACES`). `game.Name` not recorded. No script of ours exists yet - everything below is
**learned from a premade community script the user pasted on 2026-11-03**, so it is labelled
User-stated and has NOT been independently probed. Probe each fact before building on it.

## Structure (User-stated)

- `workspace.Map` is the round map. `workspace.Map.resources` exists as a sub-folder.
- `workspace.Lobby` exists.
- `ReplicatedStorage.Round` carries a **`MatchId`** attribute.
- `ReplicatedStorage.Perks` is a folder of perk entries (names are the display names).
- Killer type is read from `player:GetAttribute("SelectedKiller")` (values seen in the script:
  `Slasher`, `Veil`).
- Teams are named **`Killer`** and **`Survivors`**.
- Static rigs that must never be boxed: `AdminPunchRig`, `KillVictimPreview`, `The Hollow Menu`.

## LocalPlayer attributes (User-stated)

`CarryCapacity`, `CarryWeight`, `UpgradeStamina`, `InLobby`, `InMatch`, `SecondLife`,
`CourtLock`, `IsEntity`, `platform` (the skillcheck GUI name flips when this is `"Console"`).

## Generators (User-stated)

- Model attributes: **`RepairProgress`** (number, 0..100), **`PlayersRepairingCount`** (number),
  plus `Completed`, `Regressing`, `ProgressPaused` booleans. Some generators only carry `Progress`.
- Located in a `Folder` under `Map` whose name contains `generator` or `gens`, or under
  `Map.resources`; fall back to scanning `Map` children one and two levels deep.
- Body part for the box: `HitBox`, else `RootPart`, else `GeneratorBody`, else the first BasePart.
  Skip parts whose name contains `hitbox`, `point`, `anchor`, `handle`, `invis`.
- State derivation: `prog >= 100 or Completed` -> done; `Regressing` -> regress;
  `ProgressPaused` -> paused; `PlayersRepairingCount > 0` -> active; else idle.

## Windows (User-stated)

- Parts under `Map` whose name contains `window`.
- If any of them is named exactly `bottom`, use ONLY the `bottom` parts (one entry per window
  instead of one per pane).

## Pallets (User-stated)

- Folder `Pallets` or `Pallet` directly under `Map` (or under `Map.resources`).
- Each child is a model holding `Intact.Pallet` and `Broken.Pallet_Broken`.
- **Dropped state = `Intact.CanCollide == false`.** The script debounces it: two consecutive
  reads must agree before the state flips, because a single read can catch mid-animation.
- When dropped, draw the `Broken` part; otherwise the `Intact` part.

## Bear traps (User-stated)

- `Map.BearTrap` / `Map.BearTraps` (also searched with `FindFirstChild("BearTrap", true)`).
- Each trap instance carries an **`Armed`** boolean attribute. `false` = sprung.
- A sprung trap's victim can be read from the trap instance with one of `TrappedPlayer`,
  `TrappedUser`, `PlayerName`, `Player`, `Occupant`, `User`, `Caught` (the script tries each).
- Independent signal: a survivor's character carries **`char:GetAttribute("Trapped") == true`**,
  matched to the nearest sprung trap within ~15 studs.

## Killer / survivor attributes (User-stated)

- Killer character: `TerrorRadius`, `ChaseTargetUserId`, `BloodLust`, `MorphedKiller`, `Butcher`,
  `Beartraps`. Any one of those present means "this model is a killer".
- Killer swing window: **`LowerBodyStabilize`** flag, and **`AC_AllowedSpeed`** a number that sits
  in **20..40** during the swing. The band appears a poll earlier than the stab flag, so the band
  alone is the useful trigger.
- Survivor character: `Knocked`, `IsHooked`, `IsCarried`, `Trapped` booleans.
- Client-written self attributes: `ParrySpent`, `ParryLands`, `Parry`, `RoundScriptsActive`,
  `overridelookscript`, `nogui`. `ParryLands == 1` about 350 ms after a press is the game's own
  "the parry connected" flag.
- `char.CheckInterractable` is a child with action attributes: `action`, `isVaulting`, `isSliding`,
  `isDroppingPallet`, `isRepairing`, `isHealing`, `isUnhooking`, `isExiting`. Any of them `true`
  means the character is busy.

## Killer abilities (User-stated, from the HUD table)

| Killer | Attribute | Label | Timing |
|---|---|---|---|
| Slasher | `LakeMist` | Invis | 13 s active, 60 s cooldown |
| Slasher | `Pursuit` | Rage | 20 s active, 80 s cooldown |
| Veil | `Spears` | Spears | counter, ~15 s recharge |
| Veil | `special` | Charge | readyFlag, 30 s cooldown |
| Veil | `spearmode` | (bool) | read to know a spear is equipped |

## Skillcheck (User-stated) - needs Unsafe LuaU

- GUI: `LocalPlayer.PlayerGui.SkillCheckPromptGui.Check`, children **`Line`**, **`Goal`**,
  and for the butcher trap **`TrapGoal`**, **`TrapPerfect`**.
- Angle is read from memory at **offset `0xD0`** from each instance's `Address`, as three floats:
  `angle, sin, cos` (validate `sin^2 + cos^2 == 1` and cross-check against `math.rad(angle)`).
- Generator window is `goal+102 .. goal+116`, neutral up to `goal+159`.
- Trap window is `phase 270..286` perfect, up to `308` still good, where `phase = line - base` and
  `base` is the start angle (aligned from the written goal with `goal - 168`).
- The prompt names swap when `LocalPlayer:GetAttribute("platform") == "Console"`
  (`SkillCheckPromptGui-con` first).

## Basketball minigame (User-stated)

- `LocalPlayer.PlayerGui.Basketball.Meter.TextLabel.Text` holds the grade
  (`SLIGHTLY EARLY` / `SLIGHTLY LATE`).
- `LocalPlayer:GetAttribute("CourtLock")` flips true when a shot starts.
- Perfect release is ~`BB_METER_FILL = 0.5` of the hold, auto-tuned +-4 ms from the grade.

## Veil spear throw (User-stated)

- Ballistic solve: `speed = clamp(hold * 180, 1, 180)` studs/s, gravity **98.1**, and the launch
  origin is `camera + dir * 3` (re-solved a few times for the offset).
- Camera rotation is written in memory: 9 floats at the offset that matches `LookVector` and
  `Position` (the script scans `0x40..0x400` for it, starting at `0xD8`).

## Cautions

- **`Players:GetPlayers()` and `Players:GetChildren()` return empty mid-round** (User-stated), so
  the script finds killers in `Workspace` by attributes instead. Do not assume the Players service
  is a reliable list here.
- `LocalPlayer`'s wrapper goes stale (known VM behaviour), and `Camera.CameraSubject` is often a
  `CameraBlock` Part rather than the character.

## Open questions

- Place id, and `game.Name`.
- Whether any of the above still matches the live build - none of it was probed.
