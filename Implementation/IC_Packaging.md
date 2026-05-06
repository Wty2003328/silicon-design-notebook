# IC Packaging and Advanced Packaging — Senior Engineer Deep Dive

## Table of Contents
1. Packaging Fundamentals
2. Wire Bonding
3. Flip Chip
4. Wafer-Level Packaging
5. 2.5D Integration (Interposer)
6. 3D IC Stacking
7. Chiplet Architecture
8. HBM (High Bandwidth Memory)
9. Die-to-Die Interfaces (UCIe, BoW)
10. Thermal and Power Delivery Challenges
11. Interview Q&A (20+ Questions)

---

## 1. Packaging Fundamentals

### 1.1 Why Packaging Matters

```
The chip (die) is a fragile piece of silicon (~0.1-0.8mm thick).
Packaging provides:
  1. Mechanical protection (encapsulation)
  2. Electrical connections (die → PCB)
  3. Heat dissipation path (die → heat sink)
  4. Signal integrity (controlled impedance, low inductance)
  5. Power delivery (low-resistance supply connections)
  6. Standardized form factor (BGA, QFP, etc.)
```

### 1.2 Package Types

```
Through-hole (legacy):
  DIP (Dual In-line Package): Through-hole pins
  PGA (Pin Grid Array): 2D array of pins on bottom

Surface mount:
  QFP (Quad Flat Package): Leads on four sides, fine pitch ~0.5mm
  BGA (Ball Grid Array): Solder balls on bottom in 2D array
    - Pitch: 0.4-1.27mm
    - More I/O than QFP for same footprint
    - Better thermal/electrical performance

Advanced:
  FCBGA (Flip Chip BGA): Die face-down on substrate, BGA to PCB
  PoP (Package on Package): Stack packages vertically (mobile)
  WLCSP (Wafer-Level CSP): Package = die size, solder bumps directly on die
  SiP (System in Package): Multiple dies in one package
  CoWoS, InFO, EMIB: Advanced multi-die packaging (see sections below)
```

### 1.3 Package Substrate

```
Package substrate: Multi-layer PCB between die and solder balls

  Die bumps → RDL → package substrate → BGA balls → PCB

  Substrate layers: 4-20 layers (advanced packages)
  Core: organic (most common) or coreless (thinner, better SI)
  
  Routing:
    L/S = 8/8 μm (advanced organic substrate, 2024)
    L/S = 2/2 μm (silicon interposer RDL)
  
  Via types:
    Through-via: connects all layers (legacy)
    Micro-via: blind/buried vias, laser-drilled (~25-50 μm diameter)
    Via-in-pad: via directly under bump → higher density
```

---

## 2. Wire Bonding

### 2.1 Process

```
Wire bonding: Thin metal wire connects die pad to package lead frame

  Die pad (Al or Cu) → Gold or Cu wire (20-25 μm Ø) → Lead frame

  Bond types:
    Ball bond: Formed at die pad (first bond)
      - Wire tip melted into ball by electric flame-off (EFO)
      - Ball pressed onto pad with heat + ultrasonic energy
    
    Stitch bond (wedge bond): Formed at lead frame (second bond)
      - Wire pressed and welded to lead frame

  Process: Ball-stitch bonding (most common)
    1. EFO forms ball at wire tip
    2. Capillary presses ball onto die pad (thermosonic: heat + US + force)
    3. Capillary lifts, pays out wire in loop
    4. Capillary presses wire onto lead frame (stitch bond)
    5. Wire clamped and broken
    6. Repeat for next pad
```

### 2.2 Wire Bond Characteristics

