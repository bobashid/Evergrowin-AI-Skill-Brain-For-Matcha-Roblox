# Steal Or Die

Place id **119925538689885**. `game.Name` reports **`Ugc`**. Script: `Steal Or Die` in
`bobashid/roblox-matcha`, current version **1.1.0**. Pinned in `Hub` `PLACES` at v3.8.

## What the script does

Two sections: `ESP` (`Item ESP`, `Exit ESP`) and `Rarity` (one toggle per rarity +
`Unknown`). Item ESP boxes **active match loot only**, one marker per item, with
`rarity  name  [dist]` text coloured by rarity. Exit ESP boxes `ExitPoint` parts and
`Exit Sign - Lit` models.

## CONFIRMED by probe (2026-11-02)

- **`ReplicatedStorage.LootTemplates`** - 38 templates whose names ARE the item keys:
  `Phone, Monitor, Bag, DeskFan, Laptop, Headphones, Camera, Watch, Console, Toolbox,
  Microwave, Painting, Screwdriver, Hammer, BarricadeKit, CoffeeMug, FireExtinguisher,
  Textbooks, Keyboard, DeskLamp, Tablet, Guitar, RecordPlayer, Projector, GraphicsCard,
  ServerBlade, LabCase, HardDrive, MiniFridge, Dossier, CryptoWallet, GoldBar, Potions,
  Spellbook, JackOLantern, CrystalBall, Candelabra, Candy`.
- **`ReplicatedStorage.GameConfig` decompiles in full (635 lines)** and defines the rarity
  system: `Rarities = {{name, chance, colour}}`, `RarityColour[name]`, `RarityStroke.Secret`,
  `rollRarity(rng)`, `rarityFor(key, rng)`, `pickLootKey(list, rng)`,
  `Items[key].fixedRarity / .spawnWeight / .tool / .season`, `SecretChance = 0.0002`.
- **Rarity names, confirmed:** `Common, Uncommon, Rare, Mythic, Legendary, Secret` (from the
  `prices` tables plus `fixedRarity == "Secret"`). The ESP adds `Unknown` as a seventh bucket.
- Confirmed colours: Common `(208,62,58)`, Legendary `(228,184,62)`, Secret stroke
  `(228,228,236)`. Uncommon / Rare / Mythic colours are **my picks**, cosmetic only - the
  toggles filter on the NAME, never the colour.
- **`workspace.LootItems` is the match loot folder** - always 0 children in the lobby, populated
  during a round. That is the "active items" source; v1.1.0 reads only loot folders.
- **World items are Parts named `Item`** - 288 were present in the lobby as shop displays under
  `Promo Dump Bin`. That is shop stock, **not loot**, and v1.0.0 wrongly boxed all of it.
- **Exit markers:** Parts named `ExitPoint` (parents `Door`, `Closet`) and Models named
  `Exit Sign - Lit` under `workspace.ActiveMap`.
- **`ReplicatedStorage.ExitSpots`** holds three `CFrameValue` children named `Exit`, but
  `CFrameValue.Value` reads **nil** in this VM - exits are found by name in the workspace instead.

## v1.1.0 fixes (2026-11-03)

User: "you got the spawn down but want active items and with a rarity toggle".

1. **Active items only.** v1.0.0 walked the whole workspace for Parts named `Item` and boxed
   every lobby shop display. v1.1.0 reads only `LootItems` / `DroppedItems` / `Loot`, and if one
   of them is non-empty it uses **only** that one. Lobby = nothing boxed.
2. **One marker per item, not per part.** A single `Phone` model has `Glass, Display, FrontCam,
   Earpiece, Power, VolUp, VolDown, Port` children; v1.0.0 would have boxed all eight. The scan
   takes the model's part named `Item`, else `PrimaryPart`, else its first BasePart.
3. **Rarity resolution is layered and cached** (3 s per instance address): attributes on the item
   and up to four ancestors (any key containing `rarity` or `tier`) -> a `Value` child whose name
   contains `rarity` -> TextLabels inside the model -> `Unknown`.
4. **Rarity word matching order matters:** `uncommon` CONTAINS `common`, so matching runs
   `Legendary, Mythic, Secret, Uncommon, Rare, Common` - Common last. Matching in the natural
   rarity order would label every Uncommon as Common.
5. **Console tally on change:** `[SOD] items=N exits=M [Common=3 Unknown=5]`, printed only when
   the item count changes, so it is not spam.

## Dead ends

1. **`ReplicatedStorage.ExitSpots` `.Value`** - three `CFrameValue`s, `.Value` reads nil.
2. **`workspace.LootItems` in the lobby** - always 0 children; not a bug, items spawn per match.
3. **Reading rarity off the item Part directly** - no attributes, no children on lobby displays.
4. **Boxing every `Item` part workspace-wide** - that is the lobby shop stock (288 of them).

## Open questions

- **Where the rarity string actually sits on a live in-match item is still unconfirmed.** The
  layered lookup above covers attributes, Value children and labels, but if every item reports
  `Unknown` in a match the next step is one live item's ancestor list. The console tally line
  is what tells us which happened.

## Version history

- 1.0.0 (2026-11-02) - first build: item ESP with a per-rarity toggle set plus exit ESP. Walked
  the whole workspace for Parts named `Item`, so in the lobby it boxed the 288 shop displays.
- 1.1.0 (2026-11-03) - active items only (loot folders), one marker per item, layered rarity
  lookup with a per-instance cache, `uncommon`-contains-`common` match ordering fixed, console
  tally on change. Pushed and read back (sha `ad83262b`, 20,240 bytes).
