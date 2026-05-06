# Memory Architecture and Design — Senior Engineer Level

## 6T SRAM Cell — Transistor-Level Deep Dive

### Transistor Schematic with Sizing

```
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
```
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

```
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

```
I_M5 = Kn_access * [(VDD - Vth) * V_Q - V_Q^2/2]
I_M1 = Kn_pulldown * [(VDD - Vth) * V_Q - V_Q^2/2]
```

Wait — M1's gate is at Q_bar = VDD (assuming Q_bar hasn't changed), and M1's source is GND, drain is Q.
M5's gate is at VDD (WL=VDD), source is Q (lower voltage), drain is BL (VDD).

For M5 (in saturation if VDD - V_Q > VDD - Vth, i.e., V_Q < Vth... initially yes):
```
I_M5 ≈ (Kn_access/2) * (VDD - Vth)^2   (in saturation)
```

For M1 (in linear region since V_GS = VDD and V_DS = V_Q ≈ 0):
```
I_M1 ≈ Kn_pulldown * [(VDD - Vth) * V_Q - V_Q^2/2]
```

At equilibrium, I_M5 = I_M1:
```
(Kn_access/2) * (VDD - Vth)^2 = Kn_pulldown * (VDD - Vth) * V_Q

V_Q = (Kn_access / (2 * Kn_pulldown)) * (VDD - Vth)
    = (1 / (2 * CR)) * (VDD - Vth)
```

**Numerical example with 65nm parameters:**
```
VDD = 1.0V, Vth = 0.3V, CR = 1.33

V_Q = (1 / (2 * 1.33)) * (1.0 - 0.3) = 0.375 * 0.7 = 0.263V
```

**Is this safe?** For the cell to flip, V_Q must exceed the switching threshold of INV2 (which has Q as its input). The switching threshold of INV2 ≈ VDD * sqrt(Kn_pd/Kp_pu) / (1 + sqrt(Kn_pd/Kp_pu)). For our sizing with pull-down 2x stronger than pull-up, Vm_inv ≈ 0.4V.

V_Q = 0.263V < Vm_inv = 0.4V → **SAFE** (with margin of 0.137V).

**When does the cell flip?** If CR is reduced:
```
CR = 1.0: V_Q = 0.5 * 0.7 = 0.35V  (margin = 0.05V — dangerously close!)
CR = 0.8: V_Q = 0.625 * 0.7 = 0.44V > 0.4V → CELL FLIPS! Read-disturb failure!
```

**This is why CR > 1 is essential.** Typical designs use CR = 1.2-2.0 for adequate read margin.

### Write-Ability — Pull-Up vs Access Transistor Fight

**Setup:** Cell stores Q=1, Q_bar=0. We want to write Q=0. Drive BL=0 (strong driver), BL_bar=1. Assert WL.

**The fight:** M5 (access, WL=VDD) tries to pull Q from VDD to 0 through BL=0. But M3 (PMOS pull-up, gate=Q_bar=0, so M3 is ON) tries to keep Q at VDD.

```
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
```
V_Q = VDD * R_access / (R_access + R_pullup)

For V_Q < Vm_inv (must pull Q below switching threshold):
R_access / (R_access + R_pullup) < Vm_inv / VDD
R_pullup > R_access * (VDD - Vm_inv) / Vm_inv
```

This translates to: `(W/L)_access / (W/L)_pullup > some threshold`, which is the Pull-Up Ratio (PR).

**Numerical:** With PR = 1.5 (our sizing), M5 is 1.5x stronger than M3. The access transistor wins, pulling Q low enough to trip INV1, which then drives Q_bar high, which turns off M3 (breaking the fight), and the cell flips. Write succeeds.

**If PR < 1 (weak access, strong pull-up):** M3 wins, Q stays high, write fails.

### The Fundamental 6T Trade-Off

```
Read stability wants: Strong pull-down, WEAK access → HIGH CR
Write-ability wants:  Strong access, WEAK pull-up   → HIGH PR

But increasing access transistor strength IMPROVES PR and DEGRADES CR!
```

**This is THE fundamental tension in 6T SRAM design.** In advanced nodes (7nm, 5nm) with VDD scaling, both margins shrink, and balancing them becomes extremely challenging. This is why 8T cells are increasingly used.

### Butterfly Curve and Static Noise Margin (SNM)

The butterfly curve is constructed by plotting the voltage transfer characteristics (VTC) of the two cross-coupled inverters:

```
INV1: V_Qbar = f(V_Q)      (input Q, output Q_bar)
INV2: V_Q = g(V_Qbar)      (input Q_bar, output Q)

Plot both on the same axes: V_Q (x-axis) vs V_Qbar (y-axis)
  INV1: V_Qbar = f(V_Q)     → plot normally
  INV2: V_Q = g(V_Qbar)     → plot as V_Qbar vs V_Q (mirror)
```

The two curves intersect at three points: two stable states (Q=0, Qbar=VDD) and (Q=VDD, Qbar=0), and one metastable point (both ≈ VDD/2).

**SNM = side of the largest square that fits inside either "eye" of the butterfly curve.**