```
Materials:
  Gold wire:   Best bondability, most expensive, ~25μm Ø
  Copper wire:  95% cheaper than Au, harder to bond, requires forming gas
  Aluminum wire: Wedge bonding only, for power devices

Performance:
  Wire length:   1-5 mm typical
  Inductance:    ~1 nH/mm → significant at GHz frequencies!
  Resistance:    ~40-100 mΩ per wire
  Current:       ~100-300 mA per wire
  Frequency:     Limited to ~500 MHz-1 GHz (inductance bottleneck)

Limitations:
  - Peripheral I/O only (pads at die edge)
  - High inductance → poor SI for high-speed signals
  - Limited I/O count (perimeter-limited)
  - Sequential process → slow for high pin-count
  
  Still used for: Low-cost packages, power devices, memory (HBM is exception)
```

---

## 3. Flip Chip

### 3.1 C4 (Controlled Collapse Chip Connection)

```
Flip chip: Die flipped face-down, solder bumps connect directly to substrate

  Traditional C4 bumps:
    Material:  SnPb or Pb-free (SnAg, SnAgCu)
    Diameter:  80-100 μm
    Pitch:     150-200 μm
    I/O count: ~3,000-10,000 (area array)

  Process:
    1. Under-Bump Metallurgy (UBM) deposited on die pads
    2. Solder bumps formed (evaporation, electroplating, or stencil)
    3. Die flipped and aligned to substrate
    4. Reflow solder (peak ~250°C)
    5. Underfill (epoxy) injected between die and substrate
       - Distributes thermal stress → prevents bump fatigue

Advantages over wire bond:
  - Area array I/O (not just periphery) → much higher I/O count
  - Lower inductance (~0.1 nH per bump vs ~1 nH per wire)
  - Better thermal path (die face-down → heat conducts through bumps + substrate)
  - Higher frequency operation (lower parasitic L and R)
  - Shorter connections → less delay
```

### 3.2 Micro-Bumps

```
Micro-bumps (μbumps): For 2.5D/3D integration

  Diameter:  20-40 μm (vs C4: 80-100 μm)
  Pitch:     40-55 μm (vs C4: 150-200 μm)
  Material:  SnAg cap on Cu pillar
  I/O count: 10,000-100,000+

  Used for:
    Die-to-interposer (2.5D: CoWoS)
    Die-to-die (3D stacking)
    HBM-to-interposer

  Process:
    1. Cu pillar electroplated on die pad (~30-50 μm tall)
    2. SnAg solder cap on top (~10 μm)
    3. Bonding: thermocompression (TC) or mass reflow
    4. Underfill (capillary or pre-applied)

  Challenge: At < 40μm pitch, bridging risk increases
             Need very accurate alignment (±1-2 μm)
```

### 3.3 Copper Pillar Bumps

```
Cu pillar: Replaces traditional solder bump at fine pitch

  Structure: Cu pillar (tall, narrow) + thin solder cap

    ┌──────────┐
    │  Solder  │  ~10μm
    ├──────────┤
    │          │
    │ Cu pillar│  ~30-50μm
    │          │
    ├──────────┤
    │   UBM    │
    └──────────┘
       Die pad

  Advantages:
    - Cu has higher melting point → better EM resistance
    - Finer pitch possible (Cu doesn't spread like solder)
    - Better current handling
    - More compliant (taller pillar absorbs stress)

  Used: Standard for flip chip at 28nm and below
```

---

## 4. Wafer-Level Packaging

### 4.1 Fan-In WLP (WLCSP)

```
Fan-in: Package size = die size (or very close)

  Die with redistribution layer (RDL):
    RDL reroutes die pads from periphery to area array

  Solder balls placed directly on RDL → ready for PCB mounting

  ┌─────────────────────────────┐
  │         Die                 │
  │  ┌───┐  ┌───┐  ┌───┐      │
  │  │pad│→→│RDL│→→│ ● │ball  │
  │  └───┘  └───┘  └───┘      │
  └─────────────────────────────┘

  Advantages: Smallest form factor, lowest cost for small dies
  Limitation: I/O limited by die area (can't exceed die perimeter pitch)
  Used: Baseband, PMIC, RF front-end (mobile)
```

