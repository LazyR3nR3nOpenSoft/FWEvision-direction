# FWE-MCD-L1 — Master Context Document

FWE (Flow Works Engine) is an all-in-one free open-source engineering design and simulation platform. GPLv3 core, commercial API license for enterprise/OEM integrators. Owner: LazyR3nR3n (GitHub handle). Language: C++ core, Python scripting layer. Build: CMake. UI: Dear ImGui (viewport/panels) + Qt (outer shell).

> **For developers joining a session:** Paste this L1 doc at the start of every coding session. Then load the relevant L2 module slice(s) for the area you're working on. This is the FWE Master Context Document — it gives any Claude instance (or developer) full project context to pick up and contribute without needing prior session history.

---

## Core Architecture

FWE core is a lightweight orchestrator — owns nothing heavy, brokers everything. Four sub-mains: Model, Material, Environment, Simulate. Each is an isolated subprocess with its own memory space. One active at a time by default. Explicit keep-alive toggle for simultaneous contexts.

**FWE core owns:** scene graph, translation layer, IPC, state management, temp file system, panic function.

---

## Sub-Main Map

- **Model** → OpenCASCADE, viewport, geometry
- **Material** → material library, node canvas, physics properties
- **Environment** → scene-scale sim, always-on, node placement, audio/lighting/atmosphere
- **Simulate** → component-scale precision sim, Build→Run two-stage, data output

---

## Node-as-Initializer

Backends don't run until needed. Drop node → backend spins up. Remove node → backend releases. No nodes = nothing running. Node system is the universal interface between sim engine and human — input, output, visualization, measurement. All nodes composable, all context-agnostic. Works identically in Environment and Simulate tabs.

---

## IPC

- **Cold path** (tab switches) — full context serialization to disk via temp .fwe family files
- **Hot path** (keep-alive) — shared memory IPC or memory-mapped files
- FWE core brokers all communication. Sub-mains never reach each other's memory directly

---

## Scene Graph & State Management

FWE core owns the scene graph. Implemented as temp file system using .fwe format family. Auto-managed by default, manual override in Settings.

VirtualBox-style snapshots — per object, per tab, per session. Manual named snapshots available anytime.

State history toggle: global (default OFF) and per sub-main. OFF = limited undo, maximum performance. ON = full granular history at performance cost.

---

## Translation Layer

FWE core owned. Translates .fwe-objphys + atmospheric params into solver-specific input formats at Build step.

Geometry path: `OpenCASCADE BREP → Gmsh C API → solver mesh`

- OpenFOAM gets .msh → gmshToFoam → polyMesh
- CalculiX gets .inp directly
- SU2 gets .su2 directly

---

## Panic Function

Three-level kill system. State saved at all levels — no data loss.

**Level 1 — Node Cascade Kill:** failure propagates through dependent nodes. If an upstream node dies (e.g. fuel pump stops), all nodes that depend on it cascade off automatically. User receives a warning explaining what failed and why. Simulation continues on unaffected nodes. Auto-triggered by node-level failure conditions.

**Level 2 — Subprocess Kill:** kills only the offending tab subprocess. FWE core survives. All other tabs survive. Triggered automatically when a subprocess hits its resource threshold (RAM, CPU, or VRAM). Warning icon always visible next to each tab pill — flashes yellow on auto-kill to signal the user. Clicking the warning icon manually kills or restarts the subprocess. Threshold configurable in Settings per subprocess or system-wide.

**Level 3 — Full Panic:** program-wide hard interrupt. Kills all sub-mains simultaneously — GPU compute, sim backends, render, audio engine, Environment always-on sim engine. Triggered by hardware emergency (temperature spike, critical memory breach). Manual trigger also available (one button). State saved on trigger, no data loss.

**Subprocess Pill vs Warning Icon:** two separate UI elements, two separate jobs.
- Pill = subprocess on/off control only (user manually boots or shuts down the subprocess)
- Warning icon = health monitor and manual kill toggle (always visible next to pill, flashes yellow on auto-kill)

