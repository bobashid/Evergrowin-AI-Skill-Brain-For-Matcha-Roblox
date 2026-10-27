# Realistic Home Defense

- **Script file:** `Realistic Home Defense` in `bobashid/roblox-matcha` (current version: v7.1 - camera compensation, loads clean, no visible effect)
- **Place IDs:** 105527962898452 (source: matcha_status, 2026-10-26)
- **Hub entry:** PLACES yes, SCRIPTS keys: `realistichomedefense`, `homedefense`
- **Features:** none working yet. v1.0-v6.0 (GC stat edits) are provably impossible here; v7.0 was a syntax error; v7.1 loads but the camera route cannot win against the game's camera controller.

## Headline: two routes are dead, both for proven reasons
1. **GC / `applygc` stat editing** - `ACS_Engine.Modules.LockSettings` recursively `table.freeze`s every gun settings table, and a frozen write errors `attempt to modify a readonly table value`.
2. **Direct `Camera.CFrame` compensation** - the game forces `CameraMode = Enum.CameraMode.LockFirstPerson`, so the camera CFrame is rebuilt from mouse state by the camera controller each frame; and `RunService.BindToRenderStep`/`UnbindFromRenderStep` are **nil** in Matcha, so a script cannot register at a later render priority to write last. A one-frame correction gets overwritten before it is drawn.

## Confirmed facts
- Confirmed (2026-10-27): **the game locks the camera to first person.** `PlayerScripts.FirstPersonLock` (2098 chars) sets `v_u_3.CameraMode = Enum.CameraMode.LockFirstPerson` and re-applies it via `GetPropertyChangedSignal("CameraMode")`. Consequence: the camera is fully mouse-driven and `Camera.CFrame` is rewritten by the camera controller every frame. Proof: decompile of `FirstPersonLock`.
- Confirmed (2026-10-27): `RunService.BindToRenderStep` and `RunService.UnbindFromRenderStep` are **nil** in Matcha, so render priority cannot be controlled. `RenderStepped:Connect` works but its callback order relative to the game's camera code is undefined. Proof: matcha_exec printing both as `nil`.
- Confirmed (2026-10-27): **`LockSettings` freezes settings recursively.** 15-line decompile: `if table.isfrozen(p1) then return p1 end; table.freeze(p1); for _, v2 in pairs(p1) do if type(v2) == "table" then recurse(v2) end end; return p1`. Every `ToolStorage.<gun>.ACS_Settings` ends with `return require(...Modules.LockSettings)(v1)`. Proof: full decompile.
- Confirmed (2026-10-27): writing a frozen table errors `Matcha:4: attempt to modify a readonly table value 1`; `table.isfrozen` returns true. Proof: freeze probe.
- Confirmed (2026-10-27): `camRecoilUp` (5 entries) / `camRecoilLeft` (14) exist as GC keys but no value is a `{min,max}` pair, so v6.0's filter kept 0 entries and wrote nothing (`kept 0 of 5`, `kept 0 of 14`). Proof: v6.0 console.
- Confirmed (2026-10-27): v7.0 failed to compile - `Matcha:317: Expected ')' (to close '(' at line 311), got 'if'` - caused by `print(...)` split across lines. Fixed in v7.1, which read back clean and prints its version. Proof: user-pasted console line, then v7.1 read-back.
- Confirmed (2026-10-27): `ACS_Engine.Modules.WeaponSpawn` is a **32-byte empty stub**; it contains no `Atirar`/`FireServer`. Proof: decompile length 32, matches 0.
- Confirmed (2026-10-27): `PlayerScripts.ACS_EventHandler` (26392 chars) contains exactly **one** `FireServer` - `v_u_10.DoorEvent:FireServer(v137, 1, v_u_130)` - so the shooting code is not there; `Atirar` is only connected via `OnClientEvent`. Proof: decompile line scan.
- Confirmed (2026-10-26): `ACS_Engine` folders: `Essential, Events, FX, Assets, ArmModel, AttModels, AttModules, Modules, GrenadeModels, HITFX, AmmoModels, ToolStorage, HUD` plus `Configuration GameRules`. `Modules` holds `Utilities, Spring, Thread, Hitmarker, PartFractureModule, Ragdoll, WeaponSpawn (Script), DirCast, BreakableGlass, LockSettings`. Proof: matcha_exec listing.
- Confirmed (2026-10-26): `ToolStorage` holds Tools `HK416`, `AR-15`, `MCX`, `RPG-7`, none containing any Script/LocalScript. Proof: script-count listing.
- Confirmed (2026-10-26): a gun Tool carries `ACS_Settings` + `ACS_Animations` ModuleScripts; an equipped copy carries `RepValues` (with `BoolValue Chambered`). Proof: child listing on a held Glock 18.
- Confirmed (2026-10-26): **`require(ACS_Settings)` fails** (`false` + error string). Proof: `require ok false string`.
- Confirmed (2026-10-26): `ACS_Settings` fields: `ShootRate` (HK416 = 800 rpm), `Bullets`, `BurstShot`, `Ammo`/`AmmoInGun`/`MaxStoredAmmo`, `MinSpread`/`MaxSpread`, `camRecoil` (`camRecoilUp/Tilt/Left/Right` as `{min,max}`), `gunRecoil`, `LimbDamage`/`TorsoDamage`/`HeadDamage`, `BulletType`, `MuzzleVelocity`. Proof: full decompile.
- Confirmed (2026-10-26): `camRecoil` is a table in only **2 of 35** GC entries (9 functions, 20 unknown, 4 strings) - replacing it wholesale corrupted 33 slots, the mechanism behind "it breaks my gun". Proof: type breakdown.
- Confirmed (2026-10-26): `Player.CharacterAdded` is nil; `:Connect` on it threw `Matcha:548` and killed the tail of v4.0. Proof: user console.
- Confirmed (2026-10-26): `WeaponShop` holds `ShopConfig, WeaponCatalog, BonusChip, ArmourCatalog, Flags, PatchCatalog, ClothingCatalog, StoreSounds, GiftCatalog, GiftMenu, CosmeticCatalog` + Folders `Remotes, ArmourModels, ClothingTemplates, CosmeticModels`; `WeaponCatalog` decompiles to 41 empty tables (populated at runtime). Proof: listing + decompile.

