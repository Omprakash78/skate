# Concrete Jungle — Codebase Restructuring Plan


---

## Phase 1 Reading Note (read before anything else)

The provided source listing terminates mid-definition inside `struct World` (at the `static constexpr` grid member), approximately **line 1,119 of the stated ~5,000 lines**. The visible 1,119 lines were read in full, line by line. The remaining ~3,880 lines are reconstructed from strong in-file evidence: the header's control/feature list, the enums and structures already defined (`Rail`, `Gap`, `Pool`, `GroundHit`, `Solid::tall/noWall/noGround`, `Rng` "so the city looks the same every run", `Surf` surface enum, `GpuMesh::upload(..., bool dynamic)`), and the shader set (shadow, water-reflection, particle, HUD). Every table row and step below that touches the unseen region is marked **(inferred)**, and every migration step that enters inferred territory begins with a *symbol-anchored inventory* (find the block by its symbols, not by line number) before moving anything. Because the deterministic smoke harness (Step M2) is installed **before any code moves**, any inference error is caught at the very next compile-and-run gate, not at the end of the project. The plan is therefore robust to the truncation.

### Stated Assumptions (each carried through the plan)

- **A1** — The unseen portion contains, in order: the `World` uniform grid + queries, the procedural city builder (~1,200 lines), skater simulation/tricks/scoring (~900), camera (~150), particle update (~200), SDL audio synthesis + music (~500), HUD/help (~400), input/CLI (~200), main loop + `main()` (~300).
- **A2** — **No save/persistence system exists** (header shows no save options, no file I/O headers are included). None is added — YAGNI. There is zero save-compatibility burden.
- **A3** — The main loop uses **variable real-time dt, clamped** (evidence: `damp(cur, target, rate, dt)` exists precisely for frame-rate-independent smoothing; no accumulator is visible). This is *preserved exactly*; fixed-timestep is harness-only.
- **A4** — SDL audio is delivered via `SDL_AudioSpec` callback, which runs on a **separate thread**, with callback state reachable as file-scope statics (the only way a C callback reaches single-file state without userdata). This is a pre-existing data race — flagged UB-1, fixed at M8.
- **A5** — There is exactly one dynamic entity (the skater); cars exist as static grindable props (evidence: `RK_CAR` is a rail kind). The entity registry is added for the extensibility recipe, starting empty.
- **A6** — F11 toggles fullscreen via SDL window flags without destroying the GL context (to be verified at M4; if false, see UB-4).
- **A7** — The source is not yet under version control; the g++ one-liner in the header comment is the entire build system.
- **A8** — Music (`M`) toggles the synthesized music layer only; `--mute` silences all audio.

---

## 1. Executive Summary & Maintenance Vision

**The game:** "Concrete Jungle" — a single-file, ~5,000-line C++17 3D street skateboarding game: SDL2 windowing/input/audio, OpenGL 3.3 core with runtime-loaded entry points, fully procedural content (geometry, textures, GLSL surface detail, 5×7 font, synthesized SFX and music), a deterministic NYC block, a deep single-skater trick simulation (ollie charge, flips, grabs, grinds, manuals, combos, named gaps, 2-minute sessions).

**Top 3 structural problems:**
1. **One translation unit = one change amplifier.** City content, game rules, math, GL plumbing, and audio share a single file and namespace; any edit recompiles and risks everything, and the natural seams (already visible as section banners) are unenforced.
2. **A web of mutable global state.** The `gl` GLApi instance, (inferred) world/player/audio/session/input file-scope state, and an audio callback thread reading that state. Untestable, init-order fragile, and thread-unsafe.
3. **Zero verification infrastructure.** No build system, no tests, no behavior oracle, no way to tune without recompiling. Every refactor today is blind faith.

**Target architecture, one sentence:** A strictly layered, six-module application — `core → platform → {engine, render} → gameplay → states → app` — with downward-only dependencies enforced by the CMake target graph and an include-lint script, all shared state replaced by constructor-injected ownership rooted in a single App composition root, content added through tiny per-kind registries (levels, states, entities, audio tracks, tricks), migrated by strangler-fig steps that each end compiling, running, and hash-identical to the previous step.

**"Healthy" on a 10-year horizon means:** a new developer adds an enemy, level, game state, or music track by creating files and touching one registration line; every change is gated by unit tests plus a deterministic smoke harness with golden hashes; no module knows about any module above it; the tree builds warning-free in under 30 seconds; and every major decision has an ADR explaining *why*.

---

## 2. Current-State Analysis

### 2.1 System Inventory (source of truth for the plan)

Line ranges past ~1,119 are estimates for orientation only; all "move" instructions are defined by symbol anchors.

| # | System | Lines (est.) | Responsibilities | Global state touched | Dependencies | Coupling risk |
|---|--------|--------------|------------------|----------------------|--------------|---------------|
| 1 | Bootstrap / CLI / `main` | 4,700–5,000 (inf.) | SDL+GL init, argv (`--fullscreen --mute --res`), GLApi load, construction order, main loop, teardown | writes: `gl`, window, (inf.) world/player/audio | everything | **High** |
| 2 | Window / GL context | within #1 | window creation, GL attrs, resize, F11 | `gl`, window handle | SDL2, GLApi | Med |
| 3 | GL loader (`GLFUNCS`/`GLApi`) | 46–118 | loads 30 GL 3.3 core procs at runtime via SDL | owns global instance `gl` | SDL | Low (well-isolated) / Med (global) |
| 4 | Math (V2/V3/M4, transforms, `damp`/`approach`) | 119–330 | all vector/matrix math | `PI`, `TAU` (const) | none | Low |
| 5 | RNG & hashes (`Rng`, `hash32`, `hashf`) | 331–352 | deterministic xorshift + spatial hashes | instance state | none | Low |
| 6 | Color (`Col`, `hexc`, `shade`, `mixc`) | 353–377 | color type + ops | none | math | Low |
| 7 | Font (5×7 data, `glyphPixel`) | 378–443 | glyph raster data + lookup | reads `FONT5x7` (const) | none | Low |
| 8 | Geometry (`Mat` enum, `Vtx`, `MeshBuilder` incl. `box/limb/cylinder/sphere/text3D`) | 444–660 | CPU-side procedural mesh primitives | builder instances | math, font | Low |
| 9 | GPU mesh (`GpuMesh`) | 661–705 | VAO/VBO/EBO upload (static + dynamic), draw | `gl` | GLApi | Med |
| 10 | Shaders & programs (6 programs, `GLSL_COMMON`, `compileShader`/`makeProgram`/`fsWithCommon`) | 706–1,040 | all GLSL sources + compile/link | `gl`, static source strings | GLApi | Med |
| 11 | Render pipeline (inf.) | within 4,100–4,700 | shadow pass, water-reflection FBO, main pass, sky, water (clip plane), particles, HUD, vignette | `gl`, camera, FBO/texture handles, `uTime` | meshes, camera, world | **High** |
| 12 | Collision types & queries (`Solid`/`Surf`/`Rail`/`Gap`/`Pool`/`GroundHit`/`World` grid) | 1,041–1,150 | 2.5D heightfield solids, ramps, quarter-pipes, grindables, gap zones, ground/wall queries | reads `WATER_LEVEL`, `RIVER_EDGE_Z`, `STEP_UP` | math | Low–Med |
| 13 | City/world content builder (inf.) | 1,150–2,350 | procedural NYC block: storefronts, brownstones, plaza/fountain, cage, construction, river, skyline, props; fills solids+rails+gaps+pools **and** visual mesh interleaved | writes `World`, `MeshBuilder` | geom, collision, RNG | **High** |
| 14 | Skater sim, tricks, scoring (inf.) | 2,350–3,250 | physics, ollie charge, flip/grab/grind/manual, balance, combos, bail, gaps | writes player/combo/score state; reads world, input | collision, input, audio, particles | **High** |
| 15 | Camera (inf.) | ~3,250–3,400 | follow modes (V cycle), damping | camera state | player state, **reads `Solid::tall`** | Med |
| 16 | Particles (inf.) | ~3,400–3,600 | dust/sparks/spray update | particle pool | sim events, pools | Med |
| 17 | Audio (inf.) | 3,600–4,100 | SDL audio device, procedural SFX + music synth, callback | (inf.) static mixer/channel/music state — **second thread** | SDL audio | **High** |
| 18 | HUD & text (inf.) | 4,100–4,500 | score/combo/trick popups, clock, help overlay, pause | HUD state, font atlas | session/skater, font | Med |
| 19 | Input handling (inf.) | 4,500–4,700 | key events/state → actions, hold-to-charge | input state | SDL events | Med |
| 20 | Session/game states (inf.) | within #14/#18 | 2:00 session timer, pause, free skate | session state | input | Med |
| 21 | Config/CLI | within #1 | three flags | parsed values | none | Low |