```
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

```
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
```
6T at VDD = 0.6V: Read SNM ≈ 20 mV (essentially zero)
8T at VDD = 0.6V: Read SNM ≈ 250 mV (same as hold SNM, very robust)
```

**Disadvantage:** ~30% larger area. The two extra transistors plus the separate read bitline increase cell size from ~0.05 um^2 to ~0.065 um^2 (in 7nm).

### 10T SRAM Cell

Adds a differential read port (2 extra transistors on top of 8T, using both Q and Q_bar for read). This provides faster read (differential sensing) with the same read-decoupled advantage. Used in register files where speed is critical.

---

## DRAM — Detailed Design

### 1T1C Cell — Charge Sharing Equation

```
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
```
Q_total = C_s * V_Cs + C_BL * V_BL(precharge)

V_BL_final = Q_total / (C_s + C_BL)
           = (C_s * V_Cs + C_BL * VDD/2) / (C_s + C_BL)
```

**For stored "1" (V_Cs = VDD):**
```
V_BL_final = (C_s * VDD + C_BL * VDD/2) / (C_s + C_BL)
           = VDD/2 + (C_s / (C_s + C_BL)) * VDD/2
           = VDD/2 + delta_V

delta_V = C_s / (C_s + C_BL) * VDD/2
```

**For stored "0" (V_Cs = 0):**
```
V_BL_final = (C_s * 0 + C_BL * VDD/2) / (C_s + C_BL)
           = VDD/2 - (C_s / (C_s + C_BL)) * VDD/2
           = VDD/2 - delta_V
```

**Numerical example:**
```
Cs = 25 fF, C_BL = 250 fF, VDD = 1.0V

delta_V = 25/(25+250) * 0.5 = 25/275 * 0.5 ≈ 45.5 mV
```

Only ~45 mV of signal! This is why DRAM needs sensitive sense amplifiers.

### Sense Amplifier — Cross-Coupled Latch

```
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

### Refresh Requirements and Power Impact

```
Retention time: time for Cs to leak enough charge for the sense amp to misread
  Typical: 64 ms (DDR4), 32 ms (LPDDR5 at high temperature)

Refresh cycle: each row must be read and restored within the retention time
  For a 16 Gb DRAM with 128K rows:
    Refresh all rows in 64 ms → 1 row every 64ms / 128K ≈ 488 ns
    Each refresh takes ~50 ns (tRC)

Refresh bandwidth overhead:
  50 ns / 488 ns ≈ 10.2% of total bandwidth consumed by refresh
```

**Power impact:** In mobile DRAM (LPDDR), refresh power can be 30-40% of total DRAM power during idle (self-refresh). Techniques to reduce:
- **Per-bank refresh:** Only one bank is being refreshed at a time; other banks can be accessed
- **Targeted refresh:** Only refresh rows near a frequently-accessed row (rowhammer defense)
- **Temperature-compensated refresh:** Reduce refresh rate at low temperature (retention time is longer)

---

## Asynchronous FIFO — Complete RTL with All Signals

### Architecture Overview

```
         Write Clock Domain          │         Read Clock Domain
                                     │
  wdata ──► ┌─────────────────┐      │     ┌─────────────────┐ ──► rdata
  wen   ──► │  Write Logic    │      │     │  Read Logic      │ ◄── ren
  wrst_n──► │                 │      │     │                  │ ◄── rrst_n
            │  wr_ptr_bin     │      │     │  rd_ptr_bin      │
            │  wr_ptr_gray ───┼──sync──►   │  rd_ptr_gray ────┼──sync──►
            │                 │    │ ◄──sync┼──                │       │
            │  rd_ptr_gray_s2 │◄───┼────   │  wr_ptr_gray_s2  │◄──────┘
            │                 │    │       │                   │
  full  ◄── │  Full Logic     │    │       │  Empty Logic      │ ──► empty
  afull ◄── │  Almost Full    │    │       │  Almost Empty     │ ──► aempty
            └────────┬────────┘    │       └────────┬─────────┘
                     │             │                │
                     ▼             │                ▼
              ┌──────────────────────────────────────┐
              │      Dual-Port RAM (2^ADDR_W deep)    │
              │      Write Port A    Read Port B       │
              └──────────────────────────────────────┘
```

### Complete Synthesizable Verilog

