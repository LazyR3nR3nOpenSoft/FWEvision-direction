# FWE — Flow Works Engine | Design Document

*All-in-one free open-source engineering design and simulation platform*

---

## What It Is

Model, simulate, and iterate without ever leaving the app. No fragmented tools, no paid licenses. The simulation power already exists in open source engines — FWE is the layer that finally makes them accessible and zero hassle for the average user.

---

## Core Philosophy

- **One stop** — design and test in the same environment
- **Sub-main architecture** — each tab is its own sub-process, self-contained but connected to FWE core. Feels like its own program, talks to everything else
- **Checkpoint system** — snapshot your model anytime, experiment without fear (like VirtualBox snapshots but for 3D models)
- **Progressive disclosure** — simple on the surface, deep when you dig. Component-based means UI complexity scales with what you need, not modular installs
- **Tab-as-context** — one active context at a time by default. Switch tabs, context saves, new context loads. You're never blocked, nothing burns memory sitting idle
- **Node-as-initializer** — dropping a node spins up its backend. Removing it releases it. No nodes placed = no engine running
- **Blender-familiar viewport** that thinks in engineering terms natively
- **GPLv3 core**, commercial API license for private module builders and enterprise integrations
- **Fast lane** (Model → Simulate) and **Full lane** (Model → Material & Physics → Environment → Simulate) — nobody gets blocked
- **Everything ships complete** — simulation types are toggleable overlays, not separate installs. The physics are coupled by nature; stripping modules breaks the sim

---

## Business Model

| Tier | License | Who |
|------|---------|-----|
| FWE core | GPLv3, free forever | Everyone |
| Community modules | GPLv3, public | Anyone builds and shares |
| Private modules | Commercial API license | Companies build internally |
| FWE API spec | Licensed commercially | Unity, Unreal, studios, OEMs, any .fwe integrator |
| Verified environment presets | Commercial bundle | ISO-certified, real-world characterized |
| Priority material data | Commercial bundle | Advanced composites, high-spec materials |

*Long-term goal: `.fwe` becomes the standard format for simulation-accurate 3D assets.*

---

## Repo Structure

**Free Tier — GPLv3, always**
- Flow-Works-Engine-FWE
- fwe-materials
- fwe-environments
- fwe-nodes
- fwe-speakers
- fwe-microphones
- fwe-lights *(add later)*
- FWEvision-direction

**Commercial Bundle — via FWE-API license**
- API spec + .fwe format definition
- Verified environment presets (ISO certified, real-world characterized)
- Priority material data (advanced composites, high-spec materials)
- Private node building rights
- Org-shared library access
- Enterprise support

One API license = full bundle. Target customers: game studios, OEMs, Unity/Unreal native .fwe integration, any company building .fwe read/write into their pipeline. GPLv3 core stays free forever. Bundle is the upsell for enterprise and commercial integrators.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Viewport | Blender render engine (GPL) |
| Geometry kernel | OpenCASCADE — parametric + direct modeling |
| Fluid/aero | OpenFOAM + SU2 (user selectable), Code Saturne |
| Structural/thermal FEM | FEniCS / CalculiX / Sparselizard |
| Mechanical systems | OpenModelica |
| Combustion/Propulsion | Built natively (Engine Simulator + JSBSim as reference only) |
| Composite flow sim | OpenFOAM fiber orientation plugin |
| Mesh generation | Gmsh |
| Visualization | ParaView |
| 3D spatial audio | OpenAL Soft |
| HRTF calculation | Mesh2HRTF |
| HRTF/mic dataset standard | SOFA format |
| Viewport rendering | OpenGL |
| GPU compute — NVIDIA | CUDA |
| GPU compute — AMD | ROCm / HIP |
| GPU compute — fallback | OpenCL |
| Viewport upscaling | DLSS (NVIDIA) / FSR (AMD) / XeSS (Intel) |
| Virtual audio device | Native FWE virtual audio driver |
| Language | C++ core, Python scripting layer |
| UI | Dear ImGui (viewport/panels) + Qt (outer shell) |
| Build | CMake |

---

## Module Roadmap

All simulation types ship with FWE core — nothing is a separate install. The roadmap reflects development priority and UI surface area, not modular packaging.

