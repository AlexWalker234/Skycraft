# SkyCraft notes for MGRCraft

What MGRCraft (working name MGRCraft in `MGRR-CRAFT-PROMPT.md`) takes from SkyCraft (the rest of this
repository, read-only reference), what changes, and what goes away. Paths are relative to the repository
root. Facts about SkyCraft come from its source. Facts about MGR:R come from `RE_NOTES.md` (verified) and
are otherwise hypotheses.

## 1. The principle, unchanged

From `docs/DESIGN.md` §1: **neither game is rewritten.** Each game runs its own logic; the two mods only
translate. If a Minecraft mechanic shows up in C++, or an MGR mechanic in Java, the design has gone wrong.

MGRCraft keeps this exactly. Authority splits differently, because Minecraft is the world here:

| | SkyCraft | MGRCraft |
|---|---|---|
| World geometry | Skyrim (exported into MC collision) | **Minecraft** (nothing exported) |
| Player physics | Minecraft | Minecraft |
| Player body and animation | hidden (first person), MC skin in F5 | **MGR's real Raiden** |
| Enemies | Skyrim NPCs, mirrored as MC proxies | **Minecraft mobs**, drawn in MGR's frame |
| Combat | MC weapons hit proxies, Skyrim applies the hit | **MGR Blade Mode gives the plane, MC applies the cut** |
| Destruction | MC digs, Skyrim geometry is hidden (`Dig*`) | MGR cuts, **MC blocks break** |

## 2. Reuse almost as is

### Protocol shape (`protocol/skycraft_protocol.h`, `fabric/.../link/Proto.java`)
- One header is the source of truth; Java mirrors it by hand with offsets as constants. `kMagic` +
  `kVersion` checked on open; a mismatch refuses to link.
- Little-endian, fixed-size structs, a `static_assert` on every size.
- **Seqlock "latest value" slots** for per-frame state (`SkyState`, `McState`): writer bumps `seq` to odd,
  copies, bumps to even; reader retries until it sees the same even `seq` twice (`Link::ReadMcState`).
- **SPSC rings** with head/tail on separate 64-byte lines (`0x00` / `0x40`, data at `0x80`): input
  (Skyrim→MC), collision and render (byte rings), events (MC→Skyrim, 32-byte records).
- **Triple-buffered overlay** with one atomic `state` word (`OverlayCtl`) for MC's HUD frame.
- **Header**: both PIDs and `GetTickCount64` heartbeats. Peer is dead after 3 s of silence
  (`kMcTimeoutMs`); the Java side tolerates longer, because loading screens stall the host.
- The host creates the mapping, resets every region it owns on create (a stale mapping can survive a
  restart while MC still holds it), and publishes `magic` last with release ordering.
- The mapping gets an explicit DACL for the current user at normal integrity (`SharedWithThisUser`), so a
  host run as administrator can still be opened by a non-elevated Minecraft. **Copy this.**
- `McState` carries raw 20 Hz tick positions plus `tickQpc` and `tickMs`, so the host interpolates on its
  own frame clock. This is exactly what Blade Mode slow-motion needs: when MC runs at a lower tick rate,
  `tickMs` tells MGR how to interpolate.

### Java link (`SkyLink.java`)
- Opens the mapping with the Java FFM API (`OpenFileMappingW`, `MapViewOfFile` through `Linker`), retries
  once a second, logs the Windows error once per reason (2 = host not up yet, 5 = elevation mismatch).
- Writes `mcPid` and its heartbeat, then reads `skyrimPid`. Uses `QueryPerformanceCounter` so timestamps
  line up across processes.
- A named mutex (`<mapping>_minecraft`) stops a second Minecraft attaching.

### Input (`skse/src/Input.cpp`, `fabric/.../client/InputBridge.java`)
- The host keeps OS focus. Its input layer is filtered: a small allow-list stays with the host, everything
  else becomes `InputEvent`s in the ring. DirectInput scan codes are mapped to the codes MC expects.
- MC's hidden window is told it has focus. A `ReleaseAll`-style reset on route changes stops stuck keys.
- SkyCraft blocks the host's *player controls* rather than disabling them, because disabling them also
  broke NPC combat. Expect a similar trap in MGR.
- MGRCraft adds a third route (Blade Mode keys stay with MGR), see the prompt §6.

### Frame loop and overlay (`Game.cpp`, `Overlay.cpp`)
- Per-frame player update runs on the host's main thread; `Present` runs even when the game is paused, so
  the heartbeat lives there, not in the game update.
- Overlay pass saves and restores every pipeline state it touches. The crosshair is drawn with MC's
  invert blend in its own pass.
- Interpolation: the host renders slightly in the past so the next MC tick has always arrived, with the
  lag measured from how late ticks really arrive. Worth porting as is.

### Block rendering (`WorldRender.cpp`, `fabric/.../render/WorldExporter.java`, `SkyAtlas.java`)
- MC exports block meshes per section plus its texture atlas over the render ring. The host draws them in
  its own frame, depth-tested against the host scene. At 3,400 lines this is the biggest file and the one
  that changes most (see §3).