```verilog
module async_fifo #(
    parameter DATA_W    = 8,
    parameter ADDR_W    = 4,     // Depth = 2^ADDR_W = 16
    parameter AFULL_TH  = 2,     // Almost-full threshold (full - AFULL_TH)
    parameter AEMPTY_TH = 2      // Almost-empty threshold (empty + AEMPTY_TH)
) (
    // Write interface
    input                   wclk,
    input                   wrst_n,
    input                   wen,
    input  [DATA_W-1:0]     wdata,
    output                  full,
    output                  almost_full,

    // Read interface
    input                   rclk,
    input                   rrst_n,
    input                   ren,
    output [DATA_W-1:0]     rdata,
    output                  empty,
    output                  almost_empty
);

    localparam DEPTH = 1 << ADDR_W;
    localparam PTR_W = ADDR_W + 1;  // Extra MSB for wrap-around detection

    // =========================================================
    // Dual-port RAM
    // =========================================================
    reg [DATA_W-1:0] mem [0:DEPTH-1];

    // Write port
    always @(posedge wclk) begin
        if (wen && !full)
            mem[wr_bin[ADDR_W-1:0]] <= wdata;
    end

    // Read port (asynchronous read for FIFO — or registered for timing)
    assign rdata = mem[rd_bin[ADDR_W-1:0]];

    // =========================================================
    // Write pointer (binary and Gray)
    // =========================================================
    reg [PTR_W-1:0] wr_bin;
    wire [PTR_W-1:0] wr_bin_next;
    wire [PTR_W-1:0] wr_gray, wr_gray_next;

    assign wr_bin_next  = wr_bin + (wen & ~full);
    assign wr_gray      = wr_bin ^ (wr_bin >> 1);
    assign wr_gray_next = wr_bin_next ^ (wr_bin_next >> 1);

    always @(posedge wclk or negedge wrst_n) begin
        if (!wrst_n)
            wr_bin <= {PTR_W{1'b0}};
        else
            wr_bin <= wr_bin_next;
    end

    // Register the Gray code (for clean CDC)
    reg [PTR_W-1:0] wr_gray_reg;
    always @(posedge wclk or negedge wrst_n) begin
        if (!wrst_n)
            wr_gray_reg <= {PTR_W{1'b0}};
        else
            wr_gray_reg <= wr_gray_next;
    end

    // =========================================================
    // Read pointer (binary and Gray)
    // =========================================================
    reg [PTR_W-1:0] rd_bin;
    wire [PTR_W-1:0] rd_bin_next;
    wire [PTR_W-1:0] rd_gray, rd_gray_next;

    assign rd_bin_next  = rd_bin + (ren & ~empty);
    assign rd_gray      = rd_bin ^ (rd_bin >> 1);
    assign rd_gray_next = rd_bin_next ^ (rd_bin_next >> 1);

    always @(posedge rclk or negedge rrst_n) begin
        if (!rrst_n)
            rd_bin <= {PTR_W{1'b0}};
        else
            rd_bin <= rd_bin_next;
    end

    reg [PTR_W-1:0] rd_gray_reg;
    always @(posedge rclk or negedge rrst_n) begin
        if (!rrst_n)
            rd_gray_reg <= {PTR_W{1'b0}};
        else
            rd_gray_reg <= rd_gray_next;
    end

    // =========================================================
    // Synchronizers: Write Gray pointer → Read domain
    // =========================================================
    reg [PTR_W-1:0] wr_gray_sync1, wr_gray_sync2;

    always @(posedge rclk or negedge rrst_n) begin
        if (!rrst_n) begin
            wr_gray_sync1 <= {PTR_W{1'b0}};
            wr_gray_sync2 <= {PTR_W{1'b0}};
        end else begin
            wr_gray_sync1 <= wr_gray_reg;
            wr_gray_sync2 <= wr_gray_sync1;
        end
    end

    // =========================================================
    // Synchronizers: Read Gray pointer → Write domain
    // =========================================================
    reg [PTR_W-1:0] rd_gray_sync1, rd_gray_sync2;

    always @(posedge wclk or negedge wrst_n) begin
        if (!wrst_n) begin
            rd_gray_sync1 <= {PTR_W{1'b0}};
            rd_gray_sync2 <= {PTR_W{1'b0}};
        end else begin
            rd_gray_sync1 <= rd_gray_reg;
            rd_gray_sync2 <= rd_gray_sync1;
        end
    end

    // =========================================================
    // Empty detection (in read clock domain)
    // =========================================================
    // FIFO is empty when read Gray pointer == synchronized write Gray pointer
    assign empty = (rd_gray_next == wr_gray_sync2);

    // =========================================================
    // Full detection (in write clock domain)
    // =========================================================
    // FIFO is full when write Gray pointer matches read Gray pointer
    // with the top TWO MSBs inverted (indicating one full wrap-around apart)
    assign full = (wr_gray_next == {~rd_gray_sync2[PTR_W-1:PTR_W-2],
                                      rd_gray_sync2[PTR_W-3:0]});

    // =========================================================
    // Almost full / almost empty (using binary pointers)
    // =========================================================
    // Convert synchronized Gray pointers back to binary for arithmetic
    // (Gray-to-binary conversion)
    wire [PTR_W-1:0] rd_bin_sync;
    wire [PTR_W-1:0] wr_bin_sync;

    // Gray to binary for synchronized read pointer (in write domain)
    assign rd_bin_sync[PTR_W-1] = rd_gray_sync2[PTR_W-1];
    genvar i;
    generate
        for (i = PTR_W-2; i >= 0; i = i - 1) begin : g2b_rd
            assign rd_bin_sync[i] = rd_bin_sync[i+1] ^ rd_gray_sync2[i];
        end
    endgenerate

    // Gray to binary for synchronized write pointer (in read domain)
    assign wr_bin_sync[PTR_W-1] = wr_gray_sync2[PTR_W-1];
    generate
        for (i = PTR_W-2; i >= 0; i = i - 1) begin : g2b_wr
            assign wr_bin_sync[i] = wr_bin_sync[i+1] ^ wr_gray_sync2[i];
        end
    endgenerate

    // Fill level in write domain (for almost_full)
    wire [PTR_W-1:0] fill_level_w = wr_bin - rd_bin_sync;
    assign almost_full = (fill_level_w >= DEPTH - AFULL_TH);

    // Fill level in read domain (for almost_empty)
    wire [PTR_W-1:0] fill_level_r = wr_bin_sync - rd_bin;
    assign almost_empty = (fill_level_r <= AEMPTY_TH);

endmodule
```

