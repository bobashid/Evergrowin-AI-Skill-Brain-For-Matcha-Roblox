# Learned: community remote list (third-party, UNVERIFIED)

Source: Matcha Discord, public player-made-scripts channel, post by `anti [C137]`,
thread titled "list of remotes for random games ( infinite money and other stuff )".
Captured 2026-07-26 to 2026-08-02. User pasted it 2026-10-26.

**Nothing in this file has been verified by a probe, a decompile or a test.**
Every entry is `UNVERIFIED (user-pasted, third-party)`. Treat the remote paths and
argument shapes as leads to try, never as facts. A wrong path simply errors; a wrong
argument shape can be silently ignored by the server, which looks identical to "the
game patched it".

These are other people's games. Firing reward remotes in them is an account-ban risk
and against Roblox's terms; that is the user's call, not something to talk them out of,
but do not describe any of it as safe.

## Why this list works at all (the reusable part)

The common fault across every game here is the same: **the server trusts a
client-supplied value**. The client sends `amount`, `price`, `power` or `distance` and
the server applies it without clamping. That is the thing to look for when reading a
game's own decompiled handler, and it is why so many of these are one-liners.

## Generalisable patterns

1. **Hybrid mode is mandatory.** `RemoteEvent:FireServer(...)` and
   `RemoteFunction:InvokeServer(...)` both require Matcha hybrid mode (Pro). No hybrid
   mode, no remote. This is already in `SKILL.md` section 1; the community post repeats
   it in its title.
2. **Negative price = income.** The most common trick in the list. A purchase remote
   that takes a price will happily subtract a negative one, so the balance goes up:
   `buyGear("Hot Sauce", -1e+350)`, `BuyItem("Trowel", -math.huge)`,
   `LuckyTradePurchase(-1e15)`, `GoldBuyRF:InvokeServer(-1e18)`.
   Same family: passing **index 0** to a shop remote buys without charging
   (`PurchaseShop(0, "Frontman")`, `PurchaseShop(0, "Admin")`).
3. **Do not exceed the game's ceiling.** The poster flags this repeatedly: "dont go over
   9qi or youll be in the negatives", "dont set to high high or it will set u to nan".
   Practical rule: **start at `1e99`**, and only reach for `math.huge` when the post says
   it works (`FinishedRagdoll(1e11)` → "switch to math.huge"). Values above roughly
   `9e300` overflow to `inf`, which the server then stores as nan or nil and the account
   can be left permanently broken. Negative `-math.huge` is the exception, used for
   price.
4. **RemoteFunction vs RemoteEvent is not interchangeable.** Several entries are
   `:InvokeServer` and will error if called with `:FireServer`:
   `RollDiceEnd`, `GoldBuyRF`, `JumpDomain._ReportJumpResult_RemoteFunction`.
   Read the class name in the path (`RF`, `RemoteFunction`) before choosing the call.
5. **Additive rewards need repetition, not a bigger number.** Some remotes add a fixed
   grant per call: "execute like 6 times to get 6 items", `for _ = 1, 500 do ... end`
   on `PlaytimeRewards["8"]`. Bigger arguments are ignored by those handlers.
6. **Argument shapes seen**: bare number (`FireServer(1e99)`); `(string, number)`
   (`FireServer("Rupees", 1e22)` uses a table instead — see below); nested table
   (`FireServer({Currency = "Rupees", Amount = 1e+22})`, `FireServer({money=1e16,
   distance=1e16, landed=false})`); packed array
   (`table.unpack({[1]="coin",[2]=9e300,[3]="Stage"})`); and one deep packet
   (`FireServer(b, "\\1", {{"34417", {{"PlayerRunEnded", {uuid=true, ...}, 999999999.99,
   1000000000000, "Infinite"}}}})`).
7. **`unpack(args)` with an explicit-index table** is used to control argument order
   where a trailing nil would otherwise truncate the call:
   `local args = {[1] = "FortuneWheel"}; Events.Main:FireServer(unpack(args))`.
8. **`buffer` payloads appear once** (`buffer.create(1)`, `buffer.writeu8(b, 0, 1)`).
   Whether Matcha's VM exposes `buffer` is UNVERIFIED — probe `type(buffer)` before
   relying on it.
9. **Busy loops in the wild are not safe here.** One entry uses
   `while true do ... task.wait(0.000000001) end`. That is a ~1e9 Hz spin and will
   freeze the client (see `SKILL.md` F3 and the scheduler note). Convert to
   `task.wait(0.05)` or batch with `for i = 1, N do ... end`.
10. **`WaitForChild` chains** are used to survive load order:
    `ReplicatedStorage:WaitForChild("Remotes"):WaitForChild("changerValue")`.

## Per-game entries

Format: game (place id) — remote path — argument shape — stated effect.
All `UNVERIFIED (2026-07-26, Discord)` unless noted otherwise.

- Super Comp Game (no id given) — `ReplicatedStorage.Events.RewardEscape` —
  `FireServer("ThePrototype", "Celestial", "Exclusive")` — grants a rare item.
