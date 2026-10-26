# Learned: API facts and signatures

Proven by a probe, log or the docs. Format: `Confirmed (YYYY-MM-DD): <fact>. Proof: <source>.` When a fact corrects an `api-*` reference file, fix that line there too and note it in `learned/corrections.md`.

- Confirmed (2026-10-26): `RunService.PreRender` is nil in the current build (`type(...) == "nil"`), while `RenderStepped` and `Heartbeat` are tables and `:Connect` works. Use `RenderStepped`, not `PreRender`. Proof: `matcha_exec` printing `pre=nil render=table hb=table` in Piano Rhythm Tiles (place 101999883467300).
- Confirmed (2026-10-26): `keypress`, `keyrelease` and `isrbxactive` all exist as functions in the current build. Proof: same `matcha_exec` (`kp=function kr=function`, `active=true`).
- Confirmed (2026-10-26): a GuiObject's `AbsolutePosition` can be frozen at one value while the element is visibly animating - a tile read y = 364 on 157 consecutive samples over 4 s. Treat a never-changing `AbsolutePosition` as a VM limitation, not as proof the element is static, and time off a clock instead. Proof: two sampling probes in Piano Rhythm Tiles.
- User-stated (2026-10-26): `fireproximityprompt(prompt)` returns true on a `MountDinosaur` ProximityPrompt and completes the mount with no range check. Proof: Ride A Dino dump, header "fireproximityprompt exists and returns true on that prompt", place 93523763376122.
- User-stated (2026-10-26): `Humanoid.Sit` reads nil for a saddle mount (no Seat or VehicleSeat exists). Use a `CarriedDino` child or a seat/weld named seat, mount or saddle instead. Proof: Ride A Dino dump.
- User-stated (2026-10-26): `plot:GetAttribute('Owner')` returns the owning player's UserId as a number on Ride A Dino plots under `Workspace.Pens`. Proof: Ride A Dino dump.
- User-stated (2026-10-26): `UI.GetValue(id)` and `UI.GetValue(name)` both answer in the build used by Ride A Dino; the id form is tried first. Proof: Ride A Dino dump toggle poller.
- User-stated (2026-10-26): a `while running do ... task.wait() end` loop spins at about 60 kHz on this build and starves the scheduler. The user's own scripts poll menus at `task.wait(0.15)` and action loops at `task.wait(0.25)`. Proof: Ride A Dino poller and mount loop.