### Gray Code Pointer Crossing — Metastability Scenario

**Timing diagram showing what happens when the write pointer changes during read-domain sampling:**

```
Write domain (wclk):
  wr_gray_reg:   0100 ──────► 0110  (single bit change: bit[1] 0→1)
                           ↑
                    write happens here

Read domain (rclk):
  Synchronizer FF1 samples wr_gray_reg:
  
  Case A: FF1 samples BEFORE the write pointer changes
    FF1 captures 0100 → FF2 gets 0100 (old value, conservative)
    Empty logic sees fewer entries → reports empty or almost-empty
    SAFE: FIFO under-reports fullness (conservative)

  Case B: FF1 samples AFTER the write pointer changes
    FF1 captures 0110 → FF2 gets 0110 (new value)
    Empty logic sees actual fill level
    SAFE: correct

  Case C: FF1 samples DURING the transition (metastability!)
    Only bit[1] is changing (Gray code guarantees single-bit transition)
    FF1 either resolves to 0100 or 0110 (both valid)
    If 0100: same as Case A (conservative but safe)
    If 0110: same as Case B (correct)
    SAFE in either case!
```

**Key insight:** Because Gray code ensures only ONE bit changes, the synchronizer can only resolve to the old or new value — never to a bogus intermediate. Binary counter: if multiple bits change (e.g., 0111 → 1000, all 4 bits flip), the synchronizer could capture any of 16 intermediate values (like 1111, 0000, 1010...), most of which are completely wrong.

### Proof of Full/Empty Detection Correctness

**Empty detection (read domain): rd_gray == wr_gray_sync2**

**Why this works:** If wr_gray_sync2 shows the same value as rd_gray, then the read pointer has caught up to the write pointer. Because synchronization can only show an OLD write pointer (never newer), if wr_gray_sync2 == rd_gray, it means the write pointer was AT LEAST at rd_gray some time ago, and may have advanced further. So the FIFO has at most 0 entries (empty) or possibly more (if writes happened since the last sync). Reporting empty when there might be data is PESSIMISTIC but SAFE — the reader won't read garbage, it just waits one more cycle.

**Full detection (write domain): wr_gray == {~rd_gray_sync2[MSB:MSB-1], rd_gray_sync2[rest]}**

**Why the top-2-bit inversion?** In binary, full means wr_ptr - rd_ptr = DEPTH (wr has wrapped around exactly once more). In binary, this means the MSBs differ and the rest are equal. In Gray code:

```
Binary wr_ptr = rd_ptr + DEPTH
If rd_ptr = 0000, wr_ptr = 10000 (5-bit for depth-16)
Gray(0000) = 0000
Gray(10000) = 11000  (note: top two bits are inverted, rest same!)
```

This pattern holds for all pointer values because of how Gray code reflects at the MSB boundary. The top two MSBs of Gray code encode the "quadrant" of the binary counter, and being one full wrap apart means being in the diametrically opposite quadrant, which differs in exactly the top 2 Gray bits.

**Safety:** Since rd_gray_sync2 is a possibly-old read pointer, the FIFO may report full when the reader has actually advanced (pessimistic). This is SAFE — the writer waits unnecessarily but never overflows.

---

## Memory Compiler and Library Files

### What You Specify to a Memory Compiler

Memory compilers (e.g., ARM Artisan, Synopsys, TSMC Memory Compiler) take these inputs:

```
- Word count (depth): e.g., 1024
- Word width (bits): e.g., 32
- Number of ports: 1 (single-port), 2 (dual-port), 1R/1W
- Mux factor: column muxing ratio (4, 8, 16) — trades width for height
- Bit write enable: byte-enable or full-word only
- Power mode: high-speed, high-density, low-leakage
- Redundancy: spare rows/columns for repair
- Corner: process corners to generate .lib files for (SS, TT, FF, etc.)
```

### What You Get Back

```
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
```

### How to Read a Memory .lib File