### 4.2 Fan-Out WLP (FOWLP)

```
Fan-out: Package larger than die → more I/O possible

  Process (eWLB):
    1. Known-good dies placed on carrier face-down
    2. Molding compound fills gaps between dies
    3. RDL fabricated on molded surface → extends beyond die boundary
    4. Solder balls placed on RDL (area array larger than die)

  Advantages:
    - More I/O than fan-in (extends beyond die edge)
    - No package substrate needed → thinner
    - Good thermal and electrical performance
    - Can integrate passives (inductors, caps) in RDL

  TSMC InFO (Integrated Fan-Out):
    - Used in Apple A-series (starting A10)
    - InFO-WLP: single die fan-out
    - InFO-PoP: fan-out with package-on-package (DRAM on top)
    - InFO-L (large): multi-die fan-out (emerging for chiplets)
```

---

## 5. 2.5D Integration (Interposer)

### 5.1 Silicon Interposer

```
2.5D: Multiple dies mounted side-by-side on a silicon interposer

                    ┌──────┐  ┌──────┐  ┌──────┐
                    │ Die A│  │ Die B│  │ HBM  │
                    └──┬───┘  └──┬───┘  └──┬───┘
                       │μbumps  │         │
  ═══════════════════╤═╧═══════╧═════════╧══════╤════
  │              Silicon Interposer              │
  │     RDL (fine-pitch wiring between dies)     │
  │              TSVs (through interposer)       │
  ═══════════════════╧══════════════════════════╧════
                       │C4 bumps
                  ┌────┴──────────────┐
                  │  Package Substrate │
                  └────┬──────────────┘
                       │BGA balls
                     PCB
```

```
Key features:
  RDL pitch:       0.4-2 μm L/S (much finer than organic substrate)
  TSV diameter:    5-10 μm
  TSV pitch:       40-50 μm
  Inter-die wiring: 10,000+ connections between adjacent dies
  Bandwidth:       Several TB/s between dies

TSMC CoWoS (Chip-on-Wafer-on-Substrate):
  - First: Xilinx Virtex-7 2000T (2011)
  - Used: NVIDIA H100/B100 (GPU + HBM), AMD MI300X
  - Interposer size: up to 2-3× reticle limit (CoWoS-L uses RDL bridge)

  CoWoS variants:
    CoWoS-S: Standard silicon interposer (≤ 1 reticle)
    CoWoS-R: RDL-only interposer (organic, no TSV)
    CoWoS-L: Large interposer (2+ reticles, uses local Si bridges)
```

### 5.2 Through-Silicon Via (TSV)

```
TSV: Vertical electrical connection through a silicon wafer/die

  Fabrication:
    1. Deep Reactive Ion Etch (DRIE) → high-aspect-ratio hole
    2. Insulating liner (SiO2, ~100nm) → prevents Cu-Si contact
    3. Barrier/seed layer (TaN/Cu by PVD)
    4. Cu fill by electroplating (ECD)
    5. CMP to planarize

  TSV types by insertion point:
    Via-first:  TSV formed before FEOL (transistors)
    Via-middle: TSV formed between FEOL and BEOL (most common for interposers)
    Via-last:   TSV formed after BEOL (allows standard wafer processing)

  TSV dimensions:
    Interposer TSV: diameter 5-10 μm, depth 50-100 μm
    3D IC TSV:      diameter 1-5 μm, depth 5-50 μm
    Aspect ratio:   typically 5:1 to 20:1

  Electrical properties:
    Resistance:   ~10-50 mΩ per TSV
    Capacitance:  ~10-50 fF per TSV
    Inductance:   ~10-50 pH per TSV
    Far superior to wire bonds (R: ~100 mΩ, L: ~1 nH)

  TSV challenges:
    - Keep-out zone: No active devices near TSV (stress, Cu contamination)
    - Thermo-mechanical stress: Cu and Si have different CTE
      Cu: 17 ppm/°C, Si: 2.6 ppm/°C → stress during thermal cycling
    - Wafer thinning required for via-last (200-50 μm)
    - Cost: Si interposer adds $100-500+ per unit
```

