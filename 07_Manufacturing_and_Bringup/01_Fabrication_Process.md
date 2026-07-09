# Semiconductor Fabrication Process — Senior Engineer Deep Dive

## Table of Contents
1. Wafer Preparation
2. Oxidation
3. Lithography (DUV, EUV, Multi-Patterning)
4. Etching (Wet, Dry, Plasma)
5. Ion Implantation and Doping
6. Thin-Film Deposition (CVD, PVD, ALD, ECD)
7. Chemical Mechanical Polishing (CMP)
8. Complete CMOS Process Flow
9. FinFET and GAA Fabrication
10. BEOL (Back-End-of-Line) Processing
11. Advanced Patterning Techniques
12. Yield, Defects, and DFM
13. Reliability Mechanisms
14. Basic Packaging
15. Numbers to Memorize

---

## 1. Wafer Preparation

### 1.1 Silicon Crystal Growth

**Czochralski (CZ) Process:**
   1. Polysilicon chunks melted in quartz crucible at 1414°C
2. Seed crystal (small single-crystal Si) dipped into melt
3. Slowly pulled upward and rotated → single crystal ingot grows
4. Growth rate: ~1-2 mm/min, rotation: ~10-30 RPM

**Crystal orientation:**
   - (100): Most common for CMOS — lower interface trap density
   - (111): Used for some bipolar processes — higher mobility in certain directions

**Wafer diameters (evolution):**
   - 100mm (4") → 150mm (6") → 200mm (8") → 300mm (12") → 450mm (planned)
   - Larger wafers → more dies → lower cost per chip
   - 300mm is current mainstream (since ~2001)

### 1.2 Wafer Specifications

Typical 300mm wafer specs:
- **Diameter** — 300 ± 0.2 mm
- **Thickness** — 775 ± 25 μm
- **Resistivity** — 1-20 Ω·cm (p-type, boron-doped for NMOS bulk)
- **Crystal** — <100> orientation
- **Flatness** — TTV (Total Thickness Variation) < 2 μm
- **Defect density** — < 0.1/cm² (for advanced nodes)

- **Cost** — ~$500-1000 per bare wafer, ~$10,000-20,000 after full processing

### 1.3 Epitaxial Layer

```ascii-graph
Epi layer: Thin (2-20 μm) single-crystal Si grown on wafer surface
  - Lightly doped p-type epi on heavily doped p+ substrate
  - Provides better transistor characteristics (controlled doping)
  - Reduces latch-up (heavy substrate doping quenches parasitic BJTs)
  - Grown by Chemical Vapor Deposition (CVD) at 900-1200°C

  SiH4 (silane) → Si + 2H2  (at high temperature)
  or SiHCl3 (trichlorosilane) → Si + 3HCl
```

---

## 2. Oxidation

### 2.1 Thermal Oxidation

```ascii-graph
Si + O2 → SiO2 (dry oxidation)
  - Slow growth rate, high quality oxide
  - Used for gate oxide (critical — determines transistor performance)
  - Temperature: 800-1100°C
  - Growth rate: ~1-10 nm/hour

Si + 2H2O → SiO2 + 2H2 (wet oxidation)
  - Faster growth rate (~5-10x dry)
  - Lower quality (more defects)
  - Used for field oxide, pad oxide, masking oxide
  - Temperature: 900-1100°C
```

### 2.2 Deal-Grove Model

```text
Oxide thickness vs. time:

  x² + A*x = B*(t + τ)

  where:
    x = oxide thickness
    t = oxidation time
    τ = time offset (accounts for initial native oxide)
    A, B = rate constants (temperature-dependent)
    B/A = linear rate constant (reaction-limited, thin oxide)
    B = parabolic rate constant (diffusion-limited, thick oxide)

Two regimes:
  Thin oxide (x << A):  x ≈ (B/A) * (t + τ)     — linear growth
  Thick oxide (x >> A): x ≈ √(B * (t + τ))       — parabolic growth
                         (diffusion through existing oxide limits growth)
```

**Worked Example — Deal-Grove (Dry Oxide at 1000°C):**

Problem: Calculate the time to grow 10 nm of dry SiO2 at 1000°C on a bare silicon wafer.

Given (at 1000°C, dry O2):  A = 0.165 μm,  B = 0.0117 μm²/hr,  τ ≈ 0 (bare silicon)

Deal-Grove equation:  x² + A·x = B·(t + τ)

- **For x** = `10 nm = 0.01 μm:`
   - (0.01)² + 0.165 × 0.01 = 0.0117 × t
   - 0.0001 + 0.00165 = 0.0117 × t
   - t = 0.00175 / 0.0117 ≈ 0.15 hr ≈ 9 minutes

Note: at this thickness we are in the linear regime (x << A/2 = 0.0825 μm),
so the linear approximation gives a close answer:
t ≈ x / (B/A) = x × A/B = 0.01 × 0.165/0.0117 = 0.141 hr ≈ 8.5 min

### 2.3 Gate Oxide at Advanced Nodes

```ascii-graph
SiO2 gate oxide thickness trend:
  180nm node: tox ≈ 3.5 nm
  90nm node:  tox ≈ 1.2 nm
  45nm node:  tox ≈ 0.9 nm → approaching quantum tunneling limit!

High-k gate dielectric (since 45nm):
  HfO2 (hafnium dioxide): k ≈ 20-25 (vs SiO2 k ≈ 3.9)
  EOT (Equivalent Oxide Thickness) = t_physical × (3.9/k_highk)

  Example: 2nm physical HfO2 → EOT = 2 × (3.9/20) = 0.39 nm
  Same capacitance as 0.39nm SiO2 but much thicker physically → no tunneling

Metal gate (replaces poly-Si):
  Work function metals: TiN, TiAl, TaN
  Needed because poly-Si depletion effect adds to EOT
  "HKMG" = High-K Metal Gate (standard since 45/32nm)
```

---

## 3. Lithography

### 3.1 Photolithography Fundamentals

