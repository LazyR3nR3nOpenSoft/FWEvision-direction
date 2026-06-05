# FWE Design Document — Additions

*Additions not yet merged into main design doc*

---

## 1. Digital Twin Platform — Core Realization

FWE's value is not simply 'simulate things.' The platform models reality with sufficient fidelity that what works in FWE works in the real world. Every component requires a full parameter stack — not just geometry and a few physics values, but the entire behavioral chain of that component.

> Note: For the initial alpha launch, publicly available data may limit the full realization of this philosophy. LazyR3nR3n will make it as close as possible for accuracy-driven use cases.

### 1.1 Component Depth — Node Parameter Standards

| Category | Depth Requirement |
|----------|------------------|
| Mechanical | Full kinematic + thermodynamic chain |
| Electrical | Signal path + impedance + power |
| Electrical Classification | Classification tag (Conductor / Semiconductor / Insulator) + conductivity value (S/m). Fixed categorical field — two data points per material entry. |
| Magnetic | Classification tag (Ferromagnetic / Paramagnetic / Diamagnetic) + magnetic permeability (H/m) + magnetic saturation (T, ferromagnetic only). Saturation value required for accurate coil node simulation — defines the point where field response becomes non-linear. |
| Acoustic | Transducer physics + room interaction |
| Fluid / Aero | Flow inputs/outputs + environmental coupling |
| Lighting | DMX/fixture profile + optical physics |
| Control | MIDI/OSC/timecode + trigger/automation logic |
| Embedded & Electronic Control | Firmware behavior + communication protocols + thermal |

### 1.2 Component Examples

**Speaker Cabinet**
- Driver specs — Thiele-Small parameters (Fs, Qts, Vas, Xmax)
- Crossover network — PCB-level component values, filter order, impedance curve
- Diaphragm excursion model
- Thermal limits — voice coil heating over time
- SPL + directivity per frequency band
- Array coupling behavior when stacked/flown

**Engine (ICE)**
- Fuel input node — type, pressure, flow rate
- Combustion cycle — timing, compression ratio, ignition
- Torque/RPM curve output
- Thermal + exhaust byproduct outputs
- Engine will not start if fuel input node is disconnected — intentional behavior

**Jet Engine**
- Startup sequence — ignition, spool-up time, fuel flow ramp
- N1/N2 shaft speed nodes
- Thrust output as physical force node
- Temperature nodes — EGT monitoring

**Rotor**
- Blade pitch node — collective + cyclic (helicopter)
- RPM input from motor/engine node
- Aerodynamic output — lift, drag, torque reaction
- Wake turbulence propagation into fluid sim

**Explosion**
- Trigger node — electrical, impact, timer
- Charge parameters — type, mass, shape
- Overpressure wave propagation
- Fragmentation pattern
- Thermal pulse radius
- Toggle: simulate structural damage yes/no

**Stage Light (Moving Head / Par / Beam)**
- DMX address node
- Fixture profile — gobos, prism, zoom range, CRI
- Beam divergence + atmospheric scattering
- Thermal output
- Power draw node

---

## 2. Audio Routing Architecture

### 2.1 Two Domains, One Routing Layer

**Virtual (In-Sim)**
- Speaker arrays output SPL into the acoustic solver
- Room/environment geometry affects reflections, comb filtering, coverage
- Visualized as heatmap overlay

**Physical Hardware Routing**
- FWE acts as a virtual patch bay and virtual console
- Inputs and outputs map to real hardware via OSC/MIDI/AES67/Dante discovery
- Channel strips correspond to real analog or digital paths
- Routing works like a real console — busses, matrix outputs, aux sends

The virtual mixer is the interface for both domains. One routing layer, two domains (sim + hardware). Toggle between views as needed.

### 2.2 MIDI + Lights — Show Control

FWE supports timecode-driven show control as a first-class workflow:
- MIDI timecode (MTC) or SMPTE sync
- MIDI input node — clock, notes, CC
- Mapping MIDI events to cue triggers or parameter modulation
- Beat-reactive parameter automation — intensity, color, position
- Export show file directly to target console (MA3, Eos, Resolume)