### 5.3 Organic Interposer and Bridge

```
Alternatives to expensive silicon interposer:

Intel EMIB (Embedded Multi-die Interconnect Bridge):
  - Small silicon bridge embedded in organic substrate
  - Only where die-to-die connection is needed (not full interposer)
  - Bridge: ~4-8mm × 4-8mm, provides fine-pitch routing
  - Rest of substrate is standard organic (cheap)
  - Used in: Intel Ponte Vecchio, Sapphire Rapids

  ┌──────┐    ┌──────┐
  │ Die A│    │ Die B│
  └──┬───┘    └──┬───┘
     │            │
  ═══╧════════════╧═══════════  Organic substrate
       ┌──────────┐
       │  Si EMIB │              Embedded bridge
       │  bridge  │              (fine-pitch routing)
       └──────────┘
  ═══════════════════════════
         BGA balls

Advantages: Lower cost than full Si interposer
Disadvantages: Limited bridge area, fewer die-to-die connections
```

---

## 6. 3D IC Stacking

### 6.1 Die-to-Die Stacking

```
3D IC: Dies stacked vertically, connected by TSVs or hybrid bonding

  ┌──────────────┐
  │   Top Die    │  (e.g., SRAM cache, memory)
  │    TSVs ↕↕↕  │
  └──────┬───────┘
  ┌──────┴───────┐
  │  Bottom Die  │  (e.g., logic, processor)
  │              │
  └──────┬───────┘
         │μbumps
    ═════╧═══════  Package substrate

Benefits:
  - Shortest connections between dies → lowest power, highest bandwidth
  - Heterogeneous integration (logic + memory, analog + digital)
  - Smaller footprint than side-by-side
  - Memory-on-logic: processor accesses wide memory bus (1024+ bits)

Challenges:
  - Thermal: Top die has poor heat path (through bottom die)
  - Known-Good-Die (KGD): Must test each die before stacking
  - Yield: If any die in stack is bad → entire stack is wasted
  - Design complexity: Thermal, power delivery, signal integrity in 3D
```

### 6.2 Hybrid Bonding

```
Hybrid bonding: Cu-Cu direct bonding (no solder, no bumps)

  Process:
    1. CMP die surface (Cu pads + dielectric perfectly flat)
    2. Align and bond at room temperature (dielectric-dielectric bond)
    3. Anneal at 200-300°C → Cu pads expand and fuse (Cu-Cu bond)

  Pitch: 1-10 μm (vs μbumps: 40-55 μm) → 100× more connections!
  Density: > 1 million bonds per mm²

  Revolutionary because:
    - Eliminates solder bumps → much finer pitch possible
    - Lower resistance (Cu-Cu vs Cu-SnAg-Cu)
    - Higher bandwidth per mm² of interface

  Used in:
    - AMD V-Cache (3D stacked SRAM on top of CCD)
    - Image sensors (Sony: pixel + logic stacking)
    - Future: logic-on-logic stacking for chiplets

  Variants:
    Wafer-to-Wafer (W2W): Both wafers bonded, then diced
      - Highest alignment accuracy (< 200nm)
      - Both wafers must have same size dies (wasteful)
    
    Die-to-Wafer (D2W): Tested dies bonded to wafer
      - Allows mixed die sizes
      - KGD selection improves effective yield
      - Alignment slightly worse (~1 μm)
```

---

## 7. Chiplet Architecture

### 7.1 Why Chiplets