```ascii-graph
Process:
  1. Spin-coat photoresist on wafer
  2. Expose resist through mask (reticle) using UV light
  3. Develop (dissolve exposed or unexposed resist)
  4. Pattern transfer (etch or implant through resist openings)
  5. Strip remaining resist

  Positive resist: exposed area becomes soluble → removed during develop
  Negative resist: exposed area becomes insoluble → stays after develop

Resolution limit (Rayleigh equation):
  $CD_{min} = k_1 \times \lambda / NA$

  CD_min: minimum feature size (Critical Dimension)
  k1: process factor (theoretical minimum = 0.25, practical ≈ 0.3-0.4)
  λ: exposure wavelength
  NA: numerical aperture of lens (0.9-1.35 for immersion)

Depth of Focus:
  DOF = k2 × λ / NA²
  Higher NA → better resolution but smaller DOF → tighter process control needed
```

**Worked Example — Rayleigh Equation:**

```text
Problem: A 193 nm immersion scanner has NA = 1.35.
What is the minimum half-pitch at k1 = 0.25? At k1 = 0.35?

  CD = k1 × λ / NA

  At k1 = 0.25:  CD = 0.25 × 193 / 1.35 = 35.7 nm  (single-exposure limit)
  At k1 = 0.35:  CD = 0.35 × 193 / 1.35 = 50.0 nm  (more realistic production)

EUV comparison:
  λ = 13.5 nm, NA = 0.33, k1 = 0.3:
  CD = 0.3 × 13.5 / 0.33 = 12.3 nm
```

### 3.2 DUV (Deep Ultraviolet) Lithography

g-line (436nm):  Used for > 500nm features
i-line (365nm):  Used for 350-500nm features
KrF (248nm):     Used for 250-130nm features
ArF (193nm):     Used for 90-65nm features
ArF immersion (193nm, NA=1.35): Used for 45-7nm features (with multi-patterning)

**ArF immersion:**
   - Water (n=1.44) between lens and wafer increases effective NA
   - NA = n × sin(θ) → max NA = 1.35 (with water)
   - Resolution: 193/1.35 × k1 ≈ 40nm half-pitch (with k1 = 0.28)
   - Requires multi-patterning for sub-40nm features

### 3.3 EUV (Extreme Ultraviolet) Lithography

```ascii-graph
Wavelength: 13.5 nm (vs 193nm for ArF)
  → Much better resolution: CD_min ≈ 13.5/0.33 × 0.3 ≈ 12 nm

Key differences from DUV:
  - Reflective optics (not refractive — EUV is absorbed by all materials)
  - Mask is reflective (Mo/Si multilayer mirror, not transmissive)
  - Vacuum environment (EUV absorbed by air)
  - Light source: tin (Sn) plasma, laser-produced plasma (LPP)
    - 250W source → ~100 wafers/hour throughput
  - Very expensive ($150M+ per tool, ASML is sole supplier)

EUV adoption:
  7nm (TSMC N7+):  Limited EUV layers (fewer multi-patterning steps)
  5nm (TSMC N5):   ~14 EUV layers
  3nm (TSMC N3):   ~20+ EUV layers
  2nm and below:   High-NA EUV (NA = 0.55, resolution < 8nm)

High-NA EUV:
  - Next generation: 0.55 NA (vs current 0.33 NA)
  - ~1.7× resolution improvement (CD_min ≈ 8nm half-pitch without multi-patterning)
  - Anamorphic optics (4× in one direction, 8× in other)
  - First tools: 2025-2026 (ASML EXE:5000 series)
  - Intel first to deploy High-NA EUV in production (2025-2026)
    - Installed at Intel D1X fab in Hillsboro, Oregon
    - Targeting Intel 14A node and beyond
  - TSMC and Samsung evaluating for A16/A10 nodes (~2027+)
  - Cost: ~$350M+ per tool (vs ~$150-200M for standard EUV)
  - Key challenge: larger reticle size, new mask infrastructure needed
  - May eliminate multi-patterning on most layers at 2nm and below
```

### 3.4 Resolution Enhancement Techniques (RET)

```ascii-graph
OPC (Optical Proximity Correction):
  - Add assist features to mask to pre-compensate for optical distortion
  - Serifs (corner additions), hammerheads (line-end extensions)
  - Scattering bars (sub-resolution assist features, SRAF)
  - Model-based OPC uses simulation of optical + resist behavior

PSM (Phase Shift Mask):
  - Alternating PSM: adjacent features have 180° phase shift → destructive
    interference at boundary → sharper pattern
  - Attenuated PSM: semi-transparent (6% transmission) with 180° phase shift
  - Improves k1 factor from ~0.4 to ~0.3

SMO (Source-Mask Optimization):
  - Jointly optimize illumination source shape and mask pattern
  - Customized illumination: dipole, quadrupole, freeform
  - Most advanced: inverse lithography technology (ILT)
    - Compute optimal mask shape using inverse problem solving
    - Mask shapes look nothing like the desired pattern
    - Computationally expensive but gives best resolution
```

---

## 4. Etching

### 4.1 Wet Etching

```ascii-graph
Mechanism: Chemical reaction dissolves target material

Etchants:
  SiO2: HF (hydrofluoric acid) — BOE (buffered oxide etch)
  Si:   KOH, TMAH (anisotropic, crystal-plane dependent)
  Al:   H3PO4 + HNO3 + acetic acid
  Si3N4: hot H3PO4

Characteristics:
  - Isotropic (etches equally in all directions) for most etchants
  - Undercuts the mask → limits minimum feature size
  - High selectivity (can choose etchant that attacks one material, not another)
  - Simple, cheap, batch processing
  - Used for: cleaning, sacrificial layer removal, MEMS

Undercut problem:
         Mask
    ┌────────────┐
    │    etch     │
  ←─┤  undercut  ├─→    Isotropic etch goes sideways under mask
    │             │
    └────────────┘
  This limits resolution to ~1μm minimum with wet etch
```