A festival LD should be able to design their rig in FWE, simulate the full show with audio-reactive lighting, and export that show file to their console.

---

## 3. Embedded & Electronic Control Node Category

Because FWE already models material electrical properties, power input nodes, and signal routing — microcontrollers, PCBs, and ECUs are not special cases. They are nodes that process electrical signals according to their logic, using the same physical substrate already modeled.

### 3.1 Node Types

**Microcontroller Node**
- Clock speed, I/O pin map, voltage rails
- Firmware behavior node — upload actual firmware or define logic rules
- Communication protocol nodes — UART, SPI, I2C, CAN bus, PWM output
- Interrupt handling, watchdog timer behavior

**ECU (Automotive)**
- Sensor input nodes — MAF, O2, TPS, knock, coolant temp
- Fuel + ignition map — tune tables as actual lookup table parameters
- CAN bus output to transmission, ABS, traction control nodes
- Fault code generation when parameters go out of range

**Drone Flight Controller**
- RC input node — SBUS/CRSF/PPM
- IMU sensor fusion node — gyro + accel data
- PID loop parameters — rate, attitude, exposed as tunable
- ESC output nodes per motor
- Failsafe behavior on signal loss

**Generic PCB Node**
- Schematic import — KiCad / EasyEDA format (R&D import path)
- Component-level simulation — passives as real values, ICs as behavioral models
- Thermal map of board under load
- Signal integrity — crosstalk, trace impedance

### 3.2 Import / Export

**Import**
- KiCad .kicad_pcb / .kicad_sch
- SPICE netlists (.cir, .net) for circuit-level sim
- STEP/IGES for mechanical PCB housing
- Firmware binaries or logic definition files

**Export**
- Simulation report — signal behavior, thermal hotspots, QC flags
- Back to KiCad/EasyEDA with FWE simulation annotations
- BOM with stress-tested component ratings flagged
- Pass/fail QC report per defined test profile

### 3.3 System-Level QC Capability

Because FWE models the full system — mechanical, thermal, electrical, firmware — it can answer questions no single-domain tool can:

- Does the ECU tune hold when the engine is thermally soaked after 45 minutes?
- Does the drone FC maintain stable PID when the frame resonates at 200Hz?
- Does the speaker DSP clip when the cabinet is driven at rated power in a 40°C environment?

These questions currently require physical prototypes. FWE answers them in simulation.

### 3.4 Coupling Interfaces

Electrical ↔ Mechanical ↔ Thermal ↔ Signal

---

## 4. Measurement & Instrumentation Nodes

Measurement nodes are live simulation taps — not display widgets. Attach to any node output to read real-time parameter values during simulation. They also serve as QC checkpoints: define acceptable ranges, run the sim, FWE flags every node that went out of spec.

### 4.1 Measurement Node Types

**Mechanical**
- RPM meter — shaft speed (rotors, engines, turbines, wheels)
- Torque — Nm (engine output, drivetrain, fasteners under load)
- Force — N (thrust, lift, tension, compression)
- Displacement — mm/m (vibration amplitude, deflection)
- Angular velocity — rad/s

**Thermal**
- Temperature — K / °C (engine bay, EGT, PCB hotspot, bearing temp)
- Heat flux — W/m²
- Thermal resistance

**Electrical**
- Voltage, Current, Power — V, A, W
- Impedance — Ω (speakers, PCB traces)
- Frequency — Hz (signal, PWM)

**Acoustic**
- SPL — dB
- Frequency response — live FFT node
- Phase

**Fluid / Aero**
- Pressure — Pa / PSI / bar
- Flow rate — L/s, kg/s
- Velocity — m/s
- Density

**Lighting**
- Luminous intensity — cd
- Illuminance — lux
- Color temperature — K

### 4.2 Measurement Node Features

- Threshold alerts — flag when value goes out of defined range
- Data logging — record over sim time, exportable as CSV
- Graph overlay — live mini chart attached to the node
- Unit conversion toggle — display in any unit without affecting the sim

---

## 5. Unit System & Measurement Framework

Global unit system selector — set once per workspace, applies everywhere. Per-model and per-node overrides supported.

### 5.1 Unit Systems