Core → Material → Fluid → Thermal → Mechanical → Acoustic → Light → Vibration → Automotive [v0.2.0] → Marine [v0.2.0] → Aerospace [v0.2.0] → Drone/Small Systems → Advanced Materials (composites) → Ballistics → Instrument/VST pipeline

---

## Sub-Main Architecture

FWE core is a lightweight parent process — an orchestrator. It owns nothing heavy itself. Each tab is its own sub-main: a self-contained sub-process with its own memory space, its own initialization, its own backends. They feel like independent programs. They communicate through FWE core via context handoff.

```
FWE Core (parent — lightweight orchestrator)
├── Model sub-main        → OpenCASCADE, viewport, geometry
├── Material sub-main     → material library, node canvas, physics properties
├── Environment sub-main  → atmosphere, node placement, stage system, audio engine
└── Simulate sub-main     → backends, compute, overlays, hardware monitoring
```

**Why sub-mains, not tabs in one process:**
- One sub-main crashes — FWE core survives, all other contexts survive
- Memory is genuinely isolated — not sleeping, actually separate address spaces
- Each sub-main optimized independently — no shared overhead
- Panic function has real process-level kill switches per sub-main, not just soft interrupts
- Matches how engineers actually work — you don't model and simulate simultaneously

**Single Active Context Model:**
- One sub-main active at a time by default
- Switch tab → current context serializes and saves → new sub-main initializes
- **Explicit keep-alive toggle** — for cases where you genuinely need two contexts simultaneously (e.g. Environment + Simulate for live stage tweaking)
- No two contexts burning memory unless you explicitly force it

**Node-as-Initializer:**

Backends don't run until they're needed. Dropping a node into the scene is the trigger:
- Drop a speaker node → audio engine (OpenAL Soft) initializes
- Drop a mic node → Mesh2HRTF loads
- Drop a fluid sim overlay → OpenFOAM/SU2 spins up
- Remove the node → backend releases

No nodes placed = nothing running. The scene drives the compute, not the tab.

**Inter-Process Communication:**
- **Cold path** (tab switches) — full context serialization to disk/swap. Safe, complete, no data loss
- **Hot path** (live cross-context data during keep-alive) — shared memory IPC or memory-mapped files. Low overhead for real-time data exchange between Environment and Simulate during live stage workflows
- FWE core brokers all communication — sub-mains never reach into each other's memory directly

**Tab Context Session Saving:**

| Level | Scope | Purpose |
|-------|-------|---------|
| **Project checkpoint** | Entire project across all tabs | Snapshot this whole state before I experiment |
| **Tab context save** | Single sub-main only | Save where I am in this tab before switching |

- **Auto-save** — every tab switch triggers automatic context save of the departing sub-main. Restore is automatic on return
- **Manual save** — user explicitly snapshots a tab context. Named slots, multiple per tab
- Cross-tab dependency detection — if Model context changes after Simulate has already read it, FWE prompts 'model changed — rebuild sim?'

---

## UX Layout

**Design Language**

FWE UI is a culmination of three references — VS Code, Blender, and DaVinci Resolve. Every layout and interaction decision should feel like it belongs to all three simultaneously.

- **VS Code** — activity bar on far left, collapsible panel sections, clean dark sidebar, nothing visible until needed
- **Blender** — viewport is king, tools left, properties right, nothing wastes screen space
- **DaVinci Resolve** — tab bar at bottom switches major contexts, node graph elegance (Fusion), professional broadcast feel, panel toggle buttons in corners

**Color Scheme**

Cold system grey — professional, functional, lets the viewport and simulation data be the visual focus. Nothing competes with what is actually important. Deep cold grey body, slightly lighter panels, subtle borders for definition, cold blue accent for active states and selections. Viewport is pure black canvas. Simulation overlays use their own color-coded system on top.

**Subprocess Pill Toggles**

Each tab has a pill-shaped toggle sitting in the top right corner of its own content area. Accent color when ON (subprocess loaded and alive in memory). Grey/dark when OFF (tab exists but subprocess is dormant, nothing processing). This is the keep-alive control — not tab navigation, not a settings menu. Just the subprocess state, always visible, one click to change.

- First launch — Model pill auto-flips ON. Everything else OFF
- Pill OFF = tab is navigable but subprocess releases from RAM. Nothing computed, nothing stored
- Multiple pills ON simultaneously = explicit keep-alive. Power user decision, their hardware, their choice
- Even Model pill can be turned OFF for pure RAM freedom if needed