### 4.2 Dry Etching (Plasma Etching)

```text
Types:
  1. Physical (ion milling/sputtering):
     - Energetic ions physically knock atoms off surface
     - Anisotropic (directional) but low selectivity
     - Can damage underlying layers

  2. Chemical (plasma etching):
     - Reactive gas species react with surface
     - Higher selectivity, more isotropic
     - Example: CF4 + O2 plasma for SiO2

  3. Reactive Ion Etching (RIE):
     - Combines physical + chemical mechanisms
     - Ions accelerated toward wafer (directional component)
     - Reactive chemistry provides selectivity
     - Most common etch method for advanced CMOS

  4. Deep Reactive Ion Etching (DRIE):
     - Bosch process: alternating etch and passivation steps
     - Creates very high aspect ratio trenches (>50:1)
     - Used for TSVs, MEMS, trench capacitors

Key etch metrics:
  Etch rate:    How fast material is removed (nm/min)
  Selectivity:  Etch rate of target / etch rate of mask (or underlayer)
                Want >> 1 (e.g., 20:1 for oxide over resist)
  Anisotropy:   A = 1 - (lateral_etch / vertical_etch)
                A = 1: perfectly vertical (ideal)
                A = 0: isotropic
  Uniformity:   Variation across wafer (want < 3%)
  Profile:      Vertical sidewalls, no footing, no notching
```

### 4.3 Advanced Etch Challenges

At 7nm and below:
- Atomic Layer Etching (ALE): removes material one atomic layer at a time
- Self-limiting reactions (like ALD, but removal instead of deposition)
- Needed for fin recess, gate etch with atomic-scale precision

- Etch selectivity becomes critical:
- Gate etch must stop on thin (1-2nm) gate oxide without damage
- Spacer etch must not attack fin sidewalls
- Contact etch through multiple layers with different selectivity

- Aspect-ratio-dependent etching (ARDE):
- Dense features etch slower than isolated features
- Due to reduced ion/radical access in narrow trenches
- Requires etch bias compensation

---

## 5. Ion Implantation and Doping

### 5.1 Ion Implantation Process

```ascii-graph
Beam line implanter:
  Ion source → Mass analyzer → Accelerator → Scanner → Wafer

  1. Ion source: Gas (BF3, PH3, AsH3) ionized by electron bombardment
  2. Mass analyzer: Magnetic field separates ions by mass (selects correct species)
  3. Accelerator: Electric field accelerates ions to desired energy (1-500 keV)
  4. Scanner: Electrostatic/mechanical scanning for uniform dose across wafer
  5. Wafer: Ions penetrate surface, stopped by collisions

Key parameters:
  Species: B (p-type), P or As (n-type), In, Sb
  Energy:  Determines implant depth (Rp = projected range)
  Dose:    Ions per cm² (determines doping concentration)
  Tilt/Twist: Angle to avoid channeling (typically 7° tilt)
```

### 5.2 Implant Physics

```ascii-graph
Range and straggle (Gaussian approximation):
  N(x) = (Dose / (√(2π) × ΔRp)) × exp(-(x - Rp)² / (2 × ΔRp²))

  Rp:   Projected range (average penetration depth)
  ΔRp:  Range straggle (standard deviation)

Example (Boron into Si at 50 keV):
  Rp ≈ 170 nm
  ΔRp ≈ 58 nm
  Dose = 10^13 /cm² → peak concentration ≈ 7 × 10^17 /cm³

Channeling:
  If beam aligns with crystal axis, ions travel much deeper
  (channel between atomic rows with fewer collisions)
  Prevention: tilt wafer 7° from beam, or use pre-amorphization implant (PAI)
```

### 5.3 Activation Anneal

```ascii-graph
After implantation:
  - Dopants are NOT electrically active (interstitial positions)
  - Crystal is damaged (displaced Si atoms)
  - Must anneal to:
    1. Repair crystal damage
    2. Move dopants to substitutional lattice sites (activation)

Anneal types:
  Furnace anneal:   900-1100°C, 30-60 min (older nodes)
    - Good activation but significant diffusion (dopants move!)
    
  Rapid Thermal Anneal (RTA): 1000-1100°C, 1-10 seconds
    - Less diffusion than furnace
    - Standard for most modern processes
    
  Spike anneal:     1050-1100°C, ~1 second peak
    - Minimal diffusion, good activation
    
  Laser/Flash anneal: 1200-1300°C, ~milliseconds
    - Almost no diffusion, maximum activation
    - Surface-only heating → substrate stays cool
    - Used at advanced nodes for ultra-shallow junctions
```

### 5.4 Implants in CMOS Process

```text
Key implant steps:
  1. Well implant:       Deep, high-dose (forms n-well and p-well)
  2. Channel implant:    Sets threshold voltage (Vt adjust)
  3. Halo/pocket implant: Angled implant around source/drain to fight SCE
  4. Extension (LDD):    Shallow, lightly-doped source/drain extension
  5. Source/Drain (S/D):  Deep, heavy-dose (low resistance contacts)
  6. Contact implant:     Extra-heavy dose for ohmic contacts

At advanced nodes (FinFET):
  - Conformally doped fins require special techniques
  - Plasma doping (PLAD) for conformal doping of 3D structures
  - In-situ doping during epitaxial growth (replaces some implants)
```

---

## 6. Thin-Film Deposition

### 6.1 Chemical Vapor Deposition (CVD)

```ascii-graph
Principle: Gas-phase precursors react at hot wafer surface to form solid film

Types:
  LPCVD (Low Pressure CVD):
    - Pressure: 0.1-10 Torr, Temp: 500-900°C
    - Excellent uniformity and step coverage
    - Used for: polysilicon, Si3N4, SiO2
    - Batch processing (100+ wafers)

  PECVD (Plasma-Enhanced CVD):
    - Plasma provides energy → lower temperature (200-400°C)
    - Used for: SiO2, Si3N4, SiON, low-k dielectrics
    - Can deposit on temperature-sensitive substrates
    - Lower quality than LPCVD but adequate for many uses

  HDPCVD (High Density Plasma CVD):
    - Simultaneous deposition and sputtering
    - Excellent gap-fill capability (important for metal line gaps)
    - Used for: inter-metal dielectric (IMD) fill

  MOCVD (Metal-Organic CVD):
    - Uses metal-organic precursors
    - Used for: compound semiconductors (GaN, InP), high-k dielectrics
```

