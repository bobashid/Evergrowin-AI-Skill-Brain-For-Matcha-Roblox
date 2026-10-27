# Realistic Home Defense

- **Script file:** `Realistic Home Defense` in `bobashid/roblox-matcha` (current version: v7.0 - camera compensation)
- **Place IDs:** 105527962898452 (source: matcha_status, 2026-10-26)
- **Hub entry:** PLACES yes, SCRIPTS keys: `realistichomedefense`, `homedefense`
- **Features:** v7.0 = no recoil, implemented at the camera. v1.0-v6.0 (no spread / rapid fire / ammo / weapon giver / GC no recoil) all failed and are retired.

## Headline: stat editing cannot work here, camera editing can be tried
`ACS_Engine.Modules.LockSettings` recursively `table.freeze`s every gun settings table before use, and a frozen write raises `attempt to modify a readonly table value`. So no `getgc` / `applygc` / `setgc` weapon stat mod can ever take effect in this game. The remaining client-side lever is the camera: recoil is applied to `Camera.CFrame`, and `Camera.CFrame` is writable.

## Confirmed facts
- Confirmed (2026-10-27): **`LockSettings` freezes settings recursively.** 15-line decompile: `if table.isfrozen(p1) then return p1 end; table.freeze(p1); for _, v2 in pairs(p1) do if type(v2) == "table" then recurse(v2) end end; return p1`. Every `ToolStorage.<gun>.ACS_Settings` ends with `return require(...Modules.LockSettings)(v1)`. Proof: full decompile of `ACS_Engine.Modules.LockSettings`.
- Confirmed (2026-10-27): writing a frozen table errors `Matcha:4: attempt to modify a readonly table value 1`; `table.isfrozen` returns true. Proof: matcha_exec freeze probe.
- Confirmed (2026-10-27): `camRecoilUp` (5 entries) and `camRecoilLeft`/`camRecoilRight` (7-14 entries) exist as GC keys but **none of their values is a `{min,max}` pair**, so v6.0's pair filter kept 0 entries and wrote nothing (`kept 0 of 5`, `kept 0 of 14`, `scan done: 0 entries kept`). Proof: v6.0 console output.
- Confirmed (2026-10-27): `ACS_Engine.Modules.WeaponSpawn` is a **`Script` of 32 bytes** (an empty stub) - it contains no `Atirar` or `FireServer` reference. So it is not the firing call site. Proof: decompile length 32, matches 0.
- Confirmed (2026-10-27): `ACS_Engine.Modules` has 10 children: `Utilities, Spring, Thread, Hitmarker, PartFractureModule, Ragdoll, WeaponSpawn (Script), DirCast, BreakableGlass, LockSettings`. `ACS_Engine.Essential` holds one `LocalScript TeamTag`. Proof: matcha_exec listing.
- Confirmed (2026-10-27): `PlayerScripts.ACS_EventHandler` (26392 chars) contains exactly **one** `FireServer` - `v_u_10.DoorEvent:FireServer(v137, 1, v_u_130)` - so the shooting code is not there. `Atirar` is only connected via `OnClientEvent`. Proof: decompile line scan.
- Confirmed (2026-10-26): the game runs an ACS weapon engine. `ReplicatedStorage.ACS_Engine` contains Folders `Essential, Events, FX, Assets, ArmModel, AttModels, AttModules, Modules, GrenadeModels, HITFX, AmmoModels, ToolStorage, HUD` plus `Configuration GameRules`. Proof: matcha_exec listing.
- Confirmed (2026-10-26): `ACS_Engine.ToolStorage` holds the gun Tools `HK416`, `AR-15`, `MCX`, `RPG-7`, and **none of them contains a single Script or LocalScript**. Proof: matcha_exec script-count listing.
- Confirmed (2026-10-26): a gun Tool carries `ModuleScript ACS_Settings`, `ModuleScript ACS_Animations`; an equipped copy carries a `RepValues` folder with a `BoolValue Chambered`. Proof: matcha_exec child listing on a held Glock 18.
- Confirmed (2026-10-26): **`require(ACS_Settings)` fails** - `pcall(require, s)` returned `false` plus an error string. Proof: matcha_exec printing `require ok false string`.
- Confirmed (2026-10-26): `ACS_Settings` defines `ShootRate` (HK416 = 800 rpm), `Bullets`, `BurstShot`, `Ammo`/`AmmoInGun`/`MaxStoredAmmo`, `MinSpread`/`MaxSpread`, `camRecoil` (`camRecoilUp/Tilt/Left/Right` as `{min,max}`), `gunRecoil`, `LimbDamage`/`TorsoDamage`/`HeadDamage`, `BulletType`, `MuzzleVelocity`. Proof: full decompile of `ToolStorage.HK416.ACS_Settings`.
- Confirmed (2026-10-26): **GC keys cannot be intersected by `addr`.** 35 `camRecoil` -> 35 unique addrs, 17 `ShootRate` -> 17 unique addrs, overlap **0**. Proof: `addrOverlap camRecoil/ShootRate 0`.
- Confirmed (2026-10-26): **`camRecoil` is a table in only 2 of 35 entries** (9 functions, 20 unknown, 4 strings, 2 tables). Replacing it wholesale corrupted 33 slots - the mechanism behind "it breaks my gun". Proof: type-breakdown print.
- Confirmed (2026-10-26): `getgc` costs ~6 s per key; a periodic rescan is itself the lag. Proof: timed probes.
- Confirmed (2026-10-26): `Player.CharacterAdded` is nil here; `lp.CharacterAdded:Connect` threw `Matcha:548: attempt to index nil with 'Connect'` and killed the tail of v4.0. Proof: user-pasted console log.
- Confirmed (2026-10-26): `ReplicatedStorage.WeaponShop` holds ModuleScripts `ShopConfig, WeaponCatalog, BonusChip, ArmourCatalog, Flags, PatchCatalog, ClothingCatalog, StoreSounds, GiftCatalog, GiftMenu, CosmeticCatalog` plus Folders `Remotes, ArmourModels, ClothingTemplates, CosmeticModels`. Proof: matcha_exec listing.
- Confirmed (2026-10-26): `WeaponCatalog` decompiles to arrays of **empty tables** (`Weapons` 41 x `{}`); populated at runtime. Proof: decompile.