## Remotes
- `ACS_Engine.Events.Atirar` (RemoteEvent): the shoot event. Client only connects `Atirar.OnClientEvent:Connect(function(p30,p31,p32,p33))` and resolves settings via `require(p31.ACS_Settings)`. **The FireServer call site has never been located** - not in `ACS_EventHandler`, not in `WeaponSpawn` (empty stub). Proof: decompile scans.
- `ACS_Engine.Events.Refil` (RemoteEvent): likely the ammo refill path - untested, and unaffected by the frozen tables.
- Other Events: RemoteEvents `SVFlash, ServerBullet, DoorEvent, Equip, Refil, Grenade, HeadRot, HitEffect, Stance, Suppression, CombatLog, GunStance, Whizz, Squad, SVLaser, Drag, Surrender, Shell, DropWeapon, DropAmmo, LightBreak, Step, Hit, RepAmmo`; RemoteFunctions `Breach, AcessId, Damage`; Folders `MedSys, ACS_AI`; BindableEvent `NVG`. Proof: listing.
- `ReplicatedStorage` RemoteEvents: `RoundResults, DefenderReveal, SetAFK, AFKActivity, RoundObjective, MapPreload, VoteState, SpectateFocus, ChatAnnouncement`; RemoteFunction `ProfileRequest`. Proof: listing.

## Attributes and instance paths
- `ReplicatedStorage.ACS_Engine.ToolStorage` - the gun Tools.
- `ReplicatedStorage.ACS_Engine.Modules.LockSettings` - the freezer; do not try to patch it.
- `PlayerScripts.FirstPersonLock` - forces LockFirstPerson camera mode.
- `ReplicatedStorage.WeaponShop` - shop modules and `Remotes`.

## Quirks and traps
- **Do not write another `getgc`/`applygc` weapon script for this game.** Settings are frozen; it can never work. Say this before building.
- **Do not write another direct `Camera.CFrame` recoil script for this game.** LockFirstPerson + no `BindToRenderStep` means the camera controller overwrites it.
- A GC filter that keeps 0 entries writes nothing and looks like "no effect". Always print `kept N of M`.
- `applygc` writes into every entry carrying the key; a table into slots holding functions/strings is what broke the held gun.
- `ShootRate` is rpm (HK416 = 800), so a rapid-fire value below 800 makes the gun slower.
- `Player.CharacterAdded` is nil; never `:Connect` on it unguarded.
- A `Matcha:NNN: Expected ')'` report means the file never ran - check for a parse error before judging the approach.

## Open questions (UNVERIFIED)
- UNVERIFIED (2026-10-27): whether injecting relative mouse movement (`mousemoverel(0, +d)`) counters the camera recoil, since in LockFirstPerson the camera follows mouse input rather than CFrame writes. Risk: it fights the player's own aim and may be ignored while the cursor is locked. Test: enable, fire a burst, compare the camera climb.
- UNVERIFIED (2026-10-27): where the client code calls `Atirar:FireServer(...)`. `ACS_EventHandler` and `WeaponSpawn` are ruled out. Test: decompile remaining client modules one at a time.
- UNVERIFIED (2026-10-27): whether `Refil:FireServer()` grants ammo without server rejection. Test: fire it once, read `RepValues`.
- UNVERIFIED (2026-10-26): whether a reparented Tool from `ToolStorage` into `Backpack` is usable. Test: reparent, equip, fire, read `RepValues`.
