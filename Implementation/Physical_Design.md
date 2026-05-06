# Physical Design (PnR) — Senior Engineer Deep Dive

> Target audience: Engineers preparing for senior-level interviews at Apple, NVIDIA, AMD, Intel, Qualcomm.
> Covers the complete PnR flow with quantitative depth, RTL/ASCII diagrams, and real design trade-offs.

---

## Table of Contents

1. [ASIC Physical Design Flow Overview](#1-asic-physical-design-flow-overview)
2. [Floorplanning](#2-floorplanning)
3. [Placement](#3-placement)
4. [Clock Tree Synthesis (CTS)](#4-clock-tree-synthesis-cts)
5. [Routing](#5-routing)
6. [Physical Verification (Signoff)](#6-physical-verification-signoff)
7. [Timing Signoff](#7-timing-signoff)
8. [Advanced Topics](#8-advanced-topics)
9. [Interview Q&A](#9-interview-qa)

---

## 1. ASIC Physical Design Flow Overview

### 1.1 Complete Flow Diagram

```
                          ASIC Physical Design Flow
  ============================================================================

  Synthesized Netlist (.v)  +  SDC Constraints  +  LEF/Tech Files  +  Libraries
                                     |
                                     v
                         +------------------------+
                         |     FLOORPLANNING      |
                         | - Die size estimation  |
                         | - Macro placement      |
                         | - Power grid planning  |
                         | - IO/Pin placement     |
                         +------------------------+
                                     |
                                     v
                         +------------------------+
                         |      POWER PLAN        |
                         | - VDD/VSS rings        |
                         | - Power stripes/mesh   |
                         | - Via stacks           |
                         | - Decap insertion      |
                         +------------------------+
                                     |
                                     v
                         +------------------------+
                         |      PLACEMENT         |
                         | - Global placement     |
                         | - Legalization         |
                         | - Optimization         |
                         | - Scan reorder         |
                         +------------------------+
                                     |
                                     v
                         +------------------------+
                         |   CLOCK TREE SYNTHESIS |
                         | - Tree construction    |
                         | - Buffer insertion     |
                         | - Skew balancing       |
                         | - Post-CTS opt         |
                         +------------------------+
                                     |
                                     v
                         +------------------------+
                         |       ROUTING          |
                         | - Global routing       |
                         | - Track assignment     |
                         | - Detail routing       |
                         | - Post-route opt       |
                         +------------------------+
                                     |
                                     v
                         +------------------------+
                         |    CHIP FINISHING      |
                         | - Filler cell insert   |
                         | - Metal fill (dummy)   |
                         | - Via optimization     |
                         +------------------------+
                                     |
                                     v
                         +------------------------+
                         |   SIGNOFF CHECKS       |
                         | - STA (PrimeTime)      |
                         | - DRC (Calibre/ICV)    |
                         | - LVS (Calibre/ICV)    |
                         | - IR drop (RedHawk)    |
                         | - EM check             |
                         +------------------------+
                                     |
                                     v
                               GDSII / OASIS
                             (to foundry tapeout)
```

### 1.2 Tools Landscape

| Stage           | Synopsys             | Cadence            | Siemens (Mentor)  |
|-----------------|----------------------|--------------------|-------------------|
| Place & Route   | ICC2 / Fusion Compiler | Innovus           | Aprisa            |
| STA             | PrimeTime            | Tempus             | —                 |
| Extraction      | StarRC               | Quantus            | xCalibrate        |
| IR Drop / Power | RedHawk / PTPX       | Voltus             | —                 |
| DRC/LVS         | ICV                  | Pegasus/PVS        | Calibre           |
| CTS             | (inside ICC2/FC)     | (inside Innovus)   | —                 |

**ICC2 vs Innovus — Real-World Comparison:**
- ICC2 / Fusion Compiler: tighter integration with Design Compiler (synthesis), unified data model, stronger in timing-driven flow, better for high-frequency designs (CPU cores).
- Innovus: known for superior congestion handling, faster runtime for large flat designs, GigaPlace engine is industry-leading for placement quality.
- At 5nm/3nm: both tools produce comparable results; the differentiator is usually recipe maturity and foundry-tool certification.

### 1.3 PPA Trade-offs at Each Stage

| Stage        | Performance Lever                 | Power Lever                     | Area Lever                    |
|--------------|-----------------------------------|---------------------------------|-------------------------------|
| Floorplan    | Short wire paths for critical     | Power grid robustness           | Utilization target            |
| Placement    | Timing-driven placement           | Activity-driven placement       | Minimize whitespace           |
| CTS          | Low insertion delay               | Fewer CTS buffers, gating       | CTS cell count                |
| Routing      | Minimize detour, SI clean         | Shorter wires = less dynamic pwr| Track utilization efficiency   |
| Signoff      | Multi-corner closure              | Leakage optimization, DVFS     | Fill density, via arrays      |

---

## 2. Floorplanning

### 2.1 Die Size Estimation

The most fundamental calculation in physical design:

```
Core Area = Total Standard Cell Area / Target Utilization

Where:
  Total Standard Cell Area = sum of all cell areas from the synthesized netlist
  Target Utilization = 60-80% (technology and design dependent)
```

**Numerical Example:**

```
Given:
  - Synthesized netlist has 2M standard cells
  - Average cell area = 0.5 um^2 (at 7nm)
  - Total standard cell area = 2M * 0.5 = 1.0 mm^2
  - 4 SRAM macros, each 0.2 mm^2 = 0.8 mm^2
  - Target utilization = 70%

Core Area = (1.0 mm^2) / 0.70 + 0.8 mm^2 = 1.43 + 0.8 = 2.23 mm^2

Die Area = Core Area + IO Ring Area
  IO ring width ~ 100-200 um on each side (bump pitch dependent)
  If core is 1.49 mm x 1.49 mm (square, AR=1):
  Die = (1.49 + 0.2 + 0.2) x (1.49 + 0.2 + 0.2) = 1.89 x 1.89 = 3.57 mm^2

Seal ring, scribe line add another ~50 um per side.
```

**Why utilization is 60-80% and not higher:**
- Below 60%: wasting silicon area, longer wires, more power
- 60-70%: comfortable routing, easy timing closure, good for high-performance
- 70-80%: standard for moderate-frequency designs
- Above 80%: severe routing congestion, timing closure becomes very difficult, DRC issues
- Above 85%: nearly impossible to close timing without significant architectural changes

### 2.2 Aspect Ratio

```
Aspect Ratio (AR) = Height / Width

Ideal AR = 1.0 (square die)
Acceptable range: 0.7 to 1.4
```

**Impact of non-unity AR:**
- AR >> 1 (tall, narrow): horizontal routing tracks are short, vertical routing becomes congested, IR drop worsens along the long dimension
- AR << 1 (wide, short): opposite problem
- Non-square dies also complicate package assembly and heat dissipation

### 2.3 Core Area vs Die Area

```
  +--------------------------------------------------+
  |  Scribe Line / Dicing Area                       |
  |  +--------------------------------------------+  |
  |  |  Seal Ring (guard ring, moisture barrier)   |  |
  |  |  +--------------------------------------+  |  |
  |  |  |  IO Ring (IO pads/bumps, ESD cells)  |  |  |
  |  |  |  +------------------------------+   |  |  |
  |  |  |  |  Core Area                    |   |  |  |
  |  |  |  |                               |   |  |  |
  |  |  |  |  +-------+     +---------+   |   |  |  |
  |  |  |  |  | SRAM  |     |  SRAM   |   |   |  |  |
  |  |  |  |  | Macro |     |  Macro  |   |   |  |  |
  |  |  |  |  +-------+     +---------+   |   |  |  |
  |  |  |  |                               |   |  |  |
  |  |  |  |  [Standard Cell Rows]         |   |  |  |
  |  |  |  |  |||||||||||||||||||||||||||   |   |  |  |
  |  |  |  |  |||||||||||||||||||||||||||   |   |  |  |
  |  |  |  |  |||||||||||||||||||||||||||   |   |  |  |
  |  |  |  |                               |   |  |  |
  |  |  |  +------------------------------+   |  |  |
  |  |  |  Corner Cells (at 4 corners)        |  |  |
  |  |  +--------------------------------------+  |  |
  |  +--------------------------------------------+  |
  +--------------------------------------------------+
```

- **Corner cells**: connect the IO ring power/ground at corners, ensure continuous ESD protection path.
- **Seal ring**: metal stack ring preventing moisture ingress into the die, mandatory foundry requirement.
- **IO ring**: contains IO pads (wire bond) or RDL/bump pads (flip-chip), ESD protection cells, level shifters.

### 2.4 Macro Placement Strategy

**General principles:**
1. Place macros at periphery (edges/corners) of the core to maximize contiguous standard cell area
2. Leave adequate channels between macros for routing (minimum 10-20 um at 7nm, technology-dependent)
3. Orient macro pins facing the standard cell area (pin accessibility)
4. Group macros that share data buses to minimize wirelength
5. Consider power grid continuity — macros should not block critical VDD/VSS stripes

**Fly-line analysis:**
Before finalizing macro positions, check fly-lines (virtual connections from netlist) to estimate wire congestion:

```
  Fly-Line Analysis (before macro placement optimization)
  
  +--------+                    +--------+
  |        |~~~~~~~~~~~~~~~~~~~~|        |
  | Macro  |~~~~~~~~~~~~~~~~~~~~| Macro  |    ~~~~ = fly-lines (logical connections)
  |   A    |~~~~~~~~~~~~~~~~~~~~|   B    |    Many fly-lines = place closer together
  |        |~~~~~~~~~~~~~~~~~~~~|        |
  +--------+                    +--------+
        |~~~~~
        |~~~~~
  +--------+
  | Macro  |
  |   C    |
  +--------+
  
  After optimization: move A and B adjacent, C near A
```

**Macro orientation:**
- Most macros have pins on specific sides (e.g., address/data pins on one side, power pins on another)
- Rotate macro (R0, R90, R180, R270, MX, MY) so that signal pins face the logic that drives/receives them
- SRAM: typically place with data pins facing standard cell rows, address pins on the side

### 2.5 Power Planning

**Power grid structure:**

```
  Top-level power grid (M9-M10 or top metals):
  
  VDD ========================================== (horizontal strap, M10)
  VSS ========================================== 
  VDD ==========================================
  VSS ==========================================
  
  |  |  |  |  |  |  |  |  |  |  |  |  |  |  |  (vertical straps, M9)
  V  G  V  G  V  G  V  G  V  G  V  G  V  G  V
  D  N  D  N  D  N  D  N  D  N  D  N  D  N  D
  D  D  D  D  D  D  D  D  D  D  D  D  D  D  D
  
  These connect down through via stacks to lower-level power rails:
  
  M1: VDD ---- GND ---- VDD ---- GND ---- (standard cell power rails, in rows)
```

**Power grid design parameters — Numerical Example:**

```
Given:
  - Total chip power = 2W
  - VDD = 0.75V (7nm)
  - Target IR drop budget = 5% of VDD = 37.5 mV
  - Total current = P/V = 2/0.75 = 2.67 A

IR Drop = I * R_grid

For a simplified 1D model:
  R_grid = rho * L / (W * T)
  
  Where:
    rho = sheet resistance of metal layer
    L = length of power strap
    W = width of power strap
    T = thickness (built into sheet resistance)

Example M9 strap:
  Sheet resistance (Rs) = 0.02 ohm/sq (for thick top metal)
  Strap width = 4 um
  Strap length = 1500 um (across chip)
  Number of squares = 1500/4 = 375
  R per strap = 0.02 * 375 = 7.5 ohm
  
  But many straps in parallel:
  If 50 VDD straps: R_effective = 7.5/50 = 0.15 ohm
  IR drop from one end = 2.67A * 0.15 = 0.4V  (way too much!)
  
  But current enters from both ends and is distributed:
  Effective IR drop ≈ I*R/8 for distributed load = 2.67*0.15/8 = 50 mV
  
  Still above 37.5 mV target → need more straps or wider straps.
  Solution: 80 VDD straps, each 5 um wide, plus rings on all edges.
```

**Power strap pitch and width guidelines (7nm example):**

| Metal Layer | Typical Width | Typical Pitch | Purpose                  |
|-------------|---------------|---------------|--------------------------|
| M1          | 0.036 um      | 0.036 um      | Std cell power rails     |
| M2-M5       | 0.1-0.5 um   | 2-10 um       | Local power distribution |
| M6-M7       | 1-3 um        | 10-30 um      | Intermediate grid        |
| M8-M9       | 3-8 um        | 20-50 um      | Global grid              |
| M10+ (RDL)  | 5-15 um       | 30-80 um      | Top-level straps/rings   |

**Decap cell placement:**
- Decoupling capacitors store local charge to supply instantaneous switching current
- Place near high-switching-activity blocks (clock trees, wide buses, ALUs)
- Place in whitespace regions after placement to improve utilization
- Typical decap density: 5-15% of core area
- Critical near VDD/VSS ring connections and macro boundaries

**Power rings:**

```
  +--VDD-ring---------------------------------------+
  | +--VSS-ring-----------------------------------+ |
  | |                                             | |
  | |   Core Area with straps/mesh inside         | |
  | |                                             | |
  | +---------------------------------------------+ |
  +--------------------------------------------------+
  
  Ring width: 5-15 um (per ring), multiple VDD/VSS rings
  Via arrays at ring-to-strap intersections
```

### 2.6 Blockage Types

| Blockage Type   | Effect                                           | Use Case                        |
|-----------------|--------------------------------------------------|---------------------------------|
| Hard placement  | No cells can be placed here                      | Over macros, analog regions     |
| Soft placement  | Cells allowed but tool tries to avoid             | Near macros for routing         |
| Partial (e.g., 60%) | Limits placement density to specified %       | Around macros, congested areas  |
| Halo             | Keep-out region around macro                     | Ensure routing channels         |
| Routing blockage | No routing on specified layers in region         | Reserved layers for power, shielding |

**Halo vs Blockage:**
- Halo is relative to a macro (moves with the macro during optimization)
- Blockage is absolute (fixed coordinate region)
- Typical halo: 2-10 um around SRAM macros

### 2.7 Pin Placement

- IO pin locations drive the top-level signal routing topology
- Place pins on the side closest to the logic they connect to (minimizes wirelength)
- Clock pins: place centrally for balanced CTS
- For hierarchical designs: pin placement must match between parent and child partitions
- Pin spacing must satisfy minimum routing pitch on the pin layer

### 2.8 Floorplan Sanity Checks

1. **Channel DRC**: ensure minimum spacing between macros, macros and core boundary
2. **Placement density**: verify no region exceeds target utilization
3. **Power EM**: early check that power straps can handle expected current density
4. **Fly-line analysis**: verify no excessively long logical connections
5. **Timing estimation**: virtual route + trial placement to check feasibility

---

## 3. Placement

### 3.1 Global Placement

**Objective function — Quadratic Wirelength:**

```
The placement problem minimizes:
  
  W = Sum over all nets [ Sum over all pin pairs (i,j) in net ]
      of ( (xi - xj)^2 + (yi - yj)^2 )

This is a quadratic function → solvable via linear system:
  dW/dx_i = 0  for all cells i
  
This produces a system of linear equations:
  Q * x = b_x    (for x-coordinates)
  Q * y = b_y    (for y-coordinates)

Where Q is a connectivity matrix (Laplacian of the hypergraph).
```

**Modern placers use a hybrid approach:**
1. Quadratic/analytical placement produces an initial spread
2. Force-directed methods push cells apart to reduce overlap
3. Partitioning-based refinement (FM/KL algorithms) for further optimization
4. ePlace (Cadence) and SimPL (Synopsys) use electrostatics analogy: cells = charged particles, density = potential field

**Numerical Example — HPWL Calculation:**

```
Given a net with 4 pins at coordinates:
  Pin A: (10, 20)
  Pin B: (50, 30)
  Pin C: (30, 60)
  Pin D: (40, 10)

HPWL = (max_x - min_x) + (max_y - min_y)
     = (50 - 10) + (60 - 10)
     = 40 + 50
     = 90 units

This is a fast approximation of Steiner tree wirelength.
Actual routed wirelength is typically 1.0-1.5x HPWL.
```

### 3.2 Detailed Placement (Legalization)

After global placement, cells overlap and are not aligned to rows:

```
  Before Legalization:          After Legalization:
  
  Row 4: | A  [C]  B    |      Row 4: | A   C   B       |
  Row 3: |   [D E]      |      Row 3: |   D   E         |
  Row 2: |  [F] G    H  |  →   Row 2: |  F   G      H   |
  Row 1: | I    J  [K]  |      Row 1: | I    J    K     |
  
  [overlapping cells]           All cells snapped to rows, no overlap
```

**Legalization rules:**
- Cells must be on legal row sites (quantized x positions based on site width)
- No overlap between cells
- Minimum spacing rules between cells
- Power rail alignment: cells must alternate orientation (N/S flip) for shared VDD/VSS rails

### 3.3 Placement Optimization

**Timing-driven placement:**
- Identify critical paths from initial timing analysis
- Pull cells on critical paths closer together
- Use "virtual routing" to estimate delays
- Net weighting: assign higher weights to timing-critical nets

**Congestion-driven placement:**
- Route demand estimated by pin count and net span per GCell
- If demand > supply (available tracks), area is congested
- Spread cells from congested areas even if it slightly worsens wirelength
- Congestion penalty in placement cost function

**Power-driven placement:**
- Cluster high-activity cells to localize switching current
- Can also spread high-activity cells to distribute heat

### 3.4 Cell Padding

```
  Without padding:        With padding (1 site each side):
  
  |A|B|C|D|E|F|          |A| |B| |C| |D| |E| |F|
  
  Padding provides extra routing tracks between cells.
  Useful for high-fanout or congested regions.
  
  Typical: 1-2 sites padding for congested designs
  Impact: reduces effective utilization by 5-10%
```

### 3.5 Relative Placement (RPGroups)

Keep related cells physically close for timing or matching:

```
  RPGroup "critical_mux":
  +-------------------+
  | mux_sel_buf       |
  | mux_data0_inv     |
  | mux_inst          |
  | mux_data1_buf     |
  +-------------------+
  
  The tool places these cells within a small bounding box.
  Used for: clock muxes, critical datapaths, matched delay paths.
```

### 3.6 High Fanout Net Handling

Nets with fanout > 100-1000 are special:
- Clock nets: handled by CTS (not during placement)
- Reset nets: buffer tree inserted during placement or post-placement
- Enable signals: buffered to meet transition time requirements

```
  Before buffering (fanout = 500):
  
  driver ---+--- sink1
            +--- sink2
            +--- sink3
            ...
            +--- sink500
  
  After buffering (tree structure):
  
  driver --- buf1 ---+--- buf1a --- sink1..sink50
                     +--- buf1b --- sink51..sink100
             buf2 ---+--- buf2a --- sink101..sink150
                     ...
```

### 3.7 Congestion Analysis

**GRC (Global Routing Congestion) Map:**

```
  Congestion map (color-coded):
  
  +------+------+------+------+
  | 0.3  | 0.5  | 0.8  | 0.4  |   Numbers = demand/supply ratio
  +------+------+------+------+   
  | 0.4  | 0.9  | 1.2! | 0.6  |   > 1.0 = overflow (congestion!)
  +------+------+------+------+   0.8-1.0 = near overflow (warning)
  | 0.3  | 0.7  | 0.9  | 0.5  |   < 0.7 = comfortable
  +------+------+------+------+
  | 0.2  | 0.4  | 0.5  | 0.3  |
  +------+------+------+------+
  
  GCell (1.2!) has overflow → routing will fail or detour.
  Fix: add placement blockage, increase cell padding, resize GCells.
```

### 3.8 Placement Metrics Summary

| Metric                  | Good Target          | Indicates                    |
|-------------------------|----------------------|------------------------------|
| Total HPWL              | Minimize             | Overall wirelength quality   |
| WNS (Worst Neg Slack)   | > -100ps post-place  | Timing feasibility           |
| TNS (Total Neg Slack)   | Close to 0           | Overall timing health        |
| Congestion overflow     | 0%                   | Routability                  |
| Max density per GCell   | < 85%                | Even cell distribution       |
| Cell count              | Track vs budget      | Area sanity                  |

---

## 4. Clock Tree Synthesis (CTS)

### 4.1 CTS Goals (Priority Order)

1. **Meet DRC**: transition time (slew), capacitance, fanout limits
2. **Minimize skew**: difference in clock arrival time between any two sinks
3. **Minimize insertion delay**: total delay from clock source to sinks
4. **Minimize power**: fewer buffers = less switching power on clock network
5. **Meet OCV targets**: reduce sensitivity to on-chip variation

### 4.2 Key Definitions

```
  Clock Source (PLL output or port)
       |
       | insertion delay = 800ps
       v
  +----+----+
  |  CTS    |
  | buffers |
  +----+----+
      / \
     /   \
    v     v
  FF_A   FF_B
  
  Arrival at FF_A = 800ps
  Arrival at FF_B = 820ps
  
  Skew = |820 - 800| = 20ps
  
  Insertion delay (latency) = average arrival = 810ps
  
  Local skew: between FFs in the same clock domain that have timing paths
  Global skew: between any two FFs in the design (usually larger)
  Useful skew: intentional skew to help timing
```

### 4.3 Tree Topologies

**H-Tree:**
```
  Balanced H-tree for symmetric clock distribution:
  
            CLK_IN
               |
       +-------+-------+
       |               |
   +---+---+       +---+---+
   |       |       |       |
  FF1    FF2     FF3     FF4
  
  Each branch has equal wire length → inherently balanced.
  Used in: regular array structures, memory arrays, FPGAs.
  Downside: works poorly for non-uniform sink distributions.
```

**Balanced CTS (Standard Cell Based):**
```
  Most common in ASIC designs.
  Tool builds a buffer tree top-down or bottom-up.
  
  CLK_IN → buf → buf → buf → FF1
                    |→ buf → FF2
              |→ buf → buf → FF3
                    |→ buf → FF4
  
  Tool balances delays by:
  - Choosing buffer sizes (drive strength)
  - Inserting delay buffers on shorter paths
  - Wire snaking (adding wire length to shorter paths)
```

**Clock Mesh:**
```
  +-------+-------+-------+-------+
  |       |       |       |       |
  +---+---+---+---+---+---+---+---+   Horizontal mesh wires
  |       |       |       |       |
  +---+---+---+---+---+---+---+---+
  |       |       |       |       |
  +-------+-------+-------+-------+
  
  Multiple drivers at mesh intersections.
  Extremely low skew (< 5ps achievable).
  Very high power (mesh is always switching).
  Used in: high-performance CPUs (Intel, AMD core clocks).
```

**Fishbone:**
```
  Central spine with branches:
  
          CLK_IN
             |
  ===+===+===+===+===+===+===   (spine)
     |   |       |   |   |
    FF1  FF2    FF3 FF4  FF5     (branches to sinks)
  
  Good for long, narrow blocks.
  Easy to balance along one dimension.
```

### 4.4 CTS Algorithm Detail

**Bottom-up clustering (Deferred Merge Embedding - DME):**

```
Step 1: Start with sink locations
  FF_A(10,20), FF_B(50,20), FF_C(30,60), FF_D(70,60)

Step 2: Pair sinks and find merge point
  Pair (A,B): merge at (30, 20) — equidistant in Manhattan distance
  Pair (C,D): merge at (50, 60)

Step 3: Merge the merged points
  Merge ((30,20), (50,60)): merge at (40, 40)

Step 4: Insert buffers at merge points

  Result:
                (40,40) buf_root
                /           \
        (30,20) buf1    (50,60) buf2
         /    \          /    \
      FF_A   FF_B    FF_C   FF_D
```

### 4.5 Clock Tree Cells

CTS uses special library cells optimized for:
- **Balanced rise/fall times**: normal buffers may have unequal trise/tfall. CTS buffers are designed with balanced PMOS/NMOS sizing.
- **Low skew**: tight process variation control
- **Multiple drive strengths**: CTS library typically has 8-16 buffer sizes

```
  Typical CTS cell library:
  CLKBUF_X1, CLKBUF_X2, CLKBUF_X4, CLKBUF_X8, CLKBUF_X16
  CLKINV_X1, CLKINV_X2, CLKINV_X4, CLKINV_X8, CLKINV_X16
  
  Inverters are preferred in CTS because:
  - Inverter pair (2 inversions) has better balanced delay
  - Lower area than buffer (which is 2 inverters internally)
  - Alternating inversions naturally compensate for duty cycle distortion
```

### 4.6 Non-Default Rules (NDR) for Clock

```
  Standard signal wire:       Clock wire with NDR:
  
  |-w-|  |-s-|  |-w-|         |-2w-|  |-2s-|  |-2w-|
  |   |  |   |  |   |         |    |  |    |  |    |
  |sig|  |   |  |sig|         |clk |  |    |  |sig |
  |   |  |   |  |   |         |    |  |    |  |    |
  
  w = minimum width            2w = double width
  s = minimum spacing           2s = double spacing
  
  NDR: "double width, double spacing" (2W2S)
```

**Why NDR for clocks:**
- **Double width**: reduces wire resistance → less RC delay variation → less skew
  - R is inversely proportional to width: halved R means halved RC delay sensitivity
- **Double spacing**: reduces coupling capacitance → less crosstalk → less jitter
  - Critical because clock jitter directly impacts timing margin
- **Cost**: NDR clock wires consume 4x the routing resources of standard wires
- **Multi-cut vias**: mandatory for clock nets for reliability

**Numerical impact:**
```
Standard wire: R = 10 ohm/um, C = 0.2 fF/um, RC = 2.0 fF*ohm/um^2
NDR 2W wire:   R = 5 ohm/um,  C = 0.25 fF/um, RC = 1.25 fF*ohm/um^2  (37.5% less)
NDR 2W2S wire: R = 5 ohm/um,  C = 0.15 fF/um, RC = 0.75 fF*ohm/um^2  (62.5% less)
```

### 4.7 Useful Skew

**Concept: borrowing time from adjacent pipeline stages:**

```
  Without useful skew:
  
  FF_A --[combo: 3ns]-- FF_B --[combo: 1ns]-- FF_C
    ^                     ^                      ^
    CLK (0ps skew)       CLK (0ps skew)         CLK
    
  Period must be >= 3ns (bottleneck is path A→B)
  Path B→C uses only 1ns of 3ns period → 2ns wasted.
  
  With useful skew:
  
  FF_A --[combo: 3ns]-- FF_B --[combo: 1ns]-- FF_C
    ^                     ^                      ^
    CLK +500ps skew      CLK                    CLK -500ps skew
    
  Now FF_B captures 500ps later:
    Path A→B effective time = 3ns + 500ps = 3.5ns budget (but we only need 3ns, OK)
    Wait — we need: combo_delay < Period + skew_B - skew_A
    3ns < Period + 500ps - 0 → Period > 2.5ns
    
  Path B→C: 1ns < Period + (-500ps) - 0 → need Period > 1.5ns (still OK)
  
  Net result: can run at 2.5ns period instead of 3ns → 20% faster!
```

**Useful skew constraints:**
```
Setup: T_combo < T_clk + skew   (more time for slow paths)
Hold:  T_combo > T_hold - skew  (must still meet hold with the skew)
```

The CTS tool must balance: adding skew helps setup but hurts hold (requires more hold buffers).

### 4.8 Multi-Source CTS / Clock Mesh

For clock mesh designs:
1. Build an initial tree to drive the mesh
2. Insert mesh grid (horizontal + vertical wires)
3. Multiple tree endpoints drive different mesh intersections
4. Mesh averaging effect reduces local skew to < 5ps
5. Short-circuit current between drivers with slightly different phases is the power cost

### 4.9 CTS Metrics

| Metric              | Good Target (7nm, 1GHz) | Notes                           |
|---------------------|-------------------------|---------------------------------|
| Global skew         | < 50ps                  | Across entire clock domain      |
| Local skew          | < 20ps                  | Between related FF pairs        |
| Insertion delay     | 200-500ps               | Depends on die size             |
| Transition time     | < 80ps at sinks         | DRC requirement                 |
| # CTS buffers       | Design-dependent        | Monitor for power impact        |
| Clock tree power    | 30-40% of total dynamic | Industry-typical                |
| Duty cycle distortion | < 5%                  | Important for DDR interfaces    |

### 4.10 Post-CTS Optimization

1. **Hold fixing**: insert delay buffers on short paths to meet hold timing
   - CTS creates real clock latency → hold paths now have actual timing numbers
   - Pre-CTS hold fixing is inaccurate (ideal clocks)

2. **Useful skew optimization**: after initial CTS, tool adjusts buffer sizes / adds delay to selectively skew clock arrivals

3. **CCD (Concurrent Clock and Data)**: optimizes clock and data paths simultaneously
   - Traditional: fix clock tree, then optimize data paths
   - CCD: allows clock tree adjustments alongside data path optimization
   - 5-10% frequency improvement possible with CCD

---

## 5. Routing

### 5.1 Global Routing

```
  Chip divided into Global Routing Cells (GCells):
  
  +------+------+------+------+
  |      |      |      |      |
  | GC00 | GC01 | GC02 | GC03 |   Each GCell has:
  |      |      |      |      |   - Horizontal tracks (supply)
  +------+------+------+------+   - Vertical tracks (supply)
  |      |      |      |      |
  | GC10 | GC11 | GC12 | GC13 |   Global router finds path through
  |      |      |      |      |   GCells for each net (coarse grid)
  +------+------+------+------+
  
  Net A: GC00 → GC01 → GC11    (L-shaped route)
  Net B: GC00 → GC10 → GC11    (another L-shape)
  
  Congestion = demand / supply per GCell edge
  If demand > supply → overflow → must detour or promote to higher metal
```

### 5.2 Track Assignment

Intermediate step between global and detail routing:
- Assigns each net segment to a specific track within the GCell
- Resolves most spacing conflicts
- Results in a nearly-legal route that detail router refines

### 5.3 Detail Routing

Creates actual geometric shapes (rectangles on metal layers, vias):
- Must satisfy ALL DRC rules: minimum width, spacing, enclosure, end-of-line, etc.
- Rip-up and reroute (RRR): when conflicts arise, remove conflicting segment, find alternative
- Negotiation-based routing: PathFinder algorithm — all nets compete for tracks, penalty increases for over-used resources

### 5.4 Routing Layer Usage

```
  Metal stack (typical 7nm, 13 metal layers):
  
  M13 (AP)  ─── Ultra-thick, redistribution layer (bumps)
  M12       ─── Top global routing, power straps
  M11       ─── Global routing, power straps
  M10       ─── Global routing (prefer horizontal)
  M9        ─── Global routing (prefer vertical)
  M8        ─── Semi-global (prefer horizontal)
  M7        ─── Semi-global (prefer vertical)
  M6        ─── Intermediate (prefer horizontal)
  M5        ─── Intermediate (prefer vertical)
  M4        ─── Local routing (prefer horizontal)
  M3        ─── Local routing (prefer vertical)
  M2        ─── Local routing (prefer horizontal)
  M1        ─── Intra-cell routing (std cell internal)
  
  Lower metals: high resistance, fine pitch → short, local connections
  Upper metals: low resistance, coarse pitch → long, global connections
  
  Preferred direction alternates H/V by layer to facilitate via connections.
```

### 5.5 Antenna Effect

```
  During fabrication (plasma etching), metal layers are built bottom-up.
  When M3 is being etched, M3 wire collects charge from plasma.
  If M3 wire is long and connects to a gate (through M2, M1, poly):
  
       +----- Long M3 wire (charge collecting antenna) -----+
       |                                                      |
       Via                                                    (no connection yet
       |                                                       to upper metals)
       M2 segment
       |
       Via
       |
       M1 segment
       |
       Gate oxide  ← Charge damages thin gate oxide!
       |
       Transistor
  
  Antenna Ratio = (Metal area collecting charge) / (Gate area connected)
  
  Foundry specifies maximum antenna ratio per layer (e.g., 400:1 for M3).
```

**Antenna Fixing Methods:**

1. **Diode insertion**: add reverse-biased diode near the gate to discharge accumulated charge
   ```
   Long M3 wire --- gate
                 |
                 diode (to substrate) ← bleeds off charge
   ```

2. **Layer jumping**: break the long wire by jumping to a higher metal layer
   ```
   M3 ----+    +---- M3
          |    |
          M5---M5      (M5 processed later, breaks antenna path)
   ```

3. **Bridge insertion**: route part of the net on a different layer

### 5.6 Crosstalk

```
  Coupling capacitance between adjacent wires:
  
  Signal A ─────────────────────── (aggressor)
            ↕ Cc (coupling cap)
  Signal B ─────────────────────── (victim)
  
  When A switches, it injects noise into B through Cc.
```

**Two impacts:**

1. **Functional noise (glitch)**:
   ```
   Victim is stable at logic 0.
   Aggressor transitions 0→1.
   Coupling injects positive voltage bump on victim.
   If bump > noise margin → functional failure!
   
   Glitch amplitude ≈ Cc / (Cc + Cvictim_to_ground) * Vdd
   
   Example: Cc = 0.5fF, Cground = 1.5fF, Vdd = 0.75V
   Glitch ≈ 0.5/(0.5+1.5) * 0.75 = 0.25 * 0.75 = 0.1875V
   If noise margin = 0.2V → marginal! Need to fix.
   ```

2. **Timing impact**:
   ```
   Same-direction switching (aggressor and victim switch same way):
     Effective Cc is reduced → victim appears faster (speed-up)
     
   Opposite-direction switching:
     Effective Cc is increased → victim appears slower (slow-down)
     
   Miller effect: Cc_effective = Cc * (1 + delta_V_aggressor/delta_V_victim)
     Same direction: Cc_eff ≈ 0 (best case)
     Opposite direction: Cc_eff ≈ 2*Cc (worst case)
   ```

**Crosstalk Fixing:**

| Method         | How It Works                        | Cost              |
|----------------|-------------------------------------|--------------------|
| Spacing        | Increase distance between wires     | More routing area  |
| Shielding      | Insert VDD/VSS wire between signals | Routing resources  |
| Layer change   | Move one net to a different layer   | Via, potential detour |
| Net ordering   | Place non-switching nets adjacent   | Tool analysis time |
| Downsizing aggressor | Weaker driver = slower transition | May affect aggressor timing |
| Upsizing victim | Stronger driver = less susceptible | More power, area  |

### 5.7 Via Optimization

```
  Single-cut via:        Multi-cut via (2x1):      Multi-cut via (2x2):
  
  +---+                  +---+ +---+                +---+ +---+
  |   |                  |   | |   |                |   | |   |
  +---+                  +---+ +---+                +---+ +---+
                                                    +---+ +---+
  R = R_via              R = R_via/2                |   | |   |
                                                    +---+ +---+
  1 via: if it fails     2 vias: redundancy         R = R_via/4
  → open circuit         if 1 fails, other works
```

**Why multi-cut vias matter:**
- **Reliability**: single via failure probability ~10^-6. Two independent vias: ~10^-12
- **Resistance**: parallel vias reduce via resistance (important for IR drop)
- **EM**: current distributed across multiple vias, reduces current density per via
- **DFM requirement**: many foundries mandate multi-cut vias for signoff at 7nm and below
- **Trade-off**: multi-cut vias need more space, can cause routing congestion

### 5.8 ECO Routing

Post-signoff changes (Engineering Change Order):
1. Spare cells pre-placed in the design can be repurposed
2. Only metal layers are re-routed (no base layer change = metal-only ECO)
3. Tools perform incremental routing: fix only changed nets
4. Critical: verify LVS, DRC, timing only for changed region + neighbors

```
  ECO flow:
  Functional bug found → RTL fix → re-synthesize affected logic
  → incremental placement (using spare cells or minimal cell swaps)
  → incremental routing → re-run signoff on affected region
  → saves 2-4 weeks vs full PnR iteration
```

---

## 6. Physical Verification (Signoff)

### 6.1 DRC (Design Rule Check)

**Fundamental DRC rules:**

```
  Minimum width:    |<- w ->|     w >= w_min (e.g., 36nm for M1 at 7nm)
                    |=======|
  
  Minimum spacing:  |===| gap |===|    gap >= s_min (e.g., 36nm for M1)
  
  Minimum enclosure (via in metal):
                    +----------+
                    |  +----+  |
                    |  | via|  |    enclosure >= e_min on all sides
                    |  +----+  |
                    +----------+
  
  Minimum area:     Metal rectangle must have area >= A_min
  
  End-of-line (EOL) spacing:
                    |===|
                         ↕ EOL space (larger than regular spacing)
                    |===|
                    
  (The end of a wire needs more spacing due to lithography effects)
```

**Advanced Node DRC (7nm and below):**

- **LELE double patterning**: features on one layer split into two masks (colors)
  ```
  Original M2:    |A| |B| |C| |D|
  
  Mask 1 (blue):  |A|     |C|      (every other wire)
  Mask 2 (green):     |B|     |D|
  
  Coloring constraint: adjacent wires must be on different masks.
  If 3 mutually adjacent wires → coloring conflict → DRC error!
  ```

- **SADP (Self-Aligned Double Patterning)**: mandrel+spacer technique, creates very specific design rules
  - Tip-to-tip spacing rules (different for same-color vs different-color)
  - Cut-based rules for creating line ends

- **Via pillar rules**: vias must land on specific grid locations

### 6.2 LVS (Layout Versus Schematic)

```
  LVS Flow:
  
  Layout (GDSII)                    Schematic (Netlist from synthesis)
       |                                      |
       v                                      v
  Device Extraction                     Read Netlist
  (identify transistors,                      |
   resistors, caps)                           |
       |                                      |
       v                                      v
  Connectivity Extraction              +-----------+
  (trace metal connections)    ------> |  COMPARE  |
       |                               +-----------+
       v                                      |
  Extracted Netlist                           v
                                      PASS or FAIL
                                      (with error report)
```

**Common LVS errors:**

| Error Type       | Cause                                        | Fix                              |
|------------------|----------------------------------------------|----------------------------------|
| Short            | Two nets connected that shouldn't be         | Fix routing overlap              |
| Open             | Net has disconnected segments                | Add missing via or wire          |
| Device mismatch  | Extra/missing transistor in layout           | Check well/diffusion regions     |
| Floating net     | Net connected to nothing                     | Connect or remove                |
| Parameter mismatch | Transistor W/L differs from schematic      | Fix device sizing                |
| Missing connection | Pin not properly connected to net          | Fix pin geometry                 |

### 6.3 ERC (Electrical Rule Check)

- **Floating gate**: gate terminal not connected to any driver → undefined state, excessive leakage
- **Antenna violation**: antenna ratio exceeds limit (see Section 5.5)
- **Well connectivity**: N-well must be connected to VDD, P-well to VSS (or body bias supply)
- **ESD path check**: all IO pads have proper ESD protection path to VDD/VSS

### 6.4 Metal Density Checks and Dummy Fill

```
  CMP (Chemical Mechanical Polishing) during fabrication:
  
  Without fill:                    With fill:
  
  |==|         |==|               |==| |F| |F| |==| |F|
  +---substrate----+              +---substrate---------+
  
  Low density area polishes          Uniform density → uniform
  differently → dishing, erosion     polishing → flat surface
  
  Foundry requirement: metal density per layer must be 20-80% in any window.
  Fill insertion adds dummy metal shapes to meet density targets.
  
  Timing impact: fill adds parasitic capacitance to nearby signal wires!
  Timing-aware fill: avoid placing fill too close to timing-critical nets.
```

### 6.5 Electromigration (EM)

**Black's Equation:**
```
  MTTF = A * (J)^(-n) * exp(Ea / (k*T))
  
  Where:
    MTTF = Mean Time To Failure
    A = constant (process dependent)
    J = current density (A/cm^2)
    n = current density exponent (typically 1-2)
    Ea = activation energy (~0.7 eV for Al, ~0.9 eV for Cu grain boundary)
    k = Boltzmann constant (8.617e-5 eV/K)
    T = temperature (Kelvin)
```

**Numerical Example:**
```
Given:
  Wire on M3 at 7nm: width = 36nm, thickness = 50nm
  Wire area = 36nm * 50nm = 1800 nm^2 = 1.8e-11 cm^2
  
  EM limit for M3 = 1 MA/cm^2 = 1e6 A/cm^2
  Max DC current = J_limit * Area = 1e6 * 1.8e-11 = 18 uA (per wire!)
  
  For AC (RMS) current: limit is typically 3-10x higher due to
  self-healing during current reversal.
  
  For power straps (much wider, e.g., 2 um):
  Area = 2000nm * 100nm = 2e-9 cm^2
  Max current = 1e6 * 2e-9 = 2 mA per strap
  
  With 100 parallel straps: 200 mA total → need to verify this is sufficient.
```

**EM by metal layer:**

| Metal Layer | Typical EM Limit (DC) | Wire Width | Max Current/Wire |
|-------------|----------------------|------------|------------------|
| M1          | 0.5 MA/cm^2          | 36nm       | ~9 uA            |
| M4          | 1.0 MA/cm^2          | 40nm       | ~20 uA           |
| M8          | 2.0 MA/cm^2          | 200nm      | ~400 uA          |
| M12 (thick) | 3.0 MA/cm^2          | 2000nm     | ~30 mA           |

### 6.6 IR Drop Analysis

**Static IR Drop:**
```
Average current consumption → DC voltage drop across power grid.

V_drop = I_avg * R_grid

Measured at every node of the power grid.
Output: voltage contour map showing drop from supply to each cell.

Target: V_drop < 3-5% of VDD
  At VDD = 0.75V: max drop = 22.5-37.5 mV
```

**Dynamic IR Drop:**
```
Considers simultaneous switching of many cells (worst-case transient).

V_drop_dynamic = L * di/dt + R * i(t)

The di/dt term (inductance) causes supply bounce (Ldi/dt noise).
Much worse than static IR drop — can be 2-5x larger.

Analysis requires:
  1. VCD (Value Change Dump) from simulation for switching activity
  2. Power grid model (R, L, C network)
  3. Transient simulation of power grid

Output: time-varying voltage map, identify hotspots and worst-case time windows.
```

**IR Drop Map (ASCII representation):**
```
  +------+------+------+------+
  | 15mV | 20mV | 25mV | 18mV |   Voltage drop values at grid nodes
  +------+------+------+------+
  | 22mV | 30mV | 38mV!| 28mV |   38mV exceeds 37.5mV budget!
  +------+------+------+------+   → Need more power straps or
  | 18mV | 25mV | 32mV | 22mV |     decap cells in that region
  +------+------+------+------+
  | 10mV | 15mV | 20mV | 12mV |
  +------+------+------+------+
```

---

## 7. Timing Signoff

### 7.1 MCMM (Multi-Corner Multi-Mode)

```
  Corners (process/voltage/temperature):
  
  Corner Name     Process    Voltage     Temperature    Purpose
  ─────────────────────────────────────────────────────────────
  SS_0p675V_125C  Slow-Slow  0.675V      125°C          Setup (worst)
  FF_0p825V_m40C  Fast-Fast  0.825V      -40°C          Hold (worst)
  TT_0p75V_25C    Typical    0.750V      25°C           Nominal
  SS_0p675V_m40C  Slow-Slow  0.675V      -40°C          Setup (cold)
  FF_0p825V_125C  Fast-Fast  0.825V      125°C          Hold (hot)
  
  Modes (functional scenarios):
  - func_mode: normal operation, highest frequency
  - scan_mode: test mode, slower clock, shift/capture constraints
  - sleep_mode: low power, most clocks gated, leakage focus
  
  MCMM matrix: every corner × every mode = a "scenario"
  Each scenario has its own SDC (constraints), SPEF (parasitics), library
  
  Typical signoff: 10-30 scenarios for a complex SoC
```

### 7.2 Extraction Corners

```
  Parasitic extraction produces SPEF (Standard Parasitic Exchange Format)
  
  Extraction Corner   Resistance    Capacitance    When to Use
  ──────────────────────────────────────────────────────────────
  Cworst (Cmax)       Nominal       Maximum        Setup analysis (more delay)
  Cbest (Cmin)        Nominal       Minimum        Hold analysis (less delay)
  RCworst             Maximum       Maximum        Long wire dominated paths
  RCbest              Minimum       Minimum        Short wire / gate dominated
  
  Wire delay (Elmore): T = R * C
  - Cworst: high C → more delay → harder to meet setup
  - Cbest: low C → less delay → harder to meet hold
  - RCworst: both high → maximum wire delay (long global routes)
```

### 7.3 OCV, AOCV, POCV

**On-Chip Variation (OCV):**
```
  Cells on the same die don't all have the same delay.
  Process variation, voltage drop, temperature gradient cause differences.
  
  Traditional OCV: apply flat derate
    Data path (launch): delay * (1 + derate)    [make it slower for setup]
    Clock path (capture): delay * (1 - derate)   [make it faster for setup]
    
    Typical derate: 5-10%
    
  Problem: flat derate is pessimistic for paths with many stages.
  Statistical reasoning: variations average out over many cells.
```

**AOCV (Advanced OCV):**
```
  Derate depends on:
  1. Path depth (number of stages): more stages → smaller derate
  2. Physical distance: cells far apart → larger derate
  
  AOCV table (example):
  
  Depth    Derate (early/late)
  1        0.92 / 1.08
  2        0.93 / 1.07
  4        0.94 / 1.06
  8        0.95 / 1.05
  16       0.96 / 1.04
  32       0.97 / 1.03
  
  As depth increases, derate approaches 1.0 (less pessimistic).
```

**POCV (Parametric OCV):**
```
  Statistical approach: each cell delay has a mean and sigma.
  
  Path delay = sum of (mean_i + k * sigma_i)
  
  For independent variations:
  Path sigma = sqrt(sum of sigma_i^2)
  
  Total path delay = sum(mean_i) + k * sqrt(sum(sigma_i^2))
  
  k = 3 for 3-sigma (99.87% confidence)
  
  This is less pessimistic than AOCV for deep paths:
  AOCV assumes worst-case derate for whole path.
  POCV properly accounts for statistical cancellation.
  
  Example:
  10-stage path, each cell: mean=50ps, sigma=5ps
  
  AOCV (depth=10, derate=1.05): total = 10*50*1.05 = 525ps
  POCV (3-sigma): total = 10*50 + 3*sqrt(10*25) = 500 + 3*15.8 = 547ps
  
  At depth=100:
  AOCV (depth=100, derate=1.02): total = 100*50*1.02 = 5100ps
  POCV (3-sigma): total = 100*50 + 3*sqrt(100*25) = 5000 + 3*50 = 5150ps
  
  The relative pessimism varies; POCV gives more accurate results.
```

### 7.4 SI Timing

```
  Crosstalk impacts timing:
  
  Setup (late arrival on data):
    Victim data path slowed by opposite-switching aggressor
    → add SI delay penalty to data arrival time
    
  Hold (early arrival on data):
    Victim data path sped up by same-switching aggressor
    → subtract SI delay from data arrival time
    
  SI-aware STA tools (PrimeTime SI, Tempus):
    - Extract coupling capacitance from SPEF
    - Determine aggressor switching windows
    - Calculate worst-case SI impact on each timing path
    - Report SI-specific slack contributions
```

### 7.5 ECO Flow

```
  ECO Types:
  
  1. Functional ECO:
     - Fix a logic bug post-layout
     - Re-synthesize affected logic
     - Map to existing spare cells or minimal cell changes
     - Reroute only affected nets
     
  2. Timing ECO:
     - Fix setup/hold violations found in signoff
     - Operations: size up (faster), size down (less load), swap VT,
       insert buffer (fix transition), delete buffer (reduce delay)
     - No logic change, only cell sizing/swapping
     
  3. Metal-only ECO:
     - Change only metal routing (no cell changes)
     - Fastest turnaround
     - Limited in what it can fix
     
  ECO signoff: re-run STA, DRC, LVS only in affected region.
  Full-chip re-verification recommended for tapeout confidence.
```

---

## 8. Advanced Topics

### 8.1 FinFET-Specific Physical Design

**Fin Quantization:**
```
  In FinFET technology, transistor width is quantized:
  
  W_eff = N_fins * W_fin
  
  Where N_fins = 1, 2, 3, 4, ... (integer number of fins)
  W_fin is fixed by the process (e.g., 7nm FinFET: ~6-7nm fin width)
  
  Impact on PD:
  - Standard cell height is quantized by fin count
    Example: 7.5T cell height = 7.5 fin pitches
    (where T = track = metal pitch, which defines cell height)
  - Cell libraries come in specific heights: 6T, 7.5T, 9T
    Shorter cells: less area, but more routing congestion (fewer M2 tracks)
    Taller cells: more routing tracks, easier to close timing, but larger area
  
  6T library: ultra-dense, for area-critical designs (mobile SoCs)
  7.5T library: balanced, most common
  9T library: easy routing, for high-performance (server CPUs)
```

**CPODE (Continuous Poly on Diffusion Edge) / PODE:**
```
  In FinFET, cell boundaries have specific diffusion rules:
  
  Without CPODE:
  |  Cell A  |gap|  Cell B  |
  Fin─────────  ─────────Fin     (diffusion break = gap between cells)
  
  With CPODE:
  |  Cell A  |  Cell B  |
  Fin──────────────────Fin       (continuous diffusion, dummy gate at boundary)
  
  CPODE/PODE removes the gap → significant area savings.
  But: cells sharing a CPODE boundary are electrically connected at substrate.
  PD tools must manage CPODE-aware placement to avoid unintended sharing.
```

### 8.2 Multi-Patterning

**LELE (Litho-Etch-Litho-Etch):**
```
  For metal pitch below lithography resolution:
  
  Step 1: Print Mask A (blue features)
  Step 2: Etch Mask A features
  Step 3: Print Mask B (green features)
  Step 4: Etch Mask B features
  
  +---+   +---+   +---+   +---+
  | B |   | G |   | B |   | G |    B = Blue mask, G = Green mask
  +---+   +---+   +---+   +---+
  
  Constraint: features on same mask must satisfy single-patterning spacing.
  Features on different masks can be closer (limited by overlay accuracy).
  
  Coloring conflict example:
  
  A ─── B ─── C         If A, B, C are all mutually close:
  |           |          A→Blue, B→Green, C→? (needs Blue, but too close to A!)
  +───────────+          → Coloring conflict → must change layout
```

**SADP (Self-Aligned Double Patterning):**
```
  Uses mandrels and spacers:
  
  Step 1: Create mandrel
  ┌──┐    ┌──┐    ┌──┐
  │  │    │  │    │  │     mandrels
  └──┘    └──┘    └──┘
  
  Step 2: Deposit spacer material
  ┌┬──┬┐  ┌┬──┬┐  ┌┬──┬┐
  ││  ││  ││  ││  ││  ││   spacer on sides
  └┴──┴┘  └┴──┴┘  └┴──┴┘
  
  Step 3: Remove mandrel, spacers remain
   ┌┐  ┌┐  ┌┐  ┌┐  ┌┐  ┌┐
   ││  ││  ││  ││  ││  ││   doubled pitch!
   └┘  └┘  └┘  └┘  └┘  └┘
  
  SADP produces very regular patterns.
  Routing rules become more restrictive: specific track usage patterns.
```

### 8.3 3D IC / Chiplet Design

```
  Traditional 2D:          2.5D (Interposer):         3D (stacked dies):
  
  +----------+             +----+ +----+               +--------+
  |   Die    |             |Die1| |Die2|               | Die 2  |
  +----------+             +----+-+----+               +---||---+  ← micro-bumps
  | Package  |             |  Interposer  |            | Die 1  |  ← TSVs
  +----------+             +-----+--------+            +--------+
                           |   Package    |            | Package|
                           +--------------+            +--------+
```

**TSV (Through-Silicon Via):**
```
  TSV parameters (typical):
  - Diameter: 5-10 um
  - Pitch: 20-50 um
  - Resistance: 10-50 milliohm
  - Capacitance: 50-200 fF (significant!)
  - Keep-Out Zone (KOZ): 5-15 um around each TSV (stress-induced)
  
  PD implications:
  - TSV KOZ consumes area → reduces placement density
  - TSV capacitance adds to clock/signal loading → account in timing
  - TSV assignment: which signals/power go through TSVs
  - Thermal: vertical heat path, bottom die heats up more
```

**Micro-bumps:**
```
  Pitch: 40-100 um (much finer than C4 bumps at 130-200 um)
  Used for die-to-die connections in 3D/2.5D
  Thousands of connections possible → enables high-bandwidth interfaces
  
  Example: HBM (High Bandwidth Memory) uses ~1000+ micro-bumps per stack
  Bandwidth = 1024 bits * 2 GHz = 256 GB/s (per stack)
```

### 8.4 Hierarchical Physical Design

For designs > 500M gates:

```
  Full-chip SoC:
  +--------------------------------------------+
  |  +---------+  +---------+  +---------+     |
  |  | CPU     |  | GPU     |  | NPU     |     |
  |  | Block   |  | Block   |  | Block   |     |
  |  | (100M)  |  | (200M)  |  | (50M)   |     |
  |  +---------+  +---------+  +---------+     |
  |                                             |
  |  +---------+  +-------------------------+  |
  |  | Memory  |  |   Interconnect (NoC)    |  |
  |  | Ctrl    |  |   (100M gates)          |  |
  |  +---------+  +-------------------------+  |
  +--------------------------------------------+
  
  Hierarchical approach:
  1. Each block is implemented independently (own floorplan, P&R)
  2. Top-level integrates blocks as "black boxes" (LEF abstracts)
  3. Interface timing budgeted between blocks (timing assertions)
  
  Advantages:
  - Parallel implementation by different teams
  - Manageable database sizes (ICC2 struggles with >200M gates flat)
  - Faster iteration per block
  
  Challenges:
  - Interface timing budgeting must be accurate
  - Power grid continuity across hierarchy boundaries
  - Clock tree balancing across partitions
  - Feedthrough routing: signals passing through a block to reach another
```

---

## 9. Interview Q&A

### Q1: Walk me through the complete ASIC PD flow from netlist to GDSII.

**A:** The flow begins with a synthesized gate-level netlist, SDC timing constraints, technology LEF files, and standard cell/macro libraries. First, floorplanning establishes the die size (cell area / utilization), places macros, defines IO pin locations, and creates the power grid (VDD/VSS rings and stripes). Next, placement distributes standard cells using analytical global placement followed by legalization to snap cells to rows. The tool optimizes for timing, congestion, and power. Clock Tree Synthesis builds a balanced buffer tree for each clock domain, targeting minimum skew and insertion delay, using CTS-specific cells and NDR routing rules. Post-CTS, hold time fixing is performed. Routing proceeds in phases: global routing (GCell-level paths), track assignment, and detail routing (DRC-clean geometries). Post-route optimization fixes remaining timing/DRC issues. Chip finishing adds filler cells, metal fill for CMP density, and via optimization. Signoff runs include STA (PrimeTime, MCMM with POCV), DRC/LVS (Calibre/ICV), IR drop/EM analysis (RedHawk), and extraction correlation. After all signoff clean, GDSII is streamed out.

---

### Q2: How do you estimate die size? What utilization would you target for a high-performance CPU block vs a low-power IoT chip?

**A:** Die size = (total standard cell area / utilization) + macro area + IO ring area. For a high-performance CPU targeting >2GHz, I'd use 60-65% utilization to provide ample routing resources and minimize congestion — critical paths need direct routes without detours. For a low-power IoT chip running at 100MHz, 75-80% utilization is acceptable since timing is relaxed and area is the primary cost driver. The remaining whitespace in both cases accommodates routing tracks, buffer insertion during CTS and optimization, decap cells, and filler cells.

---

### Q3: You have a block with 20 large SRAM macros. Describe your macro placement strategy.

**A:** First, analyze fly-lines to understand dataflow between macros and logic. Group macros that share buses (e.g., cache data arrays accessed by the same controller). Place macros along the periphery to maximize contiguous standard cell area in the center. Orient macros so signal pins face the logic — for example, SRAM data pins toward the datapath, address pins toward the address decoder. Leave minimum 10-20um channels between macros for routing. Apply halos (5-10um) around each macro to prevent standard cells from crowding macro pins. Place hard blockages over macros and partial blockages (50-60% utilization) in channels. Run trial placement and routing to verify no congestion hotspots in channels. If macros have dedicated power, ensure power grid straps extend into macro channels with adequate via stacks.

---

### Q4: Explain the difference between static and dynamic IR drop. Which is harder to fix?

**A:** Static IR drop is the DC voltage drop caused by average current flowing through the resistive power grid: V_drop = I_avg × R_grid. It produces a steady-state voltage map. Dynamic IR drop is the transient voltage drop caused by sudden current surges during simultaneous switching (e.g., many FFs capturing on the same clock edge). It includes both resistive (IR) and inductive (Ldi/dt) components and can be 2-5x worse than static. Dynamic is harder to fix because it depends on switching patterns, which vary with functional scenarios. Fixes include: adding more power straps (reduces R), inserting decap cells near hotspots (local charge reservoir), staggering clock edges, reducing simultaneous switching activity, and ensuring adequate package-level decoupling capacitance.

---

### Q5: What is useful skew in CTS? When would you use it, and what are the risks?

**A:** Useful skew intentionally unbalances the clock tree to borrow time from a slack-rich stage and give it to a slack-poor stage. If a critical path has 100ps of negative setup slack, and the downstream path has 500ps of positive slack, adding 200ps of skew to the capturing FF (delaying its clock arrival) gives the critical path 200ps more time while taking 200ps from the downstream path (which can afford it). The risk is that useful skew tightens hold timing on the skewed paths — the data must still be stable for the hold time after the (now delayed) clock arrives. This typically requires inserting hold-fix buffers, which add area and power. The CTS tool must jointly optimize setup and hold across the design to find the globally optimal skew schedule.

---

### Q6: How do NDR (Non-Default Rules) help clock quality? What's the area cost?

**A:** NDR for clock nets typically specifies 2x width and 2x spacing. Double width cuts wire resistance in half, reducing RC delay and making clock delay less sensitive to process variation (less skew variation). Double spacing reduces coupling capacitance to adjacent signal wires, reducing crosstalk-induced jitter. Multi-cut vias are also mandated for reliability. The cost is significant: a 2W2S clock wire occupies roughly 4x the routing resources of a standard wire. For a large design where clock nets span the full die, this can consume 10-20% of available routing tracks on clock-heavy layers. The trade-off is worthwhile because clock quality directly impacts all timing paths in the design.

---

### Q7: Explain the antenna effect. How does the router fix antenna violations?

**A:** During fabrication, metal layers are processed sequentially (bottom-up). When a metal layer is being plasma-etched, the exposed metal acts as an antenna, collecting charge. If this metal is connected to a transistor gate through lower metals, the accumulated charge can damage the thin gate oxide. The antenna ratio (metal area / gate area) must stay below a foundry-specified limit per layer. The router fixes violations by: (1) inserting protection diodes near the gate to discharge charge (most common), (2) breaking the long wire by jumping to a higher metal layer (which is processed later, breaking the charge path), or (3) rearranging the routing topology to reduce the single-layer wire length connected to the gate. Diode insertion is simplest but requires placement of diode cells, which may need whitespace near the gate.

---

### Q8: What is the difference between DRC and LVS? Can a design pass DRC but fail LVS?

**A:** DRC checks geometric rules — spacing, width, enclosure, density — ensuring the layout is manufacturable. LVS verifies that the manufactured circuit (extracted from layout geometry) matches the intended schematic. Yes, a design can pass DRC but fail LVS. For example, two nets could be properly spaced (DRC clean) but a missing via causes an open in one net, meaning the extracted circuit has a disconnected node that doesn't match the schematic. Conversely, a short between two wires violates both DRC (spacing) and LVS (incorrect connectivity), but some shorts may be through substrate or other non-obvious paths that DRC doesn't directly check.

---

### Q9: Explain AOCV and POCV. Why are they better than flat OCV?

**A:** Flat OCV applies a fixed derate (e.g., 5%) to all cells, assuming worst-case variation on every stage. This is pessimistic for deep paths because statistical variations tend to average out over many stages. AOCV (Advanced OCV) uses depth-dependent derates: a 2-stage path might get 8% derate, while a 20-stage path gets only 3%. This reduces pessimism and allows higher operating frequencies (or lower voltage). POCV (Parametric OCV) goes further by treating each cell's delay as a distribution (mean + sigma). Path delay sigma grows as sqrt(sum of variances), which grows much slower than linearly. For a 100-stage path, POCV's relative variation is ~10x smaller than flat OCV's, enabling 5-10% better timing margins. POCV is the preferred method at 7nm and below.

---

### Q10: How do you handle crosstalk in routing? What's the timing impact?

**A:** Crosstalk is managed through SI-aware routing and extraction. The router uses shielding (inserting VSS/VDD wires between sensitive nets), spacing (widening the gap between aggressors and victims), and layer assignment (moving sensitive nets away from aggressors). Timing impact: opposite-direction switching makes the victim slower (coupling capacitance effectively doubles per Miller effect), while same-direction switching makes it faster. In STA, we analyze both for setup (add slow-down SI delay) and hold (add speed-up SI benefit). The extraction tool reports coupling capacitances in SPEF, and the timer computes worst-case aggressor alignment. Fixing priorities: clock nets first (jitter), then setup-critical data paths, then hold-critical paths.

---

### Q11: What is metal fill? Why is it needed, and what's the timing impact?

**A:** Metal fill (dummy fill) adds non-functional metal shapes to meet CMP density requirements. Without it, regions with low metal density polish differently during CMP, causing dishing (metal thinning) and erosion, leading to thickness variation and potentially shorts or opens. Foundries require 20-80% density per layer per window. The timing impact is that fill shapes add parasitic capacitance to nearby signal wires, slightly increasing delay. Timing-aware fill placement keeps fill shapes away from timing-critical nets (with a defined keep-out distance). Post-fill extraction and STA re-run are mandatory for signoff.

---

### Q12: Describe the difference between H-tree and balanced CTS. When would you choose each?

**A:** H-tree uses a recursive symmetric topology where each branch has exactly equal wire length, providing inherently zero skew (in theory). It works best for regular structures (SRAM arrays, FPGAs) where sinks are uniformly distributed. Balanced CTS (standard approach in ASIC tools) builds a buffer tree bottom-up, clustering nearby sinks and inserting buffers to equalize delays. It handles irregular sink distributions much better and is the default for digital ASIC designs. A clock mesh provides the lowest skew (<5ps) by using a grid of wires with multiple drivers, but at 3-5x the power cost of tree-based CTS. Choose H-tree for array structures, balanced CTS for general ASIC logic, and mesh for high-performance processors where skew budget is extremely tight.

---

### Q13: What is Black's equation? How does EM impact PD decisions?

**A:** Black's equation models electromigration MTTF: MTTF = A × J^(-n) × exp(Ea/(kT)), where J is current density, n ≈ 1-2, Ea is activation energy, T is temperature. Higher current density and temperature exponentially reduce wire lifetime. PD impact: power straps must be wide enough to keep J below EM limits (typically 1 MA/cm^2 for DC). Signal wires carry less current but long-running high-toggle nets (clocks) need checking. Multi-cut vias distribute current across multiple via cuts. At the end, EM analysis (RedHawk/Voltus) generates a current density map and flags violations. Fixes include widening wires, adding parallel straps, reducing load (downsizing driver fanout), or adding more via cuts.

---

### Q14: How does fin quantization affect physical design at 7nm and below?

**A:** In FinFET technology, transistor drive strength is quantized by the number of fins (1, 2, 3, etc. fins), unlike planar CMOS where width is continuous. Standard cell height is defined in terms of fin pitches — common heights are 6T, 7.5T, 9T (tracks). A 6T cell has fewer routing tracks (fewer M2 tracks available above the cell), saving area but making routing harder. A 9T cell provides more tracks, easing routing and enabling higher performance but using more area. The PD engineer must choose the cell library height based on the block's requirements: 6T for area-critical mobile chips, 7.5T for balanced designs, 9T for high-frequency server chips. Additionally, cell sizing options are discrete (1-fin steps), meaning timing optimization has coarser granularity than planar CMOS.

---

### Q15: Walk me through a timing ECO flow for fixing a setup violation found in signoff STA.

**A:** Start by analyzing the violation in PrimeTime: identify the failing path, the amount of negative slack, and which stages contribute the most delay. Strategies in priority order: (1) Swap cells to faster VT (e.g., SVT → LVT), which increases leakage but reduces delay — check that the leakage budget isn't exceeded. (2) Upsize the driving cell (e.g., BUFX2 → BUFX8) to reduce stage delay — check that load on the previous stage doesn't worsen. (3) Insert a buffer to break a long wire into two shorter segments, reducing RC delay. (4) Reroute the critical net to a higher (thicker) metal layer for lower resistance. After the ECO, run incremental placement (if cells were added), incremental routing, and re-extract parasitics. Re-run STA to verify the fix doesn't create new violations. Then re-run DRC/LVS on the affected region.

---

### Q16: What is hierarchical PD? When would you use it vs flat implementation?

**A:** Hierarchical PD partitions a large SoC into blocks, each implemented independently and then integrated at the top level. Use it when the design exceeds 100-200M gates (flat tools struggle with runtime and memory) or when multiple teams work in parallel with different schedules. Each block gets a timing budget (interface constraints at its boundary) and is implemented as a separate PnR run. The top level stitches blocks together using their LEF abstracts (physical outlines + pin locations). Challenges include interface timing budgeting (if wrong, blocks don't integrate cleanly), cross-block clock tree balancing, power grid continuity, and feedthrough signals. Flat implementation gives globally optimal results but doesn't scale beyond ~200M gates with current tools. Most modern SoCs use a mix: large regular blocks (CPU, GPU) are hierarchical, while smaller glue logic is flat.

---

### Q17: Explain double patterning coloring constraints. What happens if there's a coloring conflict?

**A:** In LELE double patterning, features on one metal layer are split onto two photomasks (two colors). Adjacent features must be on different colors because same-color features need wider spacing (single-exposure resolution limit). A coloring conflict occurs when three features are mutually close — you can't assign colors to all three without violating same-color spacing. Example: three parallel wires each within minimum space of each other form a conflict. The fix requires the router to either: increase spacing for at least one pair (to allow same-color assignment), jog one wire to create distance, or re-route entirely. Some tools support pre-coloring critical nets. DRC checks include color-aware rules (same-color spacing vs different-color spacing) and are mandatory at 7nm and below.

---

### Q18: How do you verify that the power grid is adequate before starting placement?

**A:** After floorplanning and power grid creation, run early IR drop analysis using simplified models. Use a tool like RedHawk or Voltus with estimated power consumption (from synthesis power reports or activity assumptions). Check: (1) Static IR drop at every node is within budget (typically <5% VDD). (2) EM current density on every strap and via is below limits. (3) Power grid mesh has adequate connectivity (no broken straps or missing vias). Also check: ring resistance is low enough, via arrays at strap intersections have sufficient cuts, and decap cell plan provides enough local charge storage. If violations exist, iterate on strap width, pitch, number of metal layers used, and ring dimensions before committing to placement.

---

### Q19: What is CCD (Concurrent Clock and Data) optimization?

**A:** Traditional CTS builds the clock tree first, then freezes it and optimizes data paths. CCD allows the tool to simultaneously adjust both clock tree delays and data path delays during optimization. By selectively adding or removing clock tree delay at specific endpoints (useful skew), CCD can relax critical setup paths without requiring data path changes. This provides 5-10% better frequency compared to fixed-clock-then-optimize flows. The cost is longer runtime and increased hold buffer insertion (since skewing the clock creates new hold violations). CCD is available in ICC2 (as "clock concurrent optimization") and Innovus (as "CCOpt"). It's essential for high-frequency designs where traditional CTS + data optimization can't close timing.

---

### Q20: What are the key considerations for 3D IC physical design?

**A:** (1) TSV planning: TSVs are 5-10um diameter with 5-15um keep-out zones due to mechanical stress, consuming significant area. Need to carefully budget TSV count and locations. (2) Thermal: bottom die has higher thermal resistance (heat must pass through top die or interposer), requiring thermal-aware placement. (3) Power delivery: power must reach both dies through TSVs or micro-bumps; IR drop analysis becomes 3D. (4) Timing: TSV delay (~1-5ps) and capacitance (~50-200fF) must be modeled in STA. (5) Testing: access to bottom die is through top die; need test strategies for each die independently and combined. (6) Partitioning: deciding which logic goes on which die to minimize TSV count while meeting thermal and timing requirements. (7) Alignment: micro-bump pitch (40-100um) limits the density of inter-die connections.

---

### Q21: Describe a routing congestion scenario you've encountered and how you resolved it.

**A:** [Framework answer] A common scenario: a block with 16 SRAM macros has severe congestion in the channels between macros. The GRC map shows 150% overflow on horizontal M4 in the channel regions. Root cause: all data bus wires (256-bit bus) must route through narrow channels between macros. Resolution: (1) Widened channels by moving macros further apart (traded 3% area increase). (2) Applied cell padding of 2 sites in congested regions. (3) Added routing blockages on M2/M3 in the channels to push routing to higher layers. (4) Reoriented two macros so data pins faced different directions, distributing routing demand. (5) Used placement density constraints to prevent cells from filling the channels. Result: overflow reduced to 0%, with only 1% frequency impact from slightly longer wires.

---

### Q22: What is the difference between a halo and a blockage in macro placement?

**A:** A halo is a relative keep-out zone around a macro — it moves with the macro if the macro is repositioned. It's specified as a distance from each macro edge (e.g., 5um on all sides). It prevents standard cells from being placed too close to the macro, ensuring adequate routing channels for macro pins. A blockage is an absolute coordinate-based region where placement or routing is prohibited. Blockages don't move with any object. Hard placement blockages completely prevent cell placement; soft blockages discourage but allow placement if necessary; partial blockages limit density (e.g., 50% utilization). Routing blockages prevent routing on specific layers in a region. In practice, halos are used around macros, while blockages are used for fixed regions (analog areas, special routing reservations).

---

### Q23: How does scan chain reordering interact with placement?

**A:** Scan chains are inserted during DFT (Design for Testability) synthesis, creating a serial shift register through all scan FFs. Initially, the chain order is arbitrary, meaning physically distant FFs may be adjacent in the chain, creating long scan routing. Scan chain reordering (during or after placement) rearranges the chain to follow physical proximity, reducing total scan wire length by 30-60%. This improves congestion (fewer long wires) and hold timing in scan mode (shorter wires = less delay variation). The tool groups FFs in the same scan chain that are physically nearby and chains them in a serpentine pattern across the block. Constraint: must maintain the same number of scan chains and chain lengths. Tools like ICC2 and Innovus perform this automatically as part of the placement flow.

---

### Q24: How would you handle a scenario where post-route STA shows 200ps WNS on a path that was clean post-CTS?

**A:** This 200ps degradation from post-CTS to post-route indicates significant routing issues. Debug steps: (1) Check if the path has unexpected wire detours — congestion may have caused the router to take longer paths. (2) Check crosstalk: SI analysis may show coupling delay from adjacent aggressors. (3) Check extraction: compare post-CTS estimated wire delay vs post-route extracted delay. (4) Look at specific stages: identify which cell-to-cell segments degraded most. Fixes depend on root cause: if congestion-driven detour, apply routing guides or NDR for the critical net; if crosstalk, shield the victim net or add spacing; if extraction mismatch, the pre-route estimate was too optimistic (adjust RC estimation during placement). For 200ps, likely need a combination: route the critical net on higher metals (lower R), shield it, and possibly add a pipeline register (if architecture allows) or run CCD to redistribute the timing budget.

---

### Q25: Explain the sign-off flow: what checks must pass before tapeout?

**A:** All of the following must be clean (zero violations) for tapeout:

1. **STA**: All setup and hold timing met across all MCMM scenarios (10-30 corners × modes). POCV/AOCV derating applied. SI-aware analysis. Max transition and max capacitance clean.
2. **DRC**: Zero violations on all layers, including multi-patterning (LELE/SADP) rules, density checks, and antenna rules.
3. **LVS**: Layout matches schematic with zero errors. All devices match, all nets connected correctly.
4. **ERC**: No floating gates, no well connectivity issues, ESD path complete.
5. **IR drop**: Static IR < 5% VDD at all nodes. Dynamic IR within budget for worst-case switching scenarios.
6. **EM**: All wires and vias within current density limits per Black's equation for target lifetime (e.g., 10-year reliability).
7. **Metal density**: All layers within foundry-specified density range (with dummy fill).
8. **Formal verification (LEC)**: Post-layout netlist matches post-synthesis netlist.

Additionally: power analysis confirms total power within package/board thermal limits, and the GDSII file passes foundry's incoming inspection checks.

---

### Q26: What is the impact of VDD scaling on physical design at advanced nodes?

**A:** Lower VDD (e.g., 0.5V at 3nm vs 1.0V at 28nm) impacts PD in several ways: (1) Reduced noise margins — 5% IR drop budget at 0.5V is only 25mV, demanding a denser power grid with more metal consumed for power straps. (2) Slower gates — lower VDD increases gate delay, requiring either higher performance cells (more leakage) or additional pipeline stages. (3) Increased sensitivity to variation — at 0.5V, a 25mV VT variation is 5% of VDD (vs 2.5% at 1.0V), making OCV derating more aggressive. (4) SRAM Vmin concerns — SRAMs have minimum operating voltage that may be higher than logic Vmin, requiring separate SRAM power domains or assist circuits. (5) Leakage becomes a larger fraction of total power, making VT optimization (multi-VT) critical in placement/optimization.

---

## Additional Backend Topics

### Parasitic Extraction (SPEF)

```
Parasitic extraction: Convert physical layout geometry into electrical RC network

Tools: StarRC (Synopsys), Quantus QRC (Cadence), xACT (Mentor/Siemens)

SPEF (Standard Parasitic Exchange Format):
  Industry standard file format for parasitics

  *D_NET data_bus[0] 0.4523    // net name, total cap (pF)
  *CONN
  *P data_bus[0] I              // primary port, Input
  *I u_reg/D I *C 0.012 0.005   // instance pin, input, pin cap rise/fall
  *CAP
  1 u_reg/D 0.0234              // grounded cap
  2 u_buf/Z 0.0156              // grounded cap
  3 u_reg/D u_buf/Z 0.0089      // coupling cap (for SI analysis)
  *RES
  1 u_buf/Z n1 12.5             // resistance segment (Ω)
  2 n1 u_reg/D 8.3
  *END

RC models:
  Lumped C:        Single capacitor (simplest, least accurate)
  Distributed RC:  Multiple RC segments (π-model, T-model)
  Coupled RC:      Includes coupling caps to adjacent nets (for SI)

  π-model:                    T-model:
    ──┤├──┤├──┤├──              ──┤├──R──┤├──
      C/2  R   C/2                  C    R    (distributed)

Extraction corners:
  RCbest  (Cmin, Rmin): Fastest RC → worst hold
  RCworst (Cmax, Rmax): Slowest RC → worst setup
  Cworst  (Cmax, Rmin): Maximum crosstalk noise
  Rcworst (Rmax, Cmax): Combined worst case
  
  Typically run 3-5 extraction corners × PVT corners for MCMM analysis
```

### Advanced Parasitic Considerations

```
1. Field solver vs pattern matching:
   Pattern matching: Pre-characterizes common geometries, fast (~1 hour for 50M gates)
   Field solver: 3D Maxwell equations, slow but accurate (~10-100× slower)
   Used at 7nm and below where pattern matching accuracy degrades

2. Resistance at advanced nodes:
   Cu resistivity at narrow widths >> bulk (grain boundary + surface scattering)
   At 28nm M1 pitch: R_sheet ≈ 100+ mΩ/□ (vs bulk Cu ~17 mΩ/□)
   Extraction tools use width-dependent resistivity models

3. Via resistance:
   Single-cut via: 2-10 Ω per via (technology dependent)
   Multi-cut via: R_eff = R_via / N_cuts
   Via arrays at strap intersections: model each cut separately

4. Coupling capacitance accuracy:
   Critical for SI-aware STA
   Must capture coupling to ALL adjacent nets (not just nearest)
   Coupling window: typically 3-5 neighboring tracks on each side
```

### Design for Manufacturability (DFM) Deep Dive

```
DFM is the practice of designing layouts that are robust to manufacturing variation
and maximize yield, beyond minimum DRC compliance.

1. Recommended rules (RR):
   - Minimum DRC = "will it fabricate?"
   - Recommended rules = "will it yield well?"
   - Example: min wire width = 20nm (DRC), recommended = 24nm (RR)
   - Following RR gives ~5-20% yield improvement

2. Via redundancy (critical for reliability):
   - Single-cut via failure rate: 0.01-0.1% per via
   - In a design with 100M vias: expect 10K-100K single-cut failures
   - Double via: fail rate = (0.001)² ≈ 0 (effectively zero)
   - Post-route via doubling: automatically insert redundant vias where space permits
   - Target: >95% of vias should be multi-cut

3. Litho hotspot detection:
   - Run lithography simulation (model-based OPC check) on layout
   - Identify features that are near the edge of process window
   - Fix: adjust spacing, add dummy fill, change layer
   - Tools: Calibre LFD, IC Validator DFM

4. Metal fill (dummy fill):
   - Purpose: Equalize metal density across die for uniform CMP
   - Timing-aware fill: avoid adding fill too close to timing-critical nets
     (fill adds coupling capacitance → crosstalk)
   - Typical density target: 30-70% per metal layer per window
   - Fill is inserted AFTER routing but BEFORE final extraction
   - Must re-extract and re-run STA after fill insertion

5. Wire spreading and widening:
   - After routing, spread wires in uncongested regions
   - Wider wire = lower R (performance), lower EM risk (reliability)
   - Wider spacing = lower Cc (SI improvement)
   - Automated in ICC2/Innovus post-route optimization

6. OPC (Optical Proximity Correction) awareness:
   - Designers don't do OPC, but layout choices affect OPC complexity
   - Tip-to-tip spacing: too close → OPC can't resolve → yield loss
   - Jogs: short jogs harder to print than smooth lines
   - Line ends: need sufficient extension for OPC to anchor patterns
```

### Multi-Patterning Coloring Constraints

```
At 7nm and below, SADP/LELE requires "coloring" — assigning each feature to a mask.

Rules:
  - Features closer than single-exposure resolution must be on DIFFERENT masks
  - Features on the same mask must be spaced ≥ single-exposure minimum pitch

  Example (SADP for M2 at 7nm):
    Track pitch = 28nm → single-exposure pitch = 56nm
    Adjacent tracks MUST alternate colors (masks A and B)

    Track:  1  2  3  4  5  6  7  8
    Color:  A  B  A  B  A  B  A  B

  Coloring conflict: When a net routes on adjacent tracks, it creates a
  same-net same-mask requirement that conflicts with color alternation
  → Route must jog or change layer

  Design impact:
    - Routing is constrained (not all track assignments are valid)
    - Jogs have minimum length requirements
    - Router must be color-aware (knows legal track assignments)
    - Unidirectional routing preferred (horizontal or vertical per layer, no diagonal)
    - Design rules become more complex (~5000+ rules at 7nm vs ~500 at 65nm)

For LELE:
  - Must ensure all features are 2-colorable
  - Odd cycles in layout → coloring conflict → DRC error
  - Must fix in routing (stitching, layer change, spacing increase)
```

### Advanced CTS Topics

```
1. Clock Mesh:
   - Grid of clock wires covering entire block
   - All mesh points shorted → inherent low skew
   - Requires buffers/drivers at mesh entry points
   - Very high power consumption (large capacitance)
   - Used for ultra-high-performance designs (CPUs, GPUs)

   ═══VDD═══════════════════════════
   ║   ║   ║   ║   ║   ║   ║   ║   (vertical clock mesh)
   ╠═══╬═══╬═══╬═══╬═══╬═══╬═══╣   (horizontal clock mesh)
   ║   ║   ║   ║   ║   ║   ║   ║
   ╠═══╬═══╬═══╬═══╬═══╬═══╬═══╣
   ║   ║   ║   ║   ║   ║   ║   ║

2. Concurrent Clock and Data (CCD):
   - Traditional: Build clock tree first, then optimize data paths
   - CCD: Jointly optimize clock skew AND data paths
   - Intentionally adds useful skew to borrow time for critical paths
   - Can recover 5-10% frequency without RTL changes
   - set_clock_skew_adjustment in ICC2/Innovus

3. Multi-source CTS:
   - For clock meshes: multiple buffers drive the mesh from different points
   - Reduces maximum insertion delay and skew
   - Requires careful balancing of drive points

4. CTS for multi-voltage designs:
   - Clock may cross voltage domains → need level shifters in clock path
   - Level shifters add delay and jitter → must be modeled in CTS
   - ICG cells in clock path: affects CTS topology

5. Post-CTS hold fixing:
   - After CTS, hold timing is checked (fast-corner extraction)
   - Hold violations fixed by inserting delay buffers on short paths
   - Must not create new setup violations → iterative process
   - Critical: hold buffers should be placed NEAR the launch/capture FFs
     to minimize sensitivity to OCV
```

### ECO (Engineering Change Order) Flow

```
Types of ECOs:

1. Functional ECO:
   - Fix RTL bugs found late in the flow (post-layout)
   - Modified netlist → minimal physical changes
   - Goal: Change as few cells as possible (minimize mask cost)
   - Spare cell approach: Pre-placed unused cells → can be repurposed
     without changing placement
   - Metal-only ECO: Only change metal layers (cheapest mask set)
     Requires spare cells in the right locations

2. Timing ECO:
   - Fix timing violations found during signoff
   - Cell swaps: Replace with higher-drive or lower-Vt cell
   - Buffer insertion: Add buffers on long nets
   - Gate cloning: Duplicate a high-fanout cell
   - Useful skew: Adjust clock buffer placement
   - All changes must be legalized (no overlap, on-grid)

3. ECO flow:
   Read post-route design → Apply netlist changes →
   ECO placement (place new cells in whitespace) →
   ECO routing (connect new cells, minimum wire disturbance) →
   Re-extract → Re-run STA → Re-verify LEC, DRC, LVS

4. Metal-only ECO mask costs:
   Full mask set: $5-15M at 7nm (60+ masks)
   Metal-only ECO: $1-3M (only metal layer masks changed)
   → Huge cost savings if ECO can be done in metal layers only
```

---

*End of Physical Design Deep Dive*