| System | Description |
|--------|-------------|
| SI (Metric) | mm/m, kg, N, °C/K, Pa — default |
| Imperial | in/ft, lb, lbf, °F, PSI |
| Mixed | User-defined per category — e.g. SI base with PSI for hydraulics |

### 5.2 Complete Unit Reference

**Length / Distance:** pm, nm, µm, mm, cm, dm, m, km — in, ft, yd, mi, nmi — Å — AU

**Mass:** µg, mg, g, kg, t (metric ton) — oz, lb, slug, ton (short/long)

**Force:** N, kN, MN — lbf, kip, dyn

**Pressure:** Pa, hPa, kPa, MPa, GPa — bar, mbar — psi, ksi — atm, mmHg, inHg, torr

**Temperature:** K, °C, °F, °R (Rankine)

**Torque:** N·m, kN·m, mN·m — lbf·ft, lbf·in, ozf·in

**Energy:** J, kJ, MJ, GJ — cal, kcal — Wh, kWh, MWh — BTU, eV

**Power:** W, kW, MW, GW, mW — hp (mechanical, electrical, metric) — BTU/hr

**Speed / Velocity:** m/s, km/h — mph, ft/s, kn — Mach (referenced to local speed of sound in sim environment)

**Acceleration:** m/s², g (standard gravity) — ft/s²

**Frequency:** Hz, kHz, MHz, GHz — RPM, RPS — BPM (audio/lighting sync nodes)

**Electrical:** V, mV, kV — A, mA, µA — Ω, mΩ, kΩ, MΩ — F, µF, nF, pF — H, mH, µH — S, mS

**Magnetic:** T, mT, µT, G (gauss) — A/m — Wb (weber, flux)

**Luminous / Optical:** cd, lm, lx, cd/m² (nit) — K (color temperature)

**Acoustic:** dB SPL, dB(A), dB(C) — Pa — Hz

**Angular:** °, rad, mrad — arcmin, arcsec — rev

**Volume / Flow:** mL, L, m³, cm³, mm³ — fl oz, gal (US/UK), ft³, in³ — m³/s, L/s, L/min, GPM, CFM

**Density:** kg/m³, g/cm³, g/L — lb/ft³, lb/in³

**Viscosity:** Pa·s, mPa·s (dynamic) — m²/s, cSt (kinematic)

**Data / Signal (Embedded Nodes):** bit, byte, KB, MB, GB — bps, kbps, Mbps, Gbps — Hz (clock)

All units tied to ISO/IEC standard definitions where applicable. Conversion factors hardcoded and exact — no floating point drift on unit switching.

---

## 6. Non-Metallic & Natural Material Physics Category

*Target version: Alpha 0.02.0 (or later, subject to public data availability)*

### 6.1 Overview

A new top-level category in fwe-materials for natural and biological materials, distinct from the existing NIST-seeded engineering materials.

### 6.2 Category Split

**Non-Metallic:** Wood, stone, soil, ceramics, glass, polymers, etc.

**Natural/Biological:** Soft tissue, flesh, muscle, cartilage, bone, plants, leaves, etc.

### 6.3 Architecture

FWE's existing node system handles model-parameterized behavior. Hyperelastic models (Mooney-Rivlin for soft tissue), anisotropic behavior (wood grain), and hygroscopic response (wood moisture swelling) are all expressible as node graph templates. Each material entry carries standard property data plus a reference to a behavior node template. No new architecture required.

### 6.4 Alpha 0.02.0 Scope

- Category and type split defined in fwe-materials
- Node templates built for core behavior models — hyperelastic, anisotropic, hygroscopic at minimum
- Material library entries — stubbed or empty, populated later via community PRs or dedicated research phase

### 6.5 Target Use Cases

- Hyper-realistic game environment and character physics
- Vtubing — physically grounded avatars and environments (underserved, high-visibility early adopter community)
- General simulation fidelity for organic and natural subjects

---

## 7. Materials Format Family

### 7.1 Format Overview

Three-tier material format system. The open TOML source is the canonical format — the others are compiled/encrypted derivatives. Community contributors only ever touch TOML — maintainers handle compilation.