**Global Layout Structure**

- **Top bar** — minimal menu bar only
- **Left panel** — full height, VS Code activity bar + collapsible sections, toggleable via DaVinci-style corner button
- **Center** — viewport, dominant, king of the screen
- **Right panel** — shorter (does not go full height), context-sensitive floating parameter popovers anchored here, toggleable via corner button
- **Bottom tab bar** — DaVinci style, Model / Material / Environment / Simulate / Export / Settings

All panels are individually toggleable. Collapse everything for full viewport. Both panels use scrollable parameter areas — nothing clips, nothing truncates, nothing overlaps.

**Startup**

Opens directly into Model tab. Left sidebar shows VS Code-style panel — New Project, Open Recent, Open .fwe, Installed Modules. 3D viewport already live behind it. Dismiss panel, start working.

---

## Model Tab

**Left Sidebar**
- Hand, Select, Move/Transform, Pen
- Primitives — cube, cylinder, sphere, plane, custom
- Import file, Parts library, Recent models

**Right Panel**

Object properties, transform, section/grouping tools. Define and lock model sections here — carries forward to all other tabs.

**Capabilities**
- Parametric modeling, direct modeling, both simultaneously. Geometry kernel: OpenCASCADE
- Design any geometry, any component, any machine — engines, exhausts, speakers, rotors, anything
- Group components into assemblies
- Section groups carry forward to all tabs automatically

**Import Formats v1**
- STEP, IGES, OBJ, glTF/GLB, .blend, STL, DXF
- FBX — post-launch

---

## Material & Physics Tab

Separate tab — engineers who just want to test geometry skip this entirely. No material assigned = solid geometry only. Shape-dependent sims (aero, hydro) run fine. Material-dependent sims (structural, thermal, acoustic) prompt 'assign material to run this simulation.'

**Right Panel**
- Material library — searchable, categorized, custom presets
- Auto-populate physics properties on preset select — all values pulled from validated .fwe-materials file, no dynamic calculation
- Composite layup designer — fiber type, orientation, ply sequence, resin matrix, void content
- Section assignment — group assign or individual assign
- Property override — tweak any single value on any single section without touching anything else

**Custom Material Parameters**

| Category | Parameters |
|----------|-----------|
| Mechanical | Density, yield strength, UTS, Young's modulus, Poisson's ratio, hardness |
| Thermal | Thermal conductivity, specific heat capacity, thermal expansion coefficient, melting point |
| Acoustic | Speed of sound, acoustic impedance, absorption coefficient |
| Optical | Refractive index, reflectivity, emissivity, transmittance |
| Fluid | Viscosity, surface tension |
| Electrical | Conductivity, permittivity, permeability |
| Magnetic | Classification tag + permeability + saturation |

> Note: The Additions doc covers magnetic category depth, composite layup editor, and material data handling in full detail.

---

## Node System

DaVinci Fusion-style node graph — NOT Blender-style. Flow-based, left to right, typed inputs and outputs. Reads like a pipeline — engineers trace it in 30 seconds.

**Tier 1 — Individual Nodes, Granular Control**
- Assign to specific modeled geometry
- Node respects geometry — magnet shape affects field distribution, pipe geometry affects pressure wave behavior, etc.
- Examples: Magnet, Membrane, Combustion, Coil, Exhaust nodes
- Full control down to smallest detail

**Tier 2 — Category Nodes, All-in-One**
- Pre-bundled Tier 1 nodes with sensible defaults and pre-assigned groups
- Drop on assembly, assign components to groups, tweak top level, simulate
- Examples: Speaker, Engine, Exhaust, Turbofan, Rotary, Suspension, Heat Exchanger nodes

Beginners use Tier 2. Engineers pop it open and go granular with Tier 1 underneath. Tier 2 is just Tier 1 pre-bundled — same system, two entry points. Community contributes nodes via PR to fwe-nodes.

> Note: In-depth node system specifications are in the Additions doc.

---

## Environment Tab

Environment tab is the scene-scale simulation layer. The whole scene runs live — geometry, materials, atmosphere, and all placed nodes interacting simultaneously. Viewport-first output. Particles, overlays, wave propagation visible in real time. Everything is placeable as nodes, everything configurable, everything saveable as .fwe-env-scene. Two loading modes: full scene, or main environment preset plus one model.