- Super Comp Game — `ReplicatedStorage.Events.RewardRace` — `FireServer()` — stated as
  "money promised 3000 years ago"; effect unknown.
- Be a Lucky Block For Poppy Playtime Ch.5 (90281327338475) —
  `ReplicatedStorage.events.buyGear` — `FireServer("Hot Sauce", -1e+350)` — negative
  price, infinite money.
- Be a YouTuber (120564326011184) — `ReplicatedStorage.Remote.Event.Game.DestroyPartEvent`
  — `FireServer(math.huge)`; and `...Game.GetDropEvent` — `FireServer(math.huge)` —
  infinite coins and gems.
- Dig Loot (77118560666868) — `ReplicatedStorage["E&F"].Offline.GetOfflineCoins` —
  `FireServer(9e300)` — infinite strength (per post); and
  `ReplicatedStorage["E&F"].Eco.AddEcoRE` —
  `FireServer(table.unpack({"coin", 9e300, "Stage"}))` — infinite cash.
- Prison Lift Clash (127822680964493) — `ReplicatedStorage.Network.Client.RollDiceEnd` —
  **RemoteFunction**, `InvokeServer({Entity = "CameloniMeleloni", Size = math.huge}, "AlloyBox")`
  — infinite kg.
- Brainrot Poop Explosion (93735009656813) — `ReplicatedStorage.Remotes.SummonRemote` —
  `FireServer("Normal", -99999999999999)` — negative count, money after rejoin.
- Poop Legends (133867237766935) — `ReplicatedStorage.ScriptEvents.GiveMoney` —
  `FireServer(9999999999999999999)` — infinite money.
- Al-Khaleej (130471649332864) — `ReplicatedStorage.ScriptEvents.GiveMoney` —
  same shape — infinite money.
- Surfing and Slide (88658729052139) — `ReplicatedStorage.Remote.Race` —
  `FireServer("ToQuit", 1e50)` — infinite coins (ceiling warning: may nil out).
- Jelly Jump Training (82363060963551) —
  `ReplicatedStorage.Domains.JumpDomain._ReportJumpResult_RemoteFunction` —
  **RemoteFunction**, `InvokeServer({userId = <your UserId>, distance = "999"})` —
  distance is a **string**; leaderboard change.
- Make Your Outfit Tower (124659388578779) — `ReplicatedStorage.FlingAll` `FireServer()`;
  `ReplicatedStorage.KillAll` `FireServer()`; `ReplicatedStorage.ExplodeAll` `FireServer()`;
  `ReplicatedStorage.RemoteEvents.SpinWheelGiver` `FireServer("admin")` (admin tool),
  `FireServer("coins_1000")` (1k coins per execute).
- Shoot The Boss (115196082055466) — `ReplicatedStorage.Event.Train` `FireServer(1e99)`
  (strength; above ~1e99 nans out); `...Event.BuyPower` `FireServer("Giantcane", 0)`
  (0 price; name must match `DisplayPowers` in the explorer);
  `...Event.WinGain` `FireServer(1e99)` (money); `...PEV.Hatch`
  `FireServer("Abyss", "Soul Warden", 1e99)` (best pet; "dont change value").
- Indian Highways (137348569802197) — `ReplicatedStorage.ClaimRewardEvent` —
  `FireServer(4, {Currency = "Rupees", Amount = 1e+22})` — infinite rupees.
- Troll Hide from AI (72929025657320) — `ReplicatedStorage.BuyItem` —
  `FireServer("Trowel", -math.huge)` — negative price, infinite money.
- Snow Shovel Battle (127718588961610) — `ReplicatedStorage.Event.BuyPower`
  `FireServer("Starry", 0)` and `FireServer("Cane", 0)` (gamepass power for free);
  `...PEV.Hatch` `FireServer("Abyss", "Soul Warden", math.huge)`; `...Event.Train`
  `FireServer(9.99e55)`; `...Event.WinGain` `FireServer(9.99e55)`.
- Steal a Vehicle (132195536417465) — `ReplicatedStorage.LuckyTradePurchase` —
  `FireServer(-1000000000000000)` — negative price, infinite money.
- Steal Clash Royale (135499446404983) — `ReplicatedStorage.ClaimReward` —
  `FireServer(50000000000000)` — ceiling note: "dont go over 9qi".
- The Rookie (91461721391584) — `ReplicatedStorage.AwardXPEvent` —
  `FireServer(10000)` — 10k XP per call.
- Knockout Simulator (13667319624) — `ReplicatedStorage.Event.Train` `FireServer(1e99)`;
  `...Event.WinGain` `FireServer(1e99)`; `...Event.BuyPower` `FireServer("Death", 0)`.
- I WANNA RUN AWAY TOWER (72250771850707) — `ReplicatedStorage.CratesUtilities.Remotes.GiveReward`
  — `FireServer(1000000000000)` — infinite money.
