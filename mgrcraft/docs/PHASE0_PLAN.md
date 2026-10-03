# Phase 0 plan: what's left

The feasibility report (`/mnt/project-files/mgrr-phase0-feasibility.md`, 2026-10-02) settled the static
questions from the exe on disk. A working x86 `dinput8.dll` proxy now loads in the game and keeps
`Local\MGRCraft_v1` alive. This plan covers the **runtime** questions from the prompt's §7 that are still
open, in the order they unblock each other.

Ground rules (from the prompt):
- Every experiment is a small, switchable piece of the existing `dinput8.dll` proxy (an INI flag per
  experiment, default off). It logs, and it un-hooks cleanly on exit.
- Nothing is hooked unless the exe's SHA-256 matches the pinned build (item 1).
- Addresses are written as `exe+offset` (the exe has ASLR, so absolute addresses change every run) and
  backed by a byte signature that matches exactly once.
- Results go into `RE_NOTES.md` with the evidence. Anything not yet verified is marked `UNVERIFIED`.

Tools: x32dbg, Cheat Engine 7.x, Ghidra, ReClass.NET, and **apitrace or the June 2010 DirectX SDK's PIX**
for frame captures. RenderDoc doesn't support Direct3D 9, so it can't be used here.

## Status at a glance

| # | Item | Status |
|---|---|---|
| 1 | Build identification | Mostly done. **Missing: SHA-256 pin** |
| 2 | Injection vector | Done (`dinput8.dll` proxy loads) |
| 3 | Frame hook | Next |
| 4 | Raiden entity | Open |
| 5 | Animation / state driving | Open |
| 6 | Blade Mode internals | Open (the point of the project) |
| 7 | Camera | Open |
| 8 | Rendering and depth | Open |
| 9 | Input | Imports known; device setup open |
| 10 | Scale and axes | Open |
| 11 | Risk table and verdict | After 3 to 10 |
| §5 | Host stage | Open |

---

## 1. Build identification (finish)

**Why:** the prompt forbids hooking an unrecognized build, so the hash gate is needed before item 3's
first hook.

Run in PowerShell:
```powershell
cd "C:\Program Files (x86)\Steam\steamapps\common\METAL GEAR RISING REVENGEANCE"   # or your library path
Get-FileHash -Algorithm SHA256 "METAL GEAR RISING REVENGEANCE.exe"
(Get-Item "METAL GEAR RISING REVENGEANCE.exe").VersionInfo | Format-List FileVersion,ProductVersion
```
**Paste back:** the hash and both version strings. The proxy then compares the running exe's SHA-256 (via
`BCryptHashData`) against a table holding that one hash. If it doesn't match, the proxy forwards
`DirectInput8Create` and does nothing else.

## 3. Frame hook

**Approach (no address hunting needed):** we load before the game creates its device, so patch the exe's
import of `d3d9!Direct3DCreate9` (its IAT entry). Wrap the returned `IDirect3D9`'s `CreateDevice`, then
hook the new device's vtable:

| Method | vtable index | Use |
|---|---|---|
| `Reset` | 16 | Release/recreate our `D3DPOOL_DEFAULT` resources (Alt-Tab, resolution change) |
| `Present` | 17 | Per-frame tick, heartbeat, overlay draw |
| `EndScene` | 42 | Fallback draw point if `Present` is too late |

Experiment `bFrameLog=1` logs: the `D3DPRESENT_PARAMETERS` passed to `CreateDevice` (back-buffer size and
format, depth format, windowed or not), the creating thread id, the thread calling `Present`, frames per
second every 5 s, and every `Reset`.

**Paste back:** the proxy's log after starting the game, reaching a stage, pausing, Alt-Tabbing out and back,
then quitting.

**Done when:** the heartbeat moves onto `Present` and keeps beating through menus and pauses, and Minecraft
sees it stop if the render thread hangs.