```
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

## Memory BIST — March C- Algorithm Detailed Analysis

### Fault Models

| Fault         | Abbr | Description                                | How to Detect            |
|---------------|------|--------------------------------------------|--------------------------|
| Stuck-At 0    | SA0  | Cell permanently reads 0                   | Write 1, read 1          |
| Stuck-At 1    | SA1  | Cell permanently reads 1                   | Write 0, read 0          |
| Transition 0→1| TF↑  | Cell cannot transition from 0 to 1         | Write 0, write 1, read 1 |
| Transition 1→0| TF↓  | Cell cannot transition from 1 to 0         | Write 1, write 0, read 0 |
| Coupling (inv)| CFin | Writing cell j inverts cell i              | Write j, read i          |
| Coupling (st) | CFst | Writing cell j sets cell i to fixed value  | Write j, read i          |
| Address fault | AF   | Address decoder maps two addresses to same cell | Write A, read B → should differ |

### March C- Algorithm: Step by Step

Notation: ↑ = ascending address, ↓ = descending address, (r0) = read expecting 0, (w1) = write 1.

```
Step 0: ↑↓(w0)         — Write 0 to all addresses (any order)
Step 1: ↑(r0, w1)      — Ascending: read 0 (verify), write 1
Step 2: ↑(r1, w0)      — Ascending: read 1 (verify), write 0
Step 3: ↓(r0, w1)      — Descending: read 0 (verify), write 1
Step 4: ↓(r1, w0)      — Descending: read 1 (verify), write 0
Step 5: ↑↓(r0)         — Read 0 from all addresses (final check)
```

**Complexity:** 10N operations (N = number of addresses). For 1MB SRAM: 10M operations.

### Fault Coverage Proof

**SAF detection:**
- SA0: Step 1 writes 1, Step 2 reads 1 → if SA0, read returns 0 → DETECTED
- SA1: Step 0 writes 0, Step 1 reads 0 → if SA1, read returns 1 → DETECTED

**Transition fault detection:**
- TF↑ (can't go 0→1): Step 1 writes 1 after 0, Step 2 reads 1 → fails → DETECTED
- TF↓ (can't go 1→0): Step 2 writes 0 after 1, Step 3 reads 0 → fails → DETECTED

**Coupling fault (inversion) detection:**

Consider cells i and j where writing j inverts i (CFin):
- Steps 1-2 are ascending. If j < i: j is written (step 1 or 2) before i is read (later in same step or next step). If writing j inverts i, the subsequent read of i detects the corruption.
- Steps 3-4 are descending: catches the case where j > i (j is visited first in descending order, so its write corrupts i which is visited later).
- Together, ascending + descending covers all (i, j) pairs regardless of relative ordering.

**Coupling fault (static) detection:**

CFst: writing j forces i to a fixed value (e.g., 0). Step 1 ascending writes all to 1, then step 2 reads all as 1 in ascending order. If CFst forced some cell to 0 during step 1, step 2 detects it. Descending steps similarly cover the reverse-ordered pairs.

**Address fault detection:**

If addresses A and B map to the same physical cell: writing 1 to A then 0 to B overwrites the cell. Reading A returns 0 (wrong). Steps 1-4 exercise this: write 1 ascending, then read/write patterns that would expose aliased addresses.

---

## Cache — Detailed Set-Associative Design

### Worked Example: 32KB, 4-Way, 64B Lines

**Given:**
- Total cache size: 32 KB = 32,768 bytes
- Associativity: 4-way set-associative
- Line size: 64 bytes

**Derived parameters:**
```
Number of lines = 32768 / 64 = 512 lines total
Number of sets = 512 / 4 = 128 sets
```

**Address breakdown for 32-bit address:**
```
Offset bits = log2(64) = 6 bits  (byte within a line)
Index bits  = log2(128) = 7 bits  (selects the set)
Tag bits    = 32 - 6 - 7 = 19 bits

Address: [31:13 tag | 12:6 index | 5:0 offset]
```

**Hardware structure per set:**
```
Set k:
  Way 0: [Valid][Dirty][Tag (19b)][Data (64B = 512b)]
  Way 1: [Valid][Dirty][Tag (19b)][Data (64B = 512b)]
  Way 2: [Valid][Dirty][Tag (19b)][Data (64B = 512b)]
  Way 3: [Valid][Dirty][Tag (19b)][Data (64B = 512b)]
  LRU state: 3 bits (for pseudo-LRU) or 5 bits (for true LRU)
```

**Total storage:**
```
Data:  512 lines × 64 bytes = 32 KB (the actual cache data)
Tags:  512 lines × 19 bits = 9728 bits = 1.19 KB
Valid: 512 bits
Dirty: 512 bits
LRU:   128 sets × 3 bits = 384 bits (pseudo-LRU)

Total overhead: ~1.2 KB for 32 KB cache = ~3.7% overhead
```

**Read operation (hit path):**
```
1. Extract index [12:6], tag [31:13], offset [5:0] from address
2. Use index to read all 4 ways simultaneously:
   - 4 tag comparisons (19-bit each): tag == stored_tag[way_i] AND valid[way_i]
   - 4 data reads (512 bits each, but typically only read the requested word)
3. Hit detection: one-hot vector of 4 compare results
4. MUX: select the data from the matching way
5. Use offset to extract the requested byte/word from the 64B line

Latency: Tag read + comparator + MUX ≈ 2-4 cycles for L1 cache
```

### LRU Implementation

**True LRU for 4-way:**

Track the full ordering of 4 ways. There are 4! = 24 possible orderings → need 5 bits (ceil(log2(24)) = 5).

In practice, use a 4×4 matrix:

```
Matrix M[i][j] = 1 if way i was accessed more recently than way j

