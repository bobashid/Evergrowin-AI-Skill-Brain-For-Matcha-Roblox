# Realistic Home Defense

- **Script file:** `Realistic Home Defense` in `bobashid/roblox-matcha` (current version: v5.0)
- **Place IDs:** 105527962898452 (source: matcha_status, 2026-10-26)
- **Hub entry:** PLACES yes, SCRIPTS keys: `realistichomedefense`, `homedefense`
- **Features:** no recoil, no spread, rapid fire, infinite ammo (client), weapon giver

## Confirmed facts
- Confirmed (2026-10-26): `getgamename()` returns `[UPDATE] REALISTIC HOME DEFENSE`; `matcha_status` reports `hybrid_mode: true`, so `FireServer` routes are available. Proof: matcha_status + matcha_exec print.
- Confirmed (2026-10-26): the game runs an ACS weapon engine. `ReplicatedStorage.ACS_Engine` contains Folders `Essential, Events, FX, Assets, ArmModel, AttModels, AttModules, Modules, GrenadeModels, HITFX, AmmoModels, ToolStorage, HUD` plus `Configuration GameRules`. Proof: matcha_exec listing.
- Confirmed (2026-10-26): `ACS_Engine.ToolStorage` holds the gun Tools `HK416`, `AR-15`, `MCX`, `RPG-7`, and **none of them contains a single Script or LocalScript**. Proof: matcha_exec script-count listing (`HK416 0`, `AR-15 0`, `MCX 0`, `RPG-7 0`).
- Confirmed (2026-10-26): a gun Tool carries `ModuleScript ACS_Settings`, `ModuleScript ACS_Animations`, and an equipped copy carries a `RepValues` folder with a `BoolValue Chambered`. Proof: matcha_exec child listing on a held Glock 18.
- Confirmed (2026-10-26): **`require(ACS_Settings)` FAILS here** - `pcall(require, s)` returned `false` plus an error string, never a table. The Discord "Decompile & Filter" script depends on that require succeeding, so it cannot work in this game. Proof: matcha_exec printing `require ok false string`.
- Confirmed (2026-10-26): `ACS_Settings` defines `ShootRate` (HK416 = **800, rounds per minute**), `Bullets`, `BurstShot`, `Ammo`/`AmmoInGun`/`MaxStoredAmmo`, `MinSpread`/`MaxSpread`, `camRecoil` (`camRecoilUp/Tilt/Left/Right` as `{min,max}` pairs), `gunRecoil` (same shape), `LimbDamage`/`TorsoDamage`/`HeadDamage`, `BulletType`, `MuzzleVelocity`. It returns `require(...ACS_Engine.Modules.LockSettings)(v1)`. Proof: full decompile of `ToolStorage.HK416.ACS_Settings`.
- Confirmed (2026-10-26): **GC keys cannot be intersected by `addr`.** `getgc({'camRecoil','ShootRate'})` = 52 entries: 35 `camRecoil` -> 35 unique addrs, 17 `ShootRate` -> 17 unique addrs, address overlap **0**. `addr` is the entry's own value identity, not the containing table. So "locate the real settings table with one key, write the rest only there" is impossible; v4.0/v4.1 tried exactly that and anchored on nothing. Proof: matcha_exec printing `addrOverlap camRecoil/ShootRate 0`.
- Confirmed (2026-10-26): `getgc` cost is per key (~6 s each, ~13 s for the two-key probe), so a periodic rescan is itself the lag. v4.0 rescanned every 8 s. v5.0 scans once on a button.
- Confirmed (2026-10-26): `Player.CharacterAdded` is nil here; `lp.CharacterAdded:Connect` threw `Matcha:548: attempt to index nil with 'Connect'` and killed the tail of v4.0 (no loaded print, no `_G` registration). Poll `lp.Character` instead. Proof: user-pasted console log.
- Confirmed (2026-10-26): `ReplicatedStorage.WeaponShop` holds ModuleScripts `ShopConfig, WeaponCatalog, BonusChip, ArmourCatalog, Flags, PatchCatalog, ClothingCatalog, StoreSounds, GiftCatalog, GiftMenu, CosmeticCatalog` plus Folders `Remotes, ArmourModels, ClothingTemplates, CosmeticModels`. Proof: matcha_exec listing.
- Confirmed (2026-10-26): `WeaponCatalog` decompiles to arrays of **empty tables** (`Weapons` 41 x `{}`, `Categories` 3 x `{}`) because it is populated at runtime; its `Get(id)` and `ById` are built in the module. Do not read item names from the decompile. Proof: decompile of `WeaponCatalog`.