| Format | Type | Tier | Who uses it |
|--------|------|------|------------|
| .fwe-mat | TOML, human-readable | Open / GPLv3 | Community contributors, anyone auditing data |
| .fwe-mat-com | Binary compiled | Commercial license | Studios, engineering firms, runtime use |
| .fwe-mat-ent | Binary + encrypted | Enterprise license | R&D, high-security environments, highest fidelity datasets |

> Note: This three-tier system applies to ALL format families — materials, speakers, models, environments, etc. Three versions per format type.

### 7.2 Lookup Table Architecture

Material properties that vary by condition are stored as lookup tables with standardized anchor points — not interpolated curves. Discrete measured values only, sourced from actual lab data.

- Sound absorption — standard octave bands (125Hz, 250Hz, 500Hz, 1kHz, 2kHz, 4kHz)
- Refraction — standard wavelength anchors (R/G/B primaries + UV + IR)

Each entry cites its source measurement.

### 7.3 Materials Pipeline Design

**Data Sources — GPLv3-Compatible Only**

All sources must be GPLv3-compatible — publicly available, no ToS scraping restrictions, no proprietary data. Engineering Toolbox and Matweb are confirmed incompatible and are excluded.

- Materials Project (CC BY 4.0) — mechanical, structural, elastic moduli, density. API via mp-api Python client
- NIST SRD 69 Chemistry WebBook (public domain) — thermochemical and physical properties
- NIST SRD 81 Heat Transmission Properties (public domain) — thermal conductivity and resistivity
- NIST SRD 150 Property Data Summaries (public domain) — density, elastic moduli, strength, hardness, thermal expansion
- refractiveindex.info GitHub repo (open source, CC) — optical constants n and k per wavelength. Parsed directly from YAML
- Published NDT/acoustic papers via Archive.org (public domain) — sparse damping/loss factor data

**Acoustic and Vibration Derivation**

No standalone GPLv3-compatible acoustic properties database exists. Acoustic velocity and vibration response are derived offline from mechanical properties already in the pipeline (bulk modulus, shear modulus, density, Young's modulus, Poisson's ratio). Derivation runs once as a build step — results are baked as static values into the .fwe-mat file. No live derivation at runtime.

Damping/loss factor is empirically measured and cannot be derived. Values pulled from NDT literature where available. Gaps remain null — null is honest.

**Data Confidence Tiers**

Every property in a .fwe-mat file carries a confidence tier tag:

- **static** — value pulled directly from a cited primary source. Sim runs silently
- **derived** — computed offline from other available properties. Formula cited. Sim runs with a soft warning logged in the output panel
- **fallback** — Wikipedia/Matweb sourced values (alpha only). Flagged visibly in sim output
- **null** — no source and no derivation path. FWE prompts user to continue with dynamic calculation or cancel

**Confidence hierarchy:** measured > static > literature > fallback > null

**Excel Staging and QA Flow**

All scraped and derived data lands in Excel first — one sheet per source type. Acoustic and vibration derivations are computed and appended before the manual review pass. Manual QA is one-by-one per material — to catch scraping artifacts: mismatched units, wrong material matches, formatting corruption, physically implausible values. After QA, the consolidator script merges sheets into full material profiles and outputs .fwe-mat files. The validator runs last.

---

## 8. Licensing Architecture

### 8.1 Binary License Generation Protocol

Fully offline, baked into FWE runtime. No phone home, no license server, no infrastructure dependency. Works in air-gapped environments. Each license key is a capability manifest encoding:

- Which file tiers it can decrypt (.fwe-com, .fwe-ent, or both)
- Which feature unlocks are active
- Seat count — number of installs that can activate with this key
- Hardware fingerprint slots — X allowed hardware profiles per key (similar to Windows activation)

**Key distinction:** Commercial (.fwe-com) files can be shared between any commercially licensed installs for collaboration. Enterprise (.fwe-ent) files cannot be shared outside the org — encryption is org-specific by design.

### 8.2 Revocation & Update Policy

- Minor / patch updates — keys untouched, full backward compatibility guaranteed
- Major version updates — key generation protocol may change. Commercial customers receive a migration window. Enterprise customers get extended notice
- Emergency revocation — surgical per-key blacklist baked into an update. Not a blanket wipe