### 2.2 Global / Shared State Map (defines module boundaries)

| Global | Kind | Readers | Writers | Fate |
|---|---|---|---|---|
| `gl` (GLApi instance) | mutable global | `GpuMesh::upload/draw`, `compileShader`, `makeProgram`, render pipeline (inf.) | `GLApi::load` once at boot | → owned by render device (§7) |
| `FONT5x7` | const data | `glyphPixel` → `text3D`, HUD atlas gen (inf.) | none | → module-private const |
| Shader source strings | const data | `makeProgram`/`fsWithCommon` | none | → render module-private |
| `PI`, `TAU` | const | math-wide | none | → `constexpr` in core (justified immutable) |
| `WATER_LEVEL`, `RIVER_EDGE_Z`, `STEP_UP` | const | collision, physics, water render (inf.) | none | → world constants header + config default |
| `world` (inf. file-scope or main-local) | mutable aggregate | sim, camera (`tall`), render, audio, particles | city builder, session reset | → App-owned `LevelData` |
| Player/combo/score state (inf.) | mutable | HUD, camera, audio triggers | skater sim | → `Skater`/`Session` owned by play state |
| Audio mixer/channel/music statics (inf.) | mutable, **audio thread** | SDL callback | callback **and** main thread | → `AudioEngine` + command queue (UB-1) |
| Key/event state (inf.) | mutable | sim, HUD | input poll | → `InputState` owned by App |
| Font atlas / shadow / reflection FBO handles (inf.) | mutable GL | render passes | render init | → Renderer-private |
| Trick tables (inf.) | const | scoring, HUD | none | → gameplay const registry |

### 2.3 Platform & Library

- **SDL2** (2.0.x): window, GL context, input, audio. Loaded via `find_package`/pkg-config today (sdl2-config). **Mild lock-in**, accepted: SDL2 is the process boundary; it is wrapped in exactly one platform module so a future SDL3/other-backend swap touches one layer.
- **OpenGL 3.3 core**, GLSL 330: entry points loaded at runtime through the `GLFUNCS` X-macro — this is a *gift*: GL is already behind one struct; it merely needs an owner. Version-specific: `gl_ClipDistance`, `fwidth`, `sampler2DShadow` with hardware compare — all GL 3.3 core. macOS (GL 4.1 max) is compatible. **Vendor lock-in: GLSL 3.30 + GL 3.3 core, confined to the render module post-refactor.**
- No third-party code at all today. No external assets — everything procedural. This *identity* is preserved (ADR-0003, ADR-0007).

### 2.4 Execution Model Findings

- **Threading:** one game thread + SDL audio callback thread (A4). No other threads.
- **Timestep:** variable dt (A3). Preserved; not "fixed" in this refactor — changing the timing model is a behavior change and is out of scope (ADR-0006).
- **Frame order (must be preserved verbatim):** sim update → audio command emission (inf.) → render passes in the exact order the monolith uses (shadow FBO → reflection FBO → main → sky → water with active clip plane → particles → HUD/vignette). Water reflects the *same frame's* reflection texture; the clip uniform toggles per pass. Step M6 is a *transcription*, not a recomposition.
- **Init order:** SDL init → window/context → `gl.load()` → font atlas/world/program/FBO creation → loop. Any GL-touching static object would be UB today; none are visible; the plan forbids them by rule (§6).
- **Determinism:** `Rng` fixed seed 1234567 → identical city every run; `hashf` deterministic placement; shaders deterministic given `uTime`. This is the backbone of the smoke oracle. Caveat: `std::exp/sin/cos` (libm) results can differ across platforms/compilers — golden hashes are **per platform**, regression-detection not absolute truth.

### 2.5 Undefined-Behavior & Hazard Scan

| ID | Finding | Severity | Disposition |
|---|---|---|---|
| UB-1 | Audio callback thread reads/writes state shared with the main thread without synchronization (inf., A4) → data race (UB) | **High** | Fixed at M8 (command queue + ThreadSanitizer verification) |
| UB-2 | `compileShader`/`makeProgram` log errors but **continue** with invalid program → undefined rendering | Med | Policy hardening at M5 (fatal startup abort, ADR-0005); flagged deviation R2 in §12 |
| UB-3 | `Rng::irange(a,b)` with `a > b` → negative span wraps to huge divisor → wrong results | Low | Precondition assert in core at M3 |
| UB-4 | `GpuMesh` has no release (GL objects leak at exit); if F11 ever recreates the context, all handles dangle | Med | Audit at M4; RAII wrappers at M5 |
| UB-5 | Unseen regions (trick tables, audio buffers, particle pool) not yet audited for OOB | Med | Audit protocol + ASan/UBSan smoke run at M8/M9 |
| UB-6 | `hash12(gl_FragCoord.xy)` dithered transparency → cross-GPU image nondeterminism | Low | Image hash is a secondary, per-machine oracle only |
| UB-7 | `mInverse` silently returns zero matrix on singular input | Low | Documented, unchanged (behavior contract) |