## Remotes
- `ACS_Engine.Events.Atirar` (RemoteEvent): the shoot event; the client connects `Atirar.OnClientEvent:Connect(function(p30,p31,p32,p33))` and resolves the gun settings through `require(p31.ACS_Settings)`. Arg shape for `FireServer` not yet read. Proof: decompile of `PlayerScripts.ACS_EventHandler`.
- `PlayerScripts.ACS_EventHandler` only **receives** `Atirar` (one match, `OnClientEvent`); it never calls `FireServer` for it, and the ToolStorage guns carry no scripts, so the firing client script lives somewhere not yet located. Proof: matcha_exec scanning every PlayerScripts LocalScript for `Atirar`.
- Other `ACS_Engine.Events` members: RemoteEvents `SVFlash, ServerBullet, DoorEvent, Equip, Refil, Grenade, HeadRot, HitEffect, Stance, Suppression, CombatLog, GunStance, Whizz, Squad, SVLaser, Drag, Surrender, Shell, DropWeapon, DropAmmo, LightBreak, Step, Hit, RepAmmo`; RemoteFunctions `Breach, AcessId, Damage`; Folders `MedSys, ACS_AI`; BindableEvent `NVG`. Proof: matcha_exec listing.
- `ReplicatedStorage` RemoteEvents: `RoundResults, DefenderReveal, SetAFK, AFKActivity, RoundObjective, MapPreload, VoteState, SpectateFocus, ChatAnnouncement`; RemoteFunction `ProfileRequest`. Proof: matcha_exec listing.

## Attributes and instance paths
- `ReplicatedStorage.ACS_Engine.ToolStorage` - the source of every gun Tool (weapon giver reparents from here).
- `ReplicatedStorage.WeaponShop` - shop modules and `Remotes` folder.
- No attribute reads were probed on this game yet.

## Quirks and traps
- The equipped Tool can be gone between two probes seconds apart (one probe saw `Tool Glock 18` in the character, the next saw no Tool at all). Look in `Backpack` as well, and guard every read.
- A character `Configuration` containing `ACS_Client` appeared in one probe and read `nil` in three later probes. Treat the path as unreliable and re-resolve it instead of caching it.
- `applygc` writes the value into **every** entry carrying that key, so an unfiltered recoil write pushes a recoil table into unrelated slots and visibly breaks the held gun. Filter by **value shape**: recoil keys must hold a table with 2+ of `camRecoilUp/Tilt/Left/Right` (or the `gunRecoil*` set); numeric keys must hold a number inside the plausible range. A table into a slot that held a number is what breaks the gun; a number into a number slot cannot corrupt structure.
- `ShootRate` is rounds per minute, so a "rapid fire" slider below the stock value (HK416 = 800) makes the gun slower while looking like it did nothing.
- `Player.CharacterAdded` is nil; never call `:Connect` on it unguarded. A runtime error in a pushed script kills the rest of the chunk silently.

## Open questions (UNVERIFIED)
- UNVERIFIED (2026-10-26): whether writing `ShootRate` on every plausible number entry actually raises the gun's rate, or lands on unrelated tables that happen to hold 60-3000. Test: scan, enable Rapid Fire, fire, compare observed rate; if unchanged, read entries back.
- UNVERIFIED (2026-10-26): whether the server re-applies `ShootRate`, spread or recoil after the client writes them. Test: set `ShootRate` high, fire, and compare the observed rate against a server-side hit log.
- UNVERIFIED (2026-10-26): where the client script that calls `Atirar:FireServer(...)` lives, and its argument shape. Test: find it by name (a full `getscripts` sweep times out) and decompile that one script.
- UNVERIFIED (2026-10-26): whether a reparented Tool from `ToolStorage` into `Backpack` is usable, or the server rejects it. Test: reparent one, equip, fire, read `RepValues` back.
