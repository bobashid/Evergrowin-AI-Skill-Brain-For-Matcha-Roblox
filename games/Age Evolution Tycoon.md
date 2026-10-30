# Age Evolution Tycoon

- **Script file:** `Age Evolution Tycoon` in `bobashid/roblox-matcha` (current version: v1.0)
- **Place IDs:** 96469185605358 (source: matcha_status in-game, 2026-10-29)
- **Hub entry:** PLACES yes (v3.3), SCRIPTS keys: `ageevolutiontycoon`, `ageevolution`
- **Features:** nuke with no client cooldown, auto nuke with interval slider, optional force-money for the free try, nuke-once button

## Confirmed facts
- Confirmed (2026-10-29): the 900 s nuke cooldown is **client-side only**, in the button's LocalScript. Proof: decompile of `PlayerGui.Main.GuiFrames.Troll.Buttons.ScrollingFrame.NukeForMoney.LocalScript` shows a local `v_u_3` flag and `local v_u_9 = tick() + 900` loop; the only server call is `game.ReplicatedStorage.BuyNukeWithMoney:FireServer()` with **no arguments**.
- Confirmed (2026-10-29): the money gate is also client-side (`leaderstats.Money.Value < 1000000000`), so setting the local `Money` value above 1e9 lets the click path through; the server still decides whether it actually buys. Proof: same decompile.
- Confirmed (2026-10-29): `NukeButton` is a **robux** product, not a game-money nuke - it calls `MarketplaceService:PromptProductPurchase(LocalPlayer, 3244868123)`. Proof: decompile of `...NukeButton.LocalScript`. Product id 3244868123.

## Remotes
- `BuyNukeWithMoney` (RemoteEvent) at `ReplicatedStorage.BuyNukeWithMoney`: args `none`; effect `launches the nuke for 1e9 game money`; cooldown `client-side 900 s only`. Proof: decompile 2026-10-29.
- `TrollPanelRemote` (RemoteEvent) at `ReplicatedStorage.RemoteEvents.TrollPanelRemote`: not probed. Proof: remote dump.

## Attributes and instance paths
- `game.Players.<you>.PlayerGui.Main.GuiFrames.Troll.Buttons.ScrollingFrame.NukeForMoney` - the game-money nuke button (ImageButton).
- `game.Players.<you>.PlayerGui.Main.GuiFrames.Troll.Buttons.ScrollingFrame.NukeButton` - the robux nuke button (ImageButton).
- `game.Workspace.Nuke All` - the Model that carries the nuke effect.
- Player money lives at `leaderstats.Money` (a number Value).

## Quirks and traps
- Firing `BuyNukeWithMoney` from outside the button skips the 900 s local timer entirely; that is the whole no-cooldown fix, no memory editing needed.
- Matcha has 150 RemoteEvents in `ReplicatedStorage`; most are ACS seat/throttle copies and a large `BloxbizRemotes` ad SDK. Ignore them.
- `ServerScriptService` reads empty from the client, so the server-side cooldown (if any) cannot be inspected - test the fire and read whether the nuke model appears.

## Open questions (UNVERIFIED)
- UNVERIFIED (2026-10-29): whether the server also enforces a cooldown or a real money check. Test: fire `BuyNukeWithMoney` twice in 10 s and watch for the `Nuke All` model / the nuke UI flash; if the second is rejected, the cooldown is server-side and the script cannot remove it.