**Threshold System:** FWE monitors RAM, CPU, and VRAM per subprocess. VRAM tracking respects the existing viewport/sim GPU split — tracked independently. Two modes in Settings: System-wide (one set of thresholds for all subprocesses, default) and Per-subprocess (individual threshold override per tab).

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Viewport | Blender render engine |
| Geometry | OpenCASCADE |
| Fluid/aero | OpenFOAM + SU2 + Code Saturne |
| Structural/thermal FEM | FEniCS / CalculiX / Sparselizard |
| Mechanical | OpenModelica |
| Combustion/Propulsion | Built natively (Engine Simulator + JSBSim as reference only, not bundled) |
| Mesh | Gmsh |
| Visualization | ParaView |
| Audio | OpenAL Soft |
| HRTF | Mesh2HRTF |
| HRTF standard | SOFA |
| Rendering | OpenGL |
| GPU NVIDIA | CUDA |
| GPU AMD | ROCm/HIP |
| GPU fallback | OpenCL |
| Upscaling | DLSS/FSR/XeSS |
| Language | C++ core, Python scripting layer |
| UI | Dear ImGui (viewport/panels) + Qt (outer shell) |
| Build | CMake |

---

## Build Order

Core → Model → Material → Simulate → File Format → Export → Settings first.

Module roadmap: Core → Material → Fluid → Thermal → Mechanical → Acoustic → Light → Vibration → Automotive [v0.2.0 alpha] → Marine [v0.2.0 alpha] → Aerospace [v0.2.0 alpha] → Drone → Advanced Materials → Ballistics → Instrument/VST

---

## Format Family

All formats are zip containers. Each carries exactly what it needs. Every format has three tiers: open (TOML/JSON, GPLv3), -com (binary, commercial license), -ent (binary + encrypted, enterprise license).

| Format | Contains |
|--------|---------|
| .fwe | Full project, master file |
| .fwe-env-main | Base environment only — atmosphere + boundary conditions + environment geometry/materials. What ships as presets |
| .fwe-env-scene | Full session — references .fwe-env-main + all models/nodes/measurement data/sim state |
| .fwe-obj | Geometry only |
| .fwe-objphys | Geometry + physics metadata baked together, solver-ready, one file read one memory allocation |
| .fwe-mat | Material source data, TOML, never loaded at runtime directly |
| .fwe-spkmod | Speaker model |
| .fwe-litmod | Light fixture model |
| .fwe-theme | UI theme |
| .fwe-set-pref | Full settings config |

**.fwe-objphys:** geometry and physics metadata baked in one file. Optimization decision: one file read, one memory allocation, no runtime cross-referencing. Material swapping at export/save time in Material tab, not at runtime. Binary layout optimized for solver read order.

---

## Business Model

| Tier | Details |
|------|---------|
| Community | GPLv3, free forever |
| Commercial API | Binary format SDK, .fwe-com family, seat-limited key, files shareable between licensed installs |
| Enterprise R&D | .fwe-ent family, org-specific encryption, air-gapped, files unreadable without org private key |

Zero-knowledge guarantee — LazyR3nR3n cannot open enterprise files. ECC encryption. Fully offline license validation, no phone home.

---

## GPU Compute

Two independent dropdowns — viewport and sim never compete.
- Viewport: Auto/OpenGL/DLSS/FSR/XeSS
- Sim: Auto/CUDA/ROCm/OpenCL/CPU

Auto detects on first launch.

---

## FWE-MCD-L2-Core

**Owns:** scene graph, translation layer, IPC broker, temp file system, panic function, state management, cross-tab dependency detection. Lightweight — no heavy backends. All sub-mains communicate only through core.

Cross-tab dependency: if Model changes after Simulate has read it, core prompts "model changed — rebuild sim?"

Tab context save: auto-save on every tab switch (departing sub-main serializes). Manual named slots per tab. Project checkpoint = entire project across all tabs. Tab context save = single sub-main only. Separate operations, separate UI.