### 6.2 Physical Vapor Deposition (PVD / Sputtering)

Principle: Physical transfer of material from target to wafer

**DC/RF Magnetron Sputtering:**
   1. Ar gas ionized to form plasma
2. Ar+ ions accelerated toward target (cathode)
3. Target atoms knocked off (sputtered)
4. Sputtered atoms travel to wafer and deposit

Used for: Metal films (Al, Cu barrier/seed, Ti, TiN, Ta, TaN, W)

Advantages: Good adhesion, controlled composition, alloy deposition
Disadvantages: Poor step coverage (line-of-sight process)

**Ionized PVD (iPVD):**
   - Ionize sputtered atoms using secondary plasma
   - Electric field directs ions into trenches/vias
   - Better step coverage than conventional PVD
   - Used for: barrier/seed layers in high-aspect-ratio vias

### 6.3 Atomic Layer Deposition (ALD)

```ascii-graph
Principle: Self-limiting, layer-by-layer growth

Cycle:
  1. Pulse precursor A → reacts with surface, forms monolayer, self-limits
  2. Purge (remove excess A and byproducts)
  3. Pulse precursor B → reacts with A-covered surface, completes one layer
  4. Purge
  → Repeat for desired thickness

Growth rate: ~0.5-1.5 Å per cycle (VERY slow but VERY precise)

Used for:
  - High-k gate dielectric (HfO2): precursors = HfCl4 + H2O
  - Barrier layers: TiN, TaN
  - Spacer films
  - Any application needing atomic-scale thickness control and perfect conformality

Advantages:
  - Thickness control: ±0.1 nm
  - 100% conformal (coats every surface equally)
  - Excellent uniformity (< 1% across wafer)
  - Low temperature (100-400°C)

Disadvantages:
  - Very slow (~100-300 cycles/minute)
  - Expensive precursors
  - Throughput limited
```

### 6.4 Electrochemical Deposition (ECD)

```ascii-graph
Used for: Copper metallization (since 130nm Cu interconnect)

Dual Damascene copper process:
  1. Etch trenches and vias in dielectric
  2. Deposit barrier (TaN/Ta) by PVD — prevents Cu diffusion
  3. Deposit Cu seed layer by PVD
  4. Electroplate Cu to fill trenches/vias (ECD)
  5. CMP to remove excess Cu (planarize)

ECD copper plating:
  - Wafer is cathode in CuSO4 electrolyte
  - Cu²+ ions reduced at wafer surface → Cu deposit
  - Additives control fill behavior:
    Accelerators: speed up deposition at bottom of features (bottom-up fill)
    Suppressors: slow down deposition at top (prevent void formation)
    Levelers: smooth the surface

  Bottom-up fill is critical:
    Without it → voids form inside features → reliability failure
```

---

## 7. Chemical Mechanical Polishing (CMP)

### 7.1 Process

```text
Principle: Combination of chemical dissolution and mechanical abrasion

Setup:
  Wafer (face-down) pressed against rotating polishing pad
  Chemical slurry (abrasive particles + reactive chemicals) flows between
  Pad and slurry chemistry selected for target material

CMP applications:
  1. STI CMP: Planarize after trench fill (stop on Si3N4 liner)
  2. ILD CMP: Planarize inter-layer dielectric
  3. Metal CMP: Remove excess Cu after electroplating (damascene process)
  4. Poly CMP: Planarize polysilicon (gate formation)

Preston's equation:
  Removal Rate = Kp × P × V

  Kp: Preston coefficient (material/slurry dependent)
  P:  Applied pressure
  V:  Relative velocity (pad rotation + wafer rotation)
```

### 7.2 CMP Challenges

```ascii-graph
Dishing:
  - Soft metal (Cu) polished faster than surrounding dielectric
  - Creates recessed metal surface → increased resistance
  - Worse for wide metal lines
  - Mitigation: slotting wide metals, cheesing

Erosion:
  - In dense metal regions, dielectric between lines is over-polished
  - Both metal and dielectric thinned → performance variation
  - Mitigation: dummy metal fill to equalize density

Dishing + Erosion model:
  Metal thickness loss = f(line width, pattern density, overpolish time)
  
  Design rules:
    - Metal density: 20-80% per metal layer (enforced by DRC)
    - Maximum metal width: ~10-20 μm (to limit dishing)
    - Dummy fill: automatic insertion in empty regions
```

---

## 8. Complete CMOS Process Flow

### 8.1 Simplified Flow (28nm Planar CMOS)

| Step | Process | Purpose |
|---|---|---|
| 1 | STI formation | Isolate transistors |
| 2 | Well implants | Form n-well (for PMOS) and p-well (for NMOS) |
| 3 | Vt adjust implant | Set threshold voltage |
| 4 | Gate oxide growth | Thin oxide for gate dielectric (or high-k dep) |
| 5 | Poly-Si deposition | Gate electrode material |
| 6 | Gate patterning | Define gate shapes (most critical litho step) |
| 7 | Halo implant | Counter-doping for SCE control |
| 8 | LDD implant | Lightly-doped drain extension |
| 9 | Spacer formation | Si3N4 spacers on gate sidewalls |
| 10 | S/D implant | Heavy-dose source/drain |
| 11 | Activation anneal | Activate dopants, repair damage |
| 12 | Silicidation | NiSi on gate/S/D for low contact resistance |
| 13 | Contact etch | Open holes to S/D and gate |
| 14 | W plug fill | Tungsten fill in contacts |
| 15 | M1 deposition/etch | First metal layer (Cu damascene) |
| 16 | Repeat M2-MN | Additional metal layers |
| 17 | Passivation | Protective layer (Si3N4/SiO2) |
| 18 | Pad opening | Expose bond pads |

