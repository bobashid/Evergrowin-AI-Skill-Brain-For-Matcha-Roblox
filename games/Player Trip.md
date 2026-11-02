# Player Trip

Universal script - **no place id**, it runs in any game. File: `Player Trip` in
`bobashid/roblox-matcha`. Current version **v2.2.0**. Loaded directly, not through `Hub`
(Hub picks by place id / game name, and this script is game-agnostic).

## What it does

A rebindable key (default **G**) topples the character face-down. Space, the trip key again, or
the `Get up` button stands you back up. Menu is one tab, "Player Trip", with three buttons:
`Trip`, `Get up`, `Rebind key`.

## CONFIRMED by probe (2026-11-02, place 135564683255158)

- **A true physics ragdoll is impossible in this VM.** Every lever is gone at once:
  `Motor6D.C0` reads nil and the write errors (`CFrameMeta expected, got nil`);
  `Motor6D.Part0` / `Part1` / `Enabled` are unknown properties; `Humanoid.PlatformStand` is
  unknown; `AssemblyAngularVelocity` / `AngularVelocity` / `RotVelocity` are nil and unwritable;
  `Instance.new` is nil so no BallSocketConstraint can be created.
- `AssemblyLinearVelocity` and `Velocity` DO read as Vector3 on `HumanoidRootPart` - the only
  motion levers. Unused since v2.1.0 (see below).
- The character is R6. `Torso` held **five** Motor6D joints this session (Right/Left Shoulder,
  Right/Left Hip, Neck) - the earlier note said six including `RootJoint`; the count varies with
  the rig, so never hardcode it.
- A rotated root CFrame write **persists** across physics steps (probed 2026-10-30): the humanoid
  does not snap the character upright. Gravity still pulls the position down.

## Design decisions

- **v2.1.0** - removed every velocity write. The script writes rotation only; gravity does the
  falling. The old `root.Velocity = look * 30` shove was what made it read as a launch.
- **v2.2.0** (User-stated 2026-11-02: "dont make it auto bring me back up let roblox naturally
  do") - the `CFG.down` timer and the scripted rise are **gone**. After the topple the script
  writes nothing at all: state becomes `lying`, no timer, no velocity, no restore. Recovery is
  manual only (Space / trip key / `Get up`).
  - Consequence to state in any reply: because a rotated CFrame persists, the character stays
    down until the player asks to get up, unless the game itself stands them up. Do not
    "fix" that by re-adding an auto-rise - it is the requested behaviour.
- The limb flop is kept but **self-disabling**: joints are captured only when `C0` reads as
  userdata, which it currently does not, so the console shows `joints captured: 0` and the flop
  never runs. It cannot error.

## Tuning constants

`CFG.ramp` 0.45 s topple (lower = snappier), `CFG.settle` 0.30 s rise when the player asks,
`CFG.flail` 0.85 rad limb throw, `CFG.keyVK` 0x47 (G), `CFG.recoverVK` 0x20 (Space).

## Open questions

- How the topple **looks** in game is still unverified - only that it compiles and runs.
- Whether any game stands the character up on its own after a root rotation: not observed.

## Version history

- v2.0.0 - joint-flop ragdoll attempt. **Never ran**: two statements on one line with no
  separator (`CFG.flail = v captured = false`) is a parse error, so the chunk never compiled.
- v2.0.1 - compile fix only.
- v2.0.2 - compile fix + dropped `CFrame:Lerp` from the flop (unprobed here); angles rebuilt with
  `CFrame.Angles` instead.
- v2.1.0 - gravity fall: no velocity writes, `t*t` topple curve, auto get-up after 1.1 s.
- v2.2.0 (2026-11-02) - no auto get-up; script goes silent after the topple, manual recovery
  only. Pushed and read back (sha `bc5500f8`, 14,736 bytes).