**Settings it controls:** compute backend selection, watched folders, keep-alive preferences, auto-save frequency, hardware panic thresholds (RAM/CPU/VRAM — system-wide or per-subprocess toggle), memory allocation per subprocess, CPU thread allocation, state history toggle (global + per sub-main), solver selection override per simulation type.

---

## FWE-MCD-L2-Model

Sub-main. Geometry kernel: OpenCASCADE — parametric + direct modeling simultaneously. Section groups defined here, carry forward to all other tabs automatically. On tab switch, core serializes Model state as temp .fwe file — Material tab reads from that.

**Geometry Validation:** primary validation happens here — Model tab checks geometry integrity before any other tab reads it. Catches bad geometry at the source before it reaches the mesh or solver pipeline.

**Left sidebar:** Hand, Select, Move/Transform, Pen, Primitives (cube/cylinder/sphere/plane/custom), Import, Parts library, Recent.

**Right panel:** object properties, transform, section/grouping tools.

**Import formats v1:** STEP, IGES, OBJ, glTF/GLB, .blend, STL, DXF. FBX post-launch.

**Export:** .fwe-obj + industry standard formats.

**Startup:** opens directly into Model tab. Left sidebar shows New Project/Open Recent/Open .fwe/Installed Modules. Viewport live behind it.

---

## FWE-MCD-L2-Material

Sub-main. Reads Model state from temp .fwe file via core. .fwe-objphys bake triggered on manual save or tab switch — core handles write.

**Mesh Validation:** secondary validation pass happens here during .fwe-objphys bake — confirms geometry + physics metadata are clean before baking into the solver-ready file.

**Right panel:** material library (searchable, categorized, custom presets), auto-populate physics on preset select (values from .fwe-mat, no dynamic calculation), composite layup designer, section assignment (group or individual), property override per section. Two entry modes: Simple (K, G, ν, ρ) and Advanced (full Cij tensor).

**Material parameters:** Mechanical (density, yield strength, UTS, Young's modulus, Poisson's ratio, hardness), Thermal (conductivity, specific heat, expansion coefficient, melting point), Acoustic (speed of sound, impedance, absorption), Optical (refractive index, reflectivity, emissivity, transmittance), Fluid (viscosity, surface tension), Electrical (conductivity, permittivity, permeability), Magnetic (classification tag + permeability + saturation).

No material assigned = solid geometry only. Shape-dependent sims run fine. Material-dependent sims prompt "assign material to run this simulation."

---

## FWE-MCD-L2-Environment

Sub-main. Scene-scale simulation layer. Always-on continuous — no explicit Build step. FWE core manages via temp filing per session and per edit.

Two loading modes: full scene, or main environment preset (.fwe-env-main) plus one model.

Environment IS a scene of grouped models with physics metadata attached. Every surface is a separate geometry group with its own material (.fwe-objphys). Material carries acoustic absorption, thermal, reflectivity — all of it. HRTF and SPL calculations reflect actual material properties per surface, not reverb presets.

**.fwe-env-main:** base environment only — atmosphere, boundary conditions, environment geometry + surface materials. What ships as presets (anechoic chamber, wind tunnel, still water, etc.). Reusable, shareable, context-agnostic.

**.fwe-env-scene:** full session — references .fwe-env-main + all placed models + transforms + nodes + configs + measurement data + sim state.

Node system identical to Simulate tab. Overlay nodes, measurement nodes, visualization nodes all plug and play. Particle/wave visualization = physics made visible in real time.

Bottom node graph panel: toggleable, DaVinci Fusion-style, flow left to right. 3D viewport = spatial placement. Bottom panel = logical connections. Two views of same system.

**Atmospheric parameters:** air direction, turbulence intensity, noise floor, gravity vector, light intensity, medium density/pressure/temperature.