**Fallback:** if the IAT patch misses (the device made through another path), create a throwaway device
on a hidden window inside the proxy, read its vtable, and hook the same indices.

## 4. Raiden's entity

1. In Cheat Engine, attach to the game in an open area. Unknown-initial-value scan, type **Float**.
   Walk forward, "Changed value"; stand still, "Unchanged value". Repeat until a few addresses are left.
   Three consecutive floats that move together are the position.
2. Jump. The component that rises is the **up axis** (feeds item 10).
3. On the position address: "Find out what writes to this address". Walk a few steps. The instruction
   that writes every frame is the engine's movement integration, and its register holds Raiden's object
   pointer.
4. On that object: "Find out what accesses this address", then open it in ReClass.NET. Look for the
   yaw (a float in radians that changes when turning) and velocity (three floats that are zero while
   standing).
5. Freeze the position in Cheat Engine and press a movement key. If Raiden plays his run in place without
   moving, freezing the position is enough to stop the engine moving him, which is the SkyCraft approach.

**Paste back:** for the position, yaw and velocity: `exe+offset` of the writing instruction plus the 16
bytes around it (Cheat Engine: right click, "Copy bytes + opcodes"), and the offset of each field inside
the object. Also say what happened in step 5.

**Fallback:** if freezing makes him fight back (snapping or jitter), NOP the integration write behind a
signature instead, at our own `Present` time.

## 5. Animation and state driving

The proxy already owns the input path (DirectInput, with XInput to follow), which makes option (b) from
the prompt the cheapest:

- **(b) Synthetic input:** feed MGR the stick direction and magnitude that match Minecraft's movement,
  so MGR picks and plays its own idle, walk, run and jump animations. Then overwrite the position each
  frame from Minecraft (item 4). No animation internals needed.

Test (b) first: experiment `bSyntheticStick=1` holds the left stick forward at 50% then 100% for 3 s each
and logs Raiden's position (from item 4) each frame.

**Paste back:** the log, and whether he walked then ran.

Only if (b) can't express something Minecraft does (sneaking, swimming, falling without jumping) look
for the state field:
- Cheat Engine, **4-byte** scan: idle, "Unchanged"; run, "Changed"; stop, "Changed". Keep the small
  integers. Freeze one while idle; if Raiden switches animation, that's the state id.
- "What writes to" it gives the motion-set function (option a).

Option (c), a Raiden model drawn by us from the XPS file, stays the last resort.

## 6. Blade Mode internals

1. **Gating:** in an empty area (no enemies), enter Blade Mode. Note whether it enters, whether time
   slows, and whether fuel drains. Then try with an empty fuel bar.
2. **Time scale:** Cheat Engine, Float, value **1** (exact). Enter Blade Mode, "Changed value"; leave,
   "Changed value" back to 1. Repeat. What's left is the game's time scale. Note its value during Blade
   Mode.
3. **Blade angle:** in Blade Mode, Float unknown-initial scan. Rotate the blade (mouse or right stick),
   "Changed"; hold still, "Unchanged". Look for a value that changes smoothly with the angle (radians, or
   a 2D direction pair).
4. **The cut:** on the angle address, "Find out what accesses this address", then make one slash. The
   instructions that read it only at the moment of the slash lead to the cut function. In x32dbg, set a
   breakpoint there and slash again. Note the call stack (5 to 10 frames) and the arguments: look for a
   point plus a normal (six floats) or a plane (four floats).
5. **Is a cut happening now:** the function from step 4 running is the event. Log every call with its
   arguments (experiment `bCutLog=1`, once item 4 gives a signature).

**Paste back:** what step 1 showed, the time-scale address and its value in Blade Mode, and from step 4 the
call stack, the function's `exe+offset` with 16 bytes, and its arguments for three slashes at roughly
horizontal, vertical and diagonal angles.

