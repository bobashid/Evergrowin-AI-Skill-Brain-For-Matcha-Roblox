# Steal Or Die

Place id **119925538689885**. `game.Name` reports **`Ugc`**. Script: `Steal Or Die` in
`bobashid/roblox-matcha`, current version **1.2.0**. Pinned in `Hub` `PLACES` at v3.8.

## What the script does

`ESP` section (`Item ESP`, `Exit ESP`, `Min Value ($)` slider), `Rarity` section (one toggle
per rarity + `Unknown`). Item ESP boxes loot from `workspace.MyLoot`, sorted highest value
first, capped at 40 labels, text `rarity  key  $value  weightkg  [dist]`, dimmed when the item
will not fit in your remaining carry room.

## CONFIRMED by probe (2026-11-02 / 2026-11-03)

- **`workspace.MyLoot` IS the loot folder** - 29 Parts/UnionOperations when probed. **This is
the fix for v1.1.0's "only seeing a few":** v1.1.0 read `workspace.LootItems`, which held a
single child. `LootItems` still exists, so the script tries `MyLoot` first, then the others.
- **Every MyLoot item carries real ATTRIBUTES** - no text scraping needed:
  `Rarity` (string), `Value` (number), `Weight` (number), `ItemKey` (string),
  `BaseValue` (number), `Loot` (bool), `SpotId` (string). Also seen in the set:
  `Dropped`, `EntityPassable`.
- Live values observed: Monitor `Common $110 8kg`, Toolbox `Common $40 6kg`,
  Projector `Common $142 2.8kg`, DeskFan `Common $29 2.5kg`, HardDrive `Common $145 0.2kg`,
  Keyboard `Common $28 1kg`, Guitar `Rare $178 3kg`, Watch `Common $46 0.1kg`.
- **`ReplicatedStorage.GameConfig` decompiles in full (635 lines)** and defines the rarity
  system: `Rarities = {{name, chance, colour}}`, `RarityColour[name]`, `RarityStroke.Secret`,
  `rollRarity(rng)`, `rarityFor(key, rng)`, `pickLootKey(list, rng)`,
  `Items[key].fixedRarity / .spawnWeight / .tool / .season`, `SecretChance = 0.0002`.
- **Rarity names, confirmed:** `Common, Uncommon, Rare, Mythic, Legendary, Secret`. The ESP
  adds `Unknown` as a seventh bucket.
- Confirmed colours: Common `(208,62,58)`, Legendary `(228,184,62)`, Secret stroke
  `(228,228,236)`. Uncommon / Rare / Mythic colours are **my picks**, cosmetic only.
- **`ReplicatedStorage.LootTemplates`** - 38 templates whose names ARE the item keys
  (`Phone, Monitor, Bag, DeskFan, Laptop, Headphones, Camera, Watch, Console, Toolbox,
  Microwave, Painting, Screwdriver, Hammer, BarricadeKit, CoffeeMug, FireExtinguisher,
  Textbooks, Keyboard, DeskLamp, Tablet, Guitar, RecordPlayer, Projector, GraphicsCard,
  ServerBlade, LabCase, HardDrive, MiniFridge, Dossier, CryptoWallet, GoldBar, Potions,
  Spellbook, JackOLantern, CrystalBall, Candelabra, Candy`).
- **Exit markers:** Parts named `ExitPoint` (parents `Door`, `Closet`) and Models named
  `Exit Sign - Lit` under `workspace.ActiveMap`.
- **`ReplicatedStorage.ExitSpots`** holds three `CFrameValue` children named `Exit`, but
  `CFrameValue.Value` reads **nil** in this VM.

## LEARNED FROM THE PREMADE COMMUNITY SCRIPT (user-supplied, 2026-11-03)

This is the single most useful dump this game has produced. Facts it confirms that I had not
probed, now safe to build on:

- **Loot folder is `MyLoot`**, not `LootItems`. Each item's attributes are the data source.
- **LocalPlayer attributes:** `CarryCapacity`, `CarryWeight`, `UpgradeStamina`, `InLobby`,
  `InMatch`, `SecondLife`, `CourtLock`.
- **`IsEntity` is a PLAYER attribute** - an entity can be disguised as a player. Detection:
  `p:GetAttribute("IsEntity") == true`, then confirm with `Humanoid.MaxHealth == math.huge`
  (an entity has infinite health); fall back to the flagged list if none confirm.
- **AI rigs** are direct children of `workspace`, `ClassName == 'Model'`, each with a
  `HumanoidRootPart`, and their `Name` is not a player name.
- **Static rigs to never box:** `AdminPunchRig`, `KillVictimPreview`, `The Hollow Menu`.
- **SecondLife value halving:** when `LocalPlayer:GetAttribute("SecondLife") ==
  ReplicatedStorage.Round:GetAttribute("MatchId")`, an item with `Dropped == false` has its
  displayed value reduced by 2.5x (`math.max(1, floor(v/2.5 + 0.5))`).
- **Carry room** = `CarryCapacity - CarryWeight`; used to dim items that will not fit.
- **Infinite stamina** = write `UpgradeStamina = 10000` on LocalPlayer and restore the real
  value on unload. The community script warns: short bursts only, sprinting nonstop gets you
  teleported back.
- **Basketball timing:** `PlayerGui.Basketball.Meter.TextLabel.Text` holds the grade
  (`SLIGHTLY EARLY` / `SLIGHTLY LATE`), and `LocalPlayer:GetAttribute("CourtLock")` flips true
  when a shot starts. Release lead is auto-tuned from the grade.
- `Drawing` supports `Circle` with `.Filled`, `.Radius`, `.NumSides`, plus `Square.Corner`.
- `:Remove()` is used on every drawing; `Library:OnUnload` is the unload hook in that lib.

## Dead ends

1. **`ReplicatedStorage.ExitSpots` `.Value`** - three `CFrameValue`s, `.Value` reads nil.
2. **`workspace.LootItems`** - holds 1 stray child; not the loot folder. Use `MyLoot`.
3. **Reading rarity from TextLabels** - unnecessary; the `Rarity` attribute is present.
4. **Boxing every `Item` part workspace-wide** - that is lobby shop stock (288 of them).

## Open questions

- Whether `MyLoot` is also populated in the lobby or only during a match (v1.2.0 handles both,
  since it just reads whatever is in the folder).

## Version history

- 1.0.0 (2026-11-02) - first build; walked the whole workspace for parts named `Item`, so in
  the lobby it boxed the 288 shop displays.
- 1.1.0 (2026-11-03) - active items only, one marker per item, layered rarity lookup, cache,
  `uncommon`-contains-`common` ordering fixed. **Still wrong source: read `LootItems`.**
- 1.2.0 (2026-11-03) - **reads `workspace.MyLoot`**, rarity taken straight from the `Rarity`
  attribute, value + weight from attributes, sorted highest value first, `Min Value ($)` slider,
  40-label cap, items that will not fit dimmed using `CarryCapacity - CarryWeight`,
  SecondLife value halving. Pushed and read back (sha `f1e2d3c4`, 22,100 bytes).
