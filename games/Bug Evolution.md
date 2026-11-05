# Bug Evolution (place 70650894743738)

Script: `Bug Evolution` in `bobashid/roblox-matcha`. Orb farm, void orb farm, mob hover farm.

## World layout

- `Workspace.Orbs` — Folder, ~995 children, each a **Part** named `Orb` with a real `.Position` and no children.
- `Workspace.VoidOrbs` — Folder, ~10 children, each a **Model** named `VoidOrb` with parts `Core` (1.6³), `Shimmer` (2³) and `Hit` (6³, carries two `TouchTransmitter`s). A Model has **no `.Position`**, so read `Hit`/`Core` instead. Proof: `matcha_exec` dump, this session. **This is why the first void-orb farm collected nothing** — it called `.Position` on the Model and got nil.
- `Workspace.Bugs` — Folder of ~200 Models (`Big Cricket`, `Beetle`, `Caterpillar`) with `Humanoid` + `HumanoidRootPart`, attributes `BugId`, `Kind`, `Zone`, `Scale`, `Radius`, `SizeTag`, colours. `Humanoid.MaxHealth` reads 50.
- `Workspace.ClientLegs` — leg-only Models per bug (`Beetle_Legs`), empty children.
- Other folders: `Breakables`, `RainbowOrbs`, `VoidEvent`, `Nests`, `Wisps`, `AmbientFX`.

## Damage

Confirmed (date unknown): **`Humanoid.Health` and `Humanoid.MaxHealth` are not writable** — the write raises `Unknown property: Health` (`pcall` returns false), and reading them works. `Humanoid:TakeDamage` is `nil`. Proof: two probes this session.
Consequence: a client-side "silent kill" that zeroes health is impossible in this game. Do not ship one and call it working.

Confirmed (date unknown): `ReplicatedStorage.Remotes` holds 41 entries including `Attack` (RemoteEvent), `HitConfirm`, `XPGain`, `OrbCollected`, `VoidAction` (RemoteFunction). Proof: probe listing.
UNVERIFIED (date unknown): whether `Remotes.Attack:FireServer(humanoid)` damages a bug. Test: fire it at one bug, read `Humanoid.Health` before/after. Probes were cut off before this printed.

## Notes

- `UI` exposes only `AddTab`, `RemoveTab`, `GetValue`, `SetValue`. `UI.AddTab(name, cb)` returns **nil** — build widgets inside the callback, never from a return value.
- Teleport writes (`hrp.CFrame = ...`) return `ok=true`; one probe read the character's position back unchanged 0.2 s later, so re-read before concluding a teleport is authoritative.
