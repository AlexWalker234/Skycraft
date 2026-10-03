# MGR:R reverse-engineering notes

Every fact about the game's internals, with its evidence. A row without evidence is `UNVERIFIED`.
Addresses are `exe+offset` (the exe uses ASLR) plus a byte signature that matches exactly once.

## Pinned build

| Field | Value | Evidence |
|---|---|---|
| File | `METAL GEAR RISING REVENGEANCE.exe`, 26.3 MB | Feasibility report, 2026-10-02, read from Alex's Steam install |
| SHA-256 | `7174741F46AB4C222304B0F1B9291A2281D53368016347E65D2EA404FF07AD34` (26,347,008 bytes) | Proxy's own hash in DllMain, matches `Get-FileHash` (runtime findings, 2026-10-03) |
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

## Verified (runtime, proxy logs on Alex's PC, 2026-10-03)

| Fact | Value | Evidence |
|---|---|---|
| Frame hook | IAT patch of `Direct3DCreate9`, then `CreateDevice` (IDirect3D9 slot 16), device `Reset` 16 / `Present` 17 / `EndScene` 42 | Proxy hooks installed and logging |
| Device | HAL, adapter 0, behavior 0x44 (hardware vertex processing, multithreaded), SDK 32 | `CreateDevice` arguments |
| Back buffer | 1280x720 A8R8G8B8, windowed, MSAA 2x, DISCARD | `D3DPRESENT_PARAMETERS` |
| Depth | Auto D24S8 (multisampled) | `D3DPRESENT_PARAMETERS` |
| Present interval / frame rate | Immediate; 60 fps focused, ~30 unfocused | Present timing |
| EndScene | 2 per gameplay frame | Call counts |
| Threads | One thread creates the device, polls input and calls `Present` | Thread ids |
| Keyboard | `c_dfDIKeyboard` (256 B), `DISCL_NONEXCLUSIVE \| DISCL_FOREGROUND`, polled with `GetDeviceState` | `CreateDevice` / `SetDataFormat` / call logs |
| Mouse | `DIMOUSESTATE2` (20 B), same cooperative level, polled with `GetDeviceState` | Same |
| Buffered input | Never used (`GetDeviceData` not called) | Call logs |

## Runtime (open)

| Item | Status |
|---|---|
| XInput polling with no pad connected | `UNVERIFIED` |
| Raiden's object, position, yaw, velocity | `UNVERIFIED` |
| Movement integration write | `UNVERIFIED` |
| Animation state id | `UNVERIFIED` |
| Blade Mode gating, time scale, blade angle, cut function | `UNVERIFIED` |
| Camera object and view-projection constant | `UNVERIFIED` |
| Up axis, handedness, units per meter | `UNVERIFIED` |
| Host stage and stage id | `UNVERIFIED` |