### 8.3 Key Lifecycle System

**Default — Key Versioning**
- FWE runtime maintains a registry of all past private keys
- Old encrypted files decrypt silently regardless of which key generation created them
- Zero friction, no migration step required

**Opt-in — Active Re-encryption**

For R&D and enterprise environments with strict security protocols. User-triggered manually or on major update. Two modes:
- Replace in place — file re-encrypted with current keys, old version purged
- New copy — fresh re-encrypted copy generated, original preserved until user handles it per their own security policy

**Selling Point for -ent Tier**

Not just encrypted data — auditable encrypted data with user-controlled key lifecycle. Directly addresses enterprise R&D procurement requirements.

---

## 9. Material Seeding — Alpha Decision & Session Record

### 9.1 Alpha Decision

Automated materials library seeding (AFLOW/MP pipeline) is deferred. The materials library (fwe-materials repo) is a post-funding / own-lab-verification phase deliverable. Alpha ships with a polished manual material data entry UI instead. Users bring their own data.

20–30 starter materials from AFLOW (GPLv3-compatible, CC license) will be pre-packaged as the baseline lib. Each entry tagged with source, method (DFT-computed), and verification status. UI includes inline documentation stating starter materials are DFT-sourced and suitable for preliminary simulation — production-grade results require experimental verification or user-supplied data.

Local user materials: stored locally only, never auto-synced. Users can optionally submit to community lib via PR.

### 9.2 Benchmark Material — γ-TiAl (mp-1953, P4/mmm, L1₀)

First material used to validate the seeding pipeline and .fwe-mat format. All data sourced from free, GPLv3-compatible or open-access sources.

**Mechanical (Materials Project, mp-1953, DOI: 10.17188/1194736):**

ρ = 3.87 g/cm³ | K_VRH = 113 GPa | G_VRH = 68 GPa | ν = 0.25 | E = 170 GPa (derived: 2G(1+ν))

Stiffness tensor (GPa): C11=C22=190 | C33=168 | C12=70 | C13=C23=84 | C44=C55=112 | C66=44

**Thermal (Holec et al., MDPI Materials 2019, DOI: 10.3390/ma12081292):**

αa = 11 × 10⁻⁶ K⁻¹ (a-direction) | αc = 5 × 10⁻⁶ K⁻¹ (c-direction) | Cp ≈ 668 J/kg·K

**Phase temperatures (Overton 2006, Carleton University):**

Solidus Ts = 1504°C (1777 K) — Ti-48Al composition

**Pending (flagged, non-blocking):**

κ (thermal conductivity) — known range 7–25 W/m·K (Zhang et al. 2001, Scripta Materialia), exact binary value behind paywall. Flagged as `estimated_range` in .fwe-mat. Non-blocking for mechanical/acoustic sim.

### 9.3 Calculation Protocol — Missing Data Handling

Derived values are always tagged as derived with formula cited. Truly unavailable values are flagged as pending_verification — never fabricated.

- E → derive from G and ν: `E = 2G(1+ν)`
- v_L → `√((K + 4G/3) / ρ)`
- v_S → `√(G / ρ)`
- κ missing → flag as `estimated_range` with source note

### 9.4 Manual Entry UI — Design Decisions

Two modes: Simple (K, G, ν, ρ) and Advanced (full Cij tensor). Composite support: blend two materials, entry points for both. Each field shows: value, unit, source, status (verified / derived / pending_verification). A blank .fwe-mat skeleton file (all fields empty, fully commented) ships alongside the UI so users can fill data manually in any text editor.

---

## Environment vs Simulate Clarification

**Environment tab** is scene-scale simulation. The whole scene runs live. Viewport-first output. Particles, overlays, wave propagation visible in real time.

**Simulate tab** is component-scale simulation. Precision run on one object. Data-first output — graphs, numbers, exportable logs. Optionally load an environment preset for real-world atmospheric context.

Same node system works in both tabs identically. The node is the universal interface between the sim engine and the human.

**Four Loading Modes:**

Environment tab:
- Full scene, or
- Main environment preset plus one model

Simulate tab:
- One component no environment, or
- One component plus environment preset loaded