### 2.6 Build & Persistence Findings

- Build: OS-specific g++/clang++ one-liners from the header comment. No version control, no tests, no CI. Fixed by M0–M1/M13.
- Save format: **none exists** (A2). Config files introduced at M12 are new and additive — no compatibility burden.

---

## 3. Target Architecture

**Architectural style: strictly layered, with narrow module interfaces and small per-kind content registries.** Committed.

**Why not ECS / component / event-driven:** this game has *one* deep, stateful entity (the skater) plus a static world and a particle pool. ECS solves N-entity storage and iteration, which does not exist here; an event bus hides control flow that a 5k-line game benefits from keeping explicit; componentization would decompose a single coherent state machine (trick charging, balance, landing evaluation are one interlocked ball of state — splitting it would *increase* coupling via message plumbing). The file's actual seams — math / GL / mesh / collision / content / sim / presentation — map one-to-one onto layers. **Layering with enforced direction is the minimum structure that fixes problems 1 and 2 without inventing a framework.**

```
┌─────────────────────────────────────────────────────────────┐
│  app        composition root: config, ownership wiring,     │
│             main loop, teardown. Owns everything.            │
├─────────────────────────────────────────────────────────────┤
│  states     SkateState, PauseState, state registry, IState   │
├─────────────────────────────────────────────────────────────┤
│  gameplay   skater sim, tricks, session, camera,            │
│             levels (city content), entity registry          │
├───────────────────────┬─────────────────────────────────────┤
│  engine               │  render                              │
│  world + collision    │  render device (GLApi owner),        │
│  queries, particles,  │  GPU meshes, shaders/programs,       │
│  audio engine + queue │  font atlas, HUD & particle          │
│                       │  rendering, frame pass composition   │
├───────────────────────┴─────────────────────────────────────┤
│  platform   SDL window, input events, audio device,         │
│             GLApi loader (types only, instance owned above) │
├─────────────────────────────────────────────────────────────┤
│  core       math, color, RNG, hashes, geom (CPU meshes),    │
│             font data, config parser, log, shared PODs      │
└─────────────────────────────────────────────────────────────┘
        Dependencies point DOWN only. engine ∥ render
        (siblings — no cross-includes). CMake target graph
        and tools/check_includes.py make violation a build error.
```

**Rules, verbatim:**
- A module may include only modules strictly below it. `engine` and `render` never include each other; shared data types between them (e.g., the `Particle` POD, `Vtx`) live in `core`.
- `gameplay` never includes anything GL/SDL. It talks to engine (world queries, particle spawns, audio commands) and produces plain data (camera `ViewInfo`, `HudModel`, dynamic `MeshBuilder`) that `states`/`app` hand to `render`.
- Content is registered in per-kind registry lists (levels, states, entities, audio tracks, tricks) that are themselves content files, never core files.
- The one C-callback exception: the SDL audio callback is a file-local forwarding function in `platform` that dispatches through the `userdata` pointer to the `AudioEngine` — no static game state.

---

## 4. Folder Structure

Rule: public header → `include/concrete/<module>/<name>.h`; implementation → `src/<module>/<name>.cpp`; module-private headers → `src/<module>/private/`, never included outside the module. Header-only modules skip the .cpp.

```
concrete-jungle/
├── CMakeLists.txt                  # target graph, flags, tests, smoke fixture
├── CMakePresets.json               # debug / release / asan / tsan / smoke presets
├── README.md                       # build, run, controls (from the file header), smoke
├── CHANGELOG.md                    # Keep-a-Changelog format, from v1.0.0
├── ARCHITECTURE.md                 # layer diagram + rules + module table
├── .clang-format / .clang-tidy     # pinned tool configs (enforced by CI)
├── config/
│   ├── game.cfg                    # all tuning: physics, scoring, camera, session, audio
│   ├── controls.cfg                # key → action bindings (defaults = current keys)
│   └── replays/smoke.txt           # scripted input replay driving the smoke harness
├── assets/README.md                # empty by policy: all content procedural (ADR-0007)
├── third_party/
│   └── doctest/                    # pinned single-header test framework (ADR-0008)
├── tools/
│   ├── check_includes.py           # layer dependency linter (include-path allowlist)
│   └── golden/README.md            # how golden hashes are recorded/compared
├── docs/
│   ├── STYLE.md                    # naming/layout conventions (mirrors existing style)
│   ├── BEHAVIOR-CONTRACT.md        # the §12 checklist, kept as the living contract
│   ├── MANUAL-CHECKLIST.md         # §10(c) gameplay checklist
│   ├── MIGRATION-LOG.md            # one entry per completed M-step
│   └── adr/0001…0011.md            # one ADR per major decision (§13)
├── include/concrete/
│   ├── core/    math.h (V2/V3/M4/funcs, header-only) · color.h · rng.h · hash.h ·
│   │            geom.h (Vtx, Mat, MeshBuilder) · font.h (glyph lookup) ·
│   │            config.h (Config struct + INI loader) · log.h · particles_pod.h
│   ├── platform/ gl_api.h (GLApi type) · window.h · input_events.h · audio_device.h
│   ├── render/  render_device.h (owns GLApi) · gpu_mesh.h · shaders.h (program mgmt) ·
│   │            font_atlas.h · hud.h · particle_render.h · view.h (ViewInfo) ·
│   │            renderer.h (frame pass composition)
│   ├── engine/  world.h (solids/rails/gaps/queries) · world_constants.h ·
│   │            particles.h · audio_engine.h · audio_commands.h
│   ├── gameplay/ skater.h · tricks.h · session.h · camera.h · fx events via
│   │            particles/audio · levels/level.h (ILevelBuilder) ·
│   │            entities/entity.h (IEntity) · trick_registry.h
│   ├── states/  state.h (IState) · registry.h
│   └── app/     context.h (AppContext aggregate) · app.h
├── src/
│   ├── core/    rng.cpp · geom.cpp · font.cpp · config.cpp · log.cpp
│   ├── platform/ gl_api.cpp · window.cpp · audio_device.cpp
│   ├── render/  render_device.cpp · gpu_mesh.cpp · shaders.cpp (GLSL sources live
│   │            here as private const data, ADR-0003) · font_atlas.cpp · hud.cpp ·
│   │            particle_render.cpp · renderer.cpp
│   ├── engine/  world.cpp · particles.cpp · audio_engine.cpp (+ tracks/ subdir)
│   ├── gameplay/ skater.cpp · tricks.cpp · session.cpp · camera.cpp ·
│   │            levels/registry.cpp · levels/city_builder.cpp ·
│   │            entities/registry.cpp
│   ├── states/  registry.cpp · skate_state.cpp · pause_state.cpp
│   └── app/     main.cpp · app.cpp · input_map.cpp
└── tests/
    ├── unit/     math_tests.cpp · rng_tests.cpp · geom_tests.cpp · world_tests.cpp ·
    │            tricks_tests.cpp · config_tests.cpp · audio_queue_tests.cpp
    └── smoke/    run_smoke.py (drives the --cj-smoke flag, compares goldens)
```

