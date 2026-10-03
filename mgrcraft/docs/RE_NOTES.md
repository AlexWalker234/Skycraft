# MGR:R reverse-engineering notes

Every fact about the game's internals, with its evidence. A row without evidence is `UNVERIFIED`.
Addresses are `exe+offset` (the exe uses ASLR) plus a byte signature that matches exactly once.

## Pinned build

| Field | Value | Evidence |
|---|---|---|
| File | `METAL GEAR RISING REVENGEANCE.exe`, 26.3 MB | Feasibility report, 2026-10-02, read from Alex's Steam install |
| SHA-256 | `UNVERIFIED` (not yet taken) | See PHASE0_PLAN.md item 1 |
| PE timestamp | 2014-01-28 | Feasibility report (PE header) |
| Steam app id | 235460 | `steam_appid.txt` in the game folder |

## Verified (static, from the exe on disk)

| Fact | Value | Evidence |
|---|---|---|
| Architecture | x86, PE32 (Machine 0x014C) | Feasibility report (PE header) |
| Large address aware | No: 2 GB address space | Feasibility report (PE characteristics) |
| ASLR / DEP | Both enabled | Feasibility report (DllCharacteristics) |
| DRM wrapper | None seen (no `.bind` section) | Feasibility report (section table) |
| Renderer | Direct3D 9 (`d3d9!Direct3DCreate9`), d3dx9_43 | Feasibility report (import table) |
| Keyboard and mouse | DirectInput 8 (`DINPUT8!DirectInput8Create`) | Feasibility report (import table) |
| Gamepad | XInput 1.3, ordinals 2, 3, 4 | Feasibility report (import table) |
| Injection | `dinput8.dll` proxy next to the exe loads | PC thread's Phase 1 proxy, logging from inside the process |

## Runtime (open)

| Item | Status |
|---|---|
| Thread that creates the device and calls `Present` | `UNVERIFIED` |
| Back-buffer and depth formats | `UNVERIFIED` |
| Raiden's object, position, yaw, velocity | `UNVERIFIED` |
| Movement integration write | `UNVERIFIED` |
| Animation state id | `UNVERIFIED` |
| Blade Mode gating, time scale, blade angle, cut function | `UNVERIFIED` |
| Camera object and view-projection constant | `UNVERIFIED` |
| Up axis, handedness, units per meter | `UNVERIFIED` |
| Host stage and stage id | `UNVERIFIED` |
