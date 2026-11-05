# Bug Evolution (place 70650894743738)

Script: `Bug Evolution` in `bobashid/roblox-matcha`. Orb farm, void orb farm, mob hover farm, auto eat.

## World layout

- `Workspace.Orbs` — Folder, ~995 children, each a **Part** named `Orb` with a real `.Position` and no children.
- `Workspace.VoidOrbs` — Folder, ~10 children, each a **Model** named `VoidOrb` with parts `Core` (1.6³), `Shimmer` (2³) and `Hit` (6³, carries two `TouchTransmitter`s). A Model has **no `.Position`**, so read `Hit`/`Core` instead. Proof: `matcha_exec` dump. **This is why the first void-orb farm collected nothing** — it called `.Position` on the Model and got nil.
- `Workspace.Bugs` — Folder of ~200 Models (`Big Cricket`, `Beetle`, `Caterpillar`) with `Humanoid` + `HumanoidRootPart`, attributes `BugId`, `Kind`, `Zone`, `Scale`, `Radius`, `SizeTag`, colours. `Humanoid.MaxHealth` reads 50. Each bug also has a plain part named `Body` (relevant to the eat bug below).
- `Workspace.ClientLegs` — leg-only Models per bug (`Beetle_Legs`), empty children.
- Other folders: `Breakables`, `RainbowOrbs`, `VoidEvent`, `Nests`, `Wisps`, `AmbientFX`.

## Damage

Confirmed (date unknown): **`Humanoid.Health` and `Humanoid.MaxHealth` are not writable** — the write raises `Unknown property: Health` (`pcall` returns false), and reading them works. `Humanoid:TakeDamage` is `nil`. Proof: two probes.
Consequence: a client-side "silent kill" that zeroes health is impossible in this game. Do not ship one and call it working.

Confirmed (date unknown): `ReplicatedStorage.Remotes` holds 41 entries including `Attack` (RemoteEvent), `HitConfirm`, `XPGain`, `OrbCollected`, `VoidAction` (RemoteFunction). Proof: probe listing.
UNVERIFIED (date unknown): whether `Remotes.Attack:FireServer(humanoid)` damages a bug. Test: fire it at one bug, read `Humanoid.Health` before/after. Probes were cut off before this printed.

## Movement / physics (probed)

- `HumanoidRootPart.AssemblyLinearVelocity` and `.Velocity` **are writable** (`ok=true`).
- `Humanoid.Sit` and `Humanoid.PlatformStand` are **not writable** (`Unknown property: Sit` / `PlatformStand`).
- `CFrame.new(vector3)` **raises** `CFrame.new expects 0, 3, 6, or 12 arguments` — always pass three numbers, never a Vector3.
- `Enum.HumanoidStateType` is `nil` in this build; `RunService.RenderStepped` and `Heartbeat` exist.
- Confirmed (2026-11-05, User-stated + probe): a mob-farm anchor written **below** the target does not hold — the character is shoved back up to floor level, which the user describes as "resetting to 0 / rubber banding". Mitigation shipped: re-write the CFrame on `RenderStepped` **and** zero `AssemblyLinearVelocity`/`Velocity` every frame.
- UNVERIFIED (2026-11-05): whether Matcha's `SliderInt` accepts a **negative** minimum. User: "negative doesnt work it keeps resetting to 0". Suspected clamp to 0. Workaround shipped: slider `0..max` as a *distance* plus a `Below Target` boolean; never pass a negative slider range until this is probed. User asked for the Y-distance cap to be **15** (it was 300 in v4.2 and 10 before that).

## Kill and eat

User-stated (2026-11-05): **eating a killed bug uses the same remote as attacking** (`ReplicatedStorage.Remotes.Attack`). The kill happens first, the bug ragdolls/dies where it stood, and the eat is fired at that body with the attack remote. There is no separate eat prompt to press and no corpse object with its own name.
Consequence: the mob farm does not need an eat widget wired to prompts — it needs to remember the model it killed and keep firing the attack remote at it for a few seconds. Prompt / click-detector / touch fallbacks stay only as a second path.
User-stated (2026-11-05): the **Attack Remote toggle was removed** — attack is always on while the mob farm is on.

## Auto eat

Confirmed (2026-11-05): matching a name fragment list that includes `body` matched **every bug's `Body` part** and spammed `ate Body` dozens of times a second. Rule: corpse/food matching must (a) avoid generic words like `body`, and (b) skip any instance with a `Humanoid` ancestor, i.e. a live creature's own parts.
UNVERIFIED (date unknown): what the game actually uses to eat a corpse (prompt, click detector, touch or a remote). The sweep fires all three client-side triggers; no corpse object was present during the probes. Superseded in practice by the user-stated attack-remote rule above.

## Toggle cancellation (user report, 2026-11-05)

User: "the toggle for auto orbs doesnt turn off after i toggle it off". Cause: the one-pass orb sweep checked only a global `running` flag, so switching the feature toggle off mid-pass left the sweep running for the remaining orbs.
Rule: **every in-flight loop must be given an `alive()` predicate that includes the feature's own flag**, not just the script-wide `running` flag; and a one-shot feature should clear its widget with `UI.SetValue(id, false)` behind a `pcall`. Because the getter poller can read a stale `true`, keep a latch: after a self-clearing pass, ignore polled `true` until a polled `false` arrives.

## Notes

- `UI` exposes only `AddTab`, `RemoveTab`, `GetValue`, `SetValue`. `UI.AddTab(name, cb)` returns **nil** — build widgets inside the callback, never from a return value.
- Teleport writes (`hrp.CFrame = ...`) return `ok=true`; re-read the position before concluding a teleport is authoritative.
