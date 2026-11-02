# Player Trip

Universal script - **no place id**, it runs in any game. File: `Player Trip` in
`bobashid/roblox-matcha`. Current version **v2.5.0**. Loaded directly, not through `Hub`
(Hub picks by place id / game name, and this script is game-agnostic).

## What it does

A rebindable key (default **G**) topples the character over its feet, **one shot, no get-up**.
Menu is one tab, "Player Trip", with two buttons: `Trip (one shot)` and `Rebind key`.

## CONFIRMED by probe (2026-11-02, place 135564683255158)

- **A true physics ragdoll is impossible in this VM.** Every lever is gone at once:
  `Motor6D.C0` reads nil and the write errors (`CFrameMeta expected, got nil`);
  `Motor6D.Part0` / `Part1` / `Enabled` are unknown properties; `Humanoid.PlatformStand` is
  unknown; `AssemblyAngularVelocity` / `AngularVelocity` / `RotVelocity` are nil and unwritable;
  `Instance.new` is nil so no BallSocketConstraint can be created.
- `AssemblyLinearVelocity` and `Velocity` DO read as Vector3 on `HumanoidRootPart` - the only
  motion levers. Unused since v2.1.0.
- The character is R6. `Torso` held **five** Motor6D joints this session (Right/Left Shoulder,
  Right/Left Hip, Neck) - the earlier note said six including `RootJoint`; the count varies with
  the rig, so never hardcode it.
- A rotated root CFrame write **persists** across physics steps (probed 2026-10-30): the humanoid
  does not snap the character upright. Gravity still pulls the position down.
- **Composite rotation write is safe (probed 2026-11-02):**
  `root.CFrame = CFrame.new(p.X,p.Y,p.Z) * base * CFrame.Angles(pitch, 0, roll)` returns ok with
  **0.0000 position drift**, up vector 1.000 -> 0.800 on a 0.6 rad pitch + 0.25 rad roll, and
  restores to 1.000 exactly. This is what allows a diagonal tumble instead of a straight pitch.

## v2.5.0 IS A STRICT ONE-SHOT - AND THAT SOFT-LOCKS ON PURPOSE

User request: "just one time click to fall no get up". Offered two options first (strict one-shot
vs one-shot per press); the user answered **"1"** = strict. Shipped only after stating the
consequence in the reply, before the push.

Every recovery path is **deleted**, not disabled:

    beginRise()                  removed from the file
    the 'rising' state           removed
    the Space edge in Heartbeat  removed (Space is not special)
    trip() while down            returns false
    the 'Get up' menu button     removed

After the topple the Heartbeat returns immediately every frame - no CFrame write, no velocity
write, no timer. State is `lying` forever.

**The consequence, and why it is not a bug:** a rotated root CFrame persists in this VM, so once
the character is down it stays down. `_G.PlayerTrip.stop()` does NOT restore the pose (stop only
disconnects and removes the HUD). **Rejoining is the only exit.** Do not "fix" this by re-adding
an auto-rise or a Space recovery - it is the requested behaviour.

**Rule learned:** before shipping any pose-write script with its recovery paths removed, say
plainly in the reply that the pose persists and the only exit is a rejoin, and get an explicit
confirmation. Offer the milder option (recovery on an explicit second press) first. The user
picked strict anyway - but they were told, and the note records that they were told.

## THE FLUNG FEELING WAS THE PIVOT, NOT THE CURVE (v2.4.0, 2026-11-02)

User-stated: "i just dont like how i get flung to trip cant it be natural?".

Diagnosis: every version up to v2.3.0 rotated the body about the `HumanoidRootPart`'s **own
centre**, which sits ~3 studs above the ground. A rotation about a point in mid-air sweeps the
head one way and the feet the other and never moves the body *down* - that is a spin, not a fall.
No amount of curve or ramp tuning fixes a wrong pivot, which is why v2.1.0-v2.3.0 all still felt
flung. **Rule: when a fake fall reads as a spin/fling, check the pivot before touching the curve.**

v2.4.0 (kept in v2.5.0) rotates about a world point at the feet:

    root.CFrame = CFrame.new(pivot) * tilt * CFrame.new(-pivot) * baseCF

with `pivot = (root.Position - Vector3.new(0, CFG.pivotDrop, 0))`. That composition is a rotation
about a world point, so the root follows the arc and the body tips over the feet like a felled
tree instead of pivoting in place. Still no velocity write anywhere.

- **The base CFrame is captured at trip time and rebuilt from every step**, never composed onto
  the LIVE `root.CFrame`. Composing a fixed-pivot rotation onto the live CFrame re-applies the
  rotation every frame and compounds it - the trap in the old `applyTilt`.