---

## Manual Material Creation — Three Types

Three material types, each with a tailored no-code input form. Accessed via Material tab → New Material → Select Type.

**Metallic**

Crystalline, isotropic by default. Fields: density, Young's modulus, shear modulus, bulk modulus, Poisson's ratio, yield strength, UTS, hardness, thermal conductivity, specific heat, thermal expansion, melting point, electrical conductivity, magnetic classification. Advanced mode exposes full Cij tensor for anisotropic metals. Confidence tier assigned per field.

**Non-Metallic**

Covers glass, ceramics, polymers, wood, stone, biological materials (soft tissue, bone, cartilage, skin, fur). Same base fields as metallic but with additional toggles: Hyperelastic toggle (Mooney-Rivlin coefficients) for biologicals, Hygroscopic toggle for organics, Acoustic absorption curves at standard octave bands. Anisotropic toggle reveals per-axis stiffness fields without exposing raw tensor.

**Composite**

Opens directly into the layup stack editor + weave spatial editor. Bulk properties derived from layup geometry via homogenization — not entered directly. Manual override available for users supplying measured lab data.

---

## Composite Layup + Weave Editor

**Layup stack:** each ply defined by material, fiber orientation angle, thickness, pattern type (unidirectional, woven, twill, chopped). Add, remove, reorder plies. Matrix type and volume fraction set at stack level. Confidence tier follows the weakest ply — if any ply is fallback or null, the whole composite inherits that tier.

**Weave spatial editor:** 2D interactive plane showing the actual weave structure of a selected ply. Individual cells or regions selectable — assign different fiber materials per region. Supports hybrid composites (e.g. carbon fiber base weave with Kevlar regions). Fibers color-coded, interlacing pattern preserved visually.

**Weave map is the actual solver input — not decorative.** FWE runs homogenization from the spatial fiber distribution to derive the effective Cij tensor automatically. Derived tensor stored in .fwe-mat alongside the weave map. Confidence tier: derived — geometrically grounded, more honest than a generic datasheet value.

---

## Propulsion Module

*Scope: v0.2.0 alpha. Three domains — land, air, marine. All built natively. Reference implementations inform logic only, not bundled.*

**Land (Combustion)**

Reference: Engine Simulator by Ange Yaghi. Covers ICE — fuel input, combustion cycle, torque/RPM curve, thermal + exhaust outputs. Engine will not start if fuel input node disconnected — intentional.

Tier 1 nodes: Combustion, Exhaust, Fuel Input, Ignition, Cooling. Tier 2 node: Engine.

**Air (Turbine/Jet/Rocket)**

Reference: JSBSim — under evaluation, not confirmed. Covers turbofan, turbojet, turboprop, rocket. Startup sequence, spool-up, N1/N2 shaft nodes, thrust as physical force node, EGT monitoring.

Tier 1 nodes: Compressor, Combustion Chamber, Turbine, Nozzle, Afterburner. Tier 2 nodes: Turbofan, Turbojet, Rocket Motor.

**Marine**

No reference implementation identified — fully custom. Covers diesel/petrol marine engines, shaft drive, waterjet, propeller thrust. Cavitation behavior, hull resistance coupling to fluid sim.

Tier 1 nodes: Marine Engine, Shaft, Propeller, Waterjet, Rudder. Tier 2 node: Marine Drive.

**Shared Propulsion Behavior**

All propulsion nodes output physical force vectors into mechanical/fluid sim. Exhaust and thermal outputs couple to fluid and thermal sims automatically. Failure modes — overtemp, overrev, fuel starvation — trigger automatically from material and operating limits, not hardcoded thresholds.

---

## Tech Stack Corrections & Clarifications

- Engine Simulator removed from stack table — combustion is built natively, engine-sim and JSBSim are reference implementations only, not bundled dependencies
- C++ core orchestrator, Python as scripting layer
- Tab sub-processes: C++ for Environment and Simulate (direct OpenCASCADE/OpenAL), Model tab TBD
- Core ↔ Python communication: IPC via sockets/pipes (consistent with tab sub-process architecture)

---

*GPLv3 | FWE-R3nSoftOrg*
