# Phase 0 plan: what's left

The feasibility report (`/mnt/project-files/mgrr-phase0-feasibility.md`, 2026-10-02) settled the static
questions from the exe on disk. A working x86 `dinput8.dll` proxy now loads in the game and keeps
`Local\MGRCraft_v1` alive. Runtime results so far are in `/mnt/project-files/mgrr-runtime-findings.md`
(2026-10-03). This plan covers the **runtime** questions from the prompt's §7 that are still
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
| 1 | Build identification | **Done** (SHA-256 pinned in the proxy) |
| 2 | Injection vector | **Done** (`dinput8.dll` proxy loads) |
| 3 | Frame hook | **Done** (IAT patch of `Direct3DCreate9`, Present on the game's only thread) |
| 4 | Raiden entity | **Next** |
| 5 | Animation / state driving | Open |
| 6 | Blade Mode internals | Open (the point of the project) |
| 7 | Camera | Open |
| 8 | Rendering and depth | Open |
| 9 | Input | **Done** (both devices polled with `GetDeviceState`; pads via XInput 1.3) |
| 10 | Scale and axes | Open |
| 11 | Risk table and verdict | After 3 to 10 |
| §5 | Host stage | Open |

---

## 1. Build identification: done

SHA-256 `7174741F46AB4C222304B0F1B9291A2281D53368016347E65D2EA404FF07AD34` (26,347,008 bytes), pinned in
the proxy. On a mismatch the proxy only forwards `DirectInput8Create`: no hooks, no shared memory.

## 3. Frame hook: done

The exe's IAT entry for `d3d9!Direct3DCreate9` is patched, then `CreateDevice`, then the device's
`Reset` (16), `Present` (17) and `EndScene` (42). Measured: 1280x720 windowed, A8R8G8B8, **MSAA 2x**, auto
D24S8 depth, immediate present interval (the game caps itself at 60 fps), two `EndScene`s per gameplay
frame. **One thread** creates the device, polls input and calls `Present`, so anything written at
`Present` time (Raiden's position, synthetic input) lands between two simulation steps without locks.
The heartbeat and `MgrState` are written on every `Present`.

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

The proxy already owns the input path, which makes option (b) from the prompt the cheapest. The game
polls both DirectInput devices with `GetDeviceState`, so synthetic keys and mouse deltas are an overwrite
of the caller's buffer after the real call. Keyboard only gives full-speed directions, though, and walk
versus run needs a stick magnitude. So movement goes through XInput instead: patch the exe's IAT entry for
`XInputGetState` (ordinal 2 of `xinput1_3.dll`) and, for user index 0, report a connected pad whose left
stick carries Minecraft's movement direction and speed. A real pad's other buttons pass through.

- **(b) Synthetic input:** feed MGR the stick direction and magnitude that match Minecraft's movement,
  so MGR picks and plays its own idle, walk, run and jump animations. Then overwrite the position each
  frame from Minecraft (item 4). No animation internals needed.

Test (b) first: experiment `bSyntheticStick=1` holds the virtual left stick forward at 30%, 60% and 100%
for 3 s each and logs Raiden's position (from item 4) each frame. Check first that the game polls
`XInputGetState` every frame even with no pad plugged in; if it stops polling after an
`ERROR_DEVICE_NOT_CONNECTED`, report the virtual pad as connected from the first call.

**Paste back:** the log, and at which stick values he walked and ran.

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

The depth buffer is the game's auto D24S8 surface, and it's **multisampled (MSAA 2x)**. Direct3D 9 can't
sample a multisampled depth surface or resolve it with `StretchRect`, so reading the game's depth as a
texture (INTZ) is off the table unless MSAA is turned off. The default plan is therefore to draw
Minecraft's blocks **inside the game's scene**, with the game's own depth surface bound, before its
post-processing. The frame dump finds that point.

The frame hook from item 3 gets one more switch, `bFrameDump=1`: for a single frame after F10, log
every `SetRenderTarget`, `SetDepthStencilSurface`, `Clear` and `StretchRect` with the surfaces' size and
format, plus the number of draw calls between them and which of the two `EndScene`s each belongs to. It also
logs whether `INTZ` is supported (`CheckDeviceFormat`), for the case where MSAA gets turned off.

**Paste back:** that dump (in a stage, not a menu).

What it decides:
- Where the main 3D pass ends and post-processing begins (likely around the first of the two
  `EndScene`s). That's where Minecraft's blocks get drawn, against the game's own depth surface.
- Whether the game renders into its own render targets first (then the blocks go in before it resolves
  them to the back buffer).

## 9. Input: done

`DirectInput8Create(0x800)`, then a keyboard (`c_dfDIKeyboard`, 256 bytes) and a mouse (`DIMOUSESTATE2`,
20 bytes), both `DISCL_NONEXCLUSIVE | DISCL_FOREGROUND` and both polled with `GetDeviceState` (never
`GetDeviceData`). Pads go through XInput 1.3. For the input bridge this means:
- Swallowing input from MGR is zeroing bytes in the `GetDeviceState` buffer; forwarding to Minecraft is
  copying the real buffer into the input ring first.
- Blade Mode keys stay in the buffer while Blade Mode is active (the prompt's §6 routing).
- `DISCL_FOREGROUND` means MGR loses input when unfocused, which suits SkyCraft's model (the host
  window keeps focus, Minecraft stays hidden).

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
| Game depth is multisampled, so not readable as a texture | Known | Medium | Draw blocks inside the game's scene with its depth surface bound (item 8) |
