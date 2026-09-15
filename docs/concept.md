# drap — second life (concept and research)

Living notes. Updated as we research and decide.

## Vision

Agile VR shooter with **minimal latency**, Carmack philosophy ("less latency, better performance per pixel" before fireworks). Tone references: an unreleased game in the "ILL" style, Bioshock, the HL:Alyx combat mod. 100% open source engine, no proprietary dependencies. Graphics content: AI-generated (assets aren't the bottleneck). Mid-term goal: expose the engine via **MCP** to iterate on maps/assets directly with AI tools. Possible Battlefield-Vietnam-style multiplayer mode (combined arms, latency-focused) — **separate phase**, not v1.

**This is a historic reopening of the original project (2012-2016), 10 years later.** It's not a solo experiment: the idea is to reopen it to people who test and contribute — this connects directly to the "community-built tooling" thread (maps, BSP) mentioned from the start.

**Gameplay reference: [Propagation VR](https://www.propagation-vr.com/)** (Unreal Engine) — horror/survival wave-shooter, 2-player coop, realistic gunplay with ammo/reload management, set in a subway station. It's the bar we're measuring the "feel" we want against. Concrete target: **Coop VR** (wave defense, like Propagation) + **DM VR** (PvP deathmatch, doesn't exist in Propagation, it's our own addition). Note: since it's small coop (2-4 players) against AI + small-map DM, the netcode needed is much smaller than a large-scale Battlefield Vietnam — the Quake-Wars-style combined-arms idea stays a much later phase, it doesn't block the initial design.

**Visual reference added: *Bodycam*** (2024, Unreal Engine 5) — the hyperrealistic "found-footage" shooter aesthetic: realistic light response/exposure, filmic post-processing, handheld-camera feel. Worth naming as an *aesthetic* bar (see the PBR/lighting section below for what's actually reachable on our engine family), but **its actual tech (Lumen dynamic GI + Nanite) is in direct tension with the 90Hz-or-nothing rule** — Lumen in particular is notorious for being unstable at VR framerates, which is exactly why most UE5 VR titles disable it and fall back to baked lighting. Treat Bodycam as "match the *look*," never as "adopt the tech stack."

## State of the original project (drap, 2012-2016)

- `Wintch/dhewm3`: fork of `dhewm/dhewm3`, 709 commits behind current upstream, only 11 commits of its own (minor resolution/autorun/cvar fixes, no VR).
- `Wintch/d3-base-assets`: fork of `DanielGibson/d3-base-assets`, free asset base (not the full paks — the original Doom 3 BFG is needed).
- `Wintch/drap`: SDK/compiled builds.
- Local folder `~/Documents/drap`: empty, no git yet.

## The real "ceiling" of id Software's open source

Verified against the official `id-Software` org on GitHub: the newest thing they released under GPLv3 is **Doom 3 / Doom 3 BFG (idTech4)**. idTech5 (Rage, Wolfenstein TNO) and later were never released, with no sign that will change.

- **Quake 4** (Raven): never got the same clean GPL treatment as Doom 3, only a more restrictive "SDK". The active community port **openQ4** (`themuffinator/OpenQ4`, GPL-3.0, commits this same week, 134★) is rebuilt on top of Doom3/BFG's real GPL code, not Raven's SDK — same engine lineage, not a different tier.
- **Prey 2006**: same pattern as Quake 4 (SDK + Doom3 GPL code combined by the community, e.g. `Prey2006`, `openPREY`).
- **Enemy Territory: Quake Wars**: **never released**. Only unofficial reverse engineering exists (`jmarshall23/ETQW`), legally grey — ruled out by the "no dependency on a proprietary engine" requirement. The idea of "Battlefield Vietnam on the Quake Wars engine" isn't viable as an open base; if we want that feel (large scale, vehicles, combined arms) we'd have to build the netcode ourselves on top of idTech4, using Splash Damage's GDC talks as design reference, not their code.
- **Key conclusion**: Quake4/Prey are NOT a "lower tier" relative to Doom3 — it's the same engine generation (idTech4, 2004-2006). The real decision isn't "which game", it's **which modernized fork of idTech4** we use.

## Ray tracing

- The "Quake 4 raytracing mod" you remembered (`q4rt.de`) looks like an old frameset-style site — most likely baked lighting (lightmaps computed with offline raytracing, the classic `dmap` compiler technique), not real-time GPU path tracing.
- The real hardware ray tracing reference on an id engine is **NVIDIA's Q2RTX** (`NVIDIA/Q2RTX`, custom non-GPL license but visible source, updated Dec 2025) — but that's idTech2 (Quake II, 1997), a whole generation below idTech4.
- The only idTech4 fork with an architecture seriously prepared for RT is **RBDOOM-3-BFG**: it migrated to **NVRHI** (the same DX12/Vulkan abstraction layer RT-capable renderers use), though RT itself isn't implemented there yet.

## The 60Hz vs 90Hz VR problem (more important than RT for the latency goal)

idTech4 has its game logic (usercmd, physics, script timers) tied to a fixed 60Hz tic (`USERCMD_MSEC`). Attempts to simply raise that number have existed in dhewm3 upstream for years and **remain unresolved**:

- PR #297 (2020) → draft, the author himself says it's "not even close" to mergeable.
- PR #584 (2024) → closed, broke black screens and the network protocol.
- PR #585 (2024-2026) → still in draft (August 2026), physics/ragdolls unstable above ~85fps, breaks savegames and multiplayer.
- **PR #771 (klaussilveira, open)** → the correct approach: **decouple rendering from the fixed 60Hz sim tic via interpolation** (cubic Hermite), same as Quake/Source. Game logic stays stable at 60Hz; the frame sent to the compositor runs at display refresh rate (90Hz) with interpolated/extrapolated poses. It's conceptually the same idea as Carmack's own invention (async timewarp at Oculus) — the correct answer for VR isn't forcing the sim tic to 90Hz, it's decoupling render from sim.
- RBDOOM-3-BFG already has a native `com_engineHz` (not a patch), might behave better — hasn't been thoroughly tested yet.

## idTech4 VR community reference

**Team-Beef-Studios** (DrBeef's group) is who's actually shipping VR on idTech4 today:
- **Doom3Quest** — dhewm3 + native OpenXR, commits from Sept 2026 (alive).
- **PreyVR** — same pattern, commit from March 2026.
- Both build on the VR layer from "Fully Possessed" (Doom 3 BFG mod, OpenVR/Oculus SDK, Windows-first, `KozGit/DOOM-3-BFG-VR`, last activity 2023).

They're the design reference (IK, locomotion, comfort options) even if there's no 1:1 reusable code if we end up on an engine with a different renderer (NVRHI vs classic GL).

## Tooling / community (maps, BSP)

- **DarkRadiant** (`codereader/DarkRadiant`) is still alive and maintained (updates from Feb 2026) — the de facto map editor for idTech4/Doom3 today.
- **modwiki.dhewm3.org** is the living documentation mirror (the original iddevnet is dead).
- The BSP compiler (`dmap`) and the AI one (`aas`) are part of the released engine's source code, not separate tools.

## Scope expansion: not limited to idTech

The user has their own side project (`reverb-g2`) porting VR to Linux with a **real HP Reverb G2 as a dev kit** — meaning we can measure latency/frame time on real hardware, not just in theory. **Hard rule: if it doesn't run at 90Hz, it's not good for VR.** This also broadened the scope beyond idTech4: "the best of every released engine, as long as the license allows it."

With that in mind, found a serious candidate for the **DM VR** side that changes the picture:

### ioquake3 / Quake III Arena — the cradle of id's low-latency netcode

Quake III Arena is, literally, where id Software defined the client-side prediction + snapshot interpolation standard the entire competitive shooter industry still uses. **Clean, official** GPLv2 license (`id-Software/Quake-III-Arena`'s own repo on GitHub) — without the "SDK" ambiguity Quake4/Prey have.

- **ioquake3** — the base community fork, very mature, decades of netcode refinement (Quake Live, CPMA, OpenArena all come from here).
- **Team-Beef-Studios/ioq3quest** — Quest port (same team as Doom3Quest/PreyVR).
- **Quake3VR (`ripper37/q3vr`)** — PCVR, OpenXR/SteamVR, **multiplayer with crossplay already working**, GPL-2.0, active (Feb 2026).
- **Trinity VR (`ernie/trinity-vr`)** — same lineage, **commit from yesterday** (Sept 14, 2026). This is alive right now.

In short: specifically for the **DM VR** side, working PCVR multiplayer already exists on the engine with id's best netcode heritage. No need to invent that from scratch.

### Rethinking the approach: one engine or two?

The two pillars you defined (Propagation-style coop VR, DM VR) don't necessarily call for the same engine:

- **Coop VR (wave defense vs AI, Doom/Prey setting)** → still idTech4 territory (RBDOOM-3-BFG / Prey2006+Doom3Quest-PreyVR), because that's where AI design, level scripting, and atmosphere live.
- **DM VR (PvP, minimal latency, arena)** → ioquake3/Quake3VR already solves most of the problem, with the most battle-tested netcode culture there is.

No need to decide on "one engine forever" — it's valid (and maybe more realistic) to treat them as two fronts that share a philosophy (Carmack, latency, open source) but not necessarily code.

### Legal note to keep in mind

Since you said "no limit as long as the license allows it": **id's official** releases (Doom 3, Doom 3 BFG, Quake, Quake 2, Quake 3, Wolfenstein/RTCW) are clean GPL, no ambiguity. Quake 4 and Prey 2006, on the other hand, depend on the **Raven/Human Head SDK**, which historically had more restrictive terms than pure GPL (worth reading the actual SDK license before committing to that branch, not assuming it's as free as Doom 3).

## Fundamental reframe: DRAP isn't loyal to an engine, it's a spirit

The user clarified something important: DRAP doesn't have to be loyal to idTech4's code itself — it's about the original project's **free/open, low-latency spirit**. That opens the door to non-idTech engines too, evaluated by the same bar: hard 90Hz, latency over visual fidelity, coop-vs-AI + DM PvP.

### Godot — a serious candidate outside the idTech family

- **MIT license** — as clean as it gets, none of the SDK ambiguities Quake4/Prey have.
- **Godot 4.6/4.7 (2026)**: native OpenXR 1.1, "frame synthesis" (mechanism not confirmed in detail — worth investigating whether it's real asynchronous reprojection, which would be the modern, already-solved answer to the 90Hz problem idTech4 still hasn't solved), "universal APK" support for any OpenXR headset, Forward+ (Vulkan) renderer, **RT just getting started** (initial plumbing in 4.7, not mature yet).
- **Meta is funding Godot's XR maintenance team**, and **Valve is working closely with them** — Godot has day-one support for **Steam Frame** (Valve's standalone headset, runs on Linux/SteamOS). This fits very well with your Linux/Monado/Reverb G2 stack.
- Cons: the XR team's current focus is more "mobile/standalone" than "high-performance competitive desktop shooter" — no evidence yet that it's tuned for sustained "72-90Hz in a fast PC shooter," we'd have to measure that ourselves on our own hardware.
- Multiplayer: has a built-in, actively maintained high-level API (ENet), but without the two decades of competitive netcode history the Quake3 lineage has.

### The full map now has three real fronts, not one

1. **idTech4** (RBDOOM-3-BFG / Prey2006+Doom3Quest/PreyVR) — id's literal heritage, best for coop-vs-AI mode with Doom/Prey atmosphere, but with the unsolved tic-rate problem and no RT implemented (only the path opened via NVRHI).
2. **ioquake3 / Quake3VR / Trinity VR** — the cradle of id's low-latency netcode, already with working PvP DM VR (PCVR, clean GPL-2.0), ideal specifically for the DM pillar.
3. **Godot** — modern engine, cleanest license of all, no 2004-era tech debt, active corporate backing (Meta, Valve) exactly in the area that matters most to us (XR), but without id's DNA and with RT/desktop-perf still unproven.

Not necessarily "pick one" — it's valid to use the id lineage as design/netcode reference even if the final engine is Godot, or to combine: Godot as the modern core + studying Quake3's netcode as a model to implement there.

## Decision made: id heritage, prioritize what already works at 90Hz today

The user decided: keep id heritage as the core, only port from Godot when specifically needed, and go with what **already works in VR at 90Hz today**, measured on real hardware (Reverb G2 + Linux + Monado).

**Honest check against that bar — Doom3Quest/PreyVR do NOT clear it cleanly:**
- Doom3Quest's own documentation: at 90Hz on Quest 2, "expect frame rate drops" — doesn't hold it cleanly even on its native platform.
- They're **Android/Gradle-only** projects (`build.gradle`, no Linux/SteamVR/PCVR build). Porting them to desktop is real engineering work (pulling the VR layer out of the Android packaging and into SDL2 + desktop OpenXR), not something that "already works today."

**What does clear the bar right now: Quake3VR / Trinity VR (ioquake3/Quake III Arena lineage).**
- Native build for **Linux (.deb)**, in addition to Windows.
- Documented runtime: SteamVR — but as a standard OpenXR app, in theory it should pick up whatever runtime is active (`active_runtime.json`), including **Monado directly**, without going through SteamVR. This is literally part of what you're already building in your `reverb-g2`/Monado project — worth testing as a real data point for both projects.
- Multiplayer with PC/Quest crossplay already working.
- 1999 engine (idTech3), minimal load for a modern desktop GPU — the candidate with the most real headroom to hold a clean 90Hz.
- Clean GPL-2.0 license (official id release, without Quake4/Prey's SDK ambiguity).

### First real hardware test (2026-09-14, `iashur` rig)

With the Reverb G2 connected and Monado brought up via the `reverb-g2` pipeline (`jack-in-wayland.sh 1 6dof`):

- **Monado confirms a clean 4320×2160@90.00Hz compositor lock** — a data point independent of the game, validates the sibling `reverb-g2` project's work.
- Installed **Trinity VR v1.1.2** (ioquake3/Quake3VR lineage) and pointed it straight at Monado (`XR_RUNTIME_JSON=.../openxr_monado-dev.json`, no SteamVR in the middle).
- **The OpenXR + Vulkan handshake works end to end**: Vulkan instance and device created OK via `xrCreateVulkanInstanceKHR`/`xrGetVulkanGraphicsDeviceKHR`, XR swapchain negotiated at 3024×3024 per eye. This proves `reverb-g2`'s Monado build is a valid, spec-conformant OpenXR runtime for a third-party engine — a valuable data point for that project too.
- **Bug found**: crashes creating the desktop mirror window — `VK_KHR_xlib_surface`/`VK_KHR_wayland_surface` isn't enabled on the Vulkan instance the OpenXR runtime itself creates. It's a real, reproducible bug, **different from and further along** than the already-known [q3vr issue #50](https://github.com/RippeR37/q3vr/issues/50) ("doesn't work on Wayland") — that issue's documented workaround (`SDL_VIDEODRIVER=x11`) doesn't cover this; we also tried `vr_desktopMode 0` (a cvar that exists in the binary) with no effect, the window gets created before the cvar is applied.
- Didn't get to put the headset on and verify real on-screen Hz — the app crashes before rendering anything to the headset.

**Concrete next technical step:** patch the desktop mirror window creation code in Trinity VR (add the corresponding surface extension to the instance, or skip the mirror window entirely in headless mode using `XR_MND_headless`, an extension Monado does expose) — or try the original (non-Trinity) `q3vr` binary in case it doesn't have this specific regression.

**Update:** tried the original `q3vr` (v1.0) — fails differently and worse: it defaults to OpenGL (doesn't respect `--graphics Vulkan2`), and Monado's OpenGL/GLX binding fails (`XR_ERROR_GRAPHICS_DEVICE_INVALID: glxFBConfig NULL`). Confirms the paks bundled in both releases are the **free Quake 3 Arena demo** (`DEMO_LICENSE.txt`, `Q3A_PR_v132_EULA.txt` in the package), not retail assets — no licensing issue testing with them.

Found the root cause of the Trinity VR bug in `code/vrvk/vr_vk.c`, function `VR_Vulkan_CreateInstance`: for Windows it adds `VK_KHR_WIN32_SURFACE_EXTENSION_NAME` to the Vulkan instance extensions, but **for Linux it never adds `VK_KHR_xlib_surface` or `VK_KHR_wayland_surface`** — that's why the mirror window can never create its surface regardless of which SDL backend is used. Patched (added an `#elif defined(__linux__)` branch that checks which extension the runtime offers and requests the right one), built engine-only (`-DBUILD_GAME_LIBRARIES=OFF`, didn't clone the sibling `trinity` matchmaking checkout) in `~/vr/trinity-vr-src` on `iashur`.

**Result: cleared the crash entirely.** With the patched binary (`~/vr/trinity-vr/trinityvr` + rebuilt `renderer_vulkan.so`), running directly against Monado: initialized audio, loaded the VR-aware UI QVM, computed real per-eye offsets/FOV, and **"Created stage space, centered on the head"** — meaning it's reading real headset tracking, not a stub. Process stable, no crash.

**But:** Monado's compositor log (`jack-in-wayland.log`) shows repeated `predict_next_frame_present_time: Fake pacer fell behind` warnings, in batches of ~90 consecutive skips during the run — a sign the compositor pacer is losing/skipping periods, not necessarily a clean 90Hz. Following the `reverb-g2` project's own rule ("verification is physical, someone has to put the headset on and look" — a log is never enough), **this is pending someone putting the headset on and confirming visually**: stable image or judder, and whether the game itself reports the real Hz (there's a `vr_frameTimingLog` cvar in the binary, not enabled yet in this run).

### ✅ Physical verification (2026-09-14/15): it worked, was playable

The user put the headset on and played. **First real milestone: a third-party engine we patched, running against `reverb-g2`'s Monado, playable with the headset on.** The only problem reported: the view sits way too high, "like at the very top" — a typical height/floor-calibration bug, not a rendering or framerate one.

Reviewed the `VR_Recenter` code (`vr_vk_renderer.c`): it uses `XR_REFERENCE_SPACE_TYPE_STAGE` with `y = 0.0f` (comment in the code itself: "STAGE already floors at y = 0"), exactly what the OpenXR spec calls for — the bug doesn't seem to be in the Trinity VR code we touched. More likely: (a) the floor/floor-height reported by `reverb-g2`'s Monado/WMR driver isn't calibrated the way the STAGE spec assumes, or (b) Quake 3 adds its own hardcoded player height on top of the real tracked height (a classic "double height" bug in Quake3 VR ports). Immediate fix with no rebuild: there's a `vr_heightAdjust` cvar (default 0.0) built exactly for this — lowering it to a negative value should compensate.

### ✅✅ Clean 90Hz confirmed, from the game's own log (`vr_heightAdjust -1.0`, `vr_frameTimingLog 1`)

Relaunched with `+set vr_heightAdjust -1.0 +set vr_frameTimingLog 1`. The game's internal timing log (not the compositor's) shows:

```
VR timing: 91 frames, period 11.11ms, avg 11.11ms, max 11.15ms, long(>1.5x) 0
```
repeated stably window after window (91 frames ≈ 1 second at 90fps each). **Period 11.11ms = 90.0009Hz, zero long frames (>1.5x the target period), after the single expected warmup while loading the level.** This meets the hard rule you asked for, confirmed from inside the engine — not just a compositor-side number.

### Real root cause of the height bug: not Trinity VR, it's Monado/WMR (`reverb-g2`) floor calibration

Added a temporary debug print (`HEIGHT_DEBUG`, in `vr_input.c`, already in the patched code) showing the real height the runtime reports before any adjustment. Result: **`raw_y=10.35` meters** — Monado's STAGE space is reporting the headset at ~10 meters above the "floor", when a standing person is ~1.7m. Not a Trinity VR bug or a bug in our patch: it's a floor-calibration failure in `reverb-g2`'s own Monado/WMR driver.

Why the **menu** looked fine with `vr_heightAdjust +2.0` but **gameplay** still "flew way up": the menu (`virtual_screen`) is positioned relative to the head (doesn't care about absolute height), while gameplay uses absolute height × `vr_worldscale` (32) to place the camera in the level's fixed world — with a 10m base error, that's hundreds of extra Quake units. Confirmed in code (`code/cgame/vr_cgame.c`, `CG_VR_DrawFrame`, line ~2438): `cg.refdef.vieworg[2] -= PLAYER_HEIGHT; cg.refdef.vieworg[2] += (vr->hmdposition[1] + heightOffset) * worldscale;` — the game logic is written correctly, the input data (real height) is what's wrong, coming from the engine/runtime.

**Workaround applied to keep testing today:** `vr_heightAdjust -8.65` (compensates the ~10.35m back down to ~1.9m real) — works but is a fragile patch, not the real fix. **The real fix is fixing the STAGE space floor calibration in the `reverb-g2` project** (that repo's `docs/04-lab-90hz.md` / `docs/01-bringup-monado.md` are the likely candidates to look at) — benefits everything that runs there, not just this game.

**Multiplayer:** failed to connect (not investigated further yet — no clear trace in the log from this run; could be the local `v1.1.2-dirty` build version vs. community servers/tracker, or the `trinity-tracker` we didn't set up). Pending investigation if this branch gets picked back up.

### New request: compare against other engines (AO, normal maps, POM)

The user asked to try other engines to "combine the best of all of them" — specifically **ambient occlusion, normal mapping, and parallax occlusion mapping**, which **idTech3/Quake3 (1999) doesn't have, and that's not a bug, it's the engine's generation**. Note: normal mapping isn't actually missing from any idTech4-family candidate — Doom 3 (2004) pioneered real-time normal mapping in games, it's been there from day one. AO and parallax/POM are the newer additions that vary by fork. The natural candidate is **RBDOOM-3-BFG** (PBR, soft shadows, normal maps, AO — see the engine research section above), which has no VR of its own (would need to be built), unlike the Quake3 branch which already has proven, working VR. Next step: bring up RBDOOM-3-BFG on `iashur` to evaluate its visual quality as a reference, alongside the Quake3VR branch we already tested.

**Update — built clean on `iashur` (2026-09-15).** Needed several system deps not preinstalled: `ispc` (apt), `libncurses-dev` (apt), `libopenal-dev libsdl2-dev libavcodec-dev libavformat-dev libavutil-dev libswscale-dev zlib1g-dev` (apt, batched), and DXC (Microsoft's DirectXShaderCompiler has no Debian package — downloaded the official `linux_dxc_2026_07_29.x86_x64` release binary directly and put it on `PATH`, ShaderMake's CMake only auto-detects a plain `dxc` on `$PATH` on Linux, `-DDXC_CUSTOM_PATH` alone doesn't work as documented). Full engine build succeeded with zero errors, including the entire NVRHI/Vulkan renderer backend (`renderer/NVRHI/RenderBackend_NVRHI.cpp` and friends) — real confirmation the RT-capable rendering path compiles clean on this box. Binary: `~/vr/rbdoom3bfg/neo/build/RBDoom3BFG`. No Doom 3 BFG assets owned on this machine yet, so it can't be run to completion — build-and-inspect only for now, matching the "assemble the best of what's free first, plug in drap's own assets after" sequencing the user set.

### MegaTexture — is there a chance?

Checked directly in the RBDOOM-3-BFG source we already cloned (`neo/renderer/BinaryImage.h`): a real comment reads *"Also used in a memory-mapped form for imageCPU for offline megatexture"*. Confirmed independently on Wikipedia's MegaTexture article: the technique was **first introduced in idTech4 itself**, in an early/offline form, before Splash Damage built the full real-time streaming version for Enemy Territory: Quake Wars and id took it further in Rage (idTech5). Since idTech4's GPL release genuinely includes this early groundwork, there's a real starting point here — not the full Rage-grade virtual texture streaming system (that stayed proprietary in idTech5/6, never released), but the original id Tech 4 infrastructure it grew out of.

If we want true modern sparse/virtual texturing beyond that early groundwork, **`wzhijiang/libvt`** is an independent open-source virtual texturing implementation built in the style of Carmack/Sean Barrett's technique — a from-scratch alternative rather than id's own code, worth a look if we decide virtual texturing is worth the engineering investment for large open outdoor areas.

**Much stronger lead, found by cloning Prey2006 (see below): it has real, dedicated `neo/renderer/MegaTexture.cpp` / `MegaTexture.h` source files**, not just a passing comment like RBDOOM-3-BFG. Human Head's Prey (2006) shipped actual working MegaTexture infrastructure (used for its planet-surface/portal-world terrain), also referenced from `Material.cpp`, `RenderSystem_init.cpp`, `tr_local.h`, `Model.cpp`. This is real, GPL, ready-to-study id Software MegaTexture code — genuinely the best answer to "is there a chance."

### Other good open idTech-derived engines: The Dark Mod

Found a strong one: **The Dark Mod (TDM)** — a standalone idTech4-based game/engine (started as a Doom 3 mod, went fully standalone and GPL in 2013), maintained by the same people behind DarkRadiant. Two things make it directly relevant:

- **`fholger/thedarkmod`** ("Experimental porting of Doom3 BFG features to The Dark Mod", pushed July 2025, C++, 27★) — literally porting RBDOOM-3-BFG's modern rendering work into TDM. Verified directly in its shader source (`glprogs/`): real **SSAO** shaders (`ssao.frag.glsl`, `ssao_blur`, `ssao_depth`, `ssao_depthmip`), real **parallax mapping** (`tdm_parallax.glsl`, `parallaxCubeReflect`), and soft shadow stencil shaders. It also merges in dhewm3 fixes — three idTech4 lineages converging in one codebase.
- Crucially, **TDM ships its own complete, free assets** — no retail Doom 3 needed at all, unlike RBDOOM-3-BFG or vanilla dhewm3. That directly solves the asset-ownership problem we hit with RBDOOM-3-BFG on `iashur` (no Doom 3 BFG owned on that machine).
- **`fholger/thedarkmodvr`** ("VR support for The Dark Mod", 80★) already exists — but last pushed 2022, doesn't clear our "works today" bar the way Quake3VR/Trinity VR does. Still a real design/code reference if we ever build VR on a TDM-based core.

**Concrete short-term plan:**
1. Install and test **Quake3VR/Trinity VR** on the Reverb G2 over Linux/Monado this week — first real motion-to-photon data point on our own hardware, and cross-validates the Monado/reverb-g2 stack. **(Done — see above.)**
2. For the coop-vs-AI pillar (Doom/Prey): no shortcut clears the 90Hz-on-desktop bar yet. Start from **dhewm3** (or the Prey2006 SDK+GPL desktop port) and build the VR layer + render/sim decoupling (dhewm3 PR #771's pattern) directly for Linux/OpenXR, using Doom3Quest/PreyVR/Fully Possessed's code as a design reference (IK, locomotion, comfort options) to port, not as an already-working base.

### Prey2006 — cloned and checked on `iashur`

Cloned `FriskTheFallenHuman/Prey2006` (desktop, GPL, active — pushed Sept 4 2026) to `~/vr/prey2006`. No VR here (that's `PreyVR`/Team-Beef-Studios, Android-only, same porting effort as Doom3Quest would need). What its source confirms:

- **Real MegaTexture implementation present** (see above) — the standout feature this codebase brings that RBDOOM-3-BFG/TDM don't have.
- Rendering-wise it's stock 2006-era idTech4: `base/glprogs/` has exactly one shader file (`megaTexture.vfp`, classic ARB vertex/fragment program format), normal mapping is baked into the core interaction shader (inherited from Doom 3, not a separate technique), but **no AO, no POM** — those are post-2006 additions this codebase predates.
- **Update — built clean on `iashur` (2026-09-15).** `./cmake_linux.sh gcc release` + `ninja`, zero errors across 678/678 targets. Produced `output/linux/prey06` (client) and `output/linux/prey06ded` (dedicated server), plus packaged `pak007.pk4`/`game02.pk4`. Same situation as RBDOOM-3-BFG: no retail game assets owned on this box, so build-and-inspect only for now, per the "assemble the free stuff first, plug in drap's own assets after" sequencing.

## Pending decision: engine base

Not decided yet. Real candidates:

1. **RBDOOM-3-BFG (NVRHI)** — DX12/Vulkan, native `com_engineHz`, a path to RT, most active mainline. No existing VR layer on this version — would have to be built.
2. **dhewm3 / Doom3Quest** — OpenGL, VR already proven and working (Team-Beef-Studios), but no path to RT and the tic-rate hack still unresolved upstream.
3. **openQ4** — alive, GPL-3.0, experimental Vulkan, but behind RBDOOM-3-BFG on renderer modernization, no RT.

Given the minimal-latency mandate (DX12/Vulkan has less driver overhead than classic OpenGL), the technical balance points to RBDOOM-3-BFG — but that means building the VR layer + render interpolation from scratch instead of inheriting Doom3Quest/Fully Possessed's.

### Real cost comparison: port modern rendering to Quake3, or fix 90Hz on a modern idTech4 fork? (2026-09-15)

Direct question worth answering with source code, not guesswork: since Quake3VR/Trinity VR already clears 90Hz cleanly and idTech4 doesn't, which is the cheaper path — bring Doom3-era rendering (normal maps/AO/POM) to Quake3, or finally fix the tic-rate problem on a modern idTech4 fork that already has that rendering? Checked both sides directly in the source we already have cloned on `iashur`.

**Why Quake3 clears 90Hz for free — confirmed in Trinity VR's actual source, not folklore.** In `code/cgame/cg_view.c`: `cg.frametime = cg.time - cg.oldTime` — the client (render, camera, and therefore VR head tracking) runs on its own free-running wall-clock timer, completely decoupled from the server's simulation rate (`sv_fps`, historically 20-40Hz). This is Quake III's client-side-prediction-plus-snapshot-interpolation netcode, built in 1999 for competitive multiplayer feel — VR just rides on top of an architecture that was never coupled to a fixed sim tic in the first place. It's not a fix anyone made for VR; it's foundational, 27-year-old design.

**Why "just port the rendering to Quake3" is more expensive than it looks.** Checked Trinity VR's own renderer (`code/renderervk/`, a Vulkan port, not the legacy GL path) — despite running on a modern graphics API, the lighting model is still 1999-era: `tr_light.c`'s `R_SetupEntityLighting`/`R_SetupEntityLightingGrid` is Quake3's original **light grid** (precomputed ambient/directional samples baked into the BSP, sampled per-entity), not per-pixel shading. No normal-map, parallax, or SSAO shader exists anywhere in `code/renderervk/shaders/` (checked directly — what's there is bloom/blur/fog/gamma/UI shaders, nothing material-lighting-related). Adding real normal maps + AO + POM here means building an entire per-pixel shading pipeline from scratch: tangent-space generation for geometry formats that never carried it, new material/shader-script stages, a screen-space AO pass. That's the same order of engineering RBDOOM-3-BFG spent ~13 years on, or TDM spent 22 — not a bounded port.

**Why fixing 90Hz on RBDOOM-3-BFG looks cheaper than it looks — `com_engineHz` is real infrastructure, not a renamed cvar.** Read `neo/framework/common_frame.cpp`'s actual frame-pacing loop: it's a proper accumulator-based fixed-timestep game loop (the "Fix Your Timestep" pattern) that can run **multiple game-simulation frames per render frame** based on `com_engineHz` — meaning raising it genuinely raises the simulation rate itself, not just a render-side interpolation trick. And in `neo/framework/Common_load.cpp:510`: `const float mpEngineHz = ( com_engineHz.GetFloat() < 90.0f ) ? 60.0f : 120.0f;` — multiplayer already auto-selects a real 120Hz simulation the moment you ask for ≥90. This was built by id itself for the 2012 BFG Edition to support PC/PS3/Xbox360 at different fixed framerates in the same shipped, certified console product — a fundamentally different foundation than dhewm3's six years of unmerged community attempts (PR #297/#584/#585) to bolt a variable tic onto an engine that was never designed for it.

**Conclusion: fixing 90Hz on RBDOOM-3-BFG is very likely the cheaper, lower-risk path**, because the expensive part (modern per-pixel rendering — normal maps, PBR, real SSAO, GI) is already built and working there, while the missing part (a genuine >90Hz simulation) already has serious first-party engineering behind it, not a hopeful patch. Porting rendering to Quake3 would mean re-doing a decade-plus of someone else's rendering-architecture work from a strictly older baseline. **Not yet verified hands-on** — `com_engineHz`'s real behavior at 90/120Hz (frame timing, physics stability, whether it plays nice with VR head-tracking) still needs the same rigor as the Trinity VR test (`vr_frameTimingLog`-equivalent measurement on real hardware) — blocked on the same thing as running RBDOOM-3-BFG at all: no owned BFG-format game data yet on `iashur` (see the demo-asset investigation below).

### Feature-completeness check across the three idTech4 forks (2026-09-15)

Asked directly: which fork is the most feature-complete, given years of independent modder work? Pulled current (2026) changelogs for all three and cross-checked the RBDOOM-3-BFG claim against its actual source (already cloned on `iashur`), not just search summaries.

| | **RBDOOM-3-BFG** (RobertBeckebans) | **The Dark Mod** (TDM, v2.14, March 2026) | **DIII4A / idTech4A++** (glKarin) |
|---|---|---|---|
| Renderer | **Vulkan/DX12 via NVRHI**, PBR (GGX Cook-Torrance), baked GI, Intel MOC occlusion culling | OpenGL only | GLES 2.0/3.0 only (mobile-first) |
| AO | Real SSAO (`SsaoPass.cpp`) | SSAO, extended in 2.14 to work inside subviews/mirrors too | Soft shadow mapping + GI, no dedicated AO pass found |
| Normal maps | Yes (native since Doom 3) | Yes, plus 2.14 added light-interactive decals (normal-mapped footprints/blood/bullet holes) | Yes |
| **PBR (roughness/metalness)** | **Confirmed real, source-verified (2026-09-15)**: `neo/shaders/BRDF.inc.hlsl` (actual BRDF math), auto-detects roughness/metallic maps by filename convention (`Material.cpp`), proper channel packing, image-based lighting (`ambient_lighting_IBL.ps.hlsl`, `ambient_lightgrid_IBL.ps.hlsl`), deferred G-buffer path (`gbuffer.ps.hlsl`) — a genuine modern material pipeline, not a marketing claim | **Confirmed absent, source-verified**: zero hits for roughness/metalness/GGX anywhere in `renderer/` or `glprogs/` — stays on Doom 3's classic diffuse/specular/normal model despite the 2.14 AO/POM work | Selectable Phong/Blinn-Phong/**PBR**/no-lighting modes per its own 2026 changelog (not source-verified here, web-confirmed only) |
| **POM** | **Not implemented** — verified directly in source: the only "parallax" hits in `neo/` are parallax-*corrected cubemap reflections* (a reflection-probe technique), not texture-displacement POM | **Added in 2.14 (March 2026)** — a "long requested effect," per TDM's own release notes | Not confirmed |
| Community mapping tooling | **TrenchBroom support + a standalone BSP compiler** (both added recently) — strongest dedicated tooling story of the three | DarkRadiant (mature, dedicated, same team) | None — Android-first, no editor story |
| VR | None built | `thedarkmodvr` exists but stale since 2022 | None |
| Years of continuous modding | ~13 (2013-2026) | **22 (2004-2026) — by far the longest-running idTech4 codebase that still exists**, originally a Doom 3 mod, went standalone GPL in 2013 | ~10+, very high commit frequency but mobile-porting-focused, not feature-innovation-focused |

**Conclusion — it splits by axis, and that's actually useful:**
- **Most modern rendering architecture** (the thing that actually matters for our Vulkan/low-latency VR requirement): **RBDOOM-3-BFG**, no contest. TDM and DIII4A are both stuck pre-Vulkan.
- **Most feature-complete overall, purely from accumulated modder-years**: **The Dark Mod**, clearly — 22 years of continuous, dedicated development beats everything else in the family, and it just shipped the exact feature (POM) RBDOOM-3-BFG is still missing.
- **Broadest multi-game compatibility**: DIII4A, but it's the weakest fit for us specifically (no Vulkan, no VR, mobile-first).

**Practical takeaway for drap:** RBDOOM-3-BFG stays the right *rendering/engine* base (Vulkan is a hard requirement TDM/DIII4A can't give us) — but **The Dark Mod is now confirmed as the best single feature/reference source to port from**, not just a "nice to compare against." Its POM is brand new (2026) and exactly the missing piece, and `fholger/thedarkmod` (already found earlier, see below) already proved this direction works in reverse — it ported RBDOOM-3-BFG's SSAO/parallax *into* TDM. That existing diff is effectively a working translation guide for porting TDM's new POM shader *back* into RBDOOM-3-BFG's NVRHI/HLSL pipeline, instead of writing it from scratch.

**On "realistic light rendering" (the *Bodycam*-style bar):** RBDOOM-3-BFG's PBR pipeline already has real image-based lighting and **baked global illumination** — precomputed at build time, not dynamic. That's the honest, reachable version of "realistic light" for us: good baked GI + real PBR materials + SSAO (already there) + POM (portable from TDM) gets us most of the *look* Bodycam is known for, without needing Lumen-style fully-dynamic real-time GI — which would both be a multi-year engineering project from scratch on this engine family and directly fight the 90Hz VR requirement. If we ever want *some* dynamic bounce lighting beyond static bakes, cheaper real-time-GI techniques (light probes updated at low frequency, screen-space GI as a supplement to the SSAO pass already in place) are the realistic middle ground — full Lumen-equivalent is not, at least not for v1.

### The Dark Mod — built, installed, and actually running on `iashur` (2026-09-15)

Unlike RBDOOM-3-BFG and Prey2006 (compile-only, no owned retail assets), **TDM ships its own free assets, so this is the first engine in the whole comparison we can actually run and see, not just compile.**

Build chain was heavier than the other two — TDM uses **Conan** (not vcpkg/system packages) for third-party deps (ffmpeg, tracy, fltk, openal, glfw, zlib, curl, mbedtls, jpeg, png, vorbis, ogg, pugixml, doctest, minizip, alsa), all built from source via `conan install` in an isolated venv (`~/vr/thedarkmod/.conan-venv`, no system pollution, no sudo needed for Conan itself). Needed 4 more system packages via apt (`subversion`, `mesa-common-dev`, `xorg-dev`, `libglu1-mesa-dev`). One real gotcha: `GAME_DIR` in `CMakeLists.txt` defaults to the sibling path `../darkmod`, and if that path doesn't already exist as a directory, CMake's first `cmake -E copy` (a single-file copy) silently creates it as a *file* instead — which then breaks the next step (copying `glprogs/` *into* it) and makes Make delete the just-built binary as a safety rollback. Fix: `mkdir` the sibling `darkmod` directory before building.

Game assets (5.0GB, 64 `.pk4` files) came from TDM's own `tdm_installer`, run with its documented `--unattended` flag for headless installs — except it's not fully headless, it still needs a real display connection (`Can't open display` otherwise); pointed it at the existing Wayland/XWayland session on `iashur` the same way the Trinity VR work did (`DISPLAY=:0`, `XAUTHORITY=/run/user/1000/.mutter-Xwaylandauth.*`).

**Confirmed launching cleanly**, real GPU: `OpenGL renderer: NVIDIA GeForce RTX 3060 Ti`, OpenAL audio init OK (found 256 hardware voices, HRTF enabled), GLSL shaders linking, and the version string confirms it's genuinely the current release: `The Dark Mod 2.14/64, lin` — the same version with the new POM and subview-SSAO discussed above, not an older cached build. This is now the reference engine to actually look at (screenshots/headset) for the AO/normal-map/POM visual comparison, since it's the only one of the three we can run end to end today.

### Actually seen it run — visual verification + a reusable remote-input finding (2026-09-15)

Launched the "Tears of St. Lucia" official campaign (`+map saintlucia` on the command line, to skip needing menu navigation) and got real in-level screenshots: a torch-lit alley with a barrel, hanging buckets on chains, a shovel, scattered debris, cobblestone floor, and a lantern casting soft dynamic light and shadow across a stone wall. Real prop density, real lighting falloff, visible surface detail on stone/wood — this is a genuinely good-looking idTech4 scene by 2026 standards, confirming the "22 years of modder polish" conclusion above wasn't just a changelog claim.

**Reusable finding for remote/automated testing (relevant to the later MCP-server idea too):** `xdotool`'s synthetic mouse/keyboard events (X11 `XTEST` extension) reached the TDM window (confirmed focused via `xdotool getactivewindow`) but had **zero effect** on the game — no menu navigation, no console toggle, nothing. This wasn't a TDM bug: `mutter` (the Wayland compositor on `iashur`) silently drops synthetic `XTEST` input for security reasons. The fix that actually worked: Python's `evdev` library (`pip install evdev` in a throwaway venv, no system package/sudo needed beyond the `/dev/uinput` group access the `iam` user already had) creating a virtual input device through the kernel's `uinput` interface — since this looks like *real hardware* to the compositor rather than synthetic X11 events, it goes through untouched. `ydotool` (a CLI wrapper around the same `uinput` mechanism) would do the same thing but wasn't packaged for this Debian install; the `evdev` venv route is a fine substitute and needed zero sudo. **Worth remembering for any future remote automation of these engines** (this is likely to matter again for the eventual MCP-driven iteration workflow) — `uinput`-based injection works, `XTEST`-based tools (xdotool, most "automation" tutorials) silently don't, on this kind of modern Wayland desktop.

## Free-assets survey (parallel research, 2026-09-15)

Ran three independent research threads to answer: what does drap's own original asset base actually contain, are there other free idTech4 derivatives worth knowing about, and what does the Quake4/Prey SDK license actually say (verbatim, not inference).

### What's actually in drap's own assets (`Wintch/d3-base-assets`)

This turned out to be much more substantial than assumed. Two different repos exist:

- **Upstream `DanielGibson/d3-base-assets`**: a thin SDK skeleton — 98 files, ~1.74MB. Mostly editor/utility textures (collision, clip, trigger icons, "dummy" placeholders), a handful of `.def`/`.mtr`/`.gui`/`.script` files, one test map. No models, no sounds, no real levels — explicitly incomplete (README/TODO admit missing entity defs, no main menu, "black screen"). License: **WTFPLv2** (maximally permissive), with a sub-block for ReFlex's textures (also WTFPLv2) and William Joseph's caulk/nodraw/clip textures (public domain).

- **The actual `Wintch/d3-base-assets` fork — drap's real data — is a genuinely complete, playable multiplayer base**: 864 files. 200 `.md5anim` + 13 `.md5mesh` (real character/weapon models), 149 `.ogg` sounds + 11 sound shaders, 212 `.tga` + 147 `.dds` textures, 8 particle files, **real deathmatch maps** (`dm1`, `dm4`, `dm5`, `dm6.map`), fonts, an intro video. This is not placeholder art.

**The catch:** the fork's own `COPYING.txt` explicitly flags `base/glprogs/*` (shaders), `base/fonts/*`, and `base/models/md5/*` as files that "ARE or CAN be still covered by the original Doom3 EULA" — i.e., drap's own maintainers weren't fully sure those specific pieces were clear of Doom 3's commercial license, and the README tells users to buy retail Doom 3. Weapon models are credited to a ModDB addon ("Weapon Ops for Doom 3" by bladeghost) claimed public domain, unverified. **Before "plugging in drap's assets" as planned, this needs a real pass**: confirm or replace the shaders/fonts/core models flagged as EULA-uncertain, since everything else (WTFPLv2) is genuinely free and clear.

**Update (2026-09-15) — `base/script/*` is a bigger, undisclosed instance of the same problem.** The author confirmed directly: the weapon scripts were never rewritten. Verified by diffing `Wintch/d3-base-assets` against upstream `DanielGibson/d3-base-assets`:

- Upstream's `doom_main.script` is a 25-line empty player-object stub. Upstream's `doom_util.script` header literally says *"recreated scripts from doom_util.script — (C) 2012 Daniel Gibson, WTFPLv2 — FIXME: almost all functions are missing!"* — Daniel Gibson deliberately did **not** reimplement the real content, precisely to stay clear of Doom 3's retail script data.
- Drap's fork has the **full, complete versions** instead: `doom_defs.script` (175 lines), `doom_util.script` (389 lines, every function Gibson's stub omits — `fadeOutEnt`, `crossFadeEnt`, `interpolateShaderParm`, etc.), plus `ai_base.script`, `ai_player.script`, `doom_events.script` (1228 lines), and all five `weapon_*.script` files (912 lines combined) — **3599 lines total, none of which exist in the free upstream base at all.**
- Smoking gun: `ai_player.script`'s own header comment still reads `doom_player.script` — the actual stock Doom 3 retail filename, left over from a straight extract-and-rename. Content matches known Doom 3 SDK balance values verbatim (`PISTOL_FIRERATE 0.4`, `SHOTGUN_NUMPROJECTILES 16`, etc.).
- The matching `base/def/weapon_*.def` files show the same pattern (checked `weapon_pistol.def`: stock `export fred {}` anim/sound block, not original).
- **None of this is mentioned in `COPYING.txt`**, which only flags glprogs/fonts/md5 models. So the real scope of "possibly-EULA'd content" is larger than drap's own license file discloses: essentially all game-logic scripting + weapon entity defs, not just three folders.

**What's clean, per the author's own read:** maps (`dm1`/`dm4`/`dm5`/`dm6`, built with the author's own map-editor knowledge) and — outside the flagged "core" categories — the rest of the asset base is original work. So the practical split for the revival is: **treat `base/script/*`, `base/def/weapon_*.def`, `base/glprogs/*`, `base/fonts/*`, and `base/models/md5/*` as retail-derived and replace them** (rewrite weapon scripts from scratch against the GPL'd game-logic C++, regenerate shaders for whichever renderer we land on, source new fonts/models); **keep everything else** (maps, textures, sounds, particles) as drap's own. This is a small, well-scoped rewrite — weapon scripting logic is exactly the kind of thing AI-assisted authoring (already the plan for graphical content) handles well, and it needs to happen anyway once VR interaction (motion-controlled aim/reload/throw) replaces the original 2D-mouse weapon logic.

### Two more active idTech4 forks found

- **`glKarin/com.n0n3m4.diii4a`** ("DIII4A" / idTech4A++, Harmattan Edition) — 606★, GPL-3.0, near-daily commits. A unified engine that runs Doom 3, Quake 4, Prey (2006), Doom 3 BFG, The Dark Mod, RTCW, Quake 1-3, Enemy Territory, Jedi Knight (OpenJK), and Serious Sam from **one codebase**, targeting Android/Windows/Linux. Multi-threaded GLES 2.0/3.0 renderer with soft shadow mapping, PBR/Phong lighting, global illumination, broad model-format import (obj/dae/md5/psk/iqm/gltf/fbx). No Vulkan, no VR, no RT — but worth knowing about for its breadth (one engine, many idTech4-family games) and mobile-port maturity.
- **`MadDeCoDeR/Classic-RBDOOM-3-BFG`** ("DOOM: BFA — Big Freaking Anniversary Edition") — 264★, GPL-3.0, active. A RBDOOM-3-BFG fork that runs classic Ultimate DOOM/DOOM 2/Final DOOM/Master Levels **inside** the BFG-derived engine, with rendering work not in upstream RBDOOM-3-BFG: cascaded PCF hardware shadow mapping, Half-Lambert lighting, true 64-bit HDR with adaptive tonemapping, enhanced SMAA, filmic post-processing (film grain, Technicolor grading). No RT/VR.
- Smaller, real, worth noting: **`klaussilveira/chocolate-doom3-bfg`** (preservation-focused, opposite philosophy from RBDOOM-3-BFG's feature creep) and **`Stradex/librecoop`** (66★, GPL-3.0, active Jan 2026) — **adds cooperative multiplayer to dhewm3**, which none of the other known forks provide. Directly relevant to our Coop VR pillar — worth a real look.
- Confirmed dead/not worth pursuing: vkDOOM3 (905★ but archived since 2023), fhDOOM, RobertBeckebans/TEKUUM-D3, burektech2.
- Confirmed: no additional VR-specific idTech4 fork exists beyond Team-Beef-Studios' work, and no real-time ray tracing exists anywhere in the idTech4 family — every "PBR" claim found is rasterized, not path/ray traced.

### Quake4/Prey SDK license — confirmed from the actual EULA text

Located and read the verbatim SDK license (`EULA.Development Kit.rtf`, shipped inside `themuffinator/openQ4-GameLibs` and `themuffinator/OpenPrey-GameLibs` — the Prey copy even has a leftover "ID" reference, confirming Human Head reused id's own template). **This is a standard restrictive modding EULA, not GPL-style, confirmed by direct quotes:**

- Requires owning the full retail game: mods "shall operate only with [QUAKE 4/PREY] (but not any demo, test, or other version)."
- **Non-commercial only**: "shall not rent, sell, lease, lend, offer on a pay-per-play basis, or otherwise commercially exploit or commercially distribute" — free distribution to end users only.
- No standalone redistribution, no reverse engineering of the SDK itself, automatic termination on breach.

**Implication:** the underlying idTech4 *engine* code is genuinely GPLv3 either way, but the game-logic layer that openQ4/Prey2006/PreyVR build on top of it (the SDK-derived code) stays under this restrictive, retail-gated EULA — real legal weight if we ever want something standalone/freely-distributable/commercial on that branch, not just a "mod."

## The full idTech3/idTech4 family, mapped — what each game actually innovated

The user asked to check Wolfenstein and "all released idTech games" for anything worth combining, specifically calling out FAKK2 (Heavy Metal) and Alice. Went through id Software's own GitHub org plus every known idTech3-licensee game to build a real license + innovation map, not just a list.

### Confirmed clean GPL, straight from `id-Software`'s own GitHub org

`wolf3d`, `RTCW-SP`, `RTCW-MP`, `Enemy-Territory` (**Wolfenstein: Enemy Territory**, the standalone 2003 multiplayer game — distinct from, and not to be confused with, **Enemy Territory: Quake Wars**, the idTech4 game that was *never* open-sourced, see above), `Quake`, `Quake-2`, `Quake-III-Arena`, `DOOM`, `DOOM-3`, `DOOM-3-BFG`, plus `Quake-Tools`/`Quake-2-Tools`/`GtkRadiant`. This is the full, real ceiling of what id itself has ever released — no surprises beyond what's already in this doc, except that **Wolfenstein: Enemy Territory itself (not just RTCW) is clean GPL**, which matters a lot for the community/tooling goal (see below).

### Also clean GPL, released by a different rights holder — Star Wars Jedi Knight II/Jedi Academy

Raven Software (not id) released **Jedi Outcast** and **Jedi Academy** under GPLv2 in 2013, as a tribute to the fanbase after LucasArts closed. Both are idTech3 derivatives, heavily modified by Raven: the **Ghoul2** segmented/skeletal model system (dynamic mesh attachment, dismemberment — advanced character rendering for a 2002 engine) and a real physics-driven lightsaber combat system are the two standout technical contributions. **`JACoders/OpenJK`** is the actively maintained community fork (full SP+MP for Jedi Academy, SP only for Jedi Outcast) — a fourth genuinely-GPL idTech3-family codebase alongside Quake3/RTCW/ET, worth knowing about even if saber combat itself isn't relevant to drap.

### NOT GPL — Heavy Metal: F.A.K.K.² and American McGee's Alice (confirmed via the actual EULA text)

Checked directly (`a1batross/fakk2-sdk`'s `license.rtf`, Ritual Entertainment's own text) — **this is the exact same restrictive-modding-EULA pattern as Quake4/Prey, not GPL**:
- *"You cannot sell or otherwise commercially exploit or utilize the shareware, registered/full retail version, demo or level editor in any way."*
- *"You are not permitted to copy or otherwise reproduce the SOFTWARE or ACCOMPANYING MATERIALS; modify or prepare derivative copies."*
- Personal use only, free redistribution of user-made levels only — no commercial use, no derivative redistribution.

Ritual licensed a **snapshot of Quake III's engine code from id in February 1999** (before Q3A's own GPL release existed) and built FAKK2 on top of it — that snapshot itself was never released under GPL by anyone. **American McGee's Alice (2000, Rogue Entertainment)** was built directly on FAKK2's own modified codebase, so it inherits the same restrictive status — incomplete/partial code floats around via the `zturtleman/spearmint` community project, but there's no clean open release to build on. **Practical conclusion: neither is usable as a code base**, same as Quake4/Prey — they're valid *design* references only (see below), never code we can pull from.

### What each one actually pioneered (the "map the innovations" ask)

| Game | License | What it pioneered / is genuinely good at |
|---|---|---|
| **Quake III Arena** (1999) | GPL-2.0 | The client prediction + snapshot interpolation netcode model the entire genre still uses (see the ioquake3 section above). |
| **RTCW** (2001, Gray Matter/Nerve) | GPL | First WWII shooter to popularize objective-based squad multiplayer; AI stealth/alertness system for guards in SP. Laid the groundwork Splash Damage then took further in ET. |
| **Wolfenstein: Enemy Territory** (2003, Splash Damage) | GPL | **The genre-defining class-based objective multiplayer template** (Medic/Engineer/Soldier/Covert Ops) — still one of the most respected competitive shooter communities that exists, with `ETLegacy` an actively maintained fork (v2.85, Aug 2026) and a real living anti-cheat/mod ecosystem (ETPro, ShrubBot). **This is the single best real-world reference for drap's own "reopen to community, build tooling" goal** — it's the same problem (keep a 20+ year old idTech-family game alive via community-built tools), already solved, worth studying directly. |
| **Jedi Outcast / Jedi Academy** (2002/2003, Raven) | GPL | **Ghoul2** dynamic mesh/skeletal system (dismemberment, bolt-on attachments); physics-driven melee (lightsaber) combat — the best id Tech 3-family reference if we ever want serious melee combat in DM VR beyond guns. |
| **Soldier of Fortune** (2000, Raven) | Never released — only id's underlying id Tech 2 (Quake II) is GPL; Raven's own game code isn't | Where Raven's dismemberment tech starts: the original **GHOUL** system (36 gore zones, bolt-on model attachments, compressed animation data) — id Tech 2, one generation before SOF2. |
| **Soldier of Fortune II: Double Helix** (2002, Raven) | Never officially released — Raven promised an SDK for March 2003, it never shipped; only an unofficial, non-Raven reconstruction exists (`ensiform/SOF2MPSDK`) | Moved GHOUL to id Tech 3 as **GHOUL II** — and this is the same tech Raven reused right after in Jedi Outcast/Jedi Academy. **Practical upshot: we already have legal access to SOF2's actual dismemberment engine, just through the Star Wars games' GPL release instead of SOF2's own (never-released) code** — same studio, same tech, same era, no need to chase SOF2's source specifically. |
| **Heavy Metal: F.A.K.K.²** (2000, Ritual) | Restrictive EULA, not usable as code | First to push id Tech 3 into third-person action-adventure territory (camera + melee on an engine built for first-person arena shooters) — design reference only. |
| **American McGee's Alice** (2000, Rogue) | Restrictive EULA (inherits FAKK2's), not usable as code | Not a technical/rendering innovator — its value is entirely atmosphere/art-direction/narrative design, worth keeping in mind for tone references, never for code. |

### VR angle: another Team-Beef-Studios data point

**`Team-Beef-Studios/RTCWQuest`** exists — same team as Doom3Quest/PreyVR/ioq3quest, migrated from the old Oculus VrApi SDK to OpenXR, but **Android/arm64-only** right now, same "needs real porting work, not already-working-on-desktop" caveat as Doom3Quest. `CactusVRStudios/RTCW-PCVR` is an aspirational, very early-stage attempt at a PCVR port — not something to rely on yet. Net effect: doesn't change the "Quake3VR/Trinity VR is what already clears the 90Hz-on-desktop bar today" conclusion, but confirms the same team is independently validating the same "port the VR layer to OpenXR/desktop" work pattern across the whole idTech3 family — useful precedent if we ever do that work ourselves for ET or RTCW.

## Demo-asset investigation: free game data for RBDOOM-3-BFG/dhewm3 testing (2026-09-15, in progress)

Both Doom 3 (2004) and Prey (2006) had official free demos — a possible way to get real, legally-distributable game data to actually run/test engines on `iashur` without buying anything, the same way the free Quake 3 Arena demo already unblocked Trinity VR testing.

- **Downloaded both** (via Internet Archive mirrors): `Doom3.exe` (483MB) and `Prey.exe` (470MB), now in `/mnt/resolve_test/drap-vault/demos/{doom3,prey}/`.
- **Important asymmetry, checked against RBDOOM-3-BFG's own docs**: RBDOOM-3-BFG requires **BFG Edition**-format game data specifically ("only for BFG Edition" — confirmed via its own wiki/README, no original-2004-format support). The Doom 3 (2004) demo is **not** BFG-format, so it won't unblock RBDOOM-3-BFG directly — it would only be useful for testing vanilla **dhewm3** (which we haven't cloned/built yet) or another original-format idTech4 fork. The Prey demo, by contrast, targets the same 2006 game Prey2006 (our already-built engine) already expects — much more likely to be a direct, clean match.
- **Blocked on extraction, not yet resolved**: both are old (2006-era) InstallShield-packaged `.exe` installers. `unshield` (the standard Linux tool for these) can't open either file directly — the installer data appears to be embedded as a PE resource rather than laid out as separate `.cab`/`.hdr` files, which `unshield` expects. Next step would be pulling the embedded cabinet out first (`cabextract`/`icoutils`, not yet installed) before `unshield` can run. **Have not yet been able to read either demo's actual EULA text** — so licensing (same "official demo, freely distributable" pattern as the Quake 3 demo, or something more restrictive) is still unconfirmed, not assumed clean.

## Dev environment note: storage convention on `iashur` (2026-09-15)

The root SSD on `iashur` filled up (91% full, 20GB free) after building RBDOOM-3-BFG + Prey2006 + The Dark Mod. **All drap-specific engine clones, builds, and downloaded game/demo data now live on `/mnt/resolve_test/drap-vault/`** (407GB free), symlinked back into `~/vr/<name>` so every path already written down in this doc keeps working unchanged. Applies going forward too — new engine clones/builds should be created directly in the vault path and symlinked, not dropped into `~/vr/` first.

**Scope is drap-only.** `~/vr/` also holds the user's separate `reverb-g2` project files (`monado/`, `xrizer/`, `logs/`, `proton-prefixes/`, `basalt/`, `tracy-profiler/`, `tts-voices/`, `nonsteam/`, etc.) — none of that was touched or moved.

## Ideas for later phases (don't block the engine decision)

- MCP server on top of the engine (console/rcon + asset pipeline) to iterate with AI.
- AI-generated content (textures/models/maps) — no constraints on the asset side.
- Battlefield-Vietnam-style / combined-arms multiplayer mode — evaluate after the core VR single-player is working.