**Asset Library Tiers**
- Built-in presets — ships with FWE, maintained by the project
- Community/3rd party — downloaded and imported, live in asset library
- User-created — built in Model tab, saved locally or exported to share
- Team/org shared — colleagues send .fwe-env files, import it, sits alongside your own

**Atmospheric Parameters**

All tweakable from any preset: air direction, turbulence intensity, noise floor, gravity vector, light intensity, medium density/pressure/temperature.

**Built-in Environment Presets**

| Category | Presets |
|----------|---------|
| Acoustic | Anechoic chamber (full), Hemi-anechoic, Reverberation room, Open air, Concert hall, Urban street |
| Aero | Sea level calm, High altitude, Turbulent boundary layer, Wind tunnel |
| Fluid | Still water, Ocean surface, River current, Pressurized pipe |
| Thermal | Arctic, Desert, Industrial furnace, Cryogenic |
| Optical | Dark room, Light studio, Overcast outdoor, Direct sunlight, Integrating sphere |
| Space | Full vacuum, Microgravity, Solar radiation exposure |

**Virtual Mic System**
- Place multiple mics anywhere in 3D scene — drag and drop as nodes
- Mic model library via fwe-microphones — SOFA/HRTF profiles
- Mic types: omnidirectional, cardioid, supercardioid, figure-8, shotgun, measurement, binaural dummy head
- Backend: OpenAL Soft + Mesh2HRTF
- Each mic captures independently — SPL, frequency response, polar data simultaneously

**Speaker / PA System**
- Place speaker nodes anywhere in 3D scene
- Driver library via fwe-speakers — Thiele-Small parameters seeded from public manufacturer datasheets
- Model enclosure geometry in Model tab, assign cabinet material, drop Speaker node, select driver — sim runs full frequency response, port tuning, standing waves
- **Stage mode** — place PA cabinets, line arrays, monitor wedges, run acoustic coverage map
- See dead spots, comb filtering, coverage overlap before load-in

**Lighting System**
- Draggable light nodes — intensity, color temperature, beam angle, falloff
- Camera node — drag anywhere in scene, view from any angle
- **Stage mode** — GDTF open fixture library (spotlights, floods, gobos, LEDs, moving heads, lasers)
- Custom fixture import — bring your own 3D model

**Console Integration — Control Surface Nodes**

*Lighting Console Node*
- DMX channel assignment per fixture
- Program cues, chases, scenes, timings
- Physical DMX controller plugs in — programs FWE directly in real time
- Export cue file 1:1 to real rig

*Audio Mixer Node*
- Channel strips, EQ, compression, routing, send/return
- Each speaker and mic node feeds into mixer node
- Program PA signal chain inside FWE before soundcheck
- Physical MIDI surface plugs in — controls FWE mixer directly
- OSC output — talk to any real digital console or DAW

**Player Node — Live Audio Source Injection**

FWE registers as a **native virtual audio device** on the system. Any software on the machine can select 'FWE Audio' as its output — no plugins, no rewiring, no patching required.

- DAW (FL Studio, Ableton, Reaper, etc.) — selects FWE as system audio output
- DJ software (Serato, Traktor, rekordbox) — same pipe
- Media player — drag and drop file or select FWE output
- Receives live audio stream, routes through placed speaker nodes
- Acoustic sim runs on actual live signal — real coverage map, real SPL, real comb filtering
- Headphone output comes out of FWE — engineer hears the room, not the raw mix

**Protocol & Format Support**

*Lighting:* DMX512, Art-Net, sACN (E1.31), GDTF, MVR

*Audio:* MIDI, OSC, AES67

*Dante:* deferred post-launch (patented by Audinate)

> Rule: Zero gray areas at launch.

**Bottom Node Graph Panel**

Toggleable bottom panel — hidden by default, revealed with corner button (DaVinci Fusion style). 3D viewport handles spatial node placement. Bottom panel handles logical connections between nodes. Two views of the same system.

---

## Simulate Tab

Component-scale precision simulation. One object, one material, engineering-grade data output — graphs, numbers, exportable logs. Two loading modes: one component no environment, or one component plus .fwe-env-main for real-world atmospheric context. Same node system as Environment tab.

**Visual Overlay System**

Each simulation type toggles independently. Multiple overlays active simultaneously — color coded, readable.

