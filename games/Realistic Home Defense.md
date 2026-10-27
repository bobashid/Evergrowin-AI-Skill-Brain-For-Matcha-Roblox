# Realistic Home Defense

- **Script file:** `Realistic Home Defense` in `bobashid/roblox-matcha` (current version: v7.1 - tested in game, no visible effect)
- **Place IDs:** 105527962898452 (source: matcha_status, 2026-10-26)
- **Hub entry:** PLACES yes, SCRIPTS keys: `realistichomedefense`, `homedefense`
- **Features:** none working. v1.0-v6.0 (GC stat edits) impossible here; v7.0 never compiled; v7.1 runs correctly and still has no visible effect.

## Headline: both script levers are locked, for proven reasons
1. **GC / `applygc` stat editing** - `ACS_Engine.Modules.LockSettings` recursively `table.freeze`s every gun settings table; a frozen write errors `attempt to modify a readonly table value`.
2. **Direct `Camera.CFrame` compensation** - the game forces `CameraMode = Enum.CameraMode.LockFirstPerson`, so the camera is rebuilt from mouse state every frame, and `RunService.BindToRenderStep`/`UnbindFromRenderStep` are **nil** so a script cannot write last in a frame. **Now confirmed in game, not just inferred** (see v7.1 result below).

There is no settings value a script can zero here. The practical answer for the user is Matcha's built-in Combat > Silent Aim / Trigger Bot, which makes the kick cosmetic rather than removing it.

## Confirmed facts
- Confirmed (2026-10-27): **v7.1 was exercised in game and produced no visible recoil reduction** - console showed `on=true strength=0.8 threshold=0.8`, `camera=found`, `kicks=13 lastDeg=0.92 corrections=75`. Detection and writes both ran, so the code is not dead; the camera write is overwritten by the camera controller before it is drawn. Dropping strength from 0.8 to 0 changed nothing visible. Proof: user-pasted console log, 2026-10-27.
- Confirmed (2026-10-27): **the measured per-frame kick is sub-degree** - `lastDeg=0.92` at a 0.8 deg threshold. That is the same order as ordinary mouse movement, so a pitch-delta detector at this scale mostly reacts to the player's own aim, not to recoil. Any future camera attempt would need a per-weapon kick magnitude measured first. Proof: same log.
- Confirmed (2026-10-27): **the game locks the camera to first person.** `PlayerScripts.FirstPersonLock` (2098 chars) sets `CameraMode = Enum.CameraMode.LockFirstPerson` and re-applies it via `GetPropertyChangedSignal("CameraMode")`. Proof: decompile.
- Confirmed (2026-10-27): `RunService.BindToRenderStep` and `UnbindFromRenderStep` are **nil** in Matcha; `RenderStepped:Connect` works but callback order relative to the game's camera code is undefined. Proof: matcha_exec printing both nil.
- Confirmed (2026-10-27): **`LockSettings` freezes settings recursively** - 15 lines: `if table.isfrozen(p1) then return p1 end; table.freeze(p1); for _, v2 in pairs(p1) do if type(v2) == "table" then recurse(v2) end end; return p1`. Every `ToolStorage.<gun>.ACS_Settings` ends with `return require(...Modules.LockSettings)(v1)`. Proof: full decompile.
- Confirmed (2026-10-27): writing a frozen table errors `Matcha:4: attempt to modify a readonly table value 1`; `table.isfrozen` true. Proof: freeze probe.
- Confirmed (2026-10-27): `camRecoilUp` (5) / `camRecoilLeft` (14) exist as GC keys but no value is a `{min,max}` pair, so v6.0's filter kept 0 entries and wrote nothing (`kept 0 of 5`, `kept 0 of 14`). Proof: v6.0 console.
- Confirmed (2026-10-27): v7.0 never compiled - `Matcha:317: Expected ')' (to close '(' at line 311), got 'if'` - from a `print(...)` split across lines. Fixed in v7.1. Proof: user console, then read-back.
- Confirmed (2026-10-27): `ACS_Engine.Modules.WeaponSpawn` is a **32-byte empty stub**, no `Atirar`/`FireServer`. Proof: decompile length 32, matches 0.
- Confirmed (2026-10-27): `PlayerScripts.ACS_EventHandler` (26392 chars) has exactly **one** `FireServer` - `v_u_10.DoorEvent:FireServer(v137, 1, v_u_130)` - so the shooting code is not there; `Atirar` is only `OnClientEvent`. Proof: decompile scan.
- Confirmed (2026-10-26): `ACS_Engine` folders `Essential, Events, FX, Assets, ArmModel, AttModels, AttModules, Modules, GrenadeModels, HITFX, AmmoModels, ToolStorage, HUD` + `Configuration GameRules`; `Modules` = `Utilities, Spring, Thread, Hitmarker, PartFractureModule, Ragdoll, WeaponSpawn (Script), DirCast, BreakableGlass, LockSettings`. Proof: listing.
- Confirmed (2026-10-26): `ToolStorage` Tools `HK416`, `AR-15`, `MCX`, `RPG-7`, none containing a Script/LocalScript. Proof: script-count listing.
- Confirmed (2026-10-26): a gun Tool carries `ACS_Settings` + `ACS_Animations`; an equipped copy carries `RepValues` (with `BoolValue Chambered`). Proof: child listing on a held Glock 18.
- Confirmed (2026-10-26): **`require(ACS_Settings)` fails** (`false` + error string). Proof: `require ok false string`.
- Confirmed (2026-10-26): `ACS_Settings` fields: `ShootRate` (HK416 = 800 rpm), `Bullets`, `BurstShot`, `Ammo`/`AmmoInGun`/`MaxStoredAmmo`, `MinSpread`/`MaxSpread`, `camRecoil` (`camRecoilUp/Tilt/Left/Right` as `{min,max}`), `gunRecoil`, `LimbDamage`/`TorsoDamage`/`HeadDamage`, `BulletType`, `MuzzleVelocity`. Proof: full decompile.
- Confirmed (2026-10-26): `camRecoil` is a table in only **2 of 35** GC entries (9 functions, 20 unknown, 4 strings) - replacing it wholesale corrupted 33 slots, the mechanism behind "it breaks my gun". Proof: type breakdown.
- Confirmed (2026-10-26): `Player.CharacterAdded` is nil; `:Connect` threw `Matcha:548` and killed the tail of v4.0. Proof: user console.
- Confirmed (2026-10-26): `WeaponShop` holds `ShopConfig, WeaponCatalog, BonusChip, ArmourCatalog, Flags, PatchCatalog, ClothingCatalog, StoreSounds, GiftCatalog, GiftMenu, CosmeticCatalog` + Folders `Remotes, ArmourModels, ClothingTemplates, CosmeticModels`; `WeaponCatalog` decompiles to 41 empty tables (populated at runtime). Proof: listing + decompile.