On access to way k:
  Set M[k][*] = 1  (row k, all columns)
  Set M[*][k] = 0  (column k, all rows)

LRU victim = way with all 0s in its row (accessed least recently)
```

Storage: 4×4 matrix, but diagonal is unused and matrix is antisymmetric → 4*3/2 = 6 bits per set.

For 8-way: 8*7/2 = 28 bits per set — expensive!

**Pseudo-LRU (Tree PLRU) for 4-way:**

Use a binary tree with 3 bits:
```
          bit[0]
         /      \
      bit[1]   bit[2]
      /  \      /  \
    W0   W1   W2   W3
```

Each bit points toward the "less recently used" subtree:
- bit[0] = 0: left subtree is LRU, = 1: right subtree is LRU
- bit[1] = 0: W0 is LRU of left pair, = 1: W1 is LRU
- bit[2] = 0: W2 is LRU of right pair, = 1: W3 is LRU

**On access to way k:** Set tree bits to point AWAY from k.
**On eviction:** Follow tree bits to find the LRU victim.

**Storage: only N-1 = 3 bits for 4-way** (vs 6 for true LRU).

For 8-way: 7 bits (vs 28 for true LRU). This is why PLRU is universally used for associativity ≥ 4.

**PLRU vs true LRU miss rate comparison (SPEC benchmarks, 32KB 4-way):**
```
True LRU miss rate:   ~4.2%
Pseudo-LRU miss rate: ~4.4%
Random replacement:   ~5.1%

PLRU is within 5% of true LRU for 4-way — negligible difference.
For 8-way, the gap widens slightly but PLRU is still >90% as effective.
```

### Write-Back State Machine

```
States: IDLE → TAG_CHECK → {HIT_UPDATE | MISS_ALLOCATE} → WRITEBACK → REFILL → IDLE

IDLE:
  Wait for CPU request
  → TAG_CHECK on any read/write

TAG_CHECK:
  Read tags for the indexed set
  Compare with request tag
  → HIT_UPDATE if any way matches (hit)
  → MISS_ALLOCATE if no match (miss)

HIT_UPDATE:
  Read hit: return data, update LRU
  Write hit: update cache line, set dirty bit, update LRU
  → IDLE

MISS_ALLOCATE:
  Select victim way using LRU
  Check if victim is dirty:
  → WRITEBACK if dirty (must write victim to memory first)
  → REFILL if clean (can immediately fetch new line)

WRITEBACK:
  Send victim line to next level (L2 or main memory)
  Wait for acknowledgment
  → REFILL

REFILL:
  Send request to next level for the missing line
  Wait for data
  Install new line: update tag, set valid, clear dirty
  Supply requested word to CPU (can do "critical word first")
  → IDLE
```

### Cache Coherence — MESI Protocol

MESI (Modified, Exclusive, Shared, Invalid) handles cache coherence in multi-core systems:

```
States per cache line:
  M (Modified):  Line is dirty, only copy, owned by this cache
  E (Exclusive): Line is clean, only copy, can write without bus transaction
  S (Shared):    Line is clean, multiple copies may exist
  I (Invalid):   Line is not valid in this cache
```

**State transitions:**

```
I → E: CPU read miss, no other cache has the line
        (Bus read, memory responds, line loaded as Exclusive)

I → S: CPU read miss, another cache has the line
        (Bus read, other cache responds or shares, both become Shared)

I → M: CPU write miss, no other cache has the line
        (Bus read-exclusive / read-for-ownership)

E → M: CPU write hit (silent transition, no bus traffic!)
        (This is the key advantage of E state over S)

S → I: Another CPU writes (invalidate bus transaction)
        (Snooped write-invalidate for this line)

M → I: Another CPU reads or writes
        (Must write back dirty data, then invalidate or transition to S)

M → S: Another CPU reads
        (Write back data, transition to Shared, other cache gets copy)

E → I: Another CPU reads (transition to Shared usually, or Invalid)