- Curve `t*t*(2-t)` - slow lean, fast overbalance, slight bleed off the last degrees.

## THE PASTED RAGDOLL SCRIPT CANNOT RUN HERE (probed 2026-11-02)

The user pasted a standard community ragdoll LocalScript (`Instance.new("Attachment")` +
`Instance.new("BallSocketConstraint")`, `motor.Enabled = false`, `hum.PlatformStand = true`,
`hum:ChangeState(Enum.HumanoidStateType.Physics)`). A probe the same turn confirmed **every one
of those five calls fails** on this build:

    Instance                                  -> nil (so no Attachment, no BallSocketConstraint)
    Humanoid.ChangeState                      -> nil
    Enum.HumanoidStateType                    -> nil (cannot even name Physics)
    Motor6D.Enabled  read                     -> nil
    Motor6D.Enabled  write                    -> Unknown property: Enabled
    existing BallSocketConstraints in char    -> 0  (24 Attachments exist, all game-owned)

This is the textbook internal-executor tell from `learned/patterns.md`: the mechanism itself needs
`Instance` creation, so it **cannot be ported by editing**. Do not try. Two honest answers only:
run it in an executor that exposes `Instance`, or use the rotation fall this script already does.

### The "this works" FallingDown snippet is the same dead end (2026-11-02)

A second snippet found online was:

    Humanoid:ChangeState(Enum.HumanoidStateType.FallingDown)
    RootPart.AssemblyLinearVelocity += RootPart.CFrame.LookVector * 5

It is real, ordinary Roblox - and it still cannot run here. Line 1 needs `ChangeState` (nil) and
`Enum.HumanoidStateType` (nil, so `.FallingDown` cannot even be indexed); the chunk dies before
line 2. The impulse on line 2 *would* work (`AssemblyLinearVelocity` is a readable/writable
Vector3), but a velocity impulse is precisely the "forces" the user asked to remove in v2.1.0.
Record this so a third online snippet of the same shape is answered immediately instead of
re-probed: **any snippet whose first action is `ChangeState` / `PlatformStand` / a constraint
`Instance` is a no here, regardless of how many people report it working elsewhere.**

## Design decisions

- **v2.1.0** - removed every velocity write. The script writes rotation only; gravity does the
  falling. The old `root.Velocity = look * 30` shove was what made it read as a launch.
- **v2.2.0** (User-stated: "dont make it auto bring me back up let roblox naturally do") - the
  `CFG.down` timer and the scripted rise removed; after the topple the script went silent.
- **v2.3.0** ("can you try to make it natural it doesnt force me in a direction") - random
  diagonal fall, smoothstep curve, ramp 0.60 s. Did not fix the flung feeling: pivot was still
  the root centre. See the v2.4.0 section.
- **v2.4.0** - feet pivot, captured base, `t*t*(2-t)` curve, ramp 0.65 s, roll 0.30.
- **v2.5.0** - strict one-shot: every recovery path deleted (see above).
- The limb flop is kept but **self-disabling**: joints are captured only when `C0` reads as
  userdata, which it currently does not, so the console shows `joints captured: 0` and the flop
  never runs. It cannot error.

## Tuning constants

`CFG.ramp` 0.65 s topple (lower = snappier), `CFG.pivotDrop` 2.5 studs below the root centre
(raise toward 3 = lower pivot, lower toward 1.5 = hips), `CFG.roll` 0.30 max side roll (0 =
straight forward), `CFG.flail` 0.85 rad limb throw, `CFG.keyVK` 0x47 (G). `settle`/`recoverVK`
were removed in v2.5.0.

## Open questions

- How the feet-pivot topple **looks** in game is unverified - only that it pushed and read back.
  The pivot maths is derived, not probed (see `learned/unverified.md`).
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
- v2.3.0 (2026-11-02) - random diagonal tumble (side roll), smoothstep topple curve, 0.60 s
  ramp. Pushed and read back (sha `70fc7478`, 15,439 bytes).
- v2.4.0 (2026-11-02) - **pivot moved to the feet**: `CFrame.new(pivot) * tilt * CFrame.new(-pivot) * baseCF`
  on a captured base, `t*t*(2-t)` curve, ramp 0.65 s, roll 0.30. Pushed and read back
  (sha `249a4412`, 15,790 bytes).
- v2.5.0 (2026-11-02) - **strict one-shot**: `beginRise` deleted, `rising` state deleted, Space
  recovery deleted, trip-while-down disabled, `Get up` button deleted. Rejoin is the only exit.
  Pushed and read back (sha `32e7a4ba`, 14,083 bytes).