### 8.2 STI (Shallow Trench Isolation) Process

```ascii-graph
1. Grow pad oxide (~10nm SiO2)
2. Deposit Si3N4 (~100nm) — CMP stop layer
3. Pattern and etch trenches into Si (~300-500nm deep)
4. Liner oxidation (thin thermal oxide in trench)
5. Fill trench with CVD SiO2 (HDPCVD)
6. CMP — planarize, stop on Si3N4
7. Strip Si3N4 (hot H3PO4)
8. Strip pad oxide (HF)

Result: Oxide-filled trenches isolate active areas

Cross-section:
  ┌─────┐         ┌─────┐
  │ Si  │  SiO2   │ Si  │
  │(act)│ (trench)│(act)│
  └─────┘  fill   └─────┘
  ───────┴─────────┴───────  Si substrate
```

### 8.3 Gate-First vs Gate-Last (Replacement Metal Gate)

**Gate-First (traditional):**
   - Form gate stack → S/D implant → anneal
   - Problem: High-k/metal gate must survive S/D anneal (~1050°C)
   - This degrades work function and high-k quality

**Gate-Last (Replacement Metal Gate, RMG):**
   - Form dummy poly gate → S/D implant → anneal →
   - remove dummy gate → deposit high-k + metal gate

Advantage: High-k and metal gate not exposed to high temperature
Used at: 32nm (Intel) and all subsequent FinFET nodes

Process:
1. Dummy poly gate patterned (same as gate-first up to spacer)
2. S/D epitaxy + implant + anneal
3. Deposit ILD0, CMP to expose top of dummy gate
4. Etch out dummy poly (selective to spacer and substrate)
5. Remove dummy gate oxide
6. Deposit high-k (HfO2 by ALD, ~2nm)
7. Deposit work function metals (TiN/TiAl for NMOS/PMOS)
8. Fill gate trench with W or Al
9. CMP to planarize gate metal

---

## 9. FinFET and GAA Fabrication

### 9.1 Fin Formation

```ascii-graph
1. Pattern fins using self-aligned techniques (SADP/SAQP at 7nm)
   - Fin pitch: 25-30nm at 7nm node
   - Fin width: 6-7nm
   - Fin height: 40-50nm
   - Aspect ratio > 7:1

2. STI fill between fins (same as planar STI)

3. STI recess: etch back STI oxide to expose fin above isolation
   - Exposed fin height determines effective channel width
   - Very critical etch — uniformity determines Vt matching

Cross-section after fin formation:
     ┌──┐   ┌──┐   ┌──┐   ┌──┐
     │Fi│   │Fi│   │Fi│   │Fi│
     │n │   │n │   │n │   │n │
  ───┴──┴───┴──┴───┴──┴───┴──┴───  STI level
  ════════════════════════════════  Si substrate
```

### 9.2 Gate Wrap-Around

```ascii-graph
After gate dielectric and metal deposition, the gate material
conformally wraps around the fin:

  Cross-section (perpendicular to fin):
       ┌───────────┐
       │   Gate     │
       │  ┌─────┐  │
       │  │ Fin │  │    Gate wraps 3 sides
       │  │     │  │
       └──┤     ├──┘
          │     │
          └─────┘
         STI oxide

  Gate controls fin from left, right, and top
  → Excellent electrostatic control
  → Near-ideal subthreshold slope (~65 mV/dec)
```

### 9.3 Source/Drain Epitaxy

```ascii-graph
FinFET S/D is NOT formed by implant (fins are too narrow)
Instead: Selective Epitaxial Growth (SEG)

  1. Recess fins in S/D regions (etch down below gate level)
  2. Grow epitaxial S/D:
     NMOS: Si:P (phosphorus-doped silicon) — tensile stress → ↑ electron mobility
     PMOS: SiGe:B (boron-doped SiGe) — compressive stress → ↑ hole mobility
  3. Merge neighboring fins with epi growth → lower S/D resistance

  Cross-section (along fin):
          Gate
    S/D   │   │   S/D
  ┌─────┐ │   │ ┌─────┐
  │SiGe │ │ F │ │SiGe │    (PMOS example)
  │ epi │ │ i │ │ epi │
  │     │ │ n │ │     │
  └─────┘ └───┘ └─────┘

Strain engineering:
  SiGe in PMOS: Ge larger than Si → compressive strain on channel
  Si:P in NMOS: P smaller than Si → tensile strain on channel
  Strain → changes band structure → increases carrier mobility
  Mobility enhancement: ~30-50% for PMOS, ~10-20% for NMOS
```

### 9.4 GAA Nanosheet Fabrication