```
Monolithic large die problems:
  1. Yield: Y = exp(-D*A) → large die = low yield = expensive
  2. Design cost: Full-chip reticle ~$100M+ at 3nm
  3. Reticle limit: Maximum die size ~800mm² (limited by lithography field)
  4. Inflexibility: Can't mix process nodes (logic: 3nm, I/O: 7nm, analog: 28nm)

Chiplet solution:
  Break SoC into smaller dies ("chiplets"), each optimized separately,
  then connect them in an advanced package.

  Benefits:
    - Higher effective yield (small dies yield better)
    - Mix process nodes (logic on 3nm, SerDes on 5nm, HBM on DRAM process)
    - Design reuse (same I/O chiplet across product family)
    - Modular scaling (more compute chiplets = more performance)
    - Smaller NRE per chiplet

  Examples:
    AMD EPYC (Zen): 8 CCD chiplets (5nm) + 1 IOD (6nm)
    Intel Ponte Vecchio: 47 tiles across 5 process nodes
    Apple M1 Ultra: 2 × M1 Max connected via UltraFusion
```

### 7.2 Chiplet Interconnect Standards

```
UCIe (Universal Chiplet Interconnect Express):
  - Open standard (1.0: March 2022, 1.1: August 2023, 2.0: 2025)
  - Defines physical layer, die-to-die link layer, protocol layer
  
  UCIe Standard Package (organic substrate):
    Bump pitch: 100-130 μm
    Data rate:  4-32 GT/s per lane
    Bandwidth:  28-224 Gbps per mm of die edge
    Latency:    ~2-5 ns (link layer)

  UCIe Advanced Package (silicon bridge/interposer):
    Bump pitch: 25-55 μm
    Data rate:  4-32 GT/s per lane
    Bandwidth:  165-1317 Gbps per mm of die edge
    Latency:    ~2 ns

  Protocol adapters: CXL, PCIe, AXI → protocol-agnostic transport

BoW (Bunch of Wires):
  - Simpler: parallel bus, no encoding/decoding
  - Lower latency (~1 ns)
  - Higher bandwidth density but shorter reach
  - Used for memory-to-logic interfaces

Comparison:
  | Feature       | UCIe     | BoW      | HBM PHY   |
  |---------------|----------|----------|-----------|
  | Standardized  | Yes      | Limited  | JEDEC     |
  | Latency       | 2-5 ns   | ~1 ns    | ~5-10 ns  |
  | BW density    | High     | Highest  | High      |
  | Reach         | 5-25 mm  | < 5 mm   | < 5 mm    |
  | Protocol      | Any      | Raw      | Memory    |
```

---

## 8. HBM (High Bandwidth Memory)

### 8.1 HBM Architecture

```
HBM: DRAM dies stacked on top of each other, connected via TSVs

  Stack:
    ┌──────────────┐
    │   DRAM die 7 │  (HBM3: 8-12 dies)
    │   TSVs ↕↕↕   │
    ├──────────────┤
    │   DRAM die 6 │
    │   TSVs ↕↕↕   │
    ├──────────────┤
    │     ...      │
    ├──────────────┤
    │   DRAM die 0 │
    │   TSVs ↕↕↕   │
    ├──────────────┤
    │  Base Logic  │  (test, repair, interface logic)
    └──────┬───────┘
           │ μbumps
      Si Interposer → connected to SoC/GPU die

  Interface width: 1024 bits (per stack!)
  Compare: DDR5 = 64 bits, GDDR6 = 32 bits

  HBM generations:
    | Gen  | Year | BW/stack | Capacity | Layers | Data rate |
    |------|------|----------|----------|--------|-----------|
    | HBM  | 2013 | 128 GB/s | 1 GB     | 4      | 1 Gbps    |
    | HBM2 | 2016 | 256 GB/s | 8 GB     | 4-8    | 2 Gbps    |
    | HBM2E| 2018 | 460 GB/s | 16 GB    | 8      | 3.6 Gbps  |
    | HBM3 | 2022 | 819 GB/s | 24 GB    | 8-12   | 6.4 Gbps  |
    | HBM3E| 2024 | 1.2 TB/s | 36 GB    | 8-12   | 9.6 Gbps  |
    | HBM4 | 2025 | 1.6 TB/s | 48 GB    | 12-16  | 6.4 Gbps* |

  * HBM4: wider interface (2048 bits) rather than faster data rate
```

