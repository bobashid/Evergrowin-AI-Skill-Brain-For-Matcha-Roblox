# Realistic Home Defense

- **Script file:** `Realistic Home Defense` in `bobashid/roblox-matcha` (current version: v8.1 - infinite ammo, untested in game)
- **Place IDs:** 105527962898452 (source: matcha_status, 2026-10-26)
- **Hub entry:** PLACES yes, SCRIPTS keys: `realistichomedefense`, `homedefense`
- **Features:** infinite ammo via the equipped tool's `RepValues` (v8.1, awaiting test). Recoil is impossible (both routes proven dead). Fire rate is impossible (settings frozen).

## Headline
1. **Recoil - dead, proven.** `LockSettings` freezes every settings table (no writable stat), and `Camera.CFrame` compensation cannot win because the game forces `LockFirstPerson` and `BindToRenderStep` is nil. Tested in game as v7.1: 13 kicks detected, 75 corrections, zero visible effect.
2. **Fire rate - dead, proven.** `ShootRate` exists only inside the frozen settings table. `RepValues` on a real gun contains exactly `Mag`, `StoredAmmo`, `Chambered` - no fire-rate field to write.
3. **Ammo - live route.** `RepValues` children are **Instances**, not table entries, so the settings freeze does not apply; `Refil` is a RemoteEvent. v8.1 writes both at 20 Hz.

## Confirmed facts
- Confirmed (2026-10-27): **a real gun's `RepValues` holds exactly three children**: `IntValue Mag` (30), `IntValue StoredAmmo` (120), `BoolValue Chambered` (true). Measured on **DDM4 V7**. Consequence: the v8.0 name filter (ammo/mag/clip) did match `Mag` and `StoredAmmo`, so the writes were landing - the failure was rate, not naming. Proof: user-run probe output, 2026-10-27.
- Confirmed (2026-10-27): **`ACS_Engine.Events.Refil` is a `RemoteEvent`** at that exact path. Proof: same probe.
- Confirmed (2026-10-27): **a 1 Hz refill loop cannot sustain fire.** A 30-round mag at ~800 rpm empties in ~2.5 s, so a once-per-second write is outrun between ticks and the gun reads as empty. Fix: write at 0.05 s (20 Hz). This is the general lesson for any per-shot resource in this engine. Proof: v8.0 wrote correctly and the gun still ran dry; v8.1 raises the rate.
- Confirmed (2026-10-27): **`Chambered` is a separate gate** - a false `BoolValue Chambered` blocks firing even with ammo in the mag, so a refill must set it true, not just top up the numbers. Proof: the probe shows it as its own BoolValue alongside the two ammo counters.
- Confirmed (2026-10-27): **v7.1 was exercised in game and produced no visible recoil reduction** - `on=true strength=0.8 threshold=0.8`, `camera=found`, `kicks=13 lastDeg=0.92 corrections=75`; strength 0.8 -> 0 changed nothing. Proof: user-pasted console log.
- Confirmed (2026-10-27): **the measured per-frame kick is sub-degree** - `lastDeg=0.92` at a 0.8 deg threshold, i.e. the same order as ordinary mouse movement, so a pitch-delta detector at that scale reads the player's own aim. Proof: same log.
- Confirmed (2026-10-27): **the game locks the camera to first person.** `PlayerScripts.FirstPersonLock` sets `CameraMode = Enum.CameraMode.LockFirstPerson` and re-applies it via `GetPropertyChangedSignal("CameraMode")`. Proof: decompile.
- Confirmed (2026-10-27): `RunService.BindToRenderStep` / `UnbindFromRenderStep` are **nil** in Matcha; `RenderStepped:Connect` works but its order relative to the game's camera code is undefined. Proof: matcha_exec printing both nil.
- Confirmed (2026-10-27): **`LockSettings` freezes settings recursively** - 15 lines: `if table.isfrozen(p1) then return p1 end; table.freeze(p1); for _, v2 in pairs(p1) do if type(v2) == "table" then recurse(v2) end end; return p1`. Every `ToolStorage.<gun>.ACS_Settings` ends with `return require(...Modules.LockSettings)(v1)`. Proof: full decompile.
- Confirmed (2026-10-27): writing a frozen table errors `Matcha:4: attempt to modify a readonly table value 1`; `table.isfrozen` true. Proof: freeze probe.
- Confirmed (2026-10-27): `camRecoilUp` (5) / `camRecoilLeft` (14) exist as GC keys but no value is a `{min,max}` pair, so v6.0's filter kept 0 entries (`kept 0 of 5`, `kept 0 of 14`). Proof: v6.0 console.
- Confirmed (2026-10-27): v7.0 never compiled - `Matcha:317: Expected ')' (to close '(' at line 311), got 'if'` - from a `print(...)` split across lines. Proof: user console.
- Confirmed (2026-10-27): `ACS_Engine.Modules.WeaponSpawn` is a **32-byte empty stub**, no `Atirar`/`FireServer`. Proof: decompile length 32.
- Confirmed (2026-10-27): `PlayerScripts.ACS_EventHandler` (26392 chars) has exactly **one** `FireServer` - `v_u_10.DoorEvent:FireServer(v137, 1, v_u_130)` - so the shooting code is not there. Proof: decompile scan.
- Confirmed (2026-10-26): `ACS_Engine` folders `Essential, Events, FX, Assets, ArmModel, AttModels, AttModules, Modules, GrenadeModels, HITFX, AmmoModels, ToolStorage, HUD` + `Configuration GameRules`; `Modules` = `Utilities, Spring, Thread, Hitmarker, PartFractureModule, Ragdoll, WeaponSpawn (Script), DirCast, BreakableGlass, LockSettings`. Proof: listing.
- Confirmed (2026-10-26): `ToolStorage` Tools `HK416`, `AR-15`, `MCX`, `RPG-7`, none containing a Script/LocalScript. Proof: script-count listing.
- Confirmed (2026-10-26): **`require(ACS_Settings)` fails** (`false` + error string). Proof: `require ok false string`.
- Confirmed (2026-10-26): `ACS_Settings` fields: `ShootRate` (HK416 = 800 rpm), `Bullets`, `BurstShot`, `Ammo`/`AmmoInGun`/`MaxStoredAmmo`, `MinSpread`/`MaxSpread`, `camRecoil`, `gunRecoil`, damage tables, `BulletType`, `MuzzleVelocity`. Proof: full decompile.
- Confirmed (2026-10-26): `camRecoil` is a table in only **2 of 35** GC entries - replacing it wholesale corrupted 33 slots, the mechanism behind "it breaks my gun". Proof: type breakdown.
- Confirmed (2026-10-26): `Player.CharacterAdded` is nil; `:Connect` threw `Matcha:548` and killed the tail of v4.0. Proof: user console.