```ascii-graph
Transition from FinFET to Gate-All-Around at 3nm/2nm nodes:

1. Start with a Si/SiGe superlattice on SOI:
   - Alternating layers of Si and SiGe grown by epitaxy
   - Typically 3-5 pairs of Si (channel) and SiGe (sacrificial)
   - Each Si layer will become a nanosheet channel

2. Fin patterning:
   - Pattern vertical fins from the superlattice (same SAQP as FinFET)

3. Inner spacer formation:
   - Selective etch of SiGe recess in S/D regions
   - Deposit SiN inner spacer (isolates gate from S/D)
   - Critical for reducing gate-to-S/D capacitance

4. S/D epitaxy:
   - Grow SiGe:B (PMOS) or Si:P (NMOS) epitaxially on exposed Si layers
   - Merges vertically between sheets for lower S/D resistance

5. Channel release (key GAA step):
   - Selective wet etch removes ALL SiGe sacrificial layers
   - Leaves suspended Si nanosheets (channels) in free space
   - Etch selectivity SiGe:Si > 100:1 required

6. Gate formation (RMG process):
   - ALD of interfacial SiO2 layer (~0.5nm)
   - ALD of high-k dielectric (HfO2, ~1.5nm)
   - Deposit work function metal (TiN, TiAl) by ALD/CVD
   - Gate wraps ALL 4 sides of each nanosheet simultaneously

Cross-section after gate formation:
       ┌──────────────────────────────┐
       │          Gate Metal          │
       │  ┌────┐  ┌────┐  ┌────┐    │
       │  │Si  │  │Si  │  │Si  │    │  3 nanosheets
       │  │ns1 │  │ns2 │  │ns3 │    │  fully wrapped
       │  └────┘  └────┘  └────┘    │
       │          Gate Metal          │
       └──────────────────────────────┘

Nanosheet width modulation:
  - Unlike FinFET (quantized width), nanosheet width is set by lithography
  - Width range: ~15-50nm per sheet (depends on node)
  - Wider sheets → more drive current, but more gate capacitance
  - Enables per-cell optimization of drive strength

Samsung MBCFET (Multi-Bridge Channel FET) — 3nm:
  - Nanosheets connected at ends → "bridges" between S/D
  - Production since July 2022

TSMC N2 — GAA nanosheet:
  - Production 2025-2026
  - ~15% speed or ~30% power improvement over N3
  - Uses wider nanosheets for higher drive

Intel RibbonFET — Intel 18A (1.8nm):
  - Production 2025
  - Combined with PowerVia backside power delivery
```

### 9.5 CFET (Complementary FET) — Future Beyond 2nm

```ascii-graph
CFET: Stacked NMOS over PMOS (or vice versa) in a single vertical structure

  Traditional (FinFET/GAA):
    PMOS ──┐  (side by side)
    NMOS ──┘   → each needs its own footprint

  CFET:
    ┌──────┐
    │ PMOS │  ← Top nanosheet (p-type)
    ├──────┤
    │ NMOS │  ← Bottom nanosheet (n-type)
    └──────┘
    → same function in HALF the footprint

  Benefits:
    - ~1.5-2x density improvement over GAA nanosheet
    - Reduced parasitic capacitance (shorter S/D connections)
    - Enables continued scaling below 1nm node

  Challenges:
    - Thermal budget: NMOS and PMOS processed at different temps
    - N-type and P-type channel material optimization on same stack
    - Vertical alignment and connection between stacked devices
    - Self-heating: two active devices stacked doubles heat density

  Timeline:
    - Expected for A10/A7 nodes (~2030+)
    - Intel, TSMC, Samsung all researching CFET
    - Requires monolithic 3D integration techniques

  CFET + BSPDN is the projected end-of-scaling architecture:
    - CFET provides density
    - Backside power provides clean power delivery
    - Together they may extend Moore's Law to ~2035
```

---

## 10. BEOL (Back-End-of-Line) Processing

### 10.1 Metal Interconnect Stack

Advanced node metal stack (7nm example):

| Layer | Pitch | Material | Purpose |
|---|---|---|---|
| M1 | 28nm | Cu (CoAl) | Local connections |
| M2 | 28nm | Cu | Local connections |
| M3 | 40nm | Cu | Intermediate |
| M4 | 40nm | Cu | Intermediate |
| M5-M8 | 56nm | Cu | Semi-global |
| M9-M10 | 80nm | Cu | Semi-global |
| M11-M12 | 280nm | Cu | Global (power, clock) |
| M13 (AP) | 1.6μm | Al/Cu | Top metal (pad, inductor) |

Total: 12-15 metal layers at 7nm
  Pitch doubles roughly every 2-3 layers

### 10.2 Dual Damascene Process

```ascii-graph
Standard copper interconnect formation:

1. Deposit ILD (low-k dielectric, k ≈ 2.5-3.0)
2. Pattern via holes (litho + etch)
3. Pattern trenches (litho + etch) — must align to vias
4. Deposit barrier (TaN/Ta by PVD, ~2-5nm)
5. Deposit Cu seed layer (PVD, ~10-20nm)
6. Electroplate Cu (ECD, fill trenches + vias)
7. CMP (remove excess Cu, planarize)
8. Deposit etch stop layer (SiCN, ~10-30nm)
9. Repeat for next metal layer

Cross-section:
   ┌────────────────────────────┐
   │         Cu (M2)            │
   │  ┌──────────────────────┐  │
   │  │  barrier (TaN/Ta)    │  │  Trench
   │  │                      │  │
   └──┤         ┌────┤       ├──┘
      │  via    │    │  via  │
      │  (Cu)   │    │ (Cu)  │
      └────┬────┘    └───┬───┘
   ════════╧══════════════╧════   Etch stop
   ┌──────────────────────────┐
   │         Cu (M1)          │
   └──────────────────────────┘

Dual damascene: via and trench formed together → one Cu fill step
(vs single damascene: via first, fill, then trench, fill — more steps)
```

### 10.3 Low-k Dielectrics

```ascii-graph
Why low-k:
  RC delay ∝ R × C
  C ∝ k × (area/spacing)
  Lower k → lower C → lower delay + lower crosstalk + lower power

Dielectric constant evolution:
  SiO2:       k = 3.9  (traditional)
  FSG:        k = 3.5  (fluorinated silicate glass)
  SiCOH:      k = 2.7-3.0  (carbon-doped oxide, CDO)
  Porous SiCOH: k = 2.2-2.5  (pores reduce effective k)
  Air gap:    k → 1.0  (ultimate low-k, partial integration at 10nm)

Challenges:
  - Lower k → mechanically weaker (more porous)
  - CMP damage, etch damage, moisture absorption
  - Integration with Cu (adhesion, barrier integrity)
  - Reliability: time-dependent dielectric breakdown (TDDB)
```

---

## 11. Advanced Patterning Techniques

### 11.1 Double Patterning (LELE)