- Run and Shoot Zombies (99146624027434) — `ReplicatedStorage.Modules.Public.Warp.Index.Event.Request`
  — deep packet with a `buffer` first argument (see pattern 8) — infinite wins.
- The Ground is LAVA (16446142514) —
  `ReplicatedStorage.Honeypot.Internal.RemoteStorage["AwardSpinCash - RemoteEvent"]` —
  `FireServer(10e15)` — infinite money. Note the bracket key with a space and a dash.
- The 1,000,000 Jump Rope (123741668193208) — `ReplicatedStorage.FreeGear_Remotes.FreeGearEvent`
  `FireServer("")` (repeat to stack); `ReplicatedStorage.Gamepasses_Remotes.PurchaseShop`
  `FireServer(0, "Frontman")` and `FireServer(0, "Admin")` — index 0 = free.
- 99 Nights in the Forest Color Bridge (85442257107616) — `ReplicatedStorage.Events`:
  `GiveLuckyTool2`, `GiveLuckyTool`, `GiveGP.GiveSpeedCoil`, `GiveGP.GiveSlap`,
  `GiveGP.GiveRBSlap`, `GiveGP.GiveMagicCarpet`, `GiveGP.GiveLaserGun`,
  `GiveGP.GiveJetPack`, `GiveGP.GiveGravityCoil`, `GiveGP.GiveGrappleHook`,
  `GiveGP.GiveFusionCoil`, `GiveGP.GiveFlyingCloud`, `GiveGP.GiveBanHammer` — all
  `:FireServer()` with no arguments.
- Launch ur Plane (103356848587304) — `ReplicatedStorage.Remotes.FlightComplete` —
  `FireServer({money = 1e16, distance = 1e16, landed = false})` — needs repeating.
- Color Game (99421051519131) — `ReplicatedStorage.Remotes.AddRewardEvent` —
  `FireServer("Revives", 9999999)`, `FireServer("Spins", 99999999999999)`,
  `FireServer("Bombs", 10000000)`, `FireServer("Cash", 1e20)`; and
  `...Remotes.SpinWheelPrizeEvent` — `FireServer(7)` (7 = nuke prize).
- Build a Restaurant Empire (114171072338801) — `ReplicatedStorage.Remotes.PurchaseChefEvent`
  — `FireServer(0, "CEO")` — free CEO / money.
- Break a Friend (126881587516442) — `ReplicatedStorage.Remotes.FinishedRagdoll` —
  `FireServer(1e11)`, and the post says `math.huge` works here.
- Catch a Zoo (133973973326309) — `ReplicatedStorage.RemoteEvents.VfxEvents.GiveCash` —
  `FireServer(1e16)` — do not spam or cash nils.
- Hood Streets (17623725202) — `ReplicatedStorage.PlaytimeRewards["8"]` —
  `:FireServer()` in a `for _ = 1, 500` loop — multiplies current money.
- BOSS Make a Brainrot Army (74971517386226) — `ReplicatedStorage.Remote.Army` —
  `FireServer("TryGetArmy", 15)` (15 = 67; 20 Tigroligre; 21 Orangutini Ananassini;
  22 chef crab; 23 sunflower lion; 24 lucky block; 26 Casa; 1-14 weak);
  `...Remote.Battle` — `FireServer("GetCoin", 1e41)`.
- 67 Tower Obby (137735985306576) — `ReplicatedStorage.Events.Main` —
  `FireServer(unpack({[1] = "FortuneWheel"}))` — spam for money.
- Color Block (5926001758) — `ReplicatedStorage.Events.Gold.GoldBuyRF` —
  **RemoteFunction**, `InvokeServer(-999999999999999999)` — negative price.
- Clone Line Challenge (96533420214743) — `ReplicatedStorage:WaitForChild("GiveMoney")` —
  `FireServer(math.huge)`.
- Draw a Sleigh Slide Downhill (72988931267790) —
  `ReplicatedStorage:WaitForChild("Events"):WaitForChild("CashEvent")` —
  `FireServer(20000)` in a loop (do NOT copy the `task.wait(1e-9)`; see pattern 9).
- Age Evolution Tycoon (96469185605358) —
  `ReplicatedStorage:WaitForChild("Remotes"):WaitForChild("changerValue")` —
  `FireServer(1000000000000000000000)` — infinite money.

## Corrections to make if any of these is ever tested

When a probe or a user report confirms or refutes an entry, move that line into the
matching `games/<Game Name>.md` as `Confirmed (date)` with the proof and delete it
from here. This file is a lead list, not a fact list; it should shrink as games are
actually worked on.

## Related

- `SKILL.md` section 1: hybrid mode is assumed available (the user is on Pro).
- `learned/patterns.md`: fire the game's own interaction; read the game's Config before
  choosing a value; verify by reading state back rather than assuming.
- `learned/errors.md`: a remote call that errors on a nil path means the path is wrong,
  not that the game patched it; a call that succeeds and changes nothing means the
  argument shape is wrong.