- Fluid/aero flow, Thermal, Structural stress, Acoustic pressure, Vibration, Optical/light, Electromagnetic

**Failure Visualization — Automatic**
- Thresholds pulled directly from assigned material properties (yield strength, UTS, fatigue limits)
- Color scale: Green (safe) → Yellow (approach limit) → Orange (near failure) → Red (failure)
- Failure points mode — color zones only
- Deformation mode — physical bend, warp, fracture propagation in slow motion
- Playback speed control: adjustable from real time down to 0.1x

**Hand Tool — Physical Interaction Layer**
- Grab any surface, apply directional force vector in real time
- Weight nodes — drag onto any surface, define mass, sim updates live
- Force axis locking — freeform drag or lock to specific axes

**Two-Stage Execution**
- **Build** — compiles material metadata, sets up simulation dependencies, meshes geometry
- **Run** — activates after Build. Live streaming data, real time observation

**Panic Function — Program-Wide Hard Interrupt**

Three-level kill system. State saved at all levels — no data loss.

- **Level 1 — Node Cascade Kill:** failure propagates through dependent nodes automatically. Simulation continues on unaffected nodes
- **Level 2 — Subprocess Kill:** kills only the offending tab subprocess. FWE core and all other tabs survive. Triggered automatically at resource threshold
- **Level 3 — Full Panic:** program-wide hard interrupt. Kills all sub-mains simultaneously. Manual trigger available (one button). State saved on trigger

**Two View Modes**
- **Stacked mode** — all active overlays on viewport simultaneously, toggle each independently
- **Per-tab mode** — one simulation type at a time, side panel goes deep with real-time graphs, numerical readouts, frequency plots, stress curves

**Record Function**

Record simulation playback directly from the Simulate tab. Two output types from the same recording session:
- **Viewport capture** — what you see, overlays included
- **Data capture** — raw simulation values logged over time, for R&D documentation

Capture format is set in Settings — not a per-recording decision. Set once, all recordings follow that preference.

---

## GPU Compute & Hardware Acceleration

Two independent compute dropdowns — viewport and simulation are completely separate workloads and should never compete for the same config.

**Viewport Compute**

| Setting | Description |
|---------|-------------|
| Auto (default) | Detects GPU, picks best path automatically |
| OpenGL | Universal fallback — every GPU speaks it |
| DLSS | NVIDIA upscaling — quality presets |
| FSR (FidelityFX) | AMD upscaling — AMD/Intel/any GPU |
| XeSS | Intel upscaling |

**Simulation Compute**

| Setting | Description |
|---------|-------------|
| Auto (default) | Detects GPU vendor, routes to CUDA / ROCm / OpenCL accordingly |
| CUDA | NVIDIA explicit |
| ROCm / HIP | AMD explicit |
| OpenCL | Universal fallback — Intel iGPU, older hardware, anything else |
| CPU only | Edge case |

Both dropdowns are fully independent. Panic function monitors both independently.

---

## Export Tab

**Three Modes**
- **Default** — game devs and hobbyists. Full package, one click
- **R&D** — select exactly what data exports
- **Custom Presets** — saved export configurations. Set once, reuse forever

**Export Formats v1**

| Category | Formats |
|----------|---------|
| Industry | STEP, IGES, STL, DXF |
| Game/3D | OBJ, glTF/GLB, .blend |
| Native | .fwe, .fwe-env-main, .fwe-env-scene, .fwe-obj, .fwe-objphys, .fwe-spkmod, .fwe-litmod |
| Simulation data | OpenFOAM mesh formats, VTK |
| Post-launch | FBX |

**Format-Specific Metadata Presets**

| Preset | Format | Metadata included |
|--------|--------|------------------|
| Game engine clean | glTF/GLB, .blend | Geometry + PBR only |
| CAD handoff | STEP, IGES | Geometry + dimensions only |
| 3D print ready | STL | Geometry only, manifold checked |
| Manufacturing / R&D | .fwe-objphys | Geometry + material properties |
| Stage export | MVR + OSC/MIDI | Environment + console cue data only |
| Full R&D | .fwe | Everything |
| Custom | Any | User picks exactly what goes in |

---

## Settings Tab

Everything that does not need to be in the engineer's face mid-workflow lives here.