```ascii-graph
LELE = Litho-Etch-Litho-Etch

When minimum pitch < single-exposure resolution:
  Split the pattern into two masks, each with relaxed pitch

  Mask 1:   ║   ║   ║        (every other line)
  Mask 2:     ║   ║   ║      (remaining lines)
  Combined: ║ ║ ║ ║ ║ ║      (full density)

Process:
  1. Deposit hardmask
  2. Litho + etch with Mask 1 → pattern half the features
  3. Fill/planarize
  4. Litho + etch with Mask 2 → pattern remaining features
  5. Final pattern has 2× density of single exposure

Problem: Overlay error between two exposures → pitch variation
  Overlay budget: < 2-3nm (extremely tight!)
  
Coloring: Assigning features to Mask 1 or Mask 2
  - Must be consistent (no conflicts — adjacent features on different masks)
  - This is a graph coloring problem (2-colorable)
  - Odd cycles create coloring conflicts → require design rule changes
```

### 11.2 Self-Aligned Double Patterning (SADP)

```ascii-graph
SADP eliminates overlay error by using spacer deposition:

1. Pattern mandrels (wider pitch, easy litho)
2. Deposit conformal spacer (ALD Si3N4) on mandrel sidewalls
3. Remove mandrels (selective etch)
4. Spacers remain → pitch = half of mandrel pitch!

  Step 1: Mandrels       ╔═══╗     ╔═══╗     ╔═══╗
  Step 2: Spacers        ║╔═╗║     ║╔═╗║     ║╔═╗║
  Step 3: Remove mandrel  ║ ║       ║ ║       ║ ║
                          Spacers at half-pitch!

Advantages over LELE:
  - No overlay error (spacers self-aligned to mandrels)
  - Better pitch uniformity
  - Single litho step

Used at: 14nm, 10nm, 7nm for critical layers (M1, fins)
```

### 11.3 Self-Aligned Quadruple Patterning (SAQP)

```ascii-graph
SAQP = Two rounds of SADP → 4× density

1. Pattern mandrels at relaxed pitch P
2. First spacer deposition + mandrel removal → pitch P/2
3. Use spacers as new mandrels
4. Second spacer deposition + mandrel removal → pitch P/4

Used at: 7nm, 5nm for fin patterning
  Original mandrel pitch: ~100nm
  After SAQP: ~25nm fin pitch

Challenges:
  - Two spacer depositions + four etch steps → process complexity
  - Spacer thickness uniformity critical (determines final pitch)
  - Design restrictions (not all patterns achievable)
```

### 11.4 DTCO (Design-Technology Co-Optimization)

```ascii-graph
DTCO: Jointly optimizing process technology and circuit design to maximize
PPA (Power, Performance, Area) benefits at each node.

Key DTCO techniques at advanced nodes:

1. Track height reduction:
   7.5T → 6.5T → 6T → 5.5T → 5T
   Reduces standard cell height → higher density
   Cost: fewer routing tracks per cell, more congestion
   Enabled by: smaller contacts (V0), tighter M0 pitch

2. Fin depopulation / nanosheet width optimization:
   Remove fins (or narrow nanosheets) in non-critical cells
   Saves leakage power without impacting performance
   Requires library characterization for each variant

3. Diffusion breaks:
   AB (Always-ON break): physical gap between cells → larger but safe
   SDB (Single Diffusion Break): shared diffusion → 10-15% smaller cells
   NB (No Break / continuous diffusion): smallest, CPODE technique

4. Contact over active gate (COAG):
   Place gate contact directly over the gate poly
   Eliminates separate gate contact area → 5-10% cell width reduction

5. Buried Power Rail (BPR):
   VDD/VSS routed below transistor layer in buried metal lines
   Frees M0/M1 for signal routing
   Combined with backside power delivery for maximum benefit

6. Backside Power Delivery Network (BSPDN):
   Intel PowerVia (Intel 18A, 2025):
     - Wafer thinned to expose backside
     - Deep vias (TSV-like) connect frontside to backside power grid
     - Power rails removed from frontside → M0/M1 fully for signals
     - Up to 50% IR drop reduction, ~5-10% frequency improvement
   TSMC N2P Super Power Rail (SPR, 2026):
     - Similar concept, different integration approach
   Samsung SF2 (2025): BSPDN on 2nm GAA node
```

---

## 12. Yield, Defects, and DFM

### 12.1 Yield Models

```ascii-graph
Poisson yield model (simple):
  Y = exp(-D × A)
  
  D: defect density (defects/cm²)
  A: die area (cm²)
  Y: yield (fraction of good dies)

  Example: D = 0.1/cm², A = 100 mm² = 1 cm²
  Y = exp(-0.1) = 0.905 = 90.5%

  If D = 0.5/cm² (immature process):
  Y = exp(-0.5) = 0.607 = 60.7%  → significant yield loss

Murphy's yield model (more realistic — non-uniform defect distribution):
  Y = ((1 - exp(-D×A)) / (D×A))²

Defect density at advanced nodes:
  Mature process: D ≈ 0.05-0.1 /cm²
  New process (first year): D ≈ 0.5-2.0 /cm²
  This is why small dies yield better on new processes
  (A is smaller → less chance of defect per die)
```

### 12.2 Types of Defects

```text
Particle defects:
  - Dust, contamination landing on wafer
  - Causes shorts (bridging between lines) or opens
  - Clean room class: < 1 particle (>0.1μm) per m³

Pattern defects:
  - Litho errors: mis-focus, dose variation, overlay error
  - Etch residue, incomplete etch
  - Results in CD (critical dimension) variation

Parametric defects:
  - Vt variation (doping non-uniformity)
  - Oxide thickness variation
  - Contact resistance variation
  - May not cause functional failure but timing/power impact

Systematic defects:
  - Design-dependent (layout-specific weak points)
  - Narrow line ends, dense-isolated transitions
  - Addressed by DFM (Design for Manufacturability)
```

### 12.3 DFM (Design for Manufacturability)

DFM rules ensure designs are robust to process variation:

1. Recommended rules (beyond minimum DRC):
   - Prefer wider metals where possible
   - Use uniform pattern density
   - Avoid isolated narrow features
   - Minimize line-end extensions near other features