### 8.2 HBM Integration

```
HBM is ALWAYS mounted on an interposer (2.5D):

  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
  │ HBM  │  │ HBM  │  │ GPU  │  │ HBM  │  │ HBM  │
  │stack1│  │stack2│  │ SoC  │  │stack3│  │stack4│
  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘
     │         │         │         │         │
  ═══╧═════════╧═════════╧═════════╧═════════╧═══
  │              Silicon Interposer              │
  ═══════════════════════════════════════════════

  NVIDIA H100: 1 GPU die + 6 HBM3 stacks on CoWoS
  Total HBM bandwidth: ~5 TB/s
  Interposer size: ~2500 mm² (larger than the GPU die itself!)
```

---

## 9. Die-to-Die Interfaces

### 9.1 UCIe Protocol Stack

```
UCIe protocol layers:

  Application Layer:  CXL, PCIe, AXI, custom
       │
  Protocol Layer:    Flit packing, credit-based flow control
       │
  Die-to-Die Link:   Training, retry, CRC error detection
       │
  Physical Layer:    Serialization, clock forwarding, electrical signaling
       │
  Bump Interface:    C4 (standard pkg) or μbump (advanced pkg)

Physical layer details:
  - Source-synchronous clocking (forwarded clock)
  - Single-ended signaling (not differential — saves bumps)
  - Module: 16 data lanes + 2 clock lanes + sideband
  - Standard module width: ~200 μm of die edge
  - Multiple modules per die edge → scale bandwidth

Link layer:
  - CRC-based error detection
  - Retry: replay last flit on CRC error
  - Credit-based flow control (prevents overflow)
  - Flit: 256 bits (standard) or 68 bits (latency-optimized)
```

### 9.2 Power and Signal Integrity in Die-to-Die

```
Die-to-die signaling challenges:
  1. Low voltage swing: ~0.4V (saves power) → noise-sensitive
  2. Crosstalk: dense bump array → significant coupling
  3. Impedance matching: bump + trace must match PHY impedance
  4. Power delivery: each die needs its own power bumps
  5. Clock distribution: forwarded clock must have low jitter

Power consumption:
  UCIe: ~0.5 pJ/bit (advanced package)
  Compare: PCIe Gen5: ~5 pJ/bit, DDR5: ~8 pJ/bit
  Die-to-die is 10-20× more energy-efficient than board-level I/O
```

---

## 10. Thermal and Power Delivery Challenges

### 10.1 Thermal in Multi-Die Packages

```
2.5D thermal challenge:
  - Multiple high-power dies on one interposer
  - Total power can exceed 1000W (GPU + HBMs)
  - Heat flux: 50-100 W/cm² per die
  - Interposer acts as thermal spreader (Si has good conductivity)

3D thermal challenge (much worse):
  - Top die is thermally insulated by bottom die
  - Heat must conduct through silicon, bonding interface, TSVs
  - TSVs help: Cu has 400 W/m·K (Si: 150 W/m·K)
  - But TSV density is limited → thermal resistance between dies

  Thermal resistance stack (3D):
    T_top_die = T_ambient + P_total × (Θ_top + Θ_bonding + Θ_bottom + Θ_package)
    
    Θ_bonding: 0.01-0.1 °C·mm²/W (hybrid bonding < μbumps < underfill)

Solutions:
  - Microfluidic cooling (liquid between die layers — research stage)
  - Thermal TSVs (dummy TSVs for heat conduction)
  - Power-aware floorplanning (spread hotspots across layers)
  - Active cooling (thermoelectric coolers)
```

### 10.2 Power Delivery in Multi-Die