- **Compute** — Viewport GPU dropdown, Sim compute GPU dropdown
- **Hardware** — Panic threshold config, Memory allocation per subprocess, CPU thread allocation
- **Workspace** — Watched folders + folder roles, Keep-alive preferences, Auto-save frequency
- **Updates** — Auto update toggle, Manual trigger, Per repo update control
- **Capture** — Recording format (set once, applies to all recordings)
- **Export** — Default export presets
- **Theme** — Theme picker, Import/Export .fwe-theme
- **Help** — Tooltip toggle, documentation panel

**Settings Preset Import/Export**

Import .fwe-set-pref — load a full settings configuration. Export .fwe-set-pref — share with team, post for community, or keep as backup. Huge for teams — configure one machine perfectly, share the file, every machine is identical in one click.

**Reset Controls**
- Per section Reset button — resets only that section
- Reset All Settings — nuclear option at bottom, confirmation prompt required

---

## File Format Family

All formats are zip containers sharing the same base architecture. Each carries exactly what it needs.

| Format | Contains | Who uses it |
|--------|---------|------------|
| .fwe | Full project — geometry, sim data, checkpoints, all metadata | Everyone — the master project file |
| .fwe-env-main | Base environment only — atmosphere, boundary conditions, environment geometry + materials | Preset makers, community sharing, stage teams |
| .fwe-env-scene | Full session — references .fwe-env-main + all models, nodes, measurement data, sim state | Engineers saving full working sessions |
| .fwe-obj | Geometry only | Aero/hydro engineers, game devs, anyone who just needs the shape |
| .fwe-objphys | Geometry + material properties + physics metadata | R&D, structural, thermal, acoustic |
| .fwe-mat | Material source data — TOML, human readable, never loaded at runtime directly | Material library, community contributions |
| .fwe-spkmod | Speaker model — Thiele-Small params, enclosure geometry, driver profile, frequency response | Speaker manufacturers, acoustic engineers |
| .fwe-litmod | Light fixture model — beam geometry, intensity/color profile, DMX mapping | Lighting manufacturers, stage designers |
| .fwe-theme | UI theme | Community theme sharing |
| .fwe-set-pref | Full settings configuration | Teams, enterprise IT |

Every format has three tiers: open (TOML/JSON, GPLv3), -com (binary, commercial), -ent (binary + encrypted, enterprise). See Additions doc for full tier architecture.

---

## File Format Licensing

**Fully Clean — Implement Directly**
- glTF/GLB — open source, royalty-free, Khronos Group standard
- OBJ, STL — public domain
- STEP/IGES — open ISO standards
- DXF — Autodesk publishes spec openly
- .blend — GPL, compatible with GPLv3
- DMX512, Art-Net, sACN, MIDI, OSC, AES67 — open standards
- GDTF, MVR, SOFA format, VST3 — open standards

**Deferred Post-Launch**
- FBX — needs attribution audit first
- Dante — patented by Audinate, needs legal clarity first

> Rule: Every format and protocol shipping in FWE v1 is open standard. Zero gray areas at launch.

---

## fwe-materials Library

Repo: `FWE-R3nSoftOrg/fwe-materials` — GPLv3. Primary sources only — every property value cites original paper/report and institution.

**Data Sources (GPLv3-compatible only)**
- Materials Project (CC BY 4.0) — mechanical, structural, elastic, density
- AFLOW — additional DFT-sourced data
- NIST SRD 69/81/150 (public domain) — thermal, mechanical
- refractiveindex.info GitHub (CC) — optical, YAML direct parse
- Published NDT/acoustic papers via Archive.org (public domain) — sparse damping data

**Material Data Pipeline**

`Source → Parser script → Excel (per type) → Manual QA → Consolidator script → Full profile → Validator script → .fwe-mat`

**One File Per Material**

Each material is a single .fwe-mat zip container. FWE loads only the file it needs. In-program downloader fetches individual material files on demand — no bulk library download required.

---

## FWE API

Commercial API license required for external .fwe read/write integration.

Governs:
- Full format family read/write
- Environment tab live stage data stream
- Player node / virtual audio device integration
- Console integration protocol bridge (DMX, OSC, MIDI, MVR)
- GPU compute offload interface
- Panic function hooks for external monitoring tools
- Module building rights (private nodes, verified environments, priority materials)

---

*GPLv3 | FWE-R3nSoftOrg*