## Remotes
- `ACS_Engine.Events.Atirar` (RemoteEvent): the shoot event. Client only connects `Atirar.OnClientEvent:Connect(function(p30,p31,p32,p33))` and resolves settings via `require(p31.ACS_Settings)`. **FireServer call site never located.** Proof: decompile scans.
- `ACS_Engine.Events.Refil` (RemoteEvent): likely the ammo path - untested, unaffected by frozen tables.
- Other Events: RemoteEvents `SVFlash, ServerBullet, DoorEvent, Equip, Refil, Grenade, HeadRot, HitEffect, Stance, Suppression, CombatLog, GunStance, Whizz, Squad, SVLaser, Drag, Surrender, Shell, DropWeapon, DropAmmo, LightBreak, Step, Hit, RepAmmo`; RemoteFunctions `Breach, AcessId, Damage`; Folders `MedSys, ACS_AI`; BindableEvent `NVG`. Proof: listing.
- `ReplicatedStorage` RemoteEvents: `RoundResults, DefenderReveal, SetAFK, AFKActivity, RoundObjective, MapPreload, VoteState, SpectateFocus, ChatAnnouncement`; RemoteFunction `ProfileRequest`. Proof: listing.

## Attributes and instance paths
- `ReplicatedStorage.ACS_Engine.ToolStorage` - the gun Tools.
- `ReplicatedStorage.ACS_Engine.Modules.LockSettings` - the freezer; do not patch it.
- `PlayerScripts.FirstPersonLock` - forces LockFirstPerson.
- `ReplicatedStorage.WeaponShop` - shop modules and `Remotes`.

## Quirks and traps
- **Do not write another `getgc`/`applygc` weapon script for this game.** Frozen settings; can never work.
- **Do not write another direct `Camera.CFrame` recoil script for this game.** LockFirstPerson plus no `BindToRenderStep`; tested and inert in v7.1.
- If a recoil detector reports sub-degree kicks at a sub-degree threshold, it is measuring mouse movement, not recoil.
- A GC filter keeping 0 entries writes nothing and looks like "no effect". Always print `kept N of M`.
- `applygc` writes every entry carrying the key; a table into slots holding functions/strings is what broke the held gun.
- `ShootRate` is rpm (HK416 = 800), so a rapid-fire value below 800 makes the gun slower.
- `Player.CharacterAdded` is nil; never `:Connect` on it unguarded.
- A `Matcha:NNN: Expected ')'` report means the file never ran - check for a parse error before judging the approach.

## Open questions (UNVERIFIED)
- UNVERIFIED (2026-10-27): whether `mousemoverel(0, +d)` counters camera recoil, since in LockFirstPerson the camera follows mouse input rather than CFrame writes. Risk: fights the player's aim, may be ignored while the cursor is locked. Test: enable, fire a burst, compare climb.
- UNVERIFIED (2026-10-27): where the client calls `Atirar:FireServer(...)`. `ACS_EventHandler` and `WeaponSpawn` ruled out. Test: decompile remaining client modules one at a time.
- UNVERIFIED (2026-10-27): whether `Refil:FireServer()` grants ammo without server rejection. Test: fire once, read `RepValues`.
- UNVERIFIED (2026-10-26): whether a reparented Tool from `ToolStorage` into `Backpack` is usable. Test: reparent, equip, fire, read `RepValues`.