**Built-in presets:** Acoustic (anechoic chamber, hemi-anechoic, reverb room, open air, concert hall, urban street), Aero (sea level calm, high altitude, turbulent boundary layer, wind tunnel), Fluid (still water, ocean surface, river current, pressurized pipe), Thermal (arctic, desert, industrial furnace, cryogenic), Optical (dark room, light studio, overcast, direct sunlight, integrating sphere), Space (full vacuum, microgravity, solar radiation).

**Audio:** OpenAL Soft + Mesh2HRTF. Virtual mic system — SOFA/HRTF profiles, multiple mic types. Speaker nodes — Thiele-Small, full frequency response sim. Player node — FWE registers as native virtual audio device, any app selects FWE Audio as output. Acoustic sim runs on live signal.

**Lighting:** GDTF fixture library, DMX/Art-Net/sACN. Console integration: DMX, OSC, MIDI, MVR. Show control: MTC/SMPTE sync, beat-reactive automation, export to MA3/Eos/Resolume.

Panic covers Environment always-on sim engine explicitly.

---

## FWE-MCD-L2-Simulate

Sub-main. Component-scale precision simulation. Data-first output — graphs, numbers, exportable logs.

Two loading modes: one component no environment, or one component plus .fwe-env-main for real-world atmospheric context. Environment preset in Simulate affects actual physics — not decorative.

Same node system as Environment tab. Overlay nodes, measurement nodes, visualization nodes identical.

**Build step** (owned by Simulate sub-main, translation layer owned by FWE core):
1. Geometry export → Model sub-main serializes OpenCASCADE geometry as BREP → passed to Simulate via core cold path
2. Mesh generation → Gmsh C API ingests BREP directly → outputs solver-specific format
3. Solver handoff → mesh + .fwe-objphys material + .fwe-env-main atmospheric params → solver subprocess

Run unlocks after Build completes.

**Solver routing by placed node:**
- Fluid/aero → OpenFOAM or SU2
- Structural/thermal → CalculiX or FEniCS
- Acoustic → OpenAL + Mesh2HRTF
- No node = nothing running

Solver selection: Auto by default — FWE core selects the appropriate solver based on simulation type and placed nodes. Manual override available via dropdown in Settings.

**Multi-material scenes:** .fwe-env-scene holds references to all .fwe-objphys files. Translation layer iterates through them at Build. Each already has material baked in. Each handed to appropriate solver.

**Visual overlays:** Fluid/aero flow, Thermal, Structural stress, Acoustic pressure, Vibration, Optical/light, Electromagnetic. Toggle independently, multiple simultaneously. Stacked mode (all overlays at once) and Per-tab mode (one type, deep data panel).

**Failure visualization:** thresholds from material properties. Color scale green → yellow → orange → red. Failure points mode and deformation mode. Playback speed down to 0.1x.

**Hand tool:** grab any surface, apply directional force in real time. Weight nodes. Force axis locking.

**Record function:** viewport capture + data capture simultaneously. Format set in Settings once.

Measurement node data persists in .fwe-env-scene.

**Mesh Validation Gate** — fallback validation for externally imported geometry. FWE runs element quality check via Gmsh C API (minSICN metric). Two modes: Auto (refines bad elements silently, proceeds to Run) and Manual (pauses Build, highlights problem geometry in viewport). Default: Auto. Configurable in Settings.

---

## FWE-MCD-L2-Nodes

Universal system. Same nodes work in Environment and Simulate. Context-agnostic.

**Tier 1** — individual nodes, granular control. Assign to specific geometry. Node respects geometry — shape affects physics behavior. Examples: Magnet, Membrane, Combustion, Coil, Exhaust.

**Tier 2** — category nodes, pre-bundled Tier 1 with sensible defaults. Drop on assembly, assign components, tweak top level. Examples: Speaker, Engine, Turbofan, Rotary, Suspension, Heat Exchanger. Tier 2 is just Tier 1 pre-bundled — same system, two entry points.