```
Challenge: Deliver clean power to each die through package/interposer

  Power path: VRM → PCB → BGA → Substrate → Interposer → Die
  
  Each stage adds resistance and inductance:
    PCB plane:    ~1 mΩ
    BGA balls:    ~5-10 mΩ (hundreds in parallel)
    Substrate:    ~1-5 mΩ
    C4 bumps:     ~10-20 mΩ (hundreds in parallel)
    Interposer:   ~1-5 mΩ
    μbumps:       ~5-10 mΩ

  Total PDN impedance: must be < target_impedance
    Z_target = ΔV_allowed / I_max
    
    Example: ΔV = 35 mV, I_max = 200A
    Z_target = 0.175 mΩ (extremely low!)
    
    Requires massive parallelism in bumps and vias

Decoupling strategy (multi-level):
  Level    | Location       | Effective frequency range
  ---------|----------------|-------------------------
  On-die   | MOS caps       | > 1 GHz
  On-pkg   | MIM/MOS cap    | 100 MHz - 1 GHz
  On-board | MLCC caps      | 1 MHz - 100 MHz
  On-board | Bulk caps      | < 1 MHz
```

---

## 11. Interview Q&A

**Q1: Compare wire bonding and flip chip. When would you choose each?**

Wire bonding is cheaper, peripheral-only I/O, higher inductance (~1 nH/wire), limited
to ~500 pins. Flip chip has area-array I/O, lower inductance (~0.1 nH/bump), supports
10,000+ connections, better thermal (die face-down). Choose wire bond for low-cost,
low-pin-count packages (sensors, microcontrollers). Choose flip chip for high-performance
(GPUs, CPUs, SoCs) or high-pin-count applications.

**Q2: What is a TSV? What are its key parameters?**

A Through-Silicon Via is a vertical copper connection through a silicon wafer. Key
parameters: diameter (5-10 μm interposer, 1-5 μm 3D IC), depth (50-100 μm interposer,
5-50 μm 3D), aspect ratio (5:1 to 20:1), resistance (~10-50 mΩ), capacitance (~10-50 fF),
inductance (~10-50 pH). TSVs enable vertical integration in 2.5D/3D IC packaging with
much better electrical performance than wire bonds.

**Q3: What is CoWoS and how does it work?**

TSMC CoWoS (Chip-on-Wafer-on-Substrate) is a 2.5D packaging technology. Multiple dies
(e.g., GPU + HBM stacks) are mounted on a silicon interposer with fine-pitch RDL (0.4-2μm
L/S) for die-to-die connections, and TSVs for vertical connection to the package substrate
below. CoWoS-S uses a standard silicon interposer, CoWoS-R uses RDL-only (organic),
and CoWoS-L uses a larger interposer with embedded silicon bridges.

**Q4: Why are chiplets becoming popular? What are the trade-offs?**

Chiplets improve effective yield (small dies yield better than one large die), enable
mixing process nodes (3nm logic + 7nm I/O), reduce design cost (reuse chiplets across
products), and overcome the reticle size limit. Trade-offs: die-to-die interface adds
latency (2-5 ns), bandwidth is lower than on-chip wires, packaging cost increases,
design complexity (multi-die floor planning, power delivery, thermal).

**Q5: Explain HBM and why it's critical for AI accelerators.**