### Combat bridge (`Combat.cpp`, `fabric/.../combat/*`)
- MC computes damage with vanilla code; an event carries the result to the host. The host's damage is
  refunded and sent to MC as a hurt event, so **MC owns player health**. MGRCraft keeps "MC owns
  health" and mirrors it into MGR's HUD.
- SkyCraft only trusts a hooked address after checking that the engine's own code calls it. Copy that
  habit for every MGR signature.

### Destruction (`Dig*.cpp`, `fabric/.../world/SkyDig*.java`)
- MC is authoritative for what is gone and persists it; the host only reflects it. MGRCraft reverses the
  direction of the trigger (MGR's cut asks MC to break blocks) but keeps MC as the authority.

### Tooling
- `tools/fake_skyrim.py` / `fake_guest.py`: stand-ins for each side so one half can be tested alone.
  MGRCraft gets `tools/fake_mgr.py` and `tools/fake_mc.py`.
- `skse/src/CrashLog.cpp`: unhandled-exception filter writing module + offset, the stack and a minidump,
  chained to the previous handler. Port to x86 (stack walk differs).
- `skse/src/Launcher.cpp` + `tools/minecraft-bundle/`: portable Prism instance started through Explorer
  (outside MO2's VFS). Reuse for "Minecraft starts with MGR".
- `tools/package.ps1`: release packaging.

## 3. Has to change

| Area | SkyCraft | MGRCraft | Why |
|---|---|---|---|
| Loader | SKSE | `dinput8.dll` proxy forwarding `DirectInput8Create` (built and loading, per the PC thread) | MGR:R has no script extender |
| Addresses | Address Library IDs via CommonLibSSE-NG | Signature scans, unique match, exe hash gate | No address database exists |
| Arch | x64 | **x86, verified** (Machine 0x014C) | Every DLL must be Win32 |
| Renderer | D3D11 | **D3D9, verified** (imports `Direct3DCreate9`, d3dx9_43) | Changes the whole draw path; depth readback needs INTZ or a separate pass |
| Mapping size | ~200 MB (4K overlay x3, 64 MB render ring, 32 MB collision ring) | **Must fit a 2 GB address space** | The exe is not large-address-aware (verified), and the game already uses much of its 2 GB. Today's mapping is 64 KB. Budget the full mapping at roughly 32 to 48 MB: overlay at the real back-buffer size (1080p x3 is 24 MB), a smaller render ring, no collision ring. Map views of only the regions the DLL reads if it gets tight |
| 64-bit fields | plain `u64` atomics | Same layout, but x86 needs `cmpxchg8b` for atomic 64-bit access | `std::atomic_ref<uint64_t>` handles it on MSVC/GCC; keep 64-bit fields 8-byte aligned |
| Camera | first person, MC view matrix forced | **third person**, MGR's camera follows MC's yaw/pitch; Blade Mode camera is MGR's own | Raiden is visible |
| Player | Skyrim player hidden, capsule moved | Raiden placed at MC's position, his own animations driven | Biggest unknown, Phase 0 item 4-5 |
| Combat input | MC attack key | MGR Blade Mode controls | Blade Mode is MGR input |
| Time | MC at 20 TPS | MC follows Blade Mode's time scale (`/tick rate`) | New `BladeState` slot |

## 4. Goes away

- **CollisionField / `Collision.cpp` / `SkyCollision.java` / `BlockCollisionsMixin` / `TriCollider`**:
  Minecraft's own terrain is the world. MGR's stage only has to not fight the puppet.
- **Water grid**: MC water is real water.
- **Actor table** (Skyrim NPCs → MC proxies): mobs are native to MC; the table reverses into a mob render
  table (MC → MGR), modeled on `WorldEntities`.
- **Skyrim takeover** (furniture, horses) and skill progression.
- **Mirror world** (void dimension): MGRCraft uses a normal survival world.

## 5. What this means for the MGRCraft protocol

- The PC thread's mapping is `Local\MGRCraft_v1` (64 KB, 10 Hz heartbeat), created by the MGR side. Keep
  it the single protocol; grow it region by region with SkyCraft's patterns: seqlock slots for
  `GameState` / `McState` / `BladeState`, rings for input, events, blade cuts and render data.
- Copy SkyCraft's habits that the prompt doesn't spell out: reset every MGR-owned region on create and
  publish `magic` last; an explicit DACL on the mapping; a named mutex so only one Minecraft attaches;
  the Java side tolerating a longer stall than 3 s (MGR loading screens).
- Put the offsets in one `layout.txt` that both the C++ and the Java tests check, so the two mirrors
  can't drift silently. Keep every 64-bit field 8-byte aligned and run the C++ test as a 32-bit build.
- Move the heartbeat from a timer onto `Present` once the frame hook exists: a timer thread keeps beating
  while the render thread is hung, which hides exactly the failure the heartbeat is for.