## Remotes
- `ACS_Engine.Events.Atirar` (RemoteEvent): the shoot event. Client side only connects `Atirar.OnClientEvent:Connect(function(p30,p31,p32,p33))` and resolves settings via `require(p31.ACS_Settings)`. **The FireServer call site has never been located** - not in `ACS_EventHandler`, not in `WeaponSpawn` (empty stub). Proof: decompile scans.
- `ACS_Engine.Events.Refil` (RemoteEvent): likely the ammo refill path - untested.
- Other `ACS_Engine.Events` members: RemoteEvents `SVFlash, ServerBullet, DoorEvent, Equip, Refil, Grenade, HeadRot, HitEffect, Stance, Suppression, CombatLog, GunStance, Whizz, Squad, SVLaser, Drag, Surrender, Shell, DropWeapon, DropAmmo, LightBreak, Step, Hit, RepAmio`-style names plus `RepAmmo`; RemoteFunctions `Breach, AcessId, Damage`; Folders `MedSys, ACS_AI`; BindableEvent `NVG`. Proof: matcha_exec listing.
- `ReplicatedStorage` RemoteEvents: `RoundResults, DefenderReveal, SetAFK, AFKActivity, RoundObjective, MapPreload, VoteState, SpectateFocus, ChatAnnouncement`; RemoteFunction `ProfileRequest`. Proof: matcha_exec listing.

## Attributes and instance paths
- `ReplicatedStorage.ACS_Engine.ToolStorage` - the gun Tools.
- `ReplicatedStorage.ACS_Engine.Modules.LockSettings` - the freezer. Read-only in practice; do not try to patch it.
- `ReplicatedStorage.WeaponShop` - shop modules and `Remotes` folder.
- No attribute reads were probed on this game yet.

## Quirks and traps
- **Frozen settings mean GC stat mods are dead here.** Do not write another `getgc`/`applygc` weapon script for this game. Say this first if a future request asks for one.
- A GC filter that keeps 0 entries writes nothing and silently looks like "no effect". Always print `kept N of M` and treat `kept 0` as "nothing happened".
- `applygc` writes into every entry carrying the key; a table written where 33 of 35 entries held a function/string/userdata is what broke the held gun in v1.0-v5.0.
- `ShootRate` is rpm (HK416 = 800), so a rapid-fire value below 800 makes the gun slower.
- `Player.CharacterAdded` is nil; never `:Connect` on it unguarded - a runtime error kills the rest of the chunk silently.
- `ACS_EventHandler` does not fire `Atirar`; scanning PlayerScripts for it finds only the OnClientEvent handler.
- v7.0 touches no game table at all, so it cannot break the gun the way v1.0-v5.0 did.

## Open questions (UNVERIFIED)
- UNVERIFIED (2026-10-27): whether camera pitch compensation cancels the visible kick. Test: flip No Recoil, fire a burst, press Status - `kicks detected` should be above 0 and the camera should not climb; tune Strength down if aim drags.
- UNVERIFIED (2026-10-27): whether the recoil kick per frame is large enough to separate from normal mouse movement by a pitch-delta threshold. Test: the Kick threshold slider - if normal aiming triggers kicks, raise it; if firing triggers none, lower it.
- UNVERIFIED (2026-10-27): where the client code that calls `Atirar:FireServer(...)` lives. `ACS_EventHandler` and `WeaponSpawn` are both ruled out. Test: decompile the remaining client modules by name, one per call.
- UNVERIFIED (2026-10-27): whether `Refil:FireServer()` grants ammo without server rejection. Test: fire it once with no args, read `RepValues` back.
- UNVERIFIED (2026-10-26): whether a reparented Tool from `ToolStorage` into `Backpack` is usable. Test: reparent, equip, fire, read `RepValues`.