E → S: Another CPU reads (share the line)
```

**Why E state exists (vs just MSI):**
Without E, a write to a clean exclusive line (only copy) would require a bus transaction to upgrade S→M. With E state, the write is silently promoted E→M with NO bus traffic. For workloads with mostly private data, this eliminates a huge fraction of coherence traffic.

---

## Interview Q&A — Senior-Level Depth

### Q1: Derive the read stability condition for a 6T SRAM cell.

**A:** During read, BL (at VDD) connects to the "0" storage node through the access transistor, while the pull-down NMOS holds it at GND. The access and pull-down NMOS form a voltage divider. If the access transistor is too strong, the "0" node voltage rises above the inverter switching threshold, flipping the cell. Approximate analysis: V_Q ≈ (VDD - Vth) / (2 * CR) where CR = (W/L)_pulldown / (W/L)_access. For stability, V_Q must be below the switching threshold of the feedback inverter (~0.4 * VDD for typical sizing). With VDD=1.0V, Vth=0.3V: V_Q = 0.7/(2*CR). For V_Q < 0.4V: CR > 0.875. In practice, CR ≥ 1.2-2.0 is used for adequate margin including process variation.

### Q2: Why is 8T SRAM preferred for low-VDD operation?

**A:** In 6T, the read SNM degrades with VDD because the voltage divider margin between the access and pull-down transistors shrinks. At 0.6V, read SNM is nearly zero. The 8T cell adds a separate read port (2 NMOS in series) that doesn't connect to the storage nodes during read. The read SNM equals the hold SNM (much higher), enabling reliable operation down to ~0.4V. The cost is ~30% area increase. Modern SoCs use 8T for memories that must operate across a wide voltage range (e.g., retention SRAM that stays powered during deep sleep at 0.5V).

### Q3: Explain the DRAM charge sharing equation and why sense amplifiers are critical.

**A:** When the access transistor opens, the storage capacitor (Cs ≈ 25 fF) shares charge with the bitline parasitic (C_BL ≈ 250 fF). The bitline voltage shifts by delta_V = Cs/(Cs + C_BL) * VDD/2 ≈ 45 mV. This tiny signal must be amplified to full-rail by a cross-coupled latch sense amplifier (positive feedback amplifier). The sense amp is enabled after the bitline develops sufficient differential. The read is destructive — the sense amp must write back the amplified value (restore). Sense amplifier offset (mismatch between the two sides) must be < delta_V, requiring careful transistor matching.

### Q4: Prove that the async FIFO full/empty detection is safe (no false negatives).

**A:** Empty is detected in the read domain using the synchronized write pointer, which is possibly stale (2-3 cycles old). If wr_gray_sync2 == rd_gray, either: (a) the FIFO truly is empty, or (b) the FIFO has entries but the synchronized pointer hasn't caught up. Case (b) means we report empty when we shouldn't — but this is corrected on the next cycle when the pointer updates. The reader simply waits, which is safe. We never report NOT-empty when the FIFO IS empty (which would cause reading garbage). Similarly, full is detected using a stale read pointer — we may report full when the reader has freed space, but never report NOT-full when truly full. Both errors are "pessimistic" and self-correcting.

### Q5: Why must async FIFO depth be power-of-2?

**A:** Gray code only guarantees single-bit transitions for the full 2^N sequence. For non-power-of-2 depth D, the counter wraps from Gray(D-1) to Gray(0), which may differ in multiple bits. Example: depth=5, Gray(4)=0110, Gray(0)=0000, 2 bits differ. When sampled across clock domains, any combination of the changing bits could be captured: 0110, 0010, 0100, 0000. Values 0010 (=Gray(3)=binary 2) and 0100 (=Gray(7)=binary 5) are completely wrong pointer values, causing the full/empty logic to malfunction catastrophically. Fix: use next power-of-2 depth and waste entries, or use more complex CDC schemes (e.g., handshake-based for non-power-of-2 depths).

### Q6: Explain the March C- algorithm and its fault coverage.

**A:** March C- has 6 elements totaling 10N operations: (1) ↑↓w0 (initialize), (2) ↑(r0,w1) (verify 0, write 1 ascending), (3) ↑(r1,w0) (verify 1, write 0 ascending), (4) ↓(r0,w1) (descending), (5) ↓(r1,w0) (descending), (6) ↑↓r0 (final check). It detects: all stuck-at faults (steps 2-5 read both values), all transition faults (steps write both transitions and verify), all inversion coupling faults (ascending catches j<i cases, descending catches j>i), and idempotent coupling faults. The ascending/descending pair is crucial — without both directions, coupling faults between cells with certain address relationships would be missed.

### Q7: Calculate tag/index/offset bits for a 64KB, 8-way, 128B-line cache with 64-bit addressing.

**A:** Lines = 64KB/128B = 512. Sets = 512/8 = 64. Offset = log2(128) = 7 bits. Index = log2(64) = 6 bits. Tag = 64 - 7 - 6 = 51 bits. This is a LOT of tag storage: 512 lines × 51 bits = 3.2 KB just for tags (5% of data size). In 64-bit systems, tag overhead becomes significant, which is one reason larger caches use larger line sizes (more data per tag entry) and why virtual-index-physical-tag (VIPT) schemes are used for L1 caches (reduces tag width by using virtual address for index).

### Q8: Compare true LRU with pseudo-LRU for 8-way cache.

**A:** True LRU for 8-way needs log2(8!) ≈ 15.3 → 16 bits per set (or 28 bits using the matrix method). Pseudo-LRU (tree) needs only 7 bits (N-1). The hardware for true LRU is complex — updating the full ordering on every access requires a priority encoder and shift register. PLRU uses a simple binary tree with 3 levels of MUXes. Miss rate difference: PLRU is within 1-5% of true LRU on most workloads, and the gap narrows with higher associativity (because the "wrong" eviction is less likely to matter when there are many ways). Random replacement is 10-20% worse. All modern high-performance caches use PLRU or NRU (not recently used).

### Q9: Explain MESI protocol and why the E state matters.

**A:** MESI adds an Exclusive state to the MSI protocol. A line in E is the only copy and is clean. The critical benefit: a write to an E-state line silently transitions to M without any bus traffic (no invalidation needed because no other cache has a copy). Without E (pure MSI), the same write would require a bus invalidation transaction even though no other cache has the line. For workloads with mostly private data (typical in many applications), E eliminates 20-50% of coherence bus traffic. The overhead is minimal: one extra state bit per line and slightly more complex state machine.

### Q10: What is a memory compiler and what parameters affect its output?

**A:** A memory compiler is a tool that generates custom SRAM/ROM instances from parameterized templates. You specify: word count, word width, port configuration, mux factor, redundancy, power mode. It outputs .lib (timing), .lef (physical), .v (simulation model), .gds (layout). Key trade-offs: (1) Mux factor: higher mux = shorter memory (better for physical design) but slower (longer internal bitlines). (2) Redundancy: spare rows/columns increase area but dramatically improve yield. (3) Power mode: high-speed uses larger transistors and more area; low-leakage uses high-Vth devices and power gating. A 256x32 SRAM in 7nm might have 0.5ns access time (high-speed) or 1.2ns (low-power), with 3x area difference.

### Q11: What is the "almost full" / "almost empty" flag and why is it useful?

**A:** Almost-full asserts when FIFO fill level exceeds (DEPTH - threshold). Almost-empty asserts when fill level falls below threshold. These are "early warning" flags that allow the producer/consumer to react before hitting the hard full/empty conditions. Use cases: (1) Flow control: when almost-full, send a backpressure signal to the upstream source so it can stop transmitting before data is lost. (2) Burst scheduling: when almost-empty, trigger a prefetch or DMA to refill the FIFO. (3) Performance: avoiding the full/empty conditions entirely keeps the pipeline flowing without stalls. Implementation challenge in async FIFOs: almost-full/empty require knowing the fill level, which means converting synchronized Gray pointers back to binary for arithmetic comparison — this adds a gray-to-binary converter on the critical path.

### Q12: Explain memory BIST architecture and why it's needed.

**A:** Memory BIST contains: (1) Controller FSM that sequences through the March algorithm, (2) Address generator (up/down counter for ascending/descending), (3) Data generator (produces the test patterns: 0s, 1s, checkerboard, etc.), (4) Comparator (compares read data with expected), (5) Fail register (stores failing address and data for diagnosis). BIST is needed because external ATE (Automatic Test Equipment) cannot access embedded memories directly — they're buried inside the SoC with no external pins. BIST generates all stimuli and evaluates all responses on-chip, reporting only pass/fail and optionally repair information via a serial scan chain. For a modern SoC with 100+ memory instances, BIST is the only practical way to achieve manufacturing test coverage.

### Q13: How does write-back cache handle a dirty eviction with a simultaneous cache miss?

**A:** This is a multi-step process handled by the cache controller state machine: (1) CPU misses in cache. (2) LRU selects a victim line. (3) If the victim is dirty, the cache must write it back to memory BEFORE the new line can be installed (the victim occupies the physical entry that the new line needs). (4) Write-back the dirty line: send address + data to the next level. (5) After write-back completes, issue the fill request for the missed line. (6) Receive the fill data, install in the now-free way. Optimization: **write-back buffer** — copy the dirty victim to a temporary buffer, immediately start the fill, and write the buffer back to memory in the background. This hides the write-back latency, reducing the miss penalty by 30-50%.

### Q14: Explain butterfly curve construction and SNM extraction.

**A:** The butterfly curve plots the voltage transfer characteristics (VTC) of the two cross-coupled inverters. INV1 maps V_Q → V_Qbar; INV2 maps V_Qbar → V_Q. Plot INV1's VTC normally (input on x-axis, output on y-axis). Plot INV2's VTC with its input/output swapped (so its input is on the y-axis and output on the x-axis). The two curves intersect at three points, forming two "eyes" (lobes). SNM is the side of the largest square that fits inside either eye. To measure: draw 45-degree lines tangent to both curves; the distance between the closest pair of tangent lines gives the SNM. In simulation: sweep V_Q from 0 to VDD while measuring V_Qbar from both inverters, with the SRAM cell and bitlines modeled. The read SNM includes the effect of the access transistors (WL on, BL precharged).

### Q15: What determines the access time of an SRAM memory macro?

**A:** Access time (clock-to-Q for read data) has these components: (1) Clock distribution to the wordline drivers and sense amplifiers (~10-15% of total). (2) Wordline driver + wordline RC delay (depends on memory width: wider memory = longer WL). (3) Bitcell access time (current through access + pull-down creating BL differential). (4) Bitline RC delay + sense amplifier input development (depends on memory depth: deeper = longer BL, smaller signal). (5) Sense amplifier detection + amplification (~20% of total). (6) Output driver + data mux (column mux, output register). For a typical 1K×32 SRAM in 28nm: total access time ≈ 1.0 ns. Doubling depth roughly adds 0.3-0.5 ns (longer bitlines). Doubling width adds less (~0.1-0.2 ns, longer wordlines). The mux factor trades depth for width: mux=4 reads 4× as many columns but only delivers one, making the memory shorter (fewer rows) but wider. This reduces BL delay but increases column MUX delay.