**Measurement nodes** — live simulation taps, not display widgets. Attach to any node output. Types: Mechanical (RPM, torque, force, displacement, angular velocity), Thermal (temperature, heat flux, thermal resistance), Electrical (voltage, current, power, impedance, frequency), Acoustic (SPL, FFT, phase), Fluid/aero (pressure, flow rate, velocity, density), Lighting (luminous intensity, illuminance, color temp). Features: threshold alerts, data logging (CSV export), live mini chart, unit conversion toggle.

**Embedded & Electronic Control nodes:** Microcontroller (clock, I/O, firmware behavior, UART/SPI/I2C/CAN/PWM), ECU (sensor inputs, fuel/ignition map, CAN bus output, fault codes), Drone FC (SBUS/CRSF/PPM, IMU fusion, PID tunable, ESC outputs, failsafe), Generic PCB (KiCad import, component-level sim, thermal map, signal integrity). Import: KiCad, SPICE netlists, STEP/IGES, firmware. Export: simulation report, back to KiCad, BOM, QC report.

Community contributes nodes via PR to fwe-nodes repo.

---

## FWE-MCD-L2-Formats

All formats zip containers, three tiers each (open/com/ent). See L1 for full table.

**.fwe-mat:** TOML, human-readable, GPLv3, never loaded at runtime. Source of truth. Properties stored as lookup tables at standardized anchor points — discrete measured values, no interpolation. Sound absorption at standard octave bands. Refraction at standard wavelength anchors. Confidence tiers: static (cited source, silent), derived (formula cited, soft warning), null (prompts user).

**.fwe-objphys:** geometry + physics baked. One file read, one memory allocation. Binary layout optimized for solver read order. Material swapping at save time not runtime.

**.fwe-env-main:** base environment. Presets ship as this. **.fwe-env-scene:** full session including measurement data. Embeds into master .fwe.

**License key:** capability manifest encoding file tier access, feature unlocks, seat count, hardware fingerprint slots. Fully offline. ECC encryption. Commercial: .fwe-com shareable between licensed installs. Enterprise: .fwe-ent org-specific, air-gapped, unreadable without org key. Zero-knowledge guarantee.

File format licensing: all v1 formats open standard. FBX and Dante deferred post-launch.

---

## FWE-MCD-L2-Materials-Pipeline

**Sources (GPLv3-compatible only):**
- Materials Project CC BY 4.0 (mechanical, structural, elastic, density — mp-api Python client)
- NIST SRD 69/81/150 (public domain — thermal, mechanical)
- refractiveindex.info GitHub (CC — optical, YAML direct parse)
- NDT/acoustic papers via Archive.org (public domain — sparse damping data)

**Pipeline:** source → parser script → Excel per type → manual QA → consolidator script → full profile → validator script → .fwe-mat

Acoustic/vibration derived offline from mechanical properties (bulk modulus, shear modulus, density). Derivation baked as static values — no live derivation at runtime. Damping null where unavailable — null is honest.

**Benchmark material:** γ-TiAl (mp-1953). ρ=3.87 g/cm³, K=113 GPa, G=68 GPa, ν=0.25, E=170 GPa. Thermal conductivity flagged estimated_range (paywall).

**Alpha:** 20-30 starter materials from Materials Project, DFT-sourced, tagged with verification status. Manual entry UI ships for alpha — Simple and Advanced modes. Blank .fwe-mat skeleton ships alongside. Full automated pipeline deferred post-funding.

Natural/biological materials: Alpha 0.02.0 target. Non-biological (wood, stone, soil, plant matter) and biological (soft tissue, bone, cartilage). Hyperelastic (Mooney-Rivlin), anisotropic, hygroscopic behavior as node graph templates.

---

## FWE-MCD-L2-Material — Addition

**Manual Material Creation**

Three material types, each with a tailored no-code input form. Accessed via Material tab → New Material → Select Type.

**Metallic:** Crystalline, isotropic by default. Full mechanical/thermal/acoustic/electrical/magnetic fields. Advanced mode exposes full Cij tensor for anisotropic metals. Confidence tier assigned per field.