HBM stacks 8-12 DRAM dies vertically using TSVs, providing a 1024-bit wide interface
(vs DDR5's 64 bits). This gives enormous bandwidth (>1 TB/s per stack). AI workloads
are memory-bandwidth limited — transformers need to move massive weight matrices
between memory and compute. HBM provides 5-10× the bandwidth of GDDR6 in a compact
form factor, mounted on the same interposer as the GPU/accelerator die.

**Q6: What is hybrid bonding and why is it revolutionary?**

Hybrid bonding directly connects Cu pads between two dies without solder bumps. At 1-10 μm
pitch (vs 40-55 μm for μbumps), it enables 100× more connections per area. This gives
unprecedented inter-die bandwidth and lowest latency. Used in AMD V-Cache (3D SRAM cache),
Sony image sensors, and planned for future chiplet-to-chiplet connections. The key
challenge is surface preparation — requires atomic-level flatness for reliable bonding.

**Q7: What is UCIe and how does it compare to proprietary die-to-die links?**

UCIe is an open standard for chiplet interconnect, defining physical, link, and protocol
layers. It supports both standard package (100-130 μm pitch) and advanced package
(25-55 μm pitch). Benefits: interoperability between vendors' chiplets, standard
verification methodology. Compared to proprietary links (AMD Infinity Fabric, Apple
UltraFusion): UCIe may have slightly higher overhead but enables a chiplet ecosystem
where third-party chiplets can be integrated.

**Q8: What are the thermal challenges in 3D IC stacking?**

In 3D stacking, the top die's heat must conduct through the bottom die to reach the
heat sink. This creates a higher thermal resistance path. The bonding interface (solder,
underfill, or hybrid bond) adds thermal resistance. Result: top die runs hotter, which
increases leakage and reduces reliability. Solutions: thermal TSVs (Cu-filled vias for
heat conduction), power-aware floorplanning, splitting high-power blocks across layers,
and advanced cooling (microfluidic channels).

**Q9: What is the yield advantage of chiplets?**

For Poisson defect model: Y = exp(-D×A). A monolithic 500mm² die at D=0.1/cm² yields
60.7%. Two 250mm² chiplets each yield 77.9%, and the combined good-pair probability
is 77.9%² = 60.7% — same if you need both to work. BUT: with testing, you only combine
known-good dies (KGD), so yield = Y_chiplet × (1 - test_escape_rate) ≈ 77.9%. The real
advantage grows with more chiplets and higher defect density.

**Q10: How does power delivery work in a multi-die package?**

Power flows: VRM → PCB → BGA balls → package substrate → C4 bumps → interposer (if 2.5D)
→ μbumps → die. Each die has its own power domain (possibly different voltages). The
power delivery network (PDN) impedance must be extremely low (< 0.2 mΩ for 200A at
35mV drop). This requires massive parallelism in bumps and vias, multi-level decoupling
(on-die MOS caps, on-package MIM caps, on-board MLCCs), and careful PDN design with
full-wave electromagnetic simulation.

**Q11: What is the difference between 2.5D and 3D IC?**

2.5D: Dies placed side-by-side on an interposer (silicon or organic). Dies don't stack
vertically. Inter-die connections through interposer RDL. Example: GPU + HBM on CoWoS.
3D: Dies stacked vertically, connected by TSVs or hybrid bonds through the silicon.
Much higher connection density but worse thermal and more complex. Example: AMD V-Cache,
HBM stacks. Many "3D" products are actually 2.5D with 3D memory stacks.

**Q12: What is fan-out packaging and when is it used?**

Fan-out WLP uses a reconstituted wafer where dies are embedded in mold compound with an
RDL that extends beyond the die boundary. This allows more I/O than the die area alone
would permit. TSMC InFO is a leading fan-out technology, used in Apple's A-series chips.
Fan-out is cheaper than silicon interposer for applications that don't need the extreme
die-to-die bandwidth of CoWoS. It's ideal for mobile SoCs and mid-range packages.

**Q13: What challenges exist at <10μm hybrid bonding pitch?**

At sub-10μm pitch: (1) Alignment accuracy must be <500nm (extremely tight for D2W).
(2) Surface cleanliness — a single particle can cause bond failure. (3) Pad size is so
small that Cu grain structure matters for bond quality. (4) Thermal expansion mismatch
can cause misalignment during anneal. (5) Testing at this density is challenging — can't
probe individual bonds. (6) Design rules for keep-out and dummy patterns become complex.