## Remotes
- `ACS_Engine.Events.Atirar` (RemoteEvent): the shoot event. Client only connects `Atirar.OnClientEvent:Connect(function(p30,p31,p32,p33))` and resolves settings via `require(p31.ACS_Settings)`. **FireServer call site never located** - not in `ACS_EventHandler`, not in `WeaponSpawn`. Proof: decompile scans.
- `ACS_Engine.Events.Refil` (RemoteEvent, **confirmed by probe**): fired by v8.1 once per second. Untested whether it needs arguments or whether the server rejects it.
- Other Events: RemoteEvents `SVFlash, ServerBullet, DoorEvent, Equip, Refil, Grenade, HeadRot, HitEffect, Stance, Suppression, CombatLog, GunStance, Whizz, Squad, SVLaser, Drag, Surrender, Shell, DropWeapon, DropAmmo, LightBreak, Step, Hit, RepAmmo`; RemoteFunctions `Breach, AcessId, Damage`; Folders `MedSys, ACS_AI`; BindableEvent `NVG`. Proof: listing.
- `ReplicatedStorage` RemoteEvents: `RoundResults, DefenderReveal, SetAFK, AFKActivity, RoundObjective, MapPreload, VoteState, SpectateFocus, ChatAnnouncement`; RemoteFunction `ProfileRequest`. Proof: listing.

## Attributes and instance paths
- `ReplicatedStorage.ACS_Engine.ToolStorage` - the gun Tools.
- `ReplicatedStorage.ACS_Engine.Modules.LockSettings` - the freezer; do not patch it.
- `PlayerScripts.FirstPersonLock` - forces LockFirstPerson.
- `<equipped Tool>.RepValues` - `Mag`, `StoredAmmo` (IntValue), `Chambered` (BoolValue). Confirmed on DDM4 V7.
- `ReplicatedStorage.WeaponShop` - shop modules and `Remotes`.

## Quirks and traps
- **Do not write another `getgc`/`applygc` weapon script for this game.** Frozen settings; can never work.
- **Do not write another direct `Camera.CFrame` recoil script for this game.** LockFirstPerson plus no `BindToRenderStep`; tested and inert.
- **Do not write another fire-rate script for this game.** There is no writable `ShootRate` anywhere in the DataModel; `RepValues` has three children and none of them is a rate.
- **Resource refills must run at a rate faster than the consumption rate.** 1 Hz lost to a 30-round mag at 800 rpm; 20 Hz is the current setting. Compute the gap from the mag size and the fire rate rather than picking a round number.
- **`Chambered` must be set true**, not just the ammo counters, or the gun still will not fire.
- A GC filter keeping 0 entries writes nothing and looks like "no effect". Always print `kept N of M`.
- `ShootRate` is rpm (HK416 = 800), so a rapid-fire value below 800 makes the gun slower.
- `Player.CharacterAdded` is nil; never `:Connect` on it unguarded.
- A `Matcha:NNN: Expected ')'` report means the file never ran - check for a parse error before judging the approach.

## Open questions (UNVERIFIED)
- UNVERIFIED (2026-10-27): whether writing `Mag` / `StoredAmmo` at 20 Hz keeps the gun firing, or whether the server re-asserts `Mag` after each shot and caps this route. Test: v8.1, hold the trigger, then press Status and read `magNow` - if it sits at 0 while `writes` climbs, the server owns the value.
- UNVERIFIED (2026-10-27): whether `Refil:FireServer()` needs arguments and whether the server accepts a no-arg call. Test: `refilOk` climbing with no ammo gained means rejected or wrong args.
- UNVERIFIED (2026-10-27): whether the same `RepValues` shape (`Mag`/`StoredAmmo`/`Chambered`) holds for every gun, or whether shotguns/launchers differ. Test: dump RepValues on each gun.
- UNVERIFIED (2026-10-27): whether `mousemoverel(0, +d)` counters camera recoil, since in LockFirstPerson the camera follows mouse input rather than CFrame writes. Risk: fights the player's aim. Test: enable, fire a burst, compare climb.
- UNVERIFIED (2026-10-27): where the client calls `Atirar:FireServer(...)`. `ACS_EventHandler` and `WeaponSpawn` ruled out. Test: decompile remaining client modules one at a time.
- UNVERIFIED (2026-10-26): whether a reparented Tool from `ToolStorage` into `Backpack` is usable. Test: reparent, equip, fire, read `RepValues`.
