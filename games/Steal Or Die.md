# Steal Or Die

Place id **119925538689885**. `game.Name` reports **`Ugc`**. Script: `Steal Or Die` in
`bobashid/roblox-matcha`, current version **1.0.0**. Pinned in `Hub` `PLACES` at v3.8.

## CONFIRMED by probe (2026-11-02, this session)

- **`ReplicatedStorage.LootTemplates`** - 38 templates whose names ARE the item keys:
  `Phone, Monitor, Bag, DeskFan, Laptop, Headphones, Camera, Watch, Console, Toolbox,
  Microwave, Painting, Screwdriver, Hammer, BarricadeKit, CoffeeMug, FireExtinguisher,
  Textbooks, Keyboard, DeskLamp, Tablet, Guitar, RecordPlayer, Projector, GraphicsCard,
  ServerBlade, LabCase, HardDrive, MiniFridge, Dossier, CryptoWallet, GoldBar, Potions,
  Spellbook, JackOLantern, CrystalBall, Candelabra, Candy`.
- **`ReplicatedStorage.GameConfig` is readable via `decompile`** (635 lines) and defines the
  whole rarity system: `Rarities = { {name, chance, colour}, ... }`, `RarityColour[name]`,
  `RarityStroke.Secret`, `rollRarity(rng)`, `rarityFor(key, rng)`, `pickLootKey(list, rng)`,
  `Items[key].fixedRarity / .spawnWeight / .tool / .season`, `SecretChance = 0.0002`.
- **The rarity names are `Common, Uncommon, Rare, Mythic, Legendary, Secret`** - those six
  strings are confirmed from the `prices` tables and `fixedRarity == "Secret"`. The ESP adds
  `Unknown` as a seventh bucket for items whose label could not be read.
- Confirmed colours from GameConfig: Common `(208,62,58)`, Legendary `(228,184,62)`,
  Secret stroke `(228,228,236)`. Uncommon / Rare / Mythic colours in the script are **my own
  picks**, cosmetic only (the toggles filter on the NAME).
- **World items are Parts named `Item`** - 288 of them were present in the lobby under the
  shop display models (parent e.g. `Promo Dump Bin`). A live item Part carries **no
  attributes and no children**, so the rarity must be read from an ancestor model's text
  labels (the script caches that per model `Address`, 3 s TTL).
- **`workspace.LootItems` is a Folder but is EMPTY in the lobby** - items only exist during a
  match, which is why discovery walks the whole workspace instead of that folder.
- **Exit markers:** Parts named `ExitPoint` (parents `Door`, `Closet`) and Models named
  `Exit Sign - Lit` under `workspace.ActiveMap`.
- **`ReplicatedStorage.ExitSpots` holds three `CFrameValue` children named `Exit`, but
  `CFrameValue.Value` reads `nil` in this VM** - so the exit positions are NOT available from
  that folder. Find exits by name in the workspace instead.

## Dead ends

1. **`ReplicatedStorage.ExitSpots` `.Value`** - three `CFrameValue`s, `.Value` reads nil.
2. **`workspace.LootItems` in the lobby** - always 0 children; not a bug, items spawn per match.
3. **Reading rarity off the item Part** - no attributes, no children, no `Rarity` key.

## Open questions

- Whether the ancestor-model text actually contains the rarity word in a live match (the
  labels seen in the lobby were item names). If every item reports `Unknown`, the next probe
  is one live in-match item's ancestor model descendants.

## Version history

- 1.0.0 (2026-11-02) - item ESP with one toggle per rarity (Common / Uncommon / Rare / Mythic /
  Legendary / Secret / Unknown) plus exit ESP. Item and exit toggles in an `ESP` section,
  rarity toggles in a `Rarity` section. Fixed Drawing pools (56 item slots, 12 exit slots),
  discovery refresh every 1.5 s, renderer on `RenderStepped`.