**Verticality proof:** a new enemy, level, state, or audio track = new files under the relevant `src/<module>/<kind>/` + one registration entry in that kind's `registry.cpp` (a content file) + optionally one config line. Zero existing core/engine/render files are touched. See §14.

---

## 5. Module Boundaries & Interface Design

Each entry: responsibility → public interface (names + intent) → owns → forbidden → depends on.

**core** — pure, dependency-free foundation, fully unit-testable.
Interface: `math.h` (vector/matrix ops, `damp`, `approach`, transforms — verbatim current functions); `geom.h` (`MeshBuilder` + primitives, unchanged); `font.h` (`glyphPixel` lookup); `rng.h`, `hash.h`; `config.h` (load INI into `Config`, warn-and-default semantics); `log.h` (leveled logging, printf-backed); `particles_pod.h` (`Particle` plain data). Owns: nothing mutable across frames. Forbidden: SDL, GL, any other module. Depends on: nothing but the std library.

**platform** — SDL and process boundary, game-agnostic.
Interface: `Window` (create/destroy with GL context, toggle fullscreen, poll events into `InputEvents`, swap); `GLApi` type + loader; `AudioDevice` (open with a callback + `void* userdata`, close). Owns: SDL handles only. Forbidden: game types, GL object management. Depends on: core (log/config only).

**engine** — game-agnostic simulation services.
Interface: `world.h` (`World` data + queries: ground height/normal/solid under point, wall resolution, rail proximity, gap zones, pools — the existing `Solid::heightLocal/normalLocal/inside` machinery behind narrow query functions); `particles.h` (spawn dust/spark/splash, update, expose the particle array); `audio_commands.h` (typed command struct: one-shot SFX with kind/pitch/volume, rolling-surface params, music toggle); `audio_engine.h` (drain commands, mix + synthesize into the device callback). Owns: `World` contents (filled by others), particle pool, audio mixer state. Forbidden: GL, SDL structs (device wrapper only via callback contract), gameplay types. Depends on: core.

