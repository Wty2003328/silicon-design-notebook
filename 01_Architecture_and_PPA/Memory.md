# Memory Architecture and Design — Senior Engineer Level

> **Prerequisites:** [CMOS_Fundamentals](../00_Fundamentals/CMOS_Fundamentals.md) (transistor physics, 6T SRAM cell)
> **See also:** [Cache_Microarchitecture](Cache_Microarchitecture.md), [DDR_Controller](DDR_Controller.md), [Floating_Point](../00_Fundamentals/Floating_Point.md) (ECC)

## 6T SRAM Cell — Transistor-Level Deep Dive

### 6T SRAM Bitcell — M1 through M6 Transistor Roles

The 6T SRAM cell uses six transistors organized as two cross-coupled CMOS inverters
(storage) plus two access transistors (read/write port):

| Transistor | Type | Role |
|---|---|---|
| M1 | NMOS | Pull-down for INV1. Holds Q=0 when Q_bar=1. Sized STRONG (W/L=2/1) for read stability (high CR). |
| M2 | NMOS | Pull-down for INV2. Holds Q_bar=0 when Q=1. Same sizing as M1. |
| M3 | PMOS | Pull-up for INV1. Drives Q_bar to VDD when Q=0. Sized WEAK (W/L=1/1) to allow write to overpower it (high PR). |
| M4 | PMOS | Pull-up for INV2. Drives Q to VDD when Q_bar=0. Same sizing as M3. |
| M5 | NMOS | Access transistor for BL (connects BL to Q). Gate = WL. Sized INTERMEDIATE (W/L=1.5/1). Must be weaker than M1 (CR > 1) but stronger than M3 (PR > 1). |
| M6 | NMOS | Access transistor for BL_bar (connects BL_bar to Q_bar). Gate = WL. Same sizing as M5. |

**Sizing ratio constraints (the "6T triangle"):**

- **Cell Ratio (CR)** = `(W/L)_pulldown / (W/L)_access = 2.0 / 1.5 = 1.33`
   - CR > 1 required for read stability (M1 must overpower M5 during read)
   - Typical: CR = 1.2 - 2.0

- **Pull-up Ratio (PR)** = `(W/L)_access / (W/L)_pullup = 1.5 / 1.0 = 1.5`
   - PR > 1 required for write-ability (M5 must overpower M3 during write)
   - Typical: PR = 1.2 - 2.0

**Fundamental tension:**
   - Increasing access transistor strength (M5/M6) → improves PR (write) → degrades CR (read)
   - Must find a sizing that gives adequate margin for BOTH simultaneously.
   - This becomes harder at low VDD where transistor I-V curves compress.

### Read Operation Sequence (Step-by-Step Timing)

Reading a 6T SRAM cell follows this precise sequence. We read from a cell storing Q=0, Q_bar=1:

1. **PRECHARGE** (t_precharge ~200-500 ps)
   - Precharge circuit drives both BL and BL_bar to VDD
   - Equalize transistor (between BL and BL_bar) ensures BL = BL_bar = VDD
   - WL remains LOW (M5, M6 off)
2. **WORD LINE ASSERTION** (t_WL_rise ~50-100 ps)
   - WL goes HIGH, turning on access transistors M5 and M6
   - BL connects to Q through M5; BL_bar connects to Q_bar through M6
3. **BITLINE DEVELOPMENT** (t_develop ~100-300 ps)
   - Cell has Q=0: BL (at VDD) begins to discharge through M5 and M1 toward GND
   - Cell has Q_bar=1: BL_bar (at VDD) stays near VDD (M6 connects to Q_bar=VDD, so no discharge path)
   - Differential voltage develops: BL drops, BL_bar stays high
   - Delta_V grows to ~100-200 mV (enough for sense amplifier)
4. **SENSE AMPLIFIER FIRING** (t_sense ~100-200 ps)
   - Sense amplifier enable signal goes HIGH
   - Cross-coupled latch amplifies the small differential to full rail: BL → 0 (GND), BL_bar → VDD
   - Positive feedback completes in ~100 ps
5. **DATA OUTPUT and WRITE-BACK**
   - Sense amplifier output (BL_bar) is the read data (logic "1" = cell stored Q_bar)
   - Full-rail voltage on BL/BL_bar is also driven back through M5/M6, reinforcing the cell's stored value (non-destructive read -- cell was already at Q=0)
   - Column mux selects this bitline pair for output to the read data bus
6. **WORD LINE DE-ASSERTION**
   - WL goes LOW, disconnecting cell from bitlines
   - Precharge circuit re-asserts, preparing for next read

Total read access time: t_precharge + t_WL_rise + t_develop + t_sense ≈ 0.5-1.5 ns

**Critical observation:** The read is non-destructive (unlike DRAM). The cell's cross-coupled
inverters fight the bitline discharge and maintain their state. But the cell voltage does
dip slightly during Step 3, which is the source of read-disturb risk at low VDD.

### Write Operation Sequence (Step-by-Step Timing)

Writing a 6T SRAM cell requires overpowering the feedback loop. We write Q=0 to a cell
currently storing Q=1:

1. **DRIVE BITLINES** (write driver activation ~50-100 ps)
   - Write driver forces BL = 0 (GND) for writing "0" to Q
   - Write driver forces BL_bar = VDD (reinforcing Q_bar = 1)
   - BL and BL_bar are driven STRONGLY by the write driver (much larger transistors than the cell's pull-up/pull-down)
2. **WORD LINE ASSERTION**
   - WL goes HIGH, connecting BL to Q (through M5) and BL_bar to Q_bar (through M6)
3. **FLIP Q NODE** (t_write ~200-500 ps)
   - BL = 0 pulls Q down through M5 (access transistor)
   - M3 (PMOS pull-up, gate = Q_bar = 0 initially, so M3 is ON) fights back, trying to keep Q at VDD
   - M5 must overpower M3: requires PR = (W/L)_access / (W/L)_pullup > threshold
   - With PR = 1.5, M5 wins: Q drops below the switching threshold of INV1 (Q_bar inverter)
   - INV1 input (Q) drops → INV1 output (Q_bar) rises → M3 gate goes HIGH → M3 turns OFF
   - Feedback is broken: Q continues to drop to 0, Q_bar rises to VDD
   - Cell has flipped: Q=0, Q_bar=1
4. **STABILIZATION**
   - Cross-coupled inverters reinforce the new state
   - Q_bar = VDD keeps M1 (pull-down) ON, holding Q = 0
   - Q = 0 keeps M4 (pull-up) ON, holding Q_bar = VDD
5. **WORD LINE DE-ASSERTION**
   - WL goes LOW, disconnecting cell from bitlines
   - Cell retains new state

Total write time: t_drive + t_WL + t_flip ≈ 0.5-2.0 ns

(longer than read because of the feedback fight)

**Write margin:** The worst case is writing the opposite state (flipping the cell). If VDD
is too low, M3 may overpower M5, preventing the flip. Write margin is defined as the
minimum VDD at which the write succeeds. Typical write margin: 0.5-0.6V (below nominal
VDD of 0.7-1.0V). Write-assist techniques (boosting WL voltage above VDD, or collapsing
the cell's VDD during write) can improve this margin.

**Read stability (formal):** During read, the "0" storage node forms a voltage divider
between M5 (access, in saturation) and M1 (pull-down, in linear region). The node voltage
rises to:

$$
V_Q \approx \frac{1}{2 \cdot CR} \cdot (V_{DD} - V_{th})
$$

For the cell to remain stable, $V_Q$ must be below the switching threshold of the
feedback inverter ($V_m \approx 0.4 \cdot V_{DD}$). With our sizing:

$$
V_Q = \frac{1}{2 \times 1.33} \times (1.0 - 0.3) = 0.263\,\text{V} < 0.4\,\text{V} \quad \checkmark
$$

Read margin $= V_m - V_Q = 0.4 - 0.263 = 0.137\,\text{V}$. At 3-sigma process corner
(worst-case mismatch), this margin can shrink to 20-50 mV. Below VDD = 0.7V, the margin
vanishes for the 6T cell, motivating the switch to 8T.

**Write margin (formal):** During write, M5 (access) must overpower M3 (PMOS pull-up)
to pull the storage node below the inverter switching threshold:

$$
V_Q = V_{DD} \cdot \frac{R_{M5}}{R_{M5} + R_{M3}}
$$

For write success: $V_Q < V_m$. This requires $R_{M3} > R_{M5} \cdot \frac{V_{DD} - V_m}{V_m}$,
which translates to PR > 1. With PR = 1.5, write margin is adequate at nominal VDD.
Write margin degrades at low VDD because PMOS on-resistance increases faster than
NMOS as VDD approaches Vth.

### Transistor Schematic with Sizing

```ascii-graph
                   VDD                    VDD
                    |                      |
               ┌────┴────┐           ┌────┴────┐
               │ M3 PMOS │           │ M4 PMOS │
               │ W/L=1/1 │           │ W/L=1/1 │
               └────┬────┘           └────┬────┘
                    │                      │
          Q ────────┼──────────────────────┼──────── Q_bar
                    │         ┌────┐       │
                    ├─────────┤INV2├───────┤
                    │         └────┘       │
                    │         ┌────┐       │
                    ├─────────┤INV1├───────┤
                    │         └────┘       │
               ┌────┴────┐           ┌────┴────┐
               │ M1 NMOS │           │ M2 NMOS │
               │ W/L=2/1 │           │ W/L=2/1 │
               └────┬────┘           └────┬────┘
                    │                      │
                   GND                    GND
                    
       BL                                     BL_bar
        │                                       │
   ┌────┴────┐                             ┌────┴────┐
   │ M5 NMOS │                             │ M6 NMOS │
   │ W/L=1.5/1│                            │ W/L=1.5/1│
   └────┬────┘                             └────┬────┘
        │                                       │
        └───────────── Q         Q_bar ─────────┘
                    
   WL ──── gates of M5 and M6

   INV1: M3 (PMOS pull-up) + M1 (NMOS pull-down), output = Q_bar, input = Q
   INV2: M4 (PMOS pull-up) + M2 (NMOS pull-down), output = Q, input = Q_bar
```

**Typical sizing ratios (65nm reference):**
```verilog
Pull-down NMOS (M1, M2):  W/L = 2/1    (strongest, for read stability)
Access NMOS (M5, M6):     W/L = 1.5/1  (intermediate)
Pull-up PMOS (M3, M4):    W/L = 1/1    (weakest, for write-ability)

Cell Ratio (CR) = (W/L)_pulldown / (W/L)_access = 2/1.5 = 1.33
Pull-Up Ratio (PR) = (W/L)_access / (W/L)_pullup = 1.5/1 = 1.5
```

### Read Stability — Voltage Divider Derivation

**Setup:** Cell stores Q=0 (M1 on, M3 off), Q_bar=1 (M2 off, M4 on). BL and BL_bar pre-charged to VDD. WL asserted.

**The problem:** BL is at VDD, Q is at 0V. When M5 turns on, BL (at VDD) connects to Q (at 0V) through M5. But Q is also held at 0V by M1 (pull-down NMOS, which is ON because Q_bar = VDD = gate of M1).

**Voltage divider between M5 (access) and M1 (pull-down):**

```ascii-graph
                BL (VDD)
                  │
             ┌────┴────┐
             │ M5 NMOS │  R_access = VDD / (Kn_access * (VDD - Vth)^2 / 2)
             │ (access) │     (in saturation initially)
             └────┬────┘
                  │
            Q node  ← this voltage rises during read
                  │
             ┌────┴────┐
             │ M1 NMOS │  R_pulldown = similar but with different W/L
             │(pulldown)│
             └────┬────┘
                  │
                 GND
```

**Simplified DC analysis (both in linear region for small V_Q):**

```verilog
I_M5 = Kn_access * [(VDD - Vth) * V_Q - V_Q^2/2]
I_M1 = Kn_pulldown * [(VDD - Vth) * V_Q - V_Q^2/2]
```

Wait — M1's gate is at Q_bar = VDD (assuming Q_bar hasn't changed), and M1's source is GND, drain is Q.
M5's gate is at VDD (WL=VDD), source is Q (lower voltage), drain is BL (VDD).

For M5 (in saturation if VDD - V_Q > VDD - Vth, i.e., V_Q < Vth... initially yes):
```verilog
I_M5 ≈ (Kn_access/2) * (VDD - Vth)^2   (in saturation)
```

For M1 (in linear region since $V_{GS} = V_{DD}$ and $V_{DS} = V_Q \approx 0$):
```verilog
I_M1 ≈ Kn_pulldown * [(VDD - Vth) * V_Q - V_Q^2/2]
```

At equilibrium, I_M5 = I_M1:
```verilog
(Kn_access/2) * (VDD - Vth)^2 = Kn_pulldown * (VDD - Vth) * V_Q

V_Q = (Kn_access / (2 * Kn_pulldown)) * (VDD - Vth)
    = (1 / (2 * CR)) * (VDD - Vth)
```

**Numerical example with 65nm parameters:**
```verilog
VDD = 1.0V, Vth = 0.3V, CR = 1.33

V_Q = (1 / (2 * 1.33)) * (1.0 - 0.3) = 0.375 * 0.7 = 0.263V
```

**Is this safe?** For the cell to flip, V_Q must exceed the switching threshold of INV2 (which has Q as its input). The switching threshold of INV2 ≈ VDD * sqrt(Kn_pd/Kp_pu) / (1 + sqrt(Kn_pd/Kp_pu)). For our sizing with pull-down 2x stronger than pull-up, Vm_inv ≈ 0.4V.

$V_Q = 0.263$ V $< V_{m,inv} = 0.4$ V → **SAFE** (with margin of 0.137V).

**When does the cell flip?** If CR is reduced:
```ascii-graph
CR = 1.0: V_Q = 0.5 * 0.7 = 0.35V  (margin = 0.05V — dangerously close!)
CR = 0.8: V_Q = 0.625 * 0.7 = 0.44V > 0.4V → CELL FLIPS! Read-disturb failure!
```

**This is why CR > 1 is essential.** Typical designs use CR = 1.2-2.0 for adequate read margin.

### Write-Ability — Pull-Up vs Access Transistor Fight

**Setup:** Cell stores Q=1, Q_bar=0. We want to write Q=0. Drive BL=0 (strong driver), BL_bar=1. Assert WL.

**The fight:** M5 (access, WL=VDD) tries to pull Q from VDD to 0 through BL=0. But M3 (PMOS pull-up, gate=Q_bar=0, so M3 is ON) tries to keep Q at VDD.

```ascii-graph
                VDD
                 │
            ┌────┴────┐
            │ M3 PMOS │  gate = Q_bar = 0 → M3 is ON
            │ (pull-up)│  R_pullup
            └────┬────┘
                 │
           Q node ← being pulled down
                 │
            ┌────┴────┐
            │ M5 NMOS │  gate = WL = VDD, drain = Q, source = BL = 0
            │ (access) │  R_access
            └────┬────┘
                 │
              BL = 0 (driven by write driver)
```

For write to succeed: M5 must overpower M3, pulling Q below the switching threshold of INV1.

**Voltage divider:**
```verilog
V_Q = VDD * R_access / (R_access + R_pullup)

For V_Q < Vm_inv (must pull Q below switching threshold):
R_access / (R_access + R_pullup) < Vm_inv / VDD
R_pullup > R_access * (VDD - Vm_inv) / Vm_inv
```

This translates to: `(W/L)_access / (W/L)_pullup > some threshold`, which is the Pull-Up Ratio (PR).

**Numerical:** With PR = 1.5 (our sizing), M5 is 1.5x stronger than M3. The access transistor wins, pulling Q low enough to trip INV1, which then drives Q_bar high, which turns off M3 (breaking the fight), and the cell flips. Write succeeds.

**If PR < 1 (weak access, strong pull-up):** M3 wins, Q stays high, write fails.

### The Fundamental 6T Trade-Off

```ascii-graph
Read stability wants: Strong pull-down, WEAK access → HIGH CR
Write-ability wants:  Strong access, WEAK pull-up   → HIGH PR

But increasing access transistor strength IMPROVES PR and DEGRADES CR!
```

**This is THE fundamental tension in 6T SRAM design.** In advanced nodes (7nm, 5nm) with VDD scaling, both margins shrink, and balancing them becomes extremely challenging. This is why 8T cells are increasingly used.

### Butterfly Curve and Static Noise Margin (SNM)

The butterfly curve is constructed by plotting the voltage transfer characteristics (VTC) of the two cross-coupled inverters:

- **INV1** — V_Qbar = f(V_Q)      (input Q, output Q_bar)
- **INV2** — V_Q = g(V_Qbar)      (input Q_bar, output Q)

- **Plot both on the same axes** — V_Q (x-axis) vs V_Qbar (y-axis)
- **INV1** — V_Qbar = f(V_Q)     → plot normally
- **INV2** — V_Q = g(V_Qbar)     → plot as V_Qbar vs V_Q (mirror)

The two curves intersect at three points: two stable states (Q=0, Qbar=VDD) and (Q=VDD, Qbar=0), and one metastable point (both ≈ VDD/2).

**SNM = side of the largest square that fits inside either "eye" of the butterfly curve.**

```ascii-graph
                V_Qbar
                  |
            VDD --+              /--------+
                  |            /          |
                  |          /  ← largest |
                  |        /    square    |
                  |      /   ┌─────┐     |  ← SNM
                  |    /     │     │     |
                  |  /       └─────┘     |
                  |/                      |
                  +--------+------ VDD → V_Q
                  |       /
                 ...     ...
```

**SNM values across operating conditions:**

| Condition      | VDD  | Temperature | SNM (mV) | Notes                    |
|---------------|------|-------------|----------|--------------------------|
| Hold (WL off)  | 1.0V | 25°C       | ~280     | Best case, cell isolated  |
| Read (WL on)   | 1.0V | 25°C       | ~180     | Worst operating case      |
| Read (WL on)   | 0.8V | 125°C      | ~90      | Marginal — risk of failure|
| Read (WL on)   | 0.6V | 125°C      | ~20      | Essentially zero margin   |

**This is why low-VDD SRAM is so hard.** At 0.6V, the read SNM nearly vanishes. Solutions: 8T cell, read-assist (wordline voltage reduction), or write-assist (supply boosting during write).

---

## 8T and 10T SRAM Cells

### 8T SRAM Cell — Read-Decoupled Port

```ascii-graph
Standard 6T write port (same as above):
  M1-M6 as before, with WL_write controlling M5, M6

Additional 2T read port:
                    Read BL (RBL)
                        │
                   ┌────┴────┐
                   │ M8 NMOS │  gate = Read WL (RWL)
                   └────┬────┘
                        │
                   ┌────┴────┐
                   │ M7 NMOS │  gate = Q_bar (stored complement)
                   └────┬────┘
                        │
                       GND
```

**Read operation:**
1. Pre-charge RBL to VDD
2. Assert RWL (M8 turns on)
3. If Q_bar = 1 (i.e., Q = 0): M7 is ON → RBL discharges to GND through M8+M7 → read "0"
4. If Q_bar = 0 (i.e., Q = 1): M7 is OFF → RBL stays at VDD → read "1"

**Key advantage:** The read path (M7, M8) is completely separated from the storage nodes (Q, Q_bar). Reading does NOT disturb the cell content. The read SNM equals the hold SNM (much higher than 6T read SNM).

**Write operation:** Same as 6T — uses the original access transistors M5, M6.

**Why 8T is preferred for low-VDD:**
```verilog
6T at VDD = 0.6V: Read SNM ≈ 20 mV (essentially zero)
8T at VDD = 0.6V: Read SNM ≈ 250 mV (same as hold SNM, very robust)
```

**Disadvantage:** ~30% larger area. The two extra transistors plus the separate read bitline increase cell size from ~0.05 um^2 to ~0.065 um^2 (in 7nm).

### 10T SRAM Cell

Adds a differential read port (2 extra transistors on top of 8T, using both Q and Q_bar for read). This provides faster read (differential sensing) with the same read-decoupled advantage. Used in register files where speed is critical.

### SRAM Yield and Repair

Manufacturing defects in SRAM arrays are the primary yield limiter for large SoCs. Because SRAM bitcells use the smallest feature sizes (minimum metal pitch, minimum gate length), they are disproportionately susceptible to random defects.

**Redundancy strategy:** Add spare rows (typically 2--4 per bank) and spare columns (2--8). At wafer test, defective rows/columns are replaced by fusing the address remapping.

- **Laser fuse:** permanent, blown during wafer test. High reliability, no post-fuse modification.
- **eFUSE:** electrically programmable. Can be done at package test or even in-field (for late-life repair). Smaller area but slightly less reliable than laser.

**Yield model for SRAM with repair:**

$$Y_{\text{repaired}} = \sum_{k=0}^{R} \frac{(D \times A)^k}{k!} \times e^{-D \times A}$$

where $R$ is the number of spares, $D$ is defect density, and $A$ is array area.

**Example:** 1 Mb SRAM at N5, $D = 0.3/\text{cm}^2$, cell area $= 0.05\;\mu\text{m}^2$, array area $\approx 0.05\;\text{mm}^2$. Without repair: $Y = e^{-0.3 \times 0.05} = 98.5\%$. But a large 64 MB L3 cache with 64 banks has much lower yield per bank; aggregate yield without repair can be $< 50\%$. With 4 spare rows per bank, yield recovers to $> 90\%$.

### SRAM Retention Voltage

The retention voltage ($V_{\text{retain}}$) is the minimum VDD at which the 6T cell retains data. Below this voltage, the cross-coupled inverter noise margin collapses and the stored bit can flip.

- **Typical retention voltage:** $0.4$--$0.5 \times V_{\text{DD,nominal}}$ (e.g., 0.3 V for N5 at nominal 0.7 V).
- **Trade-off:** lower retention voltage reduces leakage but increases susceptibility to soft errors and SNM degradation at high temperature.
- **Power-gating sequence:** flush pending writes $\to$ assert retention signal $\to$ lower VDD to $V_{\text{retain}}$ $\to$ ... $\to$ raise VDD $\to$ deassert retention $\to$ resume operation. The transition takes 5--50 $\mu$s.

### eDRAM (Embedded DRAM)

The 1T1C cell (one transistor + one capacitor) is much smaller than a 6T SRAM cell: $\sim 0.02\;\mu\text{m}^2$ at N5, roughly 3--4x denser. The trade-off is the need for periodic refresh (every 2--8 ms), consuming bandwidth and power.

| Attribute | SRAM (6T) | eDRAM (1T1C) |
|-----------|-----------|--------------|
| Cell area (N5) | ~0.05 $\mu\text{m}^2$ | ~0.02 $\mu\text{m}^2$ |
| Access latency | 1--3 ns | 5--10 ns |
| Refresh required | No | Every 2--8 ms |

**Use cases:** IBM POWER L4 caches, Intel server processor eDRAM L4 (Crystalwell). eDRAM is attractive as a last-level cache where density matters more than latency. The access latency is 2--4x slower than SRAM (access + restore cycle), but the much larger capacity per unit area makes it viable for large shared caches.

### HBM (High Bandwidth Memory) and GDDR

**HBM:** 3D-stacked DRAM with a wide interface (1024-bit per stack), 3--8 stacks per GPU. HBM3E delivers 960 GB/s per stack with 36 GB capacity. Connected via TSV interposer. Used in AI accelerators (H100, B200).

**GDDR6X:** 384-bit interface per GPU, 24 Gbps per pin, $\sim$1.15 TB/s. Used in gaming/graphics GPUs (RTX 4090). Much cheaper than HBM but lower bandwidth.

**Key difference:** HBM trades pin count for per-pin speed (wide + slow vs. narrow + fast). HBM per-pin speed: $\sim$6 Gbps. GDDR6X: $\sim$24 Gbps.

### SRAM Soft Error Rate (SER)

Cosmic rays (neutrons) and alpha particles (from packaging materials) can flip SRAM bits. SER $\approx$ 100--1000 FIT/Mb at terrestrial altitude (1 FIT = 1 failure per $10^9$ device-hours).

**Mitigation strategies:**

- **ECC:** SECDED corrects 1-bit, detects 2-bit errors. Overhead: 7+1=8 bits per 64-bit word (12.5% for SECDED). For a 1 MB cache: 128 KB of ECC storage.
- **Parity with scrubbing:** periodic read-check-correct cycle. Prevents accumulation of single-bit errors that could exceed ECC correction capability.
- **Physical shielding:** overburden concrete for data centers reduces neutron flux by 2--10x depending on depth.

---

## DRAM — Detailed Design

### 1T1C Cell — Charge Sharing Equation

```ascii-graph
    BL (precharged to VDD/2)
     │
  ┌──┴──┐
  │NMOS │  gate = WL
  └──┬──┘
     │
    [C_s]  storage capacitor (Cs ≈ 20-30 fF)
     │
    V_plate (= VDD/2, cell plate common to all cells)
```

**Read — charge sharing:**

Before WL assert: V_BL = VDD/2, V_Cs = VDD (stored "1") or 0 (stored "0").

After WL assert, charge sharing between Cs and C_BL (bitline parasitic, ~200-400 fF):
```verilog
Q_total = C_s * V_Cs + C_BL * V_BL(precharge)

V_BL_final = Q_total / (C_s + C_BL)
           = (C_s * V_Cs + C_BL * VDD/2) / (C_s + C_BL)
```

**For stored "1" (V_Cs = VDD):**
```verilog
V_BL_final = (C_s * VDD + C_BL * VDD/2) / (C_s + C_BL)
           = VDD/2 + (C_s / (C_s + C_BL)) * VDD/2
           = VDD/2 + delta_V

delta_V = C_s / (C_s + C_BL) * VDD/2
```

**For stored "0" (V_Cs = 0):**
```verilog
V_BL_final = (C_s * 0 + C_BL * VDD/2) / (C_s + C_BL)
           = VDD/2 - (C_s / (C_s + C_BL)) * VDD/2
           = VDD/2 - delta_V
```

**Numerical example:**
```verilog
Cs = 25 fF, C_BL = 250 fF, VDD = 1.0V

delta_V = 25/(25+250) * 0.5 = 25/275 * 0.5 ≈ 45.5 mV
```

Only ~45 mV of signal! This is why DRAM needs sensitive sense amplifiers.

### Sense Amplifier — Cross-Coupled Latch

```ascii-graph
          VDD
           │
      ┌────┴────┐     ┌────┴────┐
      │ MP1 PMOS│     │ MP2 PMOS│
      └────┬────┘     └────┬────┘
           │               │
    BL ────┼───────────────┼──── BL_bar
           │               │
      ┌────┴────┐     ┌────┴────┐
      │ MN1 NMOS│     │ MN2 NMOS│
      └────┬────┘     └────┬────┘
           │               │
          GND (via enable switch)
```

**Operation:**
1. BL and BL_bar develop a small differential (±45 mV)
2. Enable the sense amp (connect to VDD and GND)
3. Positive feedback amplifies the differential:
   - If BL > BL_bar: MN1 conducts more → BL_bar pulled down → MP1 gate goes low → MP1 conducts more → BL pulled up → reinforces
4. Within ~1 ns, BL → VDD and BL_bar → 0 (or vice versa)

**This is a destructive read:** The original charge on Cs is disturbed by charge sharing. The sense amplifier must write the amplified value back to the cell (**restore operation**).

### Sense Amplifier Design -- Voltage-Mode vs Current-Mode

#### Voltage-Mode Sensing (Most Common)

The standard cross-coupled latch sense amplifier described above operates in **voltage
mode**. It detects the voltage difference between BL and BL_bar after charge sharing.

**Bitline swing time calculation:**

```verilog
The sense amplifier detects when delta_V >= V_sense_threshold (~10-20 mV).

Bitline development time (voltage-mode):
  t_dev = C_BL * V_sense_threshold / I_cell

where:
  C_BL = total bitline capacitance (~200-400 fF for 256-512 rows)
  I_cell = cell read current through access transistor (~20-50 uA in 7nm)
  V_sense_threshold = minimum detectable voltage (~15 mV)

t_dev = 300 fF * 15 mV / 30 uA = 150 ps

But this is only the detection time. Full rail-to-rail amplification:
  t_amp = C_BL * VDD / I_sense_amp

  I_sense_amp = ~100-200 uA (sense amp transistor drive)
  t_amp = 300 fF * 1.0V / 150 uA = 2.0 ns

Total sense time = t_dev + t_amp ≈ 150 ps + 2.0 ns ≈ 2.2 ns
```

**Why differential sensing rejects common-mode noise:**

BL and BL_bar are routed as a tightly-coupled differential pair on the same metal layer,
with matched parasitic capacitance and resistance. Any external noise source (power
supply ripple, capacitive coupling from adjacent bitlines, substrate noise) couples
equally into both BL and BL_bar, appearing as a common-mode shift. The sense amplifier
responds only to the **difference** between BL and BL_bar, rejecting the common-mode
component. This is why DRAM can detect a ~45 mV signal in a noisy array environment:

```verilog
BL signal:      VDD/2 + 45 mV + noise
BL_bar signal:  VDD/2 - 45 mV + noise
Difference:     90 mV (noise cancels!)

Common-mode rejection ratio (CMRR) of cross-coupled sense amp:
  CMRR ≈ gm * R_load ≈ 30-40 dB (voltage ratio of 30-100x)
  A 100 mV common-mode noise appears as only 1-3 mV differential error
```

#### Current-Mode Sensing

An alternative that detects current flow rather than voltage swing:

Instead of waiting for bitline voltage to develop:
Current-mode sense amp measures the difference in discharge current
between BL and BL_bar.

I_BL  (stored "1") = C_s * VDD / (C_s + C_BL) * (discharge rate)
I_BL_bar (stored "0") ≈ 0

The current difference appears in ~50-100 ps (much faster than voltage-mode).

**Advantages:**
   - Faster detection: 50-100 ps vs 150+ ps for voltage-mode
   - Less bitline swing required (lower power)

**Disadvantages:**
   - More complex circuit (requires matched current mirrors)
   - Higher offset sensitivity (transistor mismatch affects current more)
   - Typically used only in fast SRAM (register files) not DRAM

Most DRAM uses voltage-mode due to simplicity and robustness.
Fast SRAM register files may use current-mode for sub-ns read latency.

---

### DRAM Read and Write Operation Sequences

#### DRAM Read Sequence (Step-by-Step)

1. **PRECHARGE** (tRP ~13.75 ns for DDR4-3200)
   - Bitlines BL and BL_bar are equalized to VDD/2 by precharge circuit
   - All sense amplifiers in the bank are reset
2. **ACTIVATE** (wordline assertion)
   - Controller issues ACT command with row address
   - Wordline for the selected row goes HIGH
   - All access transistors in the row turn on simultaneously
   - Charge sharing begins between each cell capacitor (Cs) and its bitline (C_BL)
   - For a stored "1": BL rises above VDD/2 by delta_V ≈ 45 mV
   - For a stored "0": BL drops below VDD/2 by delta_V ≈ 45 mV
   - Sense amplifier detects the differential: BL vs BL_bar diverge
3. **SENSE AND RESTORE** (tRCD includes this)
   - Sense amplifier fires, amplifying the ~45 mV signal to full rail
   - Full-rail voltage is driven back through the access transistor into the cell capacitor, RESTORING the charge (destructive read requires restore)
   - The entire row is now latched in the sense amplifier array (row buffer)
4. **COLUMN READ** (tCL ~13.75 ns)
   - Controller issues READ command with column address
   - Column decoder selects the appropriate sense amplifier output
   - Selected data is driven onto the DQ pins through the I/O gating and output driver
   - Burst of 8 data transfers (BL8) on both clock edges over 4 clock cycles
5. **PRECHARGE** (if closing the row)
   - Controller issues PRECHARGE command
   - Wordline goes LOW, disconnecting cells
   - Bitlines equalized back to VDD/2
   - Row buffer data is lost (cells have been restored in Step 3)

Total read latency (row miss): tRP + tRCD + tCL = ~41 ns (DDR4-3200)

Total read latency (row hit):  tCL only = ~13.75 ns

#### DRAM Write Sequence (Step-by-Step)

1. **ACTIVATE** (same as read)
   - Open the row, sense amplifiers latch row data
2. **COLUMN WRITE** (tCWL ~12 ns)
   - Controller issues WRITE command with column address
   - Write driver forces the selected bitline pair to the new data value
   - The sense amplifier for the selected column is overridden by the write driver
   - The cell capacitor is charged/discharged to the new value through the access transistor
3. **WRITE RECOVERY** (tWR ~15 ns)
   - After the write burst completes, the written cell needs time to fully charge the capacitor to the correct voltage level
   - The sense amplifier must hold the written value during this time
   - The row must remain open for tWR before precharge
4. **PRECHARGE**
   - After tWR has elapsed, controller can issue PRECHARGE
   - Row buffer data (with the updated column) is written back to all cells in the row

Total write latency (row miss): tRP + tRCD + tCWL + tWR = ~54 ns

### DRAM Refresh -- Detailed Mechanism

#### Charge Leakage

A DRAM cell stores a bit as charge on a capacitor (Cs ≈ 25-30 fF). This charge leaks
away through several paths:

**Leakage current sources:**
   1. Subthreshold leakage through the access transistor (NMOS gate = off, but
small Ids flows): I_sub ≈ 1-10 fA per cell (temperature dependent)

2. Gate leakage through the access transistor dielectric: I_gate ≈ 0.1-1 fA
(reduced in high-k metal gate processes)

3. Junction leakage at the drain of the access transistor: I_junc ≈ 0.5-5 fA

4. Capacitor dielectric leakage (tunneling through the capacitor dielectric):
I_cap ≈ 0.1-2 fA (worse with thinner dielectrics for higher density)

Total leakage: I_total ≈ 2-20 fA per cell (at 85°C)

Time to lose 50% of charge (from VDD to VDD/2):
t_leak = Cs * (VDD/2) / I_total
= 25 fF * 0.5V / 10 fA
= 12.5 fC / 10 fA
= 1.25 seconds (at room temperature)

But the sense amplifier threshold is much tighter than 50%:
The cell must retain enough charge for the sense amplifier to distinguish
"1" from "0" reliably. If delta_V < V_sense_min (~15 mV), the cell is "lost."

Effective retention time: ~64 ms at 85°C (JEDEC standard)
At 45°C: retention time is ~2-4x longer (lower leakage)
At 105°C: retention time is ~2-4x shorter (higher leakage)

#### Refresh Commands

```verilog
1. All-Bank Refresh (RAS-only refresh):
   Command: CS_n=0, RAS_n=0, CAS_n=0, WE_n=0, A10=1
   All banks must be precharged (idle) before refresh
   Duration: tRFC (all-bank refresh cycle time)
   During tRFC: NO commands can be issued to ANY bank

   tRFC values by density:
   | Density | tRFC (ns) | Rows refreshed per command |
   |---------|-----------|---------------------------|
   | 4 Gb    | 260       | ~8                        |
   | 8 Gb    | 350       | ~8                        |
   | 16 Gb   | 550       | ~8                        |
   | 24 Gb   | 650       | ~8                        |

2. Per-Bank Refresh (DDR4 optional):
   Only one bank is refreshed at a time
   Other 15 banks remain available for read/write
   tRFC_pb ≈ 140 ns (much shorter than all-bank)
   16 per-bank refreshes needed to cover all banks

3. Auto-Refresh:
   Controller issues REFRESH command, DRAM internally manages row counter
   Row counter increments automatically after each refresh
   8192 refresh commands per tREFW (64 ms)
   tREFI = 64 ms / 8192 = 7.8125 us average interval

4. Self-Refresh:
   DRAM enters low-power mode, internal oscillator generates refreshes
   No controller involvement needed
   Used in sleep/standby modes
   tCKE must be low for at least tCKESR before self-refresh is entered
   Exit latency: tXSR (self-refresh exit time, ~100-200 ns)

5. Fine Granularity Refresh (FGR, DDR4):
   1x mode: 8192 commands per 64 ms (standard, tREFI = 7.8 us)
   2x mode: 16384 commands per 64 ms (tREFI = 3.9 us, more frequent)
   4x mode: 32768 commands per 64 ms (tREFI = 1.95 us, even more frequent)
   Higher modes reduce tRFC (fewer rows per command) but increase total
   refresh overhead. Used at high temperature to prevent data loss.
```

#### Refresh Overhead at Different Densities

```verilog
Refresh overhead = tRFC / tREFI (fraction of time DRAM is unavailable)

| Density | tRFC (ns) | tREFI (us) | Overhead | Bandwidth Lost |
|---------|-----------|------------|----------|----------------|
| 4 Gb    | 260       | 7.8125     | 3.3%     | ~0.85 GB/s per 25.6 GB/s channel |
| 8 Gb    | 350       | 7.8125     | 4.5%     | ~1.15 GB/s |
| 16 Gb   | 550       | 7.8125     | 7.0%     | ~1.80 GB/s |
| 24 Gb   | 650       | 7.8125     | 8.3%     | ~2.13 GB/s |
| 32 Gb*  | 800       | 7.8125     | 10.2%    | ~2.62 GB/s |

* Projected for future DDR5 densities

At 16 Gb density, 7% of total bandwidth is consumed by refresh.
For a DDR5-5600 channel (44.8 GB/s peak):
  Refresh overhead = 44.8 * 0.07 = 3.14 GB/s lost to refresh
  That's equivalent to ~49,000 cache line misses per second that cannot be served

Same-bank refresh (DDR5) mitigates this:
  Refresh bank 0 of all bank groups simultaneously
  Banks 1-3 in each bank group remain available
  Effective overhead: tRFC_sb / tREFI ≈ 200ns / 7.8us ≈ 2.6% (much lower)
```

### Refresh Requirements and Power Impact

Retention time: time for Cs to leak enough charge for the sense amp to misread
Typical: 64 ms (DDR4), 32 ms (LPDDR5 at high temperature)

Refresh cycle: each row must be read and restored within the retention time
For a 16 Gb DRAM with 128K rows:
Refresh all rows in 64 ms → 1 row every 64ms / 128K ≈ 488 ns
Each refresh takes ~50 ns (tRC)

**Refresh bandwidth overhead:**
   - 50 ns / 488 ns ≈ 10.2% of total bandwidth consumed by refresh

**Power impact:** In mobile DRAM (LPDDR), refresh power can be 30-40% of total DRAM power during idle (self-refresh). Techniques to reduce:
- **Per-bank refresh:** Only one bank is being refreshed at a time; other banks can be accessed
- **Targeted refresh:** Only refresh rows near a frequently-accessed row (rowhammer defense)
- **Temperature-compensated refresh:** Reduce refresh rate at low temperature (retention time is longer)

#### DRAM Refresh Bandwidth Overhead — Worked Calculation

```verilog
For DDR4-3200 single channel (25.6 GB/s peak) with 16 Gb chips:

Refresh parameters:
  tREFW = 64 ms (refresh window)
  8192 refresh commands required per tREFW
  tRFC = 550 ns (16 Gb, all-bank refresh)
  tREFI = 7.8125 us (average interval)

Time spent in refresh per 64 ms window:
  Total refresh time = 8192 * 550 ns = 4,505,600 ns = 4.506 ms
  Overhead = 4.506 / 64.0 = 7.04%

Bandwidth lost to refresh:
  25.6 GB/s * 0.0704 = 1.80 GB/s lost

  In terms of cache lines (64 B) not served:
    1.80 GB/s / 64 B = 28.1 million cache lines per second that cannot be served

For DDR5-5600 (44.8 GB/s peak) with 24 Gb chips:
  tRFC = ~700 ns (projected for 24 Gb)
  Overhead = 8192 * 700 ns / 64 ms = 8.96%
  Bandwidth lost: 44.8 * 0.0896 = 4.01 GB/s

With same-bank refresh (DDR5 SBR):
  tRFC_sb = ~250 ns
  Only 4 bank numbers x tRFC_sb per refresh round (not all banks)
  Banks 1-3 remain available during each SBR
  Effective overhead: ~2.5-3% (much better than all-bank 8.96%)
```

#### Auto-Refresh vs Self-Refresh — Detailed Comparison

**Auto-refresh (controller-managed):**
   - Controller issues REFRESH command every tREFI (7.8 us average).
   - DRAM internally increments a row counter and refreshes the next set of rows.
   - Controller must track timing: can postpone up to 8x tREFI, but must catch up.
   - DRAM is fully operational between refreshes (commands to other banks allowed).

Pros: Controller controls when refresh happens (can schedule during idle).
Cons: Controller complexity; must guarantee all refreshes within tREFW.

**Self-refresh (DRAM-managed):**
   - Controller asserts CKE low and issues SELFREF command.
   - DRAM enters low-power mode with internal oscillator.
   - Internal refresh counter generates refreshes autonomously.
   - No controller involvement; all I/O pins are quiescent (maximum power savings).
   - Exit latency: tXSR = 100-200 ns (must complete any in-progress refresh before exit).

Pros: Maximum power saving; no controller overhead during sleep.
Cons: Long exit latency; no external access during self-refresh.

**Power comparison:**
   - Active idle (CKE high, no commands):  ~80 mW per x8 device (DDR4-3200)
   - Auto-refresh (periodic REF):          ~65 mW average (refresh spikes to ~200 mW)
   - Self-refresh (CKE low):               ~15-30 mW (DRAM handles refresh internally)
   - Power-down (CKE low, no refresh):     ~5 mW (data lost! Only for deep sleep)

**Typical mobile use:**
   - Active → auto-refresh during normal operation
   - Screen off → enter self-refresh (DRAM retains data, minimal power)
   - Deep sleep → power-down (DRAM data lost, restore from flash on wake)

---

## Asynchronous FIFO — moved

The complete async-FIFO treatment (architecture, binary↔Gray proofs, full/empty detection proof, production-quality RTL, worked pointer-state example, reset handling, non-power-of-2 depths) lives in [Async_Circuit_Design](../03_Frontend_RTL_and_Verification/Async_Circuit_Design.md) §5 — it is a CDC structure first and a memory second. This page keeps the SRAM/DRAM device view.

---

## Memory Compiler and Library Files

### What You Specify to a Memory Compiler

Memory compilers (e.g., ARM Artisan, Synopsys, TSMC Memory Compiler) take these inputs:

- Word count (depth): e.g., 1024
- Word width (bits): e.g., 32
- Number of ports: 1 (single-port), 2 (dual-port), 1R/1W
- Mux factor: column muxing ratio (4, 8, 16) — trades width for height
- Bit write enable: byte-enable or full-word only
- Power mode: high-speed, high-density, low-leakage
- Redundancy: spare rows/columns for repair
- Corner: process corners to generate .lib files for (SS, TT, FF, etc.)

### What You Get Back

1. .lib (Liberty): Timing models for STA
   - Setup/hold times for address, data, write-enable relative to clock
   - Clock-to-Q (access time) for read data output
   - Leakage power per state (all zeros, all ones, average)
   - Dynamic energy per read/write operation

2. .lef (Library Exchange Format): Physical abstract for P&R
   - Cell outline (width, height)
   - Pin locations (metal layer, coordinates)
   - Blockage layers

3. .v (Verilog model): Behavioral model for simulation
   - Functional model with timing annotation
   - X-propagation on timing violations
   - Usually includes backdoor read/write for verification

4. .gds (GDSII): Full layout for manufacturing

5. .sdf (Standard Delay Format): Back-annotated timing for gate-level sim

6. .spice: Transistor-level netlist for analog simulation (optional)

### How to Read a Memory .lib File

```verilog
Key parameters in a .lib timing group:

cell (SRAM_1024x32) {
  area : 25000;  // in um^2
  
  pin (CLK) {
    clock : true;
    capacitance : 0.05;  // pF
  }
  
  pin (Q[31:0]) {
    timing () {
      related_pin : "CLK";
      timing_type : rising_edge;
      cell_rise (lookup_table) { ... }
      cell_fall (lookup_table) { ... }
      // Access time: typically 0.8-2.0 ns for embedded SRAM in 28nm
    }
  }
  
  pin (D[31:0]) {
    timing () {
      related_pin : "CLK";
      timing_type : setup_rising;
      rise_constraint (lookup_table) {
        // Setup time: typically 0.1-0.3 ns
      }
      fall_constraint (lookup_table) {
        // Setup time
      }
    }
    timing () {
      related_pin : "CLK";
      timing_type : hold_rising;
      rise_constraint (lookup_table) {
        // Hold time: typically 0.05-0.15 ns
      }
    }
  }
  
  leakage_power () {
    value : 150;  // uW (typical for 1K x 32 in 28nm)
  }
}
```

**Practical reading tips:**
- Access time is the clock-to-Q delay of the output pins — this sets the memory read latency in your pipeline.
- Setup and hold on address pins are usually tighter than data pins (address decode is on the critical path).
- The "mux factor" affects aspect ratio: higher mux = shorter, wider memory (better for layout but slower).
- Always check worst-case corner (SS, 0.9V, 125°C) for timing, best-case (FF, 1.1V, -40°C) for hold.

---

## Memory BIST — moved

March algorithms (element notation, March C- walkthrough, fault models, and the fault-coverage proof) live with the rest of test: [DFT_and_ATPG](../06_Signoff/DFT_and_ATPG.md) §7 *Memory BIST*.

---

## Cache — moved

Cache organization and microarchitecture have their own page: [Cache_Microarchitecture](Cache_Microarchitecture.md) — the worked 32KB/4-way/64B geometry example is in its §1, replacement policies (LRU/PLRU/RRIP) in §7, write policies + write-back FSM in §3, and MESI/MOESI coherence in §9 (protocol-level view: [CPU_Architecture](CPU_Architecture.md) §8).

---

## 9. Multi-Port SRAM Design

### Port Configurations and Cell Topologies

Multi-port SRAM extends the basic 6T/8T cell by adding additional access transistors for independent read and write operations within a single cycle. The number and type of ports fundamentally determine the cell structure, area, and timing characteristics.

**1R1W (Single-Port, one read + one write, time-multiplexed):**

The simplest configuration. A standard 6T cell with a single pair of access transistors (M5, M6) and a single wordline. Read and write share the same port and cannot occur simultaneously. One complete memory access (read or write) occupies one clock cycle. The bitline pair (BL, BL_bar) is shared between read and write operations.

```verilog
6T cell with single port:
  1 wordline (WL)
  1 bitline pair (BL, BL_bar)
  6 transistors total
  Area: ~0.05 um^2 (7nm reference)
```

**2R1W (Two independent reads + one write):**

Requires separate read and write access paths. The write port uses the standard M5/M6 access transistors with WL_write. The two read ports each add a dedicated 2T read stack (similar to the 8T read port), with independent read wordlines (RWL0, RWL1) and read bitlines (RBL0, RBL1).

```verilog
2R1W cell topology:
  Write: M5, M6 + WL_write, BL/BL_bar (from 6T core)
  Read port 0: M7, M8 + RWL0, RBL0
  Read port 1: M9, M10 + RWL1, RBL1
  Total: 10 transistors
  Area: ~0.08 um^2 (7nm)
```

The read ports are read-decoupled: they do not disturb the storage nodes Q and Q_bar, so read stability equals the hold SNM regardless of how many read ports are active simultaneously.

**2R2W (Full dual-port):**

Two completely independent ports, each capable of reading or writing. This requires two full write paths and two full read paths. The cell has two write wordlines (WL_A, WL_B) and two write bitline pairs (BLA/BLA_bar, BLB/BLB_bar), plus two read wordlines and two read bitlines if read-decoupled ports are used.

Full dual-port (8T cell variant):
- **Port A write** — WL_A, BLA, BLA_bar (M5, M6)
- **Port B write** — WL_B, BLB, BLB_bar (M5b, M6b)
- **Read port A** — RWL_A, RBL_A (2T)
- **Read port B** — RWL_B, RBL_B (2T)
- **Total** — 10-12 transistors
- **Area** — ~0.10 um^2 (7nm)

Concurrent write conflict: if both ports write to the same address in the same cycle, the result is indeterminate. A multi-port SRAM controller must implement write-conflict detection (compare addresses of both write ports) and arbitration logic.

### Port Scaling Analysis

Area scales quadratically with the number of ports. Each port adds one wordline (horizontal wire spanning the full row width) and one or two bitlines (vertical wires spanning the full column height). The transistor count per cell increases linearly, but the routing congestion for the additional wordlines and bitlines forces the cell to grow in both dimensions.

$$A_{cell}(P) \approx A_{6T} \cdot \left(1 + \alpha \cdot P^2\right)$$

where $P$ is the total port count and $\alpha \approx 0.05$--$0.10$ depends on the process node and layout style. The quadratic term arises because each new port requires both a new horizontal wire (wordline) and vertical wires (bitlines), and these must route through the cell without overlapping existing metal traces.

**Practical limits:** Beyond 8--12 ports, the cell area and wire delay become prohibitive. A 16-port SRAM cell in 7nm would be approximately 8--10x the area of a single-port cell, and the resistive-capacitive (RC) delay on the long wordlines and heavily loaded bitlines would push access time beyond 2--3 ns, negating the benefit of parallel access.

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk", "nodeSpacing": 60, "rankSpacing": 60, "htmlLabels": false}}}%%
flowchart TD
    A["Port Requirement"] --> B{"Port count P"}
    B -->|"P <= 2"| C["Single/bank SRAM: 6T-10T cell, direct implementation"]
    B -->|"2 < P <= 6"| D["Banked multi-port SRAM: 2-4 banks, crossbar interconnect"]
    B -->|"6 < P <= 12"| E["Heavily banked SRAM: 8+ banks, multi-cycle arbitration"]
    B -->|"P > 12"| F["Register file via flip-flops or distributed register clusters"]
    C --> G["Area: 1x baseline, Access: 0.5-1.0 ns"]
    D --> H["Area: 2-4x baseline, Access: 0.8-1.5 ns"]
    E --> I["Area: 4-8x baseline, Access: 1.5-3.0 ns, Bank conflict possible"]
    F --> J["Area: 10-50x equivalent SRAM, Access: 0.2-0.5 ns per bank"]
```

### Column Multiplexing

Column multiplexing reduces the number of sense amplifiers by sharing one sense amplifier across multiple adjacent columns. A column mux ratio of $M$ means $M$ cells in a row share a single sense amplifier.

Column mux = 4:
- **Physical columns** — 128 (128 bitcells per row)
- **Logical words** — 32 bits (128 / 4 = 32 words per row, each 32 bits wide)
- **Sense amplifiers** — 128 / 4 = 32

- **Read** — WL asserted, all 128 bitlines develop differential, 4:1 MUX selects one of every 4 bitlines for each sense amplifier

- **Address** — extra 2 bits (log2(4)) in column address select which of the 4 bitlines routes to the sense amplifier

Column multiplexing trades sense amplifier area for taller memory (more rows for the same capacity). Common mux ratios: 2, 4, 8, 16. Higher mux ratios produce physically narrower and taller memories, which can improve floorplan fit in some SoC layouts but increase bitline length and access time.

### GPU Register File Application

GPU shader cores require register files with very high port counts. A typical GPU thread in a warp/wavefront issues multiple operands per cycle, requiring simultaneous reads of several source registers.

- **Example** — 4-wide GPU SIMD lane, 3-source instruction format
- **Per-cycle register access** — 4 threads x 3 sources = 12 reads
- **Plus results** — 4 threads x 1 result = 4 writes
- **Total** — 12R + 4W = 16 ports (if implemented as monolithic SRAM)

This is far beyond practical SRAM port limits. The solution is **banking**: divide the register file into $B$ banks, each with $P/B$ ports. A 16-bank register file with 1R1W per bank can service up to 16 reads and 16 writes per cycle, provided no two accesses target the same bank (bank conflict). The compiler or hardware scheduler inserts bank-conflict avoidance by assigning registers to banks in a round-robin or conflict-free pattern.

---

## 10. Register File Design

### Multi-Port Register File for Superscalar CPUs

A superscalar processor with issue width $W$ needs to read $W$ source operands and write back up to $W/2$ results per cycle (assuming roughly half the instructions produce results). The architectural register file therefore requires at least $W$ read ports and $W/2$ write ports.

```verilog
Port count derivation for a 4-wide out-of-order machine:
  Instructions per cycle: up to 4 (integer + FP mix)
  Source operands per instruction: 2 (typical RISC: src1, src2)
  Total read ports: 4 instructions x 2 sources = 8R
  Result ports: up to 4 results = 4W (some instructions do not write back)
  Total: 8R + 4W = 12 ports on the architectural register file
```

For a physical register file with register renaming (typical in modern OoO cores), the file is larger because all committed and in-flight destination registers occupy physical entries.

Physical register file parameters (example: ARM Cortex-A78 class):
- **Entry count** — 128-160 physical registers (32 arch + 96-128 rename buffers)
- **Data width** — 64 bits per entry
- **Read ports** — 8-10 (for 4-5 issue width, 2 sources each)
- **Write ports** — 4-6 (for execution unit writebacks)
- **Total ports** — 12-16

### Multi-Port Register File — Detailed Implementation (moved from the DRAM section)

#### How Multiple Read Ports Are Implemented

Each additional read port requires a **separate pair of bitlines** (or a separate 2T
read stack) for every bitcell in the array. The bitcell physically grows to accommodate
the additional bitline routing and access transistors.

**1R1W register file (6T cell):**

```ascii-graph
                   VDD
                ┌──┴──┐
        BL ──── M5    M6 ──── BL_bar
                │  ╳  │     (cross-coupled inverters M1-M4)
                └──┬──┘
               GND (via pull-down)
        WL ──── gates of M5, M6

1 read port (shared with write via BL/BL_bar)
1 write port (same access transistors)
6 transistors, 1 wordline, 1 bitline pair
```

**3R2W register file (16T cell):**

- **Write ports** — 2 independent write paths
- **Write Port A** — WLA, BLA/BLA_bar (access transistors M5a, M6a)
- **Write Port B** — WLB, BLB/BLB_bar (access transistors M5b, M6b)

- **Read ports** — 3 independent read-only paths (8T-style read-decoupled)
- **Read Port 0** — RWL0, RBL0 (transistors M7a, M8a -- gated by Q_bar)
- **Read Port 1** — RWL1, RBL1 (transistors M7b, M8b -- gated by Q_bar)
- **Read Port 2** — RWL2, RBL2 (transistors M7c, M8c -- gated by Q_bar)

- **Total transistors** — 6 (storage) + 2x2 (write ports) + 3x2 (read ports) = 16T
- **Total wordlines** — 2 write + 3 read = 5
- **Total bitlines** — 2 write pairs + 3 read singles = 7

Area per cell ≈ 16T/6T * 0.05 um^2 ≈ 0.13 um^2 (in 7nm)

#### Write Conflict Resolution

When two write ports target the same address simultaneously, the result is undefined
without arbitration. A multi-port register file controller must implement:

```verilog
Write conflict detection:
  For all pairs of write ports (i, j):
    if (W_addr[i] == W_addr[j]) && W_enable[i] && W_enable[j]:
      // CONFLICT! Resolve by priority
      if (W_priority[i] > W_priority[j]):
        W_enable[j] = 0   // suppress port j
      else:
        W_enable[i] = 0   // suppress port i

In an OoO processor:
  Port priorities are assigned by the scheduler (e.g., older instruction wins)
  The scheduler knows both write addresses before issue, so conflicts are
  resolved before the write reaches the register file

Alternative: some designs allow simultaneous writes to the same address
  only if they write non-overlapping byte lanes (checked via WSTRB comparison)
```

#### Area Scaling With Ports

```verilog
Cell area scales approximately as O(Ports^2):
  Each port adds 1 wordline (horizontal) + 1-2 bitlines (vertical)
  Cell must grow in BOTH dimensions to route the additional wires

  A(P) ≈ A_6T * (1 + k * P)^2  where k ≈ 0.1-0.15

  P=1 (1R1W):  A ≈ 0.05 um^2 (6T)
  P=2 (1R1W):  A ≈ 0.07 um^2 (8T)
  P=3 (2R1W):  A ≈ 0.10 um^2 (10T)
  P=5 (3R2W):  A ≈ 0.13 um^2 (16T)
  P=6 (4R2W):  A ≈ 0.18 um^2 (18T)
  P=8 (6R2W):  A ≈ 0.25 um^2 (22T)
  P=12 (8R4W): A ≈ 0.40 um^2 (30T) -- extremely large

Access time also scales poorly with ports:
  Each additional bitline adds capacitance to the sense node
  Each additional wordline adds gate loading to the decoder
  P=2:  ~0.5 ns access
  P=6:  ~0.8 ns access
  P=12: ~1.5 ns access (often multi-cycle)

Typical OoO CPU configurations:
  ARM Cortex-A78:  128-entry x 64b, 4R2W (banked 2x2R1W)
  Apple M1:       ~350-entry x 64b, estimated 6R3W (banked)
  AMD Zen 4:      ~180-entry x 64b, estimated 4R2W (banked)
  Intel Golden Cove: ~280-entry x 64b, estimated 6R4W (banked)
```

### Register File Banking

To avoid the area and delay explosion of a monolithic 12+ port SRAM, the register file is split into multiple banks with fewer ports each. The key trade-off is port count reduction versus bank-conflict stalls.

```verilog
Monolithic: 128 entries x 8R4W, 12 ports
  Area (7nm): ~0.015 mm^2
  Access time: ~1.5 ns (heavily loaded bitlines)

2-bank: 2 x 64 entries x 4R2W, 6 ports per bank
  Area (7nm): ~0.010 mm^2 total (2 banks x 0.005)
  Access time: ~0.9 ns
  Bank conflict rate: ~15% (requires both sources in same bank)

4-bank: 4 x 32 entries x 2R1W, 3 ports per bank
  Area (7nm): ~0.008 mm^2 total
  Access time: ~0.6 ns
  Bank conflict rate: ~35% (higher because fewer banks to distribute across)
```

The optimal banking factor balances area savings against the performance loss from bank conflicts. Most out-of-order cores use 2--4 banks for the integer register file and 2 banks for the floating-point/SIMD register file.

### Read Port Optimization: Bypass (Forwarding)

When a write and a read target the same register in the same cycle, the read does not need to wait for the write to complete and propagate through the SRAM. Instead, a **bypass network** (also called forwarding or write-through) compares the read address against all pending write addresses. If a match is found, the write data is forwarded directly to the read port, saving one cycle of latency.

```verilog
Bypass logic per read port:
  For each read address R_addr[i]:
    For each write port j (0 to W-1):
      if (R_addr[i] == W_addr[j]) && W_enable[j]:
        R_data[i] = W_data[j]    // bypass: use write data directly
      else:
        R_data[i] = SRAM_read[i]  // normal: read from register file

  Hardware: W comparators per read port, W:1 MUX
  Total for 8R4W: 8 x 4 = 32 comparators + 8 four-input MUXes
```

Bypass reduces the effective read latency from 2 cycles (1 write + 1 read) to 1 cycle, which is critical for back-to-back dependent instruction execution in pipelines.

### SRAM vs Flip-Flop Register File

The choice between SRAM-based and flip-flop-based register files depends on the entry count and target frequency.

| Parameter | Flip-Flop RF (32 entries) | SRAM RF (128+ entries) |
|-----------|--------------------------|------------------------|
| Read latency | 1 gate delay (~50 ps) | 1-2 clock cycles (~500 ps - 1 ns) |
| Write latency | 1 gate delay | 1 cycle (setup to clock edge) |
| Area per bit (7nm) | ~0.5 um^2 | ~0.05 um^2 (100x denser) |
| Total area (8R4W) | ~0.15 mm^2 (32x64b) | ~0.015 mm^2 (128x64b) |
| Power (active) | Very high (clocked FFs) | Lower (only accessed rows consume) |
| Multi-port | Easy (just wire to all FF outputs) | Expensive (adds ports to SRAM cell) |

Small register files (16--32 entries) with high port counts are almost always implemented as flip-flop arrays because the read latency advantage outweighs the area penalty. Large register files (64+ entries) use SRAM because the area and power of flip-flops scale linearly with entry count while SRAM scales much more favorably.

**Area calculation example: 32-entry 8R4W register file in 7nm**

```verilog
Flip-flop approach:
  32 entries x 64 bits = 2048 flip-flops
  Each DFF in 7nm: ~0.25 um^2 (with clock gating)
  8 read ports: 8 x 64-bit 32:1 MUX per port = 8 x 64 x 32 x ~0.01 um^2
  4 write ports: 4 x 64-bit decoder + enable logic
  Total estimate: ~0.01-0.02 mm^2

SRAM approach (for comparison):
  32-entry x 64-bit x 12-port (8R4W) custom SRAM
  Cell area: ~0.3 um^2 per bit (12-port cell is very large)
  Total cell array: 32 x 64 x 0.3 = 614 um^2
  Plus periphery (decoders, sense amps, MUXes): ~3x cell area
  Total estimate: ~0.002 mm^2 (smaller but much slower than FF)
```

---

## 11. Error Correction Codes for Memory

### SECDED -- Hamming Code

Single Error Correction, Double Error Detection (SECDED) is the most common ECC scheme for SRAM caches and register files. It is built on Hamming codes with an additional overall parity bit.

**Hamming(72,64): 8 check bits for 64 data bits**

The code word structure places check bits at power-of-2 positions ($2^0, 2^1, 2^2, \ldots, 2^7$) within a 72-bit code word. Data bits fill the remaining 64 positions.

```verilog
Bit positions (1-indexed):
  Check bits:  1,  2,  4,  8, 16, 32, 64  (7 Hamming check bits)
  Parity bit:  0 (overall parity, position 0 or appended at MSB)
  Data bits:   all other positions (3,5,6,7,9,10,11,12,13,...)

  Total: 64 data + 7 Hamming check + 1 overall parity = 72 bits
```

**Syndrome computation:**

Each check bit $C_i$ covers all data bit positions whose binary representation has a 1 in bit position $i$. On read, the syndrome is computed by XORing the received check bits against the check bits recomputed from the received data.

$$S = C_{received} \oplus C_{computed}$$

The syndrome $S$ is a 7-bit value (8-bit if including overall parity $P$):

- $S = 0$, $P = 0$: No error detected
- $S \neq 0$, $P = 1$: Single-bit error. $S$ gives the bit position; flip that bit to correct
- $S \neq 0$, $P = 0$: Double-bit error detected. Cannot correct; must flag as uncorrectable error
- $S = 0$, $P = 1$: Error in the overall parity bit only; correct by flipping $P$

**Single-bit error correction example:**

```verilog
Stored code word (64 data + 8 check/parity):
  [P C1 C2 D3 C4 D5 D6 D7 C8 D9 D10 ... D64]

Suppose bit 13 (data bit) flips during storage.

Readback:
  Syndrome S = 13 in binary = 0001101
  Overall parity P = 1 (parity changed by the single-bit flip)

  Interpretation: S = 13 != 0, P = 1 => single-bit error at position 13
  Correction: flip bit 13
```

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk", "nodeSpacing": 60, "rankSpacing": 60, "htmlLabels": false}}}%%
flowchart TD
    A["Read code word from memory"] --> B["Compute syndrome S from received data"]
    B --> C{"S == 0?"}
    C -->|"Yes"| D{"Overall parity P OK?"}
    D -->|"Yes"| E["No error -- use data"]
    D -->|"No"| F["Parity bit error only -- correct P"]
    C -->|"No"| G{"P == 1 (odd parity)?"}
    G -->|"Yes"| H["Single-bit error at position S: flip bit S to correct"]
    G -->|"No"| I["Double-bit or multi-bit error: cannot correct, flag SDC"]
```

### ECC Overhead in SRAM Caches

For a 64-bit data path with SECDED, 8 check bits are stored alongside each 64-bit word. This represents a 12.5% storage overhead.

```verilog
L1 data cache (32KB, 64B lines):
  Per line: 64 bytes = 512 bits data
  ECC: 512/64 x 8 = 64 check bits = 8 bytes per line
  Total storage per line: 72 bytes (12.5% overhead)
  Total cache: 32KB x 1.125 = 36 KB physical SRAM

  Note: L1 caches sometimes omit ECC for latency reasons,
  relying on parity (1-bit error detect only) for speed.

L2 cache (256KB, 64B lines):
  Same 12.5% ECC overhead
  256KB x 1.125 = 288 KB physical SRAM

  Some L2/L3 caches use longer code words (e.g., 128+8 or 256+9)
  to reduce the overhead percentage at the cost of more complex decode.
```

### BCH Codes for Multi-Bit Correction

Bose--Chaudhuri--Hocquenghem (BCH) codes generalize Hamming codes to correct $t$ bit errors, where $t > 1$. The number of check bits grows as $m \cdot t$ where $m = \lceil \log_2(n) \rceil$ and $n$ is the code word length.

```verilog
BCH(511, 484) for t=2 (double-bit correction):
  Data bits: 484
  Check bits: 511 - 484 = 27
  Overhead: 27/511 = 5.3%

BCH(511, 457) for t=3 (triple-bit correction):
  Data bits: 457
  Check bits: 511 - 457 = 54
  Overhead: 54/511 = 10.6%
```

BCH codes are used in NAND flash memory (where multi-bit errors are common due to program/erase cycling and retention loss) and in large server-class SRAM caches where silent data corruption from multi-bit soft errors is unacceptable. The decode latency is significantly higher than Hamming (requires solving a polynomial equation over a Galois field), typically 3--10 cycles.

### Scrubbing

Soft errors (caused by alpha particles, cosmic rays, or neutron strikes) can flip bits in SRAM. If left uncorrected, multiple single-bit errors can accumulate in the same code word, eventually exceeding the correction capability of the ECC.

**Scrubbing** periodically reads every memory location, checks the ECC, and corrects any single-bit errors before a second error can accumulate in the same code word.

**Scrubbing rate calculation:**
   - Soft error rate (SER): 100 FIT per Mb (Failures In Time = errors per 10^9 device-hours)

For a 32 MB L3 cache:
Total bits: 32 x 10^6 x 8 = 256 Mbits (data)
With ECC: 256 x 1.125 = 288 Mbits (total)

Expected errors: 288 Mbits x 100 FIT / 10^9 = 0.0288 errors/hour
= 1 error every ~34.7 hours

For SECDED to work, we need to scrub fast enough that the
probability of TWO errors in the same 72-bit word between
scrubs is negligibly small.

With full scrub every 24 hours:
Mean errors between scrubs: 0.0288 x 24 = 0.69 errors (across ALL 4M words)
Probability of any double error: ~10^-7 per scrub cycle (very safe)

Hardware scrubbers walk through the cache or SRAM array during idle cycles, reading each line, checking the syndrome, and writing back corrected data if a single-bit error is found. This is transparent to software and has no performance impact if scheduled during idle periods.

### Silent Data Corruption (SDC)

When the ECC detects an error but cannot correct it (multi-bit error exceeding the correction capability), the system must decide how to respond. The worst outcome is **silent data corruption**: the error is not detected at all, and corrupted data propagates to the processor and potentially to persistent storage.

**SDC scenarios:**
   - No ECC: any single or multi-bit error is SDC
   - Parity only: single-bit error detected, multi-bit may be SDC (even number of flips)
   - SECDED: single-bit corrected, double-bit detected (CE), triple+ may be SDC
   - BCH(t=3): up to 3-bit corrected, 4-bit detected, 5+ may be SDC

SDC rate target for server processors: < 1 SDC per 10^17 device-hours
(roughly 1 SDC per 10 million server-years)

Mitigation strategies for SDC include: chipkill (distributing each memory word across multiple DRAM chips so a single-chip failure causes at most 1 bit per code word), end-to-end data integrity (CRC/ECC checked at every transfer point from DRAM to cache to register file), and lockstep execution (two cores execute the same code and compare results).

---

## 12a. SRAM Scaling Challenges at 3nm/2nm

### 6T Bitcell Scaling Limits

As SRAM scales from 5nm to 3nm and 2nm (gate-all-around / GAA nanosheet transistors), the fundamental 6T bitcell faces compounding challenges:

| Parameter | 7nm | 5nm | 3nm | 2nm (projected) |
|---|---|---|---|---|
| **6T cell area (um^2)** | 0.027 | 0.021 | 0.015 | 0.010 |
| **VDD min (read-stable)** | 0.7V | 0.65V | 0.55V | 0.45V |
| **Read SNM (mV, VDD min)** | 120 | 90 | 60 | 40 |
| **Write margin (mV)** | 200 | 160 | 110 | 80 |
| **Cell ratio (CR)** | 1.5 | 1.3 | 1.1 | 1.0 (marginal) |
| **Process variation (sigma Vth)** | 30 mV | 35 mV | 45 mV | 55 mV |

**Key scaling challenges:**

1. **Read/write margin degradation:** As transistors shrink, the cell ratio (CR = pulldown/access strength) degrades because the access transistor does not scale as favorably. At 2nm, CR approaches 1.0, leaving almost no margin between read stability and write-ability. The fundamental tension (read wants weak access, write wants strong access) becomes nearly unresolvable at these nodes.

2. **Process variation sensitivity:** Random dopant fluctuation (RDF) and line-edge roughness (LER) increase with scaling. The standard deviation of Vth (sigma_Vth) scales as $1/\sqrt{W \cdot L}$, meaning smaller transistors have larger Vth variation. For a 6T cell at 2nm, a 55 mV sigma_Vth means 1-sigma read-margin variation is ~40% of the nominal margin -- requiring 5-6 sigma design guardbanding that consumes most of the available margin.

3. **Bitline leakage:** At 3nm/2nm, subthreshold leakage through unaccessed cells on the same bitline becomes significant. A 256-row bitline has 255 off-state access transistors leaking, collectively contributing a DC offset that reduces the sense amplifier's effective signal swing. Solutions include: read-bitline precharge to a reference voltage (instead of VDD) to reduce leakage-induced offset, and bitline segmentation (split a 256-row column into 4x 64-row segments with local-global bitline hierarchy).

4. **8T cell adoption:** At 3nm and below, the read-decoupled 8T cell is increasingly necessary to maintain adequate read SNM. The 30% area penalty of 8T is offset by the ability to operate at lower VDD (0.45V vs 0.55V for 6T), reducing active power by ~33% ($P \propto VDD^2$).

5. **Assist circuits:** Read-assist (wordline under-drive: reduce WL voltage during read to weaken the access transistor, increasing CR) and write-assist (negative bitline: drive BL below GND during write to strengthen the access transistor's pull-down) are mandatory at 3nm/2nm. These add 5-10% area overhead for charge pump circuits.

**Read-assist wordline under-drive:**
   - Normal read: V_WL = VDD = 0.55V -> access transistor strong -> CR ~1.1
   - With under-drive: V_WL = VDD - 0.15V = 0.40V -> access weaker -> effective CR ~1.4
   - Read SNM improvement: ~30-40 mV (significant at these margins)

**Write-assist negative bitline:**
   - Normal write: BL = 0V (GND) -> access pulls Q toward GND
   - With negative BL: BL = -0.2V -> stronger pull on Q -> write margin +50 mV
   - Implementation: on-chip charge pump generates -0.2V rail, gated to BL during write

---

## 12b. Compute-in-Memory (CIM / PIM)

### Overview

Compute-in-Memory (CIM) or Processing-in-Memory (PIM) integrates arithmetic operations directly into the memory array, eliminating the data movement between memory and processor that dominates energy consumption in data-intensive workloads (AI inference, graph analytics, scientific computing).

**Traditional von Neumann:**
   - Memory Array -> Read data -> Bus -> ALU -> Result -> Bus -> Write back
   - Energy per operation: ~100-1000 pJ (dominated by data movement)
   - Data movement energy: ~10-100x the energy of the computation itself

**Compute-in-Memory:**
   - Memory Array + integrated compute logic
   - Input applied to wordlines/bitlines -> result appears on sense amplifiers
   - Energy per operation: ~1-10 pJ (10-100x improvement)
   - No data leaves the memory array for the computation

### SRAM-Based CIM: Bitline Computing

SRAM-based CIM performs analog accumulation directly on the bitlines by simultaneously activating multiple wordlines and interpreting the resulting bitline voltage as a multi-bit analog value.

```verilog
Basic SRAM CIM operation (1-bit multiply-accumulate):

  Bitline computing:
    RBL is precharged to VDD.
    Multiple wordlines (RWL0..RWL3) are asserted simultaneously.
    Each activated cell conditionally discharges RBL through its read port.

    For an 8T cell (read-decoupled):
      RWL_i = 1, Q_bar_i = 1 -> cell i discharges RBL (contributes current)
      RWL_i = 1, Q_bar_i = 0 -> cell i does NOT discharge RBL

    RBL voltage drop is proportional to the number of cells that discharge:
      V_RBL = VDD - n * delta_V
      where n = number of cells with (RWL=1 AND Q_bar=1)
      and delta_V = per-cell discharge contribution

    This computes: n = sum(X_i AND W_i) for i = 0..3
    where X_i = RWL_i (input) and W_i = Q_bar_i (stored weight)
    = dot product of binary input vector and binary weight vector

  Multi-bit extension:
    For 4-bit weights and 4-bit inputs:
    - Weight bits stored across 4 rows (W[3:0] in rows 0-3)
    - Input bits applied to 4 wordlines (X[3:0] on RWL0-3)
    - Bitline voltage encodes: sum(X_i * W_i) for one column
    - Analog-to-digital converter (ADC) on the bitline converts voltage to digital count

  Array-level accumulation:
    128 columns compute 128 independent dot products in parallel
    = 128 MACs per cycle per subarray
    Throughput: 128 TOPS (binary) at 1 GHz
```

**SRAM CIM energy model:**

Energy per binary MAC operation in SRAM CIM:
RBL discharge: ~0.5 fJ (capacitance x voltage swing)
ADC per column: ~5 fJ (moderate-resolution SAR ADC)
WL driver: ~0.1 fJ per WL

Total per MAC: ~5-10 fJ (vs ~100 pJ for SRAM read + ALU + writeback)
CIM advantage: 10,000x lower energy per MAC

**Limitations:**
   - ADC precision limits compute precision: 3-4 bit ADC is practical
   - Process variation causes column-to-column offset (requires calibration)
   - Limited to integer/fixed-point; floating-point requires software decomposition
   - Analog noise accumulates with array size (practical limit: 128-256 columns)

### ReRAM/RRAM for In-Memory Computing

Resistive RAM (ReRAM or RRAM) stores data as resistance states (low-resistance = "1", high-resistance = "0") and naturally supports analog computation because Ohm's law provides multiplication ($V \times G = I$, where G = conductance = stored weight) and Kirchhoff's current law provides accumulation (currents sum at a node).

RRAM crossbar array for matrix-vector multiplication:

Input voltages V[0]..V[N-1] applied to wordlines
RRAM cells at crosspoints store conductance G[i][j]
Bitline currents: I[j] = sum(V[i] * G[i][j], i=0..N-1)

This computes Y = G * X in a single step (O(1) time complexity)
where G is the weight matrix, X is the input vector

Energy per MAC: ~1 fJ (current-based, no ADC needed for binary)
Array size: 128x128 crossbar practical (~2x2 um per cell)

**Challenges:**
   - Limited endurance: 10^6 - 10^9 write cycles (vs 10^15+ for SRAM)
   - Conductance drift: resistance changes over time (retention error)
   - Non-ideal linearity: conductance does not scale perfectly with voltage
   - Fabrication: RRAM requires additional mask layers beyond CMOS logic
   - Programming: write-verify cycles needed for precise weight setting

### MRAM: STT-MRAM and SOT-MRAM for Last-Level Cache

MRAM (Magnetoresistive RAM) uses magnetic tunnel junctions (MTJs) to store data non-volatilely. Two variants are relevant for cache memory:

**STT-MRAM (Spin-Transfer Torque MRAM):**

```verilog
STT-MRAM cell:
  1T + 1 MTJ (Magnetic Tunnel Junction)
  MTJ structure: Free layer / MgO tunnel barrier / Fixed layer
  Resistance states:
    Parallel (P):  low resistance  = "0"
    Anti-parallel (AP): high resistance = "1"

  Write: current through MTJ flips free layer orientation
    Write current: ~50-100 uA per cell
    Write latency: 5-20 ns (current-induced switching)
    Write energy: ~0.1-1 pJ per bit

  Read: sense resistance via small read current
    Read latency: 2-5 ns (similar to SRAM sense amplifier)
    Read energy: ~0.01 pJ per bit

Area: ~30-50 nm^2 per MTJ + 1 NMOS access transistor
  Cell area (7nm): ~0.03 um^2 (comparable to 6T SRAM)
  Advantage: non-volatile (retains data with zero power)
```

**STT-MRAM for last-level cache (LLC):**

| Parameter | SRAM (6T, 7nm) | STT-MRAM (1T-1MTJ, 7nm) |
|---|---|---|
| Cell area | 0.021 um^2 | 0.025 um^2 (comparable) |
| Read latency | 2-4 ns | 3-6 ns (slightly slower) |
| Write latency | 0.5-1 ns | 10-35 ns (significantly slower) |
| Endurance | > 10^16 cycles | 10^12-10^15 cycles |
| Standby power | ~10-100 uW/Mb (leakage) | ~0 (non-volatile, zero leakage) |
| Retention | Requires power | > 10 years at 85C |

**Key trade-off:** STT-MRAM eliminates standby power (critical for large LLCs that are idle most of the time) but write latency is 10-30x worse than SRAM. Write-aware LLC management policies (write-buffering, write-bypass for non-dirty lines) mitigate this.

**SOT-MRAM (Spin-Orbit Torque MRAM):**

```verilog
SOT-MRAM cell:
  1T + 1 SOT channel + 1 MTJ
  Write: current through heavy-metal SOT channel (not through MTJ)
    Write current: ~50-200 uA
    Write latency: 1-5 ns (much faster than STT, comparable to SRAM!)
    Write energy: ~0.05-0.5 pJ per bit

  Read: same as STT-MRAM (sense MTJ resistance)
    Read latency: 2-5 ns

  Key advantage: write current does NOT pass through the MTJ tunnel barrier,
  so endurance is effectively unlimited (> 10^16 cycles, matching SRAM).
  Write speed is 3-10x faster than STT-MRAM.

Area: ~0.04-0.06 um^2 (larger than STT due to additional SOT channel)
  Trade-off: faster write and unlimited endurance at 2x area cost
```

SOT-MRAM is projected to replace SRAM in last-level caches (L3/L4) in advanced nodes where SRAM leakage dominates total chip power. The combination of near-SRAM write speed, zero standby power, and unlimited endurance makes it attractive for server processors with 100+ MB LLCs that consume 30-50W of leakage power in SRAM.

### eMRAM as Embedded NVM

Embedded MRAM (eMRAM) uses STT-MRAM technology integrated into the logic CMOS process as an on-chip non-volatile memory replacement for embedded flash (eFlash).

```verilog
eMRAM characteristics:
  Capacity: 8-64 MB per die (integrated in Back-End-Of-Line BEOL metal stack)
  Read latency: 10-30 ns (random access, byte-addressable)
  Write latency: 50-200 ns (block write for efficiency)
  Endurance: 10^6 - 10^8 cycles (sufficient for firmware / config storage)
  Retention: > 10 years at 125C

  Use cases:
    - Secure boot ROM replacement (store boot code in eMRAM, updateable)
    - Firmware storage (microcontroller code, updateable in-field)
    - Configuration fuses (replace eFuses with re-programmable eMRAM)
    - AI weight storage (persistent model parameters in edge devices)
    - Instant-on memory (retain system state across power cycles)

  Integration: eMRAM sits in the interconnect metal layers (BEOL), not
  in the front-end-of-line (FEOL) with logic transistors. This means
  it can be added to any logic process (5nm, 3nm, 2nm) without
  modifying the FEOL transistor stack -- just add 2-3 extra BEOL masks.
```

---

## 12c. Near-Memory Processing: HBM-PIM

### Samsung HBM-PIM (Aquabolt-XL with Programmable Logic)

HBM-PIM integrates programmable compute logic directly into the HBM (High Bandwidth Memory) stack, adjacent to the DRAM banks. This enables computation to occur where the data resides, avoiding the energy and latency of moving data across the HBM-to-host interface.

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk", "nodeSpacing": 60, "rankSpacing": 60, "htmlLabels": false}}}%%
flowchart TD
    subgraph HBM["HBM stack — 8-high 3D-stacked DRAM (layers 0–7)"]
        B0["Bank 0"] --- P0["PIM-0<br/>ALU + MAC"]
        B1["Bank 1"] --- P1["PIM-1<br/>ALU + MAC"]
        B2["Bank 2"] --- P2["PIM-2<br/>ALU + MAC"]
    end
    classDef b fill:#dbeafe,stroke:#1d4ed8,color:#000
    classDef p fill:#fde68a,stroke:#b45309,color:#000
    class B0,B1,B2 b
    class P0,P1,P2 p
```

Each DRAM bank has an adjacent PIM unit that reads local bank data at DRAM-internal bandwidth (~256 GB/s per bank group), not limited by HBM I/O bandwidth. Per-PIM compute: 16-bit and 8-bit integer MAC (quantized inference), SIMD over 32 elements in parallel, a 256–1024 B local register file, and a microcoded instruction memory (load, MAC, store, reduce).

**HBM-PIM performance model:**

```verilog
Samsung Aquabolt-XL HBM-PIM (HBM2E-based):
  HBM stack bandwidth: 460 GB/s (host interface)
  Internal DRAM bank bandwidth: ~4x host BW = ~1.8 TB/s
  PIM compute: 128 INT8 MAC units per channel x 16 channels
             = 2048 MACs per cycle at 300 MHz = 1.2 TOPS per HBM stack

  For GEMM (matrix multiply) on 4096x4096 matrices:
    Traditional: read both matrices from HBM -> GPU -> write result back
      Data movement: ~100 GB, at 460 GB/s = ~0.22 s (bandwidth-bound)
    HBM-PIM: compute in HBM, only read result
      Data movement: ~0 (compute where data lives)
      Compute time: ~0.05 s (compute-bound at 1.2 TOPS)
      Speedup: ~4x, energy reduction: ~10x

Key use case: AI inference where model parameters are too large for on-chip SRAM.
  A 175B-parameter model at INT8: ~175 GB of weights.
  Must stream from HBM. With PIM, the streaming and compute happen simultaneously
  at the DRAM bank level, hiding the bandwidth bottleneck.
```

**Programming model challenges:**

- The host GPU/CPU must issue PIM-specific commands (via a command queue in the HBM controller). These are not standard DRAM commands -- they are custom opcodes that the PIM unit decodes and executes.
- Data layout: weights must be pre-arranged in DRAM bank order to maximize bank-level parallelism. Scatter-gather patterns are inefficient.
- Synchronization: the host must wait for PIM operations to complete before reading results. This requires a completion signal mechanism (e.g., a status register in the HBM controller that the host polls).

---

## 12d. DRAM Process Scaling Limits

### Capacitor Scaling

The DRAM storage capacitor must maintain sufficient charge (~25-30 fF) for reliable sensing despite shrinking cell area. At 1znm and beyond, capacitor scaling is approaching fundamental limits.

```verilog
DRAM capacitor scaling approaches:

  1x nm node: ~15.5 um^2 cell footprint, 25 fF capacitor
  1y nm node: ~12.5 um^2 cell footprint, ~22 fF
  1z nm node: ~10.0 um^2 cell footprint, ~18 fF
  1a nm node: ~8.0 um^2 cell footprint (projected), ~15 fF

  Capacitor height must increase to maintain capacitance as footprint shrinks:
    1x nm: ~1.0 um tall capacitor pillar
    1z nm: ~1.5 um tall (aspect ratio > 30:1, extremely difficult to etch)
    1a nm: ~2.0 um tall (aspect ratio > 50:1, near manufacturing limit)

  High-k dielectric materials (ZrO2, Al2O3-based) have been adopted to
  boost capacitance per unit area, but further k-value improvements are
  approaching diminishing returns.

  If capacitance drops below ~10 fF:
    Sense margin: delta_V < 20 mV (below reliable sense amplifier detection)
    Retention time drops below 64 ms at 85C (violates DDR spec)
    Soft error rate increases (cosmic ray charge overwhelms smaller capacitor)
```

### Access Transistor Scaling

The DRAM access transistor (1T in the 1T1C cell) faces scaling challenges similar to logic transistors, but with additional DRAM-specific constraints.

**Access transistor requirements:**
   - Low leakage: must hold capacitor charge for 64+ ms
   - I_off < 1 fA per cell (extremely low compared to logic transistors)
   - Sufficient I_on for sense amplifier detection
   - I_on > 30 uA per cell (to discharge bitline in ~10 ns)
   - Low Vth variation: sense margin is tiny, so Vth spread must be < 50 mV

Scaling challenge: reducing transistor size increases I_off (leakage).
DRAM uses high-Vth transistors (~0.8V threshold) to minimize leakage,
but this limits I_on and slows access time.

Solutions:
- Recessed channel array transistor (RCAT): wrap gate around a recessed
   - channel to increase effective channel length without increasing footprint.
   - Saddle-fin / U-shape gate: 3D gate structure for increased drive current
   - at the same footprint.
   - These are DRAM-specific transistor architectures not used in logic CMOS.

### Implications for Memory Controllers

As DRAM scaling slows (doubling time increasing from ~2 years to ~4+ years), memory controllers must extract more performance from the same DRAM technology:

- **Higher burst lengths and wider channels:** DDR5 BL16 (128 bytes per READ) and MRDIMM multiplexing maximize per-command data transfer.
- **Same-bank refresh (SBR):** DDR5's SBR refreshes a subset of banks while others remain active, reducing refresh-induced latency spikes.
- **Error-aware scheduling:** As capacitor charge decreases, the controller must handle increasing soft-error rates, potentially scheduling more aggressive scrubbing or using stronger ECC (BCH or on-die ECC + system ECC).
- **Thermal-aware refresh:** High-density DRAM stacks (HBM, 3DS DIMMs) require temperature-compensated refresh rates. The controller monitors on-die thermal sensors and adjusts tREFI dynamically.

---

## 12. Content-Addressable Memory (CAM/TCAM)

### CAM Cell Design

A CAM cell stores a data bit and compares it against an input search bit in parallel across all entries. The basic binary CAM cell is built from a standard 6T SRAM cell with an additional 4T comparison circuit.

```verilog
10T CAM cell (6T SRAM core + 4T compare):

  Storage: M1-M6 (standard 6T cross-coupled inverters + access)
           Stores bit value Q and Q_bar

  Compare circuit:
           Search line SL ---- gate of M9 (NMOS)
           Search line_bar SL_bar ---- gate of M10 (NMOS)

           M9 drain ---- M7 (NMOS, gate = Q_bar) ---- Match line (ML)
           M10 drain -- M8 (NMOS, gate = Q) ----------- Match line (ML)

           M7, M9 series: conducts when Q_bar=1 AND SL=1 (mismatch)
           M8, M10 series: conducts when Q=1 AND SL_bar=1 (mismatch)

  Match line (ML): precharged to VDD
    If stored bit == search bit: neither path conducts, ML stays HIGH (match)
    If stored bit != search bit: one path conducts, ML discharges (mismatch)
```

For an $N$-entry CAM, the match line spans all $N$ entries. If ANY entry has a mismatch, its discharge path pulls the shared match line LOW. Only if ALL bits in ALL positions of an entry match does the match line remain HIGH.

### Binary CAM vs Ternary CAM

**Binary CAM:** Each cell stores 0 or 1 and matches only the exact same value. Used for exact-match lookups: MAC address tables, ARP caches, TLB tag comparisons.

**Ternary CAM (TCAM):** Each cell stores one of three states -- 0, 1, or X (don't care). The X state matches any input value. TCAM is implemented with two storage bits per cell, encoding the three states.

TCAM cell encoding (2 storage bits per cell):

| Stored value | Bit_A | Bit_B | Matches |
|---|---|---|---|
| 0 | 0 | 1 | search = 0 only |
| 1 | 1 | 0 | search = 1 only |
| X (don't care) | 0 | 0 | search = 0 or 1 (always match) |
| (invalid) | 1 | 1 | not used |

TCAM cell: 12T-16T (two SRAM storage cells + compare logic)
  Area: ~2x binary CAM cell

### TCAM Application: IP Routing (Longest Prefix Match)

IP routing tables store destination prefixes of varying lengths. A prefix 192.168.0.0/16 means "match the first 16 bits exactly, ignore the last 16 bits." This maps directly to TCAM: the first 16 bits are stored as 0/1 values, and the remaining 16 bits are stored as X (don't care).

```verilog
Routing table example:
  Entry 0: 10.0.0.0/8     -> Port A  (8-bit prefix, 24 X's)
  Entry 1: 10.1.0.0/16    -> Port B  (16-bit prefix, 16 X's)
  Entry 2: 10.1.2.0/24    -> Port C  (24-bit prefix, 8 X's)
  Entry 3: 0.0.0.0/0      -> Port D  (0-bit prefix, 32 X's = default route)

Search for 10.1.2.5:
  Entry 0 matches (10.x.x.x matches 10.0.0.0/8)
  Entry 1 matches (10.1.x.x matches 10.1.0.0/16)
  Entry 2 matches (10.1.2.x matches 10.1.2.0/24)  <- longest prefix
  Entry 3 matches (default route)

  Result: Entry 2 wins (longest prefix = most specific match)
  Priority encoder selects the highest-priority (lowest-index) matching entry
  where entries are ordered by prefix length (longest prefix = highest priority)
```

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk", "nodeSpacing": 60, "rankSpacing": 60, "htmlLabels": false}}}%%
flowchart TD
    A["Incoming packet: Destination IP 10.1.2.5"] --> B["TCAM Search: broadcast search key to all entries"]
    B --> C["Parallel compare across all entries"]
    C --> D["Match lines: Entry 0 MATCH, Entry 1 MATCH, Entry 2 MATCH, Entry 3 MATCH"]
    D --> E["Priority Encoder: select longest prefix, highest priority match"]
    E --> F["Output: Entry 2, Next hop Port C"]
```

### TCAM Power Consumption

TCAM is extremely power-hungry because every entry compares its stored value against the search key simultaneously. All match lines are precharged every cycle, and mismatching entries discharge their match lines through the comparison transistors.

```verilog
Power comparison (per bit, same process node):
  SRAM read (1 port):  1x (reference)
  Binary CAM search:   5-8x
  TCAM search:         10-20x

Reason for high TCAM power:
  1. All N entries are active every search cycle (SRAM activates 1 row)
  2. Match line precharge: N x C_ML x VDD^2 per search
  3. Search line capacitance: each SL drives N gates per column
  4. Comparison transistors: switching activity in mismatching entries
```

**Power reduction techniques:**

- **Selective precharge:** Only precharge entries in a subset of the TCAM bank per cycle, trading throughput for power. Common in low-power networking chips (2--4 searches per routing decision instead of 1).
- **Partitioned TCAM:** Divide the TCAM into blocks by prefix length. Only search the relevant block(s) based on the first few bits of the key. Reduces active entries by 50--80%.
- **Search line gating:** Disable search lines for columns that are "don't care" in all entries of a block, reducing switching capacitance.

### TCAM Sizing in Networking Chips

```verilog
Typical TCAM allocations in a datacenter switch ASIC (2024 generation):
  IPv4 routing table:     128K entries x 32-bit key = 4 Mb
  IPv6 routing table:     32K entries x 128-bit key = 4 Mb
  ACL rules:              64K entries x 144-bit key (with metadata) = 9 Mb
  QoS classification:     16K entries x 80-bit key = 1.3 Mb
  Total TCAM:             ~18 Mb

  At 10-20x SRAM power per bit:
  18 Mb TCAM power at 1 GHz search rate = 5-15 W (significant portion of chip TDP)

  Physical area (7nm): ~18 Mb x 0.2 um^2/bit = ~3.6 mm^2 (TCAM cell area)
  With periphery: ~8-10 mm^2 total
```

### Alternative: Hash-Based Lookup in SRAM

For applications where exact match suffices (or where multiple hash probes are acceptable), SRAM with hash-based lookup offers much lower power at the cost of variable lookup latency.

```verilog
Hash lookup vs TCAM comparison:

  TCAM:
    Latency: 1 cycle (deterministic, parallel compare)
    Power: very high (all entries active)
    Area: 2-3x SRAM per bit
    Supports prefix match: yes (via don't-care bits)

  SRAM hash table:
    Latency: 2-5 cycles (hash compute + SRAM read + chain/probe)
    Power: low (only accessed SRAM rows consume energy)
    Area: standard SRAM density
    Supports prefix match: no (exact match only, or limited via multiple tables)

  Hybrid approach:
    Use TCAM for prefix matching (routing, ACLs)
    Use SRAM hash for exact match (MAC tables, flow caches)
    Typical split in switch ASICs: 30% TCAM, 70% SRAM hash tables
```

### Interview-Style Questions

**Q: Why does CAM cell area scale poorly with port count compared to SRAM?**

**A:** SRAM ports add wordlines and bitlines, which increase cell area quadratically. CAM adds search lines (analogous to bitlines for comparison) and match lines (analogous to wordlines for output). Each additional search dimension requires its own comparison transistor stack per cell, and each additional match dimension requires another match line. Since CAM is already 10T-16T per cell (versus 6T for SRAM), adding multi-search capability quickly makes the cell impractically large. This is why TCAM is almost always single-search (one lookup per cycle per bank) whereas SRAM supports multi-port access.

**Q: Calculate the storage overhead for protecting a 64-byte cache line with SECDED ECC.**

**A:** A 64-byte line contains 512 data bits. With Hamming-based SECDED, the number of check bits $r$ satisfies $2^r \geq 512 + r + 1$. For $r = 10$: $2^{10} = 1024 \geq 523$. So 10 check bits per 512-bit code word. In practice, the line is divided into eight 64-bit sub-words, each protected by its own Hamming(72,64) code: $8 \times 8 = 64$ check bits = 8 bytes per 64-byte line. Overhead: 8/64 = 12.5%. Each sub-word is independently correctable, so up to 8 single-bit errors per line can be corrected (one per sub-word).

**Q: Why does a 4-wide OoO processor need 8 read ports on the physical register file even though it only has 4 source operands per cycle?**

**A:** The physical register file serves both the rename stage (which reads architectural register mappings) and the issue stage (which reads source operand values). In many designs, the wakeup/select logic reads the register file speculatively. Additionally, some instructions (like conditional moves or fused multiply-add) read 3 source operands. Accounting for these cases: 4 instructions x 2--3 sources = 8--12 read ports. Conservative designs allocate 8R for a 4-wide machine. Some processors reduce this by using operand caching (cache recently-read values at the issue queue) to avoid re-reading the register file for dependent instructions.