2. Via redundancy:
   - Double/triple vias wherever space permits
   - Single-cut via has ~0.01-0.1% fail rate
   - Double via: fail rate² → effectively zero
   - DRC may flag single-cut vias as DFM violations

3. Metal slotting/cheesing:
   - Wide metals must have slots cut in them
   - Prevents dishing during CMP
   - Reduces metal density variation

4. Dummy fill:
   - Insert non-functional metal shapes in empty regions
   - Ensures uniform metal density for CMP
   - Typically 20-80% density target per layer
   - Auto-generated by fill tools (Calibre, IC Validator)

5. Litho-friendly design:
   - Prefer manhattan (vertical/horizontal) routing
   - Avoid jogs shorter than minimum resolution
   - Use line-end extensions at tips
   - Respect SADP/LELE coloring constraints

---

## 13. Reliability Mechanisms

### 13.1 TDDB (Time-Dependent Dielectric Breakdown)

```text
Gate oxide degrades under electric field stress over time.
Time to breakdown follows:  t_BD ∝ exp(γ × (E_BD − E_ox))

  γ:  field acceleration factor (~1-3 decade·cm/MV)
  E_ox: operating oxide field

Design rule: maximum oxide field ≤ 3-4 MV/cm for 10-year lifetime.

At N5 with EOT = 0.9 nm and VDD = 0.7 V:
  E = 0.7 / (0.9 × 10⁻⁷) = 7.8 MV/cm — approaching the reliability limit.
This is why high-k dielectrics are essential (thicker physical layer at same EOT).
```

### 13.2 NBTI (Negative Bias Temperature Instability)

```text
PMOS threshold voltage increases (degrades) when gate is held at negative bias
(VGS = −VDD) at elevated temperature. Vth shift follows a power law:

  ΔVth ∝ (t_stress)^n,  where n ≈ 0.16-0.25

This causes timing degradation over the chip's lifetime.
Designers add 5-10% timing guardband for NBTI.
Recovery occurs when stress is removed (partial, not complete).
```

### 13.3 HCI (Hot Carrier Injection)

```text
High-energy carriers gain enough energy from the lateral electric field near
the drain to be injected into the gate oxide. Causes Vth shift and
transconductance degradation. Worse at higher VDD and shorter channels.

Mitigation:
  - LDD (lightly-doped drain) structures reduce peak lateral field
  - Lower VDD at advanced nodes naturally reduces HCI
  - Graded junction profiles
```

### 13.4 Electromigration (EM)

```text
Momentum transfer from current-carrying electrons to metal atoms causes
progressive void formation (open) or hillock formation (short).

Black's equation:  MTTF = A × J^(−n) × exp(Ea / kT)

  J:   current density
  n:   ≈ 1-2
  Ea:  ≈ 0.5-0.7 eV for Cu

Design rule: J_max ≈ 1-3 mA/μm for Cu interconnect at 105°C, 10-year lifetime.
Cu has ~100× better EM resistance than Al due to higher activation energy.
```

### 13.5 ESD (Electrostatic Discharge)

```text
Brief voltage spikes (HBM: 2 kV, CDM: 500V) can destroy thin gate oxides.
On-chip ESD protection circuits clamp these transients.
(Detailed ESD design covered in the dedicated ESD section.)
```

---

## 14. Basic Packaging

### 14.1 Wire Bonding

```verilog
25 μm Au (or Al, Cu) wire bonded from die pad to package leadframe.
  Bandwidth limit:   2-4 GHz (inductance limited)
  Inductance:        0.5-1 nH per bond
  Pad pitch:         ≥ 50 μm
  Advantages:        Low cost, mature, high yield
  Limitations:       Only peripheral I/O, inductance limits high-speed signals
```

### 14.2 Flip Chip (C4)

```text
Solder bumps (C4 = Controlled Collapse Chip Connection) on die surface.
Die is flipped and bonded face-down onto substrate.
  Bump pitch:        50-200 μm
  Inductance:        < 0.1 nH (much lower than wire bond)
  Advantages:        Better power delivery, higher I/O density, area-array pads,
                     shorter signal paths, better thermal path
  Used in:           All high-performance designs (CPU, GPU, SoC)
```

### 14.3 Wafer-Level Packaging (WLP)

```text
Packaging performed at wafer level before dicing.
  Fan-in WLP:   Package size ≈ die size (limited I/O count)
  Fan-out WLP:  Encapsulates die in mold compound, RDL routes I/O beyond die edges
                 Used for mobile SoCs (higher I/O density than fan-in)
  Advantages:   Direct board attach without interposer, lowest cost for high volume
  Used in:      Mobile/IoT devices, RF modules
```

See IC_Packaging.md for the full treatment of advanced packaging (2.5D, 3D, chiplets, TSVs).

---

## 15. Numbers to Memorize

| Parameter | Value |
|---|---|
| Si melting point | 1414°C |
| Dry O2 growth rate at 1000°C (linear regime) | ~2.5 nm/min |
| Wet O2 growth rate at 1000°C | ~20 nm/min |
| DUV wavelength (ArF immersion) | 193 nm |
| EUV wavelength | 13.5 nm |
| NA (ArF immersion) | 1.35 |
| NA (EUV current) | 0.33 |
| NA (High-NA EUV) | 0.55 |
| Resolution at EUV 0.33 NA, k1=0.3 | ~12 nm |
| Reticle size (standard EUV) | 858 mm² |
| Reticle size (High-NA EUV) | 429 mm² |
| Implant dose range | 10¹¹ – 10¹⁶ ions/cm² |
| CMP removal rate (oxide) | 100-300 nm/min |
| N5 minimum metal pitch (M1) | ~28 nm |
| N5 minimum metal pitch (M4) | ~40 nm |
| Defect density (N5 production) | ~0.1-0.3 /cm² |
| Yield (Poisson) | Y = exp(−D×A) |
| Yield (negative binomial) | Y = (1 + D×A/α)^(−α) |

---