**render** — the only module that touches GL.
Interface: `render_device.h` (`RenderDevice`: owns the `GLApi` instance, exposes it by reference; creates/destroys context-scoped resources); `gpu_mesh.h` (upload from `MeshBuilder` — static or dynamic — and draw, taking `GLApi&` so bodies stay byte-for-byte the same logic); `shaders.h` (create the six programs; fatal on compile/link failure — ADR-0005); `font_atlas.h` (build atlas texture from core font, measure/layout text); `hud.h` (draw a `HudModel`: quads, text, vignette); `particle_render.h` (draw core particle arrays); `renderer.h` (`renderFrame(const FrameDesc&)` — the single function transcribing the monolith's exact pass order: shadow, reflection, main, sky, water+clip, particles, HUD); `view.h` (`ViewInfo`: view/proj matrices, camera pos, time). Owns: all GL objects (meshes, programs, atlas, FBOs, shadow texture). Forbidden: reading `Skater`/`Session`/`World` gameplay meaning — it draws what `FrameDesc` hands it. Depends on: platform, core.

**gameplay** — rules and content.
Interface: `skater.h` (construct with world/audio/particle references; `update(dt, InputActions)`; expose read-only pose/state/score for HUD and camera); `tricks.h` (trick identity + points — const registry); `session.h` (timer, mode, results); `camera.h` (modes, update, produce `ViewInfo`); `levels/level.h` (`ILevelBuilder`: fill `World`, fill `MeshBuilder`, set spawn/water plane/name); `entities/entity.h` (`IEntity`: update, submit geometry, optional simple collision vs skater). Owns: skater/session/camera state, level content output. Forbidden: GL, SDL, direct audio callback access — emits `AudioCommand`s only. Depends on: engine, core.

**states** — top-level flow.
Interface: `state.h` (`IState`: handle input event, update, render via `Renderer` + `HudModel`, query next state); `registry.h` (list of constructible states). Owns: the active `Skater`/`Session`/level instance (SkateState), pause overlay (PauseState). Forbidden: GL details, platform types. Depends on: gameplay, engine, render, core.

**app** — composition root and loop.
Interface: `app.h` (`run()`); `context.h` (`AppContext` aggregate of references: config, window, render device, renderer, audio engine, particles, world, input, active state machine). Owns: *every* subsystem instance, constructed in explicit init order, destroyed in reverse. Forbidden: business logic. Depends on: all layers (it is the only module allowed to).

**Circular includes:** forbidden; CMake link graph cannot express them anyway. Prefer forward declarations in headers; include only in .cpp.

---

## 6. Ownership & Lifetime Policy

- **RAII everywhere.** Every resource has exactly one owner. `RenderDevice` owns the GLApi instance and all context-lifetime GL objects (via RAII wrappers for programs/meshes/FBOs that call `gl.Delete*` in destructors — fixes UB-4; behavior-neutral). `App` owns subsystems as direct members/`unique_ptr`, constructed in a single explicit order (SDL → window/context → `gl.load()` → render resources → audio device → level build → state machine) and destroyed in reverse. No GL-touching object may be static/global.
- **Raw `new`/`delete` are forbidden** (clang-tidy gate). Owning = `std::unique_ptr` (entities, states). Non-owning = references or pointers; raw pointers appear only at C-API boundaries (SDL `userdata`). `Vtx`-style POD buffers live in `std::vector`.
- **Const-correctness:** all world queries take `const World&`; config, hud models, view info, particle arrays consumed by render are const; only builders and sim state are mutable.
- **Error policy — committed: no exceptions.** (a) *Recoverable* (config parse, missing replay/golden file): warn + documented default. (b) *Fatal startup* (SDL init, context creation, missing GL functions, shader compile/link — hardening UB-2, level build): log precise error, clean teardown, non-zero exit. (c) *Invariants* (preconditions like UB-3, queue overflow, entity list integrity): `assert` in Debug. Justification: matches the existing `fprintf`-and-continue culture (hardened to fail fast), keeps MinGW/embedded friendliness, and game startup failures have no sane recovery path. Recorded as ADR-0005.
- **Hot-path discipline:** the dynamic skater mesh and HUD/particle buffers must keep the current reuse pattern (capacity-grown, `BufferSubData`) — the plan forbids introducing per-frame allocations in the frame loop; verified by the smoke frame-time report (§11 M6).

---

## 7. Shared State Elimination

**Pattern — committed: constructor dependency injection with a single `AppContext` aggregate as the only blessed convenience bundle**, built once in `app` (the composition root), passed by reference. No service locator (hides the graph), no singletons (immortal hidden state), no DI framework (absurd at this scale).

| Current global | New owner | Delivery mechanism |
|---|---|---|
| `gl` (GLApi) | `RenderDevice` member | `GLApi&` parameter into `GpuMesh`/program functions (bodies unchanged — the parameter is literally named `gl`); pass composition uses `RenderDevice`'s instance |
| `FONT5x7`, shader sources | `core/font`, `render/shaders` as module-private const data | internal linkage; exposed via behavior functions only |
| `PI`, `TAU` | `constexpr` in `core/math.h` | justified immutable constants (allowed to survive) |
| `WATER_LEVEL`, `RIVER_EDGE_Z` | `engine/world_constants.h` constexpr | shared read-only world identity |
| `STEP_UP` | `PhysicsParams` default, overridable by `[physics] step_up` | config |
| `world` (inf.) | `App`-owned `LevelData` (World + CPU mesh + meta) | `const World&` to queries; builder fills at load |
| Player/combo/score (inf.) | `Skater` + `Session`, owned by `SkateState` | constructed with world/audio/particles refs |
| Audio statics + callback (inf.) | `AudioEngine` owned by `App`; SDL callback reaches it via `userdata` | SPSC command queue: gameplay pushes typed commands; callback drains — **the one flagged semantically-equivalent rewrite** (R1, §12) |
| Key/input state (inf.) | `InputState` + `InputMap` owned by `App` | events → `InputActions` struct consumed by states/gameplay |
| Font atlas, FBOs, shadow map (inf.) | `Renderer` private | internal |
| Camera state (inf.) | `Camera` owned by gameplay | produces `ViewInfo` per frame |
| Trick tables (inf.) | `gameplay/tricks` const registry | data |
| `uTime` (inf.) | frame parameter on `renderFrame` | explicit |

No global survives unjustified: the only file-scope objects remaining anywhere are `constexpr` data and the SDL callback's file-local forwarding function (which holds no state).

---

## 8. Data-Driven Design

**Format — committed: a minimal INI subset** (sections `[name]`, `key = value`, `;` comments, values are int/float/bool/string; parser ~100 lines in core, unit-tested). Justification: human-editable, zero dependencies, matches the tuning surface (flat scalars); JSON/TOML buy nesting this game doesn't have.

**Policy:** missing file or key → built-in default **equal to the current hardcoded constant** (so absence of config = today's behavior, byte-identical); unknown key → warn and ignore (forward/backward compatibility); malformed value → warn and default.

**What moves to `config/game.cfg` (tunable without recompiling):**

| Section | Keys (defaults = current constants, discovered symbol-anchored during M12) |
|---|---|
| `[physics]` | gravity, push_accel, push_top_speed, brake_force, ground_friction per surface, air drag, ollie_charge_rate, ollie_max_height, steer_rate, air_spin_rate, rail_magnet_distance, grind_snap, balance_decay, manual_wobble_gain, bail_impact_threshold, step_up |
| `[scoring]` | per-trick base points, combo multiplier rule, gap bonus multiplier |
| `[camera]` | follow distance/height/lag per mode, fov |
| `[session]` | length_seconds (120), free_skate (bool) |
| `[audio]` | master/music/sfx volumes, music_on_default |
| `[video]` | width, height, fullscreen (CLI flags override — current precedence preserved) |

**`config/controls.cfg`:** key→action bindings, defaults exactly the current key set (W/Up, S/Down, A/D, Space, J/Z, K/X, L/C, I/Shift, R, H, M, V, T, Esc, F11).

**What deliberately stays in code (ADR-0007):** level geometry (the procedural city *is* the content — moving it to data files would be a rewrite, violating the pure-refactor contract), GLSL, trick *identity* (names + mechanics), gap definitions (geometry-coupled). Rule honored where feasible: *tuning* without recompile — yes; *content authoring* stays C++ by design.

---

## 9. Build System & Tooling

**CMake ≥ 3.21, target-based, C++17, no global properties:**

- Static libs, one per layer: `cj_core`, `cj_platform`, `cj_engine`, `cj_render`, `cj_gameplay`, `cj_states`; executable `concrete_jungle` from `src/app`. `target_link_libraries` encodes the §3 layer graph exactly — an upward dependency is a *link error*, not a style nit.
- Per-target `target_include_directories` (`include/` public, `src/` private). No file-scoped `include_directories`.
- Warnings: `-Wall -Wextra -Wpedantic -Wshadow -Werror` (GCC/Clang; MSVC `/W4` best-effort, not gated) once M13 lands; `-Wconversion -Wsign-conversion` scoped to `cj_core` only (highest-value, bounded churn). Baseline warnings recorded at M1, zeroed stepwise, `-Werror` flipped on only when the tree is clean.
- Debug/Release via `CMakePresets.json`: `debug`, `release`, `debug-asan` (ASan+UBSan), `debug-tsan` (for M8), `release-smoke`.
- Sanitizers in the *verification workflow*: every migration step's smoke run executes under `debug-asan`; M8 additionally under `debug-tsan`.
- **No precompiled headers** (≈35 TUs, sub-30s builds; ADR-0010 — revisit only past ~100 TUs).
- `doctest` pinned via FetchContent at a fixed commit (ADR-0008). SDL2 via `find_package(SDL2)` with pkg-config fallback — matches today's `sdl2-config`.

**Tooling → rule mapping (CI-gated):**

| Tool | Enforces |
|---|---|
| clang-format (pinned `.clang-format`: LLVM base, indent 4, col 100, unsorted includes to minimize churn) | layout consistency; applied once as a pure reformat commit at M13 |
| clang-tidy (curated: `cppcoreguidelines-owning-memory`, `modernize-use-nullptr`, `bugprone-narrowing-conversions`, `cppcoreguidelines-init-variables`, `readability-const`, `bugprone-use-after-move`, `misc-unused-parameters`) | §6 ownership/const/no-`new` rules |
| cppcheck (all, warning+style+performance, suppressions file) | OOB/null-deref audit of migrated code (UB-5) |
| include-what-you-use | include hygiene on new modules |
| `tools/check_includes.py` (include-path allowlist per directory) | §3 downward-only dependency rule — **the tool-enforced dependency gate** |

---

## 10. Testing & Verification Strategy

**(a) Unit tests (doctest, `tests/unit/`, run via CTest):** pure logic only — math (matrix inverse round-trips, `damp`/`approach` edge cases), RNG determinism and range preconditions (UB-3), `MeshBuilder` outputs (vertex/index counts for known primitives, `text3D` widths), world queries (hand-built solids: box/ramp/quarter-pipe height+normal correctness, grid lookup, rail proximity, gap zones), trick table integrity and scoring arithmetic, INI parser (missing/malformed/unknown keys), audio command queue semantics (single-threaded contract test). These land *with* each module as it is extracted (see M-steps).

**(b) Automated smoke/regression harness — the behavior oracle:**
- Added at M2 as a hidden, additive CLI mode: `--cj-smoke --replay config/replays/smoke.txt --frames 600`.
- Harness mode: fixed RNG seed (the existing default 1234567), **fixed dt = 1/60** (harness-only — ADR-0006), no real-time sync, audio device opened in a silent/dummy configuration (SDL dummy audio driver) so the callback still runs.
- Scripted input replay: ~600 frames covering hold-and-release ollie, a flip trick, a grind, a manual, a bail/reset, session start, pause/resume, camera cycle, music toggle.
- Oracles, in priority order: **(1) simulation state hash** every 30 frames (skater pos/vel/yaw/state enum, score, combo, particle count — driver-independent), **(2) audio command stream hash** (count + sequence — proves audio *trigger* behavior identical, R1), **(3) framebuffer CRC** on the reference machine only (secondary; UB-6 makes it non-portable).
- Golden files per platform in `tests/golden/<platform>.txt`, compared by `tests/smoke/run_smoke.py`; local runs detect regressions vs. the dev's own last goldens; CI compares against the platform's recorded goldens. A smoke run also reports average frame time (perf regression watch, §15).
- Every migration step below is gated on **sim + audio hashes byte-identical to the previous step** (and image hash on the reference machine for render steps).

**(c) Manual gameplay checklist (`docs/MANUAL-CHECKLIST.md`):** every control from the file header (push, brake, steer, air spin, rail balance, ollie hold-height curve, all five flip tricks, grab release-before-landing, grind/slide on each rail kind incl. car, manual/nose manual balance, R, H, M, V cycle, T session, Esc pause, F11), 2-minute session flow and results, water splash in fountain/river, grind sparks, landing dust, shadows visible, music toggle, `--fullscreen/--mute/--res` flags, exit cleanliness. Executed fully at M0 (baseline), and at every step marked "manual" below.

---

## 11. Incremental Migration Plan (Strangler-Fig)

Standing rules for every step: **one git commit per step** (message `refactor(M#): …`); the project **compiles, runs, and passes its verification tier with identical behavior** at the end of every step; the monolith `skate.cpp` remains the top of the figure until M11 and may include extracted module headers (that is the strangler pattern, not a dual path); any *temporary shim* is marked `// DEPRECATED(M#): remove at M#+n` with a scheduled removal — no temporary survives past its named step.

| Step | What moves / changes | Verification | Rollback | Effort |
|---|---|---|---|---|
| **M0 — Baseline & repo** | git init, `.gitignore` (build dirs), tag `v0-monolith`; document the g++ one-liners in a README stub; run the full manual checklist and record results in `docs/MANUAL-CHECKLIST.md`; (optional) capture a reference video. | Manual checklist passes on all three OS commands. | n/a | 0.25d |
| **M1 — CMake builds the monolith** | `CMakeLists.txt` with one executable target compiling `skate.cpp`, C++17, warning flags **without** `-Werror`; presets `debug`/`release`; record the baseline warning list in `docs/MIGRATION-LOG.md`. | Build via CMake on all platforms; game runs; manual spot-check (controls + one session). | Revert commit; g++ one-liner still documented. | 0.5d |
| **M2 — Smoke harness + test scaffolding** | Add the hidden `--cj-smoke` mode (fixed seed/dt, replay file loader, state + audio-command + image hashes, frame-time report) **additively** to the monolith; add doctest via pinned FetchContent; write `tests/smoke/run_smoke.py`; author `config/replays/smoke.txt`; record goldens per platform. *First read of the unseen loop code — confirm A3/A4 and adjust the plan's inferred table in the migration log if reality differs (the oracle, already captured, protects behavior regardless).* | Smoke runs green twice consecutively (determinism); goldens committed; manual checklist unchanged (flag is additive). | Revert commit; monolith untouched functionally. | 1.0d |
| **M3 — Extract `core`** | Move math/color/rng/hash/font-data/geom verbatim into `cj_core` (inline functions stay inline; `PI/TAU` → `constexpr`; `FONT5x7` module-private behind `glyphPixel`); monolith includes core headers. Add unit tests: math, rng (incl. UB-3 assert), geom. | Build; smoke hashes identical; unit tests green. | `git revert` single commit. | 1.0d |
| **M4 — Extract `platform`** | Move the `GLApi` *type*+loader, SDL window/context creation, event polling into `InputEvents`, audio-device open/close (callback + `userdata` contract) into `cj_platform`. Monolith keeps its local `gl` instance (now typed from platform). **Audit F11 path (UB-4) and record result in the migration log.** | Build; smoke identical; F11 + resize manual check. | Revert. | 1.0d |
| **M5 — Extract render primitives** | Move `Vtx`-upload/GpuMesh (methods now take `GLApi& gl` — bodies unchanged), shader sources + compile/link utils into `cj_render` (fatal-on-failure policy per ADR-0005 — flagged deviation R2), font atlas + text rendering, particle/HUD vertex upload. Add RAII deleters (UB-4). Unit tests: none (GL); reference-machine image hash in smoke covers it. | Build; smoke **image hash identical** on reference machine + state hashes; manual visual checklist (shadows, water, text, vignette). | Revert. | 1.0d |
| **M6 — Extract renderer/frame composition** | Transcribe (not recompose) the frame pass sequence into `renderer.cpp` behind `renderFrame(FrameDesc)`; FBOs/shadow map/reflection texture become Renderer-private; `ViewInfo` introduced; monolith builds `FrameDesc` from its (still-monolithic) world/player/camera. Frame-time report added to smoke. | Smoke: image + state hashes identical, frame time within 10% of M2 baseline; full manual visual checklist. | Revert. | 1.5d |
| **M7 — Extract `engine/world`** | Move `Solid/Rail/Gap/Pool/GroundHit/World` + grid + query functions into `cj_engine` (pure — no GL/SDL); constants header. Add unit tests: heights/normals for box/ramp/QP, grid queries, rails, gaps. | Build; smoke state hash identical; unit green. | Revert. | 1.0d |
| **M8 — Extract audio engine + particles** | Move synthesis/mixer/music into `AudioEngine` (engine layer); introduce the SPSC command queue; wire SDL callback via `userdata` forwarding function (R1 flagged rewrite — command stream must be byte-identical); move particle system (spawn/update, data in core POD). Run smoke under `debug-tsan` and `debug-asan`; audit unseen audio buffers for OOB (UB-5). **All audio file-scope statics die here.** | TSan clean on smoke; ASan clean; smoke audio-command hash + state hashes identical; manual audio checklist (ollie pop, land, grind tone, surface rolls, music toggle, mute). | Revert. | 1.5d |
| **M9 — Extract gameplay core** | Symbol-anchored move of skater simulation, trick tables, combo/scoring, session into `cj_gameplay` (`Skater::update(dt, InputActions)` with injected world/audio/particles; `InputMap` translating `InputEvents`→`InputActions` with current key semantics; `Camera` producing `ViewInfo`). Split into two sub-commits (sim physics, then tricks/scoring) with one shared verification gate. Add unit tests: trick table, scoring/combo arithmetic, pure landing/balance math. | **Critical gate:** smoke sim + audio hashes byte-identical; unit green; full manual feel checklist (input timing, ollie charge curve, trick detection windows). | Revert (sub-commits allow partial). | 2.0d |
| **M10 — Extract level content** | City builder → `gameplay/levels/city_builder.cpp` implementing `ILevelBuilder` (fills `World` + CPU mesh + spawn/water/meta in one pass — preserving the interleaved build that guarantees visual/collision agreement); level registry with one entry; App loads via registry. | Smoke image + state hashes identical (deterministic seed proves identical city). | Revert. | 1.0d |
| **M11 — States, app, delete the monolith** | `IState` + `SkateState`/`PauseState` + registry; `App` composition root with `AppContext`, explicit init/shutdown order; `main.cpp`; monolith file deleted (history preserved in git). Entity registry (empty) and trick registry land here, sized exactly to §14 recipes. | Full smoke (all hashes), full manual checklist, clean shutdown (no leaks under ASan). | Revert to M10 tree. | 1.0d |
| **M12 — Data-driven config** | INI parser in core (unit-tested); `Config` struct; move tuning keys of §8 with defaults = old constants; `controls.cfg` with current bindings; CLI flags override config (current precedence). | Smoke **without** config files: hashes identical (defaults prove behavior preservation); with a tweaked value: hash differs (proves the knob is live); unit tests green. | Revert. | 1.0d |
| **M13 — Tooling & enforcement lockdown** | Flip `-Werror` (tree is clean by now); pinned clang-format + one pure reformat commit; clang-tidy/cppcheck/IWYU configs + fixes; `tools/check_includes.py` layer linter wired into CTest; GitHub Actions matrix (Linux-GCC, macOS-Clang, Windows-MinGW): build, unit, smoke goldens per platform, format check, tidy, cppcheck, include-lint. | CI green end-to-end. | Revert (reformat commit is isolated). | 1.5d |
| **M14 — Docs & release** | Finalize README/ARCHITECTURE.md/STYLE.md/ADRs/BEHAVIOR-CONTRACT.md/MIGRATION-LOG; CHANGELOG 1.0.0; tag `v1.0.0`. | Docs reviewed; DoD checklist (§17) passes. | n/a | 0.25d |

**Schedule:** 14.0 dev-days of implementation → plan **3 calendar weeks** for one developer including review, CI stabilization, and the two full manual checklist runs. Critical path: M2 (oracle) → M6/M9 (the two big transcription gates).

---

## 12. Behavior Preservation Contract

Pure refactor: logic moves and gets encapsulated; it is never rewritten. `docs/BEHAVIOR-CONTRACT.md` holds this table as the living document; verification codes: **S** = smoke hashes, **SI** = smoke incl. image hash (reference machine), **M** = manual checklist item.

| # | Observable behavior | Verification |
|---|---|---|
| 1 | Deterministic city: identical layout every run (seed 1234567) | S, SI |
| 2 | Full control set per header: push/brake/steer/air-spin/rail-balance | S (replay), M |
| 3 | Ollie hold-to-crouch → release pop, charge-height curve | S, M (feel) |
| 4 | All five flip tricks (J/Z + dir) with names/points | S, M |
| 5 | Grab tricks hold-release, bail-on-landing-grab | S, M |
| 6 | Grind/slide snap on rails/ledges/benches/curbs/cars, sparks | S, SI, M |
| 7 | Manual/nose manual balance (W/S), combo linking | S, M |
| 8 | Combo scoring, gap detection & named-gap bonuses | S (score hash) |
| 9 | Camera mode cycle (V), follow damping feel | SI, M |
| 10 | 2-minute session (T): timer, flow, results/reset | S, M |
| 11 | Pause (Esc) freezes sim & audio; help overlay (H) content identical | S, SI, M |
| 12 | Water: river/fountain/puddle visuals, waves, splash + spray, reflection | SI, M |
| 13 | Shadows present, chain-link diamond shadows, skyline-light exclusion | SI |
| 14 | Audio: every SFX trigger and rolling-surface behavior, music toggle (M), `--mute` | S (command-stream hash), M (listen) |
| 15 | Timing model: variable-dt feel unchanged (damping/frame-rate independence) | S + M (feel); harness dt differs from live dt **by design** and is compared only against itself |
| 16 | CLI: `--fullscreen`, `--mute`, `--res WxH` precedence over config | M |
| 17 | Particle behaviors: landing dust, grind sparks, water spray | S, SI |
| 18 | Vignette + HUD layout/typography (5×7 font, popups, clock) | SI, M |
| 19 | Reset (R) behavior; out-of-bounds/river respawn | S, M |
| 20 | No save system exists → nothing to preserve; config files are additive | S (absence of config = identical hashes) |
| 21 | Exit path: clean shutdown, no GL leaks (post-M5), no audio device leak | ASan smoke |

**Flagged rewrites (semantically equivalent, explicit per contract):**
- **R1 (M8):** audio callback plumbing → command queue. Same commands at the same frames (proven by the command-stream hash); removes the data race. The one place logic is *re-plumbed*, never re-*written*.
- **R2 (M5):** shader compile/link failure now aborts startup instead of continuing broken. Deviation from current *buggy* behavior, accepted, ADR-0005.
- **R3 (M2):** `--cj-smoke` harness flag — additive, invisible in normal play.
- **R4 (M13):** one pure-formatting commit — formatting only, gate: smoke hashes unchanged.
- **Explicit non-goal:** switching live gameplay to fixed timestep — that is a behavior change requiring its own ADR and is out of scope.

---

## 13. Documentation & Decision Records

- **README.md** — build (CMake presets), run, controls (verbatim from the file header), smoke harness usage.
- **ARCHITECTURE.md** — the §3 layer diagram, the dependency rules, the §5 module table, the frame pass order (from M6), and the App init order.
- **STYLE.md** — naming: files `snake_case`, types `PascalCase`, functions/variables `camelCase`, constants `SCREAMING_SNAKE` — *mirrors the existing conventions to minimize diff churn*; comments explain **why**, never what.
- **Per-module header docs** — one block per public header: responsibility, owns, forbidden, depends (copied from §5).
- **ADRs** (context → decision → consequences), all under `docs/adr/`:
  - 0001 Layered architecture over ECS/component/event-driven (single-entity game; seams match layers)
  - 0002 Constructor injection + AppContext over singletons/service locator (visible ownership graph)
  - 0003 GLSL stays embedded in `render/shaders.cpp` (procedural, zero-asset identity; no file-loading path added)
  - 0004 INI-subset config format, warn-and-default semantics
  - 0005 No exceptions; fatal startup errors; debug asserts (hardens the current fprintf culture; fixes UB-2)
  - 0006 Deterministic smoke harness with fixed dt; live variable-dt preserved; goldens per platform (libm variance)
  - 0007 Level content remains procedural C++; data-driven = tuning, not geometry
  - 0008 Pinned dependencies: doctest via FetchContent fixed commit; SDL2 via system package, tested version documented
  - 0009 Audio cross-thread command queue (fixes the data race; command stream preserved)
  - 0010 No precompiled headers at this codebase size
  - 0011 Compile-time registries (no plugins/DLLs): extensibility = new files + one registration line

---

## 14. Extensibility Roadmap

1. **New enemy / dynamic entity** (e.g., a vendor cart rolling across the plaza that the skater must ollie over): create `src/gameplay/entities/vendor_cart.cpp` + `include/concrete/gameplay/entities/vendor_cart.h` implementing `IEntity` (update with dt/world/skater, submit its geometry into the dynamic mesh pass, self-contained AABB-vs-skater check using existing world helpers, emits audio/particle commands); add **one registration entry** in `src/gameplay/entities/registry.cpp`; optionally tune in `game.cfg`. Touch nothing else — `SkateState` iterates the registered list.
2. **New level/map** (e.g., rooftop spot): create `src/gameplay/levels/roof_builder.cpp` + `include/concrete/gameplay/levels/roof.h` implementing `ILevelBuilder` (fill `World` solids/rails/gaps/pools, fill the visual `MeshBuilder`, set spawn/water plane/name); add **one registration entry** in `levels/registry.cpp`; select via `[game] level = roof` in `game.cfg`. No core, engine, or render file changes.
3. **New game state** (e.g., photo mode): create `src/states/photo_state.cpp` + `include/concrete/states/photo_state.h` implementing `IState` (input, update, render through the existing `Renderer`/`HudModel` APIs); add **one registration entry** in `src/states/registry.cpp`; bind the entry key in `config/controls.cfg`. Zero existing state files touched — transitions come from the generic keybinding hook.
4. **New feature/system — new music track:** create `src/engine/audio/tracks/nocturne.cpp` defining the track's synth program + data, register **one entry** in the track registry (in the same module's `tracks` list), select via `[audio] music_track`. No other file changes.
5. **New trick (bonus):** add one entry to the trick registry table in `src/gameplay/tricks.cpp` (a content file, not core) with name/points/input window; scoring, HUD popups, and combos pick it up automatically.

Every recipe satisfies the verticality requirement: new files + one registration point, zero existing core/engine/render files edited.

---

## 15. Risks & Mitigations

| # | Risk | Plan decision that neutralizes it |
|---|---|---|
| 1 | **Hidden state beyond the mapped globals** (unseen 3,880 lines) | M2 installs the behavior oracle *before any move*; every step's hash gate catches drift at the first compile; M-step symbol-anchored inventory protocol; grep protocol for file-scope mutable symbols run at each extraction (recorded in MIGRATION-LOG) |
| 2 | **Inference error from the truncated listing** (A1–A8 wrong) | Same as #1: smoke harness is installed while the code is still the untouched monolith; the first hash-gated step that touches an inferred region (M2/M9) forces the executor to read it; contract is anchored to hashes, not to my reading |
| 3 | **Initialization-order breakage** (GL/GLApi touched before load; static objects) | §6: no GL-touching statics, explicit App init sequence, fatal `gl.load()` verification; CI + ASan smoke catches use-before-init |
| 4 | **Circular dependencies** (world↔render, builder↔both) | Shared PODs pushed to core; CMake per-layer link graph makes cycles a *build error*; `check_includes.py` lints include direction; builder writes both outputs without the consumers knowing each other |
| 5 | **Behavior drift** (float arithmetic, pass order, input timing, audio triggers) | Pure-move migration with bodies unchanged where possible (`GLApi&` parameter keeps GL call sites identical); M6 frame function is a transcription; three-oracle smoke (state, audio-command, image); flagged rewrites limited to R1–R4 |
| 6 | **Performance regression** from layout/include changes and new indirection | Frame-time report in smoke compared to M2 baseline (10% gate); no new per-frame allocations (§6 hot-path rule); dynamic-mesh reuse preserved |
| 7 | **Pre-existing UB** (UB-1 audio race, UB-2 shader-continue, UB-3 irange, UB-4 GL leaks/context, UB-5 unseen OOB) | Each mapped to its fixing step: M8 (race + TSan), M5 (fatal shaders + RAII), M3 (assert), M4 audit, M8/M9 audits under ASan |
| 8 | **Golden flakiness across drivers/libm** (UB-6, libm variance) | Primary oracle is the sim-state hash; image hash is secondary and per-machine; goldens recorded per CI platform |

---

## 16. Long-Term Maintenance Policy

- **Versioning:** semantic versioning, starting at **v1.0.0 at M14** (the refactor is a pure-behavior change → no major bump of user-visible semantics; the pre-refactor monolith is tagged `v0-monolith` for history). Public module headers = the de-facto API: additive changes = minor, breaking = major.
- **CHANGELOG:** Keep-a-Changelog format, updated in the same PR as the change; CI can later check its presence.
- **Deprecation policy:** a deprecated path survives **at most one migration step or one release** — never two permanent ways to do anything. Temporary strangler shims (M3–M11 monolith includes, the local `gl` instance until M6) carry `DEPRECATED(M#)` comments with named removal steps. Releases never ship a shim.
- **Dependency policy:** pin and minimize. `doctest` at a fixed commit; SDL2 via system package with the tested version documented in README; **no new runtime dependency without an ADR**; vendored code only under `third_party/` with a provenance note.
- **CI gate list (must stay green before any change is accepted):** clean build (warnings-as-errors, Linux/macOS/Windows matrix) · unit tests · smoke goldens (state + audio hashes, per platform) · clang-format check · clang-tidy · cppcheck · include-layer lint. A PR may update goldens only with an explicit justification diff reviewed against the behavior contract.

---

## 17. Definition of Done

- [ ] Warning-free build (`-Wall -Wextra -Wpedantic -Wshadow -Werror`) on the full CI matrix; Debug and Release presets both build.
- [ ] Zero unjustified globals: the only file-scope objects are `constexpr` data and the stateless SDL callback forwarder (auditable via the migration log's final grep protocol).
- [ ] Dependency rules tool-enforced: CMake per-layer target graph + `check_includes.py` green; no circular includes.
- [ ] All unit tests green; smoke harness green on all CI platforms against recorded goldens, including ASan/UBSan runs and a TSan run over the audio path.
- [ ] Behavior contract (§12) verified in full: all 21 rows, all four flagged rewrites documented and hash-proven.
- [ ] Frame time within 10% of the M2 baseline in the smoke report.
- [ ] README, ARCHITECTURE.md, STYLE.md, BEHAVIOR-CONTRACT.md, MIGRATION-LOG.md, and ADRs 0001–0011 written; every public header carries its module doc block.
- [ ] All four (plus one) extensibility recipes of §14 dry-run verified: a scratch branch adds a trivial new state and a trivial new audio track registration and compiles without touching any core/engine/render file.
- [ ] `v1.0.0` tagged; CHANGELOG started; monolith deleted; repository history preserves the full migration, one commit per M-step.

*End of plan.*