**Fallback:** if the cut plane never exists as data (the mesh is cut directly from the blade's trail),
build the plane ourselves from the blade's start and end directions at the slash plus the camera
position. Report this before Phase 4, because it changes how exact cuts can be.

## 7. Camera

1. Cheat Engine: the camera position is three floats that move when the camera orbits but Raiden doesn't.
   "What writes to" leads to the camera update.
2. Frame capture (apitrace or PIX): on one draw call of Raiden, look at the vertex shader constants for a
   4x4 view-projection matrix. Note the constant register.
3. Enter Blade Mode and repeat step 1: does the same camera object change, or does another take over?

**Paste back:** the camera position's address and writer (`exe+offset` + bytes), the constant register
holding view-projection, and step 3's answer.

## 8. Rendering and depth

The `bFrameLog` hook from item 3 gets one more switch, `bFrameDump=1`: for a single frame after F10, log
every `SetRenderTarget`, `SetDepthStencilSurface`, `Clear` and `StretchRect` with the surfaces' size and
format, plus the number of draw calls between them. It also logs whether `INTZ` and `DF24` depth textures
are supported (`CheckDeviceFormat`).

**Paste back:** that dump (in a stage, not a menu).

What it decides:
- Where the main 3D pass ends and post-processing begins. That's where Minecraft's blocks get drawn,
  against the game's own depth buffer.
- Whether that depth is a plain surface (then the blocks share it in the same pass) or already a
  readable texture.

## 9. Input

Already known from the imports: DirectInput 8 (keyboard and mouse) and XInput 1.3 by ordinal (gamepad).
Still open is how the game reads them.

Experiment `bInputLog=1` in the proxy wraps `IDirectInput8::CreateDevice` and logs each device's GUID,
data format, cooperative level, and whether the game polls (`GetDeviceState`) or reads buffered input
(`GetDeviceData`). It also patches the exe's IAT entries for XInput ordinals 2, 3 and 4 and logs the first
call to each.

**Paste back:** the log after a minute in a stage with keyboard, mouse and (if you have one) a pad.

## 10. Scale and axes

1. Up axis from item 4, step 2.
2. Handedness: walk toward the screen's right. Of the two remaining position components, note which
   changed and its sign. With the up axis that fixes handedness.
3. Size: stand Raiden next to a wall corner, note his position, then walk to an edge one known length
   away (a crate or a floor tile repeated several times) and note the difference. Or read the top of his
   head (a bone near the top of the skeleton in ReClass) minus his feet.
4. From that, `kUnitsPerBlock` is Raiden's height in game units divided by 1.8. Write the conversion
   functions (C++ and Java) with unit tests for a point, yaw, pitch and a plane normal.

**Paste back:** the numbers from steps 1 to 3.

## §5. Host stage

1. From the main menu, list which VR Missions are available on your save, and which story chapters you
   can select.
2. Load the emptiest candidate. Note: are there enemies, scripted events, a forced camera, checkpoints?
3. Later (once item 4 exists): find the stage id in memory by loading two stages and scanning for an
   integer that differs. That's how the proxy will load the stage automatically.

**Paste back:** the lists from step 1 and your notes from step 2.

## 11. Risk table and verdict

Written after items 3 to 10 come back, ranked by likelihood and impact, ending in "go as designed", "go
with changes" or "stop and rethink". Current read:

| Risk | Likelihood | Impact | Note |
|---|---|---|---|
| Blade Mode plane isn't available as data | Medium | High | The fallback (plane from the blade direction) works but is less exact |
| Freezing Raiden's position makes him jitter | Medium | Medium | Patch the integrator instead |
| No room in the 2 GB address space | Medium | High | Keep the mapping small; never apply a LAA patch without asking |
| Blade Mode can't be entered without an enemy | Low to medium | High | An invisible dummy target, or the gating flag |
| D3D9 depth isn't readable | Low | Medium | Draw blocks in the same pass with the game's depth surface |