**Non-Metallic:** Covers glass, ceramics, polymers, wood, stone, biological materials (soft tissue, bone, cartilage, skin, fur). Hyperelastic toggle (Mooney-Rivlin coefficients) for biologicals, Hygroscopic toggle for organics, Acoustic absorption curves at standard octave bands. Anisotropic toggle reveals per-axis stiffness fields without exposing raw tensor.

**Composite:** Opens directly into the layup stack editor + weave spatial editor. Bulk properties derived from layup geometry via homogenization — not entered directly. Manual override available for users supplying measured lab data.

**Composite Layup + Weave Editor**

Layup stack: each ply defined by material, fiber orientation angle, thickness, pattern type (unidirectional, woven, twill, chopped). Add, remove, reorder plies. Matrix type and volume fraction set at stack level. Confidence tier follows the weakest ply.

Weave spatial editor: 2D interactive plane showing the actual weave structure of a selected ply. Individual cells or regions selectable — assign different fiber materials per region. Supports hybrid composites. Fibers color-coded, interlacing pattern preserved visually.

**Weave map is the actual solver input — not decorative.** FWE runs homogenization from the spatial fiber distribution to derive the effective Cij tensor automatically. Derived tensor stored in .fwe-mat alongside the weave map. Confidence tier: derived.

---

## FWE-MCD-L2-UX

Design language: VS Code + Blender + DaVinci Resolve simultaneously. Cold system grey body, slightly lighter panels, cold blue accent, pure black viewport canvas. Sim overlays use their own color-coded system.

**Global layout:** Top bar (minimal menu only), Left panel (full height, VS Code activity bar + collapsible sections, toggleable), Center (viewport, dominant), Right panel (shorter, context-sensitive parameter popovers, toggleable), Bottom tab bar (DaVinci style — Model / Material / Environment / Simulate / Export / Settings).

**Subprocess pill toggles:** pill per tab, top right of content area. Accent = ON (subprocess alive). Grey = OFF (dormant). First launch: Model pill auto-ON, everything else OFF. Multiple pills ON = explicit keep-alive, user's choice.

Per-tab quick export buttons in side panel. Export tab = master bus for full project output, batch, custom presets. Settings tab = everything that doesn't need to be mid-workflow. Update system: startup checks all repos, one popup (Update All / Choose / Skip).

---

## FWE-MCD-L2-Propulsion

Scope: v0.2.0 alpha. Three domains — land, air, marine. All built natively. Reference implementations inform logic only, not bundled.

**Land (Combustion):** Reference: Engine Simulator by Ange Yaghi. Covers ICE — fuel input, combustion cycle, torque/RPM curve, thermal + exhaust outputs. Engine will not start if fuel input node disconnected — intentional. Tier 1 nodes: Combustion, Exhaust, Fuel Input, Ignition, Cooling. Tier 2 node: Engine.

**Air (Turbine/Jet/Rocket):** Reference: JSBSim — under evaluation, not confirmed. Covers turbofan, turbojet, turboprop, rocket. Startup sequence, spool-up, N1/N2 shaft nodes, thrust as physical force node, EGT monitoring. Tier 1 nodes: Compressor, Combustion Chamber, Turbine, Nozzle, Afterburner. Tier 2 nodes: Turbofan, Turbojet, Rocket Motor.

**Marine:** No reference implementation identified — fully custom. Covers diesel/petrol marine engines, shaft drive, waterjet, propeller thrust. Cavitation behavior, hull resistance coupling to fluid sim. Tier 1 nodes: Marine Engine, Shaft, Propeller, Waterjet, Rudder. Tier 2 node: Marine Drive.

**Shared Propulsion Behavior:** All propulsion nodes output physical force vectors into mechanical/fluid sim. Exhaust and thermal outputs couple to fluid and thermal sims automatically. Failure modes — overtemp, overrev, fuel starvation — trigger automatically from material and operating limits, not hardcoded thresholds.

---

*GPLv3 | FWE-R3nSoftOrg*
