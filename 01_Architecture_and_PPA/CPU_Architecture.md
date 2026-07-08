# CPU Architecture -- The Complete Interview Bible

## Table of Contents
1. Five-Stage Pipeline -- Detailed Microarchitecture
2. Pipeline Hazards with Instruction-Level Examples
3. Forwarding (Bypassing) -- Complete MUX Diagrams
4. Branch Prediction -- Deep Dive
5. Tomasulo's Algorithm -- Step-by-Step Execution
6. Superscalar and Register Renaming
7. Memory Hierarchy -- AMAT, Cache Design, Store Buffer, and MLP
8. Cache Coherence -- MESI with All Transitions
9. Virtual Memory and TLB
10. Performance Analysis
11. Interview Q&A (20+ Questions)
12. Fetch Unit Microarchitecture
13. RISC-V Instruction Examples
14. SMT Resource Partitioning
15. Exception Pipeline in Out-of-Order Execution
16. Memory Ordering and Consistency Models
17. Micro-op Fusion

---

## 1. Five-Stage Pipeline -- Detailed Microarchitecture

### 1.1 Stage 1: Instruction Fetch (IF)

```text
Inputs:  PC (Program Counter), Branch prediction signals
Outputs: Instruction bits, PC+4, predicted branch target

Components:
  - PC register (updated every cycle)
  - PC MUX: selects between:
    * PC + 4 (sequential)
    * Branch target (from branch predictor or resolved branch)
    * Exception/interrupt vector
  - Instruction Memory / I-Cache
  - Branch Target Buffer (BTB) lookup (parallel with I-Cache)
  - Next-PC logic

Operation:
  1. Send PC to I-Cache
  2. Simultaneously look up PC in BTB
  3. If BTB hit and BHT predicts taken: next PC  = BTB target
  4. If BTB miss or not taken: next PC           = PC + 4
  5. Fetch instruction from I-Cache (1 cycle for cache hit)
  6. On I-Cache miss: pipeline stalls, fill from L2 (10-20 cycles)

Pipeline register IF/ID:
  {instruction[31:0], PC+4, predicted_target, prediction_direction}
```
### 1.2 Stage 2: Instruction Decode / Register Read (ID)

```text
Inputs:  Instruction from IF/ID register
Outputs: Control signals, register values, immediate, decoded opcode

Components:
  - Instruction decoder (opcode -> control signals)
  - Register file (read ports: rs1, rs2; write port from WB stage)
  - Immediate generator (sign-extend, shift for different formats)
  - Hazard detection unit (detects RAW hazards, generates stalls)
  - Branch comparator (for branch-in-ID optimization)
  - Control signal generation:
    * ALUSrc (register or immediate)
    * ALUOp (add, sub, and, or, slt, etc.)
    * MemRead, MemWrite
    * RegWrite
    * MemtoReg (ALU result or memory data)
    * Branch type

Operation:
  1. Decode instruction opcode and function fields
  2. Read rs1 and rs2 from register file
  3. Generate immediate value
  4. Check for data hazards (compare rs1/rs2 with EX/MEM/WB destinations)
  5. If load-use hazard detected: insert pipeline bubble (stall IF, ID)

Pipeline register ID/EX:
  {rs1_data, rs2_data, immediate, rd, ALUOp, control_signals, PC+4}
```
### 1.3 Stage 3: Execute (EX)

```text
Inputs:  Operands from ID/EX register, forwarded values
Outputs: ALU result, branch decision, target address

Components:
  - ALU (arithmetic, logic, comparison)
  - Forwarding MUX on each ALU input (select: ID/EX, EX/MEM forward, MEM/WB forward)
  - Branch target adder (PC + immediate)
  - Branch resolution (compare register values)
  - Address calculation (for loads/stores: base + offset)

Operation:
  1. Select ALU operands (from register file or forwarded values)
  2. Execute ALU operation
  3. For branches: compare operands, determine taken/not-taken
  4. If branch misprediction: flush IF and ID stages, correct PC
  5. For loads/stores: compute effective address = rs1 + immediate

Pipeline register EX/MEM:
  {ALU_result, rs2_data (for store), rd, control_signals, branch_taken, target}
```
### 1.4 Stage 4: Memory Access (MEM)

```text
Inputs:  ALU result (address), store data from EX/MEM register
Outputs: Memory read data or ALU pass-through

Components:
  - Data Memory / D-Cache
  - Memory read/write control
  - Write buffer (for store buffering)

Operation:
  1. For loads: send address to D-Cache, receive data (1 cycle if hit)
  2. For stores: write data to D-Cache (or write buffer)
  3. For ALU instructions: pass ALU result through (no memory access)
  4. On D-Cache miss: pipeline stalls, fill from L2

Pipeline register MEM/WB:
  {ALU_result, mem_read_data, rd, control_signals}
```
### 1.5 Stage 5: Write Back (WB)

Inputs:  MEM/WB register
Outputs: Write to register file

**Components:**
   - Result MUX: select between ALU result and memory data
   - Register file write port

**Operation:**
   1. Select result (MemtoReg: memory data or ALU result)
2. Write to destination register rd (if RegWrite is asserted)
3. Write occurs on the first half of the clock cycle (allows
same-cycle read in ID stage -- "write-first" register file)

---

## 2. Pipeline Hazards with Instruction-Level Examples

### 2.1 RAW (Read After Write) -- True Data Dependency

```text
Instruction sequence (MIPS):
  I1: ADD  $1, $2, $3       # Writes $1 in WB (cycle 5)
  I2: SUB  $4, $1, $5       # Reads $1 in ID (cycle 3) -- $1 not yet written!
  I3: AND  $6, $1, $7       # Reads $1 in ID (cycle 4) -- $1 not yet written!
  I4: OR   $8, $1, $9       # Reads $1 in ID (cycle 5) -- $1 written this cycle (OK with write-first RF)

Pipeline without forwarding:
  Cycle:  1    2    3    4    5    6    7    8
  I1:    IF   ID   EX  MEM  WB
  I2:         IF   ID  stall stall EX  MEM  WB    <- 2-cycle stall for $1
  I3:              IF  stall stall ID   EX  MEM  WB
  I4:                   stall stall IF   ID   EX  MEM  WB

With forwarding (see Section 3):
  Cycle:  1    2    3    4    5    6    7    8
  I1:    IF   ID   EX  MEM  WB
  I2:         IF   ID   EX  MEM  WB               <- Forward from EX/MEM
  I3:              IF   ID   EX  MEM  WB          <- Forward from MEM/WB
  I4:                   IF   ID   EX  MEM  WB     <- Normal read (WB to ID same cycle)

  No stalls! Forwarding eliminates 2 stall cycles.
```
### 2.2 Load-Use Hazard (Forwarding Can't Solve It)

```text
  I1: LW   $1, 0($2)        # Load: data available END of MEM (cycle 4)
  I2: ADD  $4, $1, $5       # Needs $1 at BEGINNING of EX (cycle 3)

Pipeline:
  Cycle:  1    2    3    4    5    6    7
  I1:    IF   ID   EX  MEM  WB
  I2:         IF   ID  stall EX  MEM  WB     <- 1 stall UNAVOIDABLE
  I3:              IF  stall ID   EX  MEM  WB

Why forwarding can't help:
  I1 produces data at end of MEM (cycle 4 clock edge).
  I2 needs data at beginning of EX (cycle 3 clock edge).
  Data isn't ready yet! Must wait 1 cycle.

  After the stall: I2's EX is in cycle 4, and I1's MEM result is
  available at the end of cycle 4 -> can forward from MEM/WB to EX.

Compiler optimization (load delay slot scheduling):
  I1: LW   $1, 0($2)
  I3: AND  $6, $7, $8                        <- Independent instruction moved here
  I2: ADD  $4, $1, $5                        <- Now 1 cycle later, no stall needed
```
### 2.3 Control Hazards (Branch Penalty)

```text
  I1: BEQ  $1, $2, target   # Branch resolved in EX (cycle 3)
  I2: ADD  $3, $4, $5       # Fetched speculatively (may be wrong)
  I3: SUB  $6, $7, $8       # Fetched speculatively (may be wrong)

If branch is taken (mispredicted as not-taken):
  Cycle:  1    2    3    4    5
  I1:    IF   ID   EX  MEM  WB
  I2:         IF   ID  flush               <- Wrong path, flushed
  I3:              IF  flush               <- Wrong path, flushed
  target:               IF   ID   EX ...   <- Correct path starts

Branch penalty = 2 cycles (for branch resolved in EX)

If branch resolved in ID (some implementations):
  Penalty = 1 cycle

If branch resolved in MEM (deeply pipelined):
  Penalty = 3 cycles
```
### 2.4 WAR and WAW (Out-of-Order Only)

```text
WAR (Write After Read) - anti-dependency:
  I1: ADD  $1, $2, $3       # Reads $2 in cycle 2
  I2: SUB  $2, $4, $5       # Writes $2 in cycle 6

In-order pipeline: No hazard (I1 reads $2 before I2 writes it).
Out-of-order: I2 might execute before I1 if I2's operands are ready first.
  If I2 writes $2 before I1 reads it -> I1 gets wrong value.
Solution: Register renaming. I2 writes to physical reg P47 (renamed),
  I1 still reads from P12 (old mapping of $2). No conflict.

WAW (Write After Write):
  I1: ADD  $1, $2, $3       # Writes $1
  I2: SUB  $1, $4, $5       # Also writes $1

In-order: No hazard (I1 writes before I2, final value is from I2).
Out-of-order: I1 might write after I2, leaving $1 with I1's value (wrong!).
Solution: Register renaming. I1 writes P47, I2 writes P63 (different physical regs).
  Architectural register $1 maps to P63 after I2. P47 is freed later.
```
---

## 3. Forwarding (Bypassing) -- Complete MUX Diagrams

### 3.1 Forwarding Paths

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk", "nodeSpacing": 60, "rankSpacing": 60, "htmlLabels": false}}}%%
flowchart TD
    R1["ID/EX.rs1_data"] --> MA["MUX A"]
    EM1["EX/MEM.ALU_result"] --> MA
    MW1["MEM/WB.result"] --> MA
    FA["Forward_A ctrl"] --> MA
    MA --> AA["ALU Input A"]
    R2["ID/EX.rs2_data"] --> MB["MUX B"]
    EM2["EX/MEM.ALU_result"] --> MB
    MW2["MEM/WB.result"] --> MB
    FB["Forward_B ctrl"] --> MB
    MB --> AB["ALU Input B"]
    classDef mux fill:#fde68a,stroke:#b45309,color:#000
    classDef src fill:#dbeafe,stroke:#1d4ed8,color:#000
    classDef out fill:#dcfce7,stroke:#15803d,color:#000
    class MA,MB mux
    class R1,R2,EM1,EM2,MW1,MW2,FA,FB src
    class AA,AB out
```

Each MUX selects one of three inputs: `00` normal (register file via ID/EX); `01` forward from EX/MEM (1-cycle forward, ALU→ALU); `10` forward from MEM/WB (2-cycle forward, ALU→ALU or MEM→ALU). Load-to-use forwards from MEM/WB to EX after one stall cycle.

### 3.2 Forwarding Control Logic

```verilog
// Forward A (ALU input 1)
always @(*) begin
    if (EX_MEM_RegWrite && (EX_MEM_Rd != 0) && (EX_MEM_Rd == ID_EX_Rs1))
        ForwardA = 2'b01;  // Forward from EX/MEM
    else if (MEM_WB_RegWrite && (MEM_WB_Rd != 0) && (MEM_WB_Rd == ID_EX_Rs1))
        ForwardA = 2'b10;  // Forward from MEM/WB
    else
        ForwardA = 2'b00;  // No forwarding
end

// Forward B (ALU input 2) -- same logic with Rs2
always @(*) begin
    if (EX_MEM_RegWrite && (EX_MEM_Rd != 0) && (EX_MEM_Rd == ID_EX_Rs2))
        ForwardB = 2'b01;
    else if (MEM_WB_RegWrite && (MEM_WB_Rd != 0) && (MEM_WB_Rd == ID_EX_Rs2))
        ForwardB = 2'b10;
    else
        ForwardB = 2'b00;
end

// Priority: EX/MEM forward takes priority over MEM/WB forward
// (EX/MEM has the most recent value)
```
### 3.3 Hazard Detection Unit (for Load-Use Stall)

```text
// Detect load-use hazard: current ID stage reads what EX stage is loading
wire load_use_hazard = ID_EX_MemRead &&
                       ((ID_EX_Rd == IF_ID_Rs1) || (ID_EX_Rd == IF_ID_Rs2));

// When hazard detected:
//   1. Insert NOP into ID/EX register (bubble)
//   2. Stall IF and ID stages (don't advance PC, don't update IF/ID register)
//   3. After 1 cycle, the load result is available for forwarding from MEM/WB
```
---

## 4. Branch Prediction -- Deep Dive

### 4.1 CPI Impact of Branches

```text
CPI = CPI_base + Branch_Frequency * Misprediction_Rate * Penalty

Example:
  CPI_base          = 1 (ideal pipeline)
  Branch_Frequency  = 20% (1 in 5 instructions is a branch)
  Penalty           = 2 cycles (branch resolved in EX)

  With always-not-taken prediction (50% accuracy for conditional branches):
    CPI = 1 + 0.2 * 0.5 * 2 = 1.2  (20% performance loss)

  With 2-bit predictor (90% accuracy):
    CPI = 1 + 0.2 * 0.1 * 2 = 1.04  (4% loss)

  With tournament predictor (97% accuracy):
    CPI = 1 + 0.2 * 0.03 * 2 = 1.012  (1.2% loss)

  With deep pipeline (20 stages, penalty  = 15 cycles):
    At 97% accuracy: CPI                  = 1 + 0.2 * 0.03 * 15 = 1.09  (9% loss!)
    Even excellent prediction hurts with deep pipelines.
```

### 4.2 Two-Bit Saturating Counter -- FSM with Transitions

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk", "nodeSpacing": 60, "rankSpacing": 60, "htmlLabels": false}}}%%
stateDiagram-v2
    direction LR
    ST: Strongly Taken (11) · predict T
    WT: Weakly Taken (10) · predict T
    WNT: Weakly Not-Taken (01) · predict NT
    SNT: Strongly Not-Taken (00) · predict NT
    ST --> ST: Taken
    ST --> WT: Not Taken
    WT --> ST: Taken
    WT --> WNT: Not Taken
    WNT --> WT: Taken
    WNT --> SNT: Not Taken
    SNT --> WNT: Taken
    SNT --> SNT: Not Taken
```

Prediction rule: MSB = 1 → predict Taken; MSB = 0 → predict Not Taken. For a loop that runs 100×, a 1-bit predictor makes 2 mispredictions (first entry NT, last exit T); a 2-bit predictor makes only 1 (at exit — the weakly-taken state absorbs the first exit and stays taken for re-entry).

### 4.3 Branch History Table (BHT) Organization

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk", "nodeSpacing": 60, "rankSpacing": 60, "htmlLabels": false}}}%%
flowchart TD
    PC["PC"] --> LB["Lower bits · PC[11:2]"]
    LB --> BHT["BHT array<br/>Entry 0 … Entry N-1<br/>(each a 2-bit counter)"]
    BHT --> PR["Prediction<br/>(Taken / Not Taken)"]
    classDef s fill:#dbeafe,stroke:#1d4ed8,color:#000
    class PC,LB,BHT,PR s
```

BHT size is typically 1K–16K entries; the index is `PC[11:2]` for 1K entries (ignore the byte offset, align to instructions). The main problem is **aliasing** — different branches map to the same entry, with a collision rate that depends on table size and branch frequency.

### 4.4 BTB (Branch Target Buffer) Structure

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk", "nodeSpacing": 60, "rankSpacing": 60, "htmlLabels": false}}}%%
flowchart TD
    PC["PC"] --> TAG["BTB Tag RAM"]
    TAG --> Q{"Tag match?"}
    Q -->|Yes| DATA["BTB Data RAM"] --> TGT["Target address<br/>(next PC if taken)"]
    Q -->|No| NT["Predict Not Taken<br/>(or use BHT separately)"]
    classDef s fill:#dbeafe,stroke:#1d4ed8,color:#000
    classDef d fill:#fde68a,stroke:#b45309,color:#000
    class PC,TAG,DATA,TGT,NT s
    class Q d
```

Each BTB entry holds `{tag (upper PC bits), target_address, valid, branch_type}`; typical size 256–4K entries, 2–4-way set-associative. The BTB answers *where* to fetch if taken; the BHT answers *whether* to take (target vs PC+4). Both are needed for full prediction.

### 4.5 Gshare Predictor

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk", "nodeSpacing": 60, "rankSpacing": 60, "htmlLabels": false}}}%%
flowchart TD
    PC["PC"] --> X(("XOR"))
    GHR["GHR — global<br/>history register"] --> X
    X --> IDX["Index"] --> PHT["PHT — pattern history table<br/>(2-bit counters)"]
    PHT --> P["Prediction"]
    classDef s fill:#dbeafe,stroke:#1d4ed8,color:#000
    class PC,GHR,IDX,PHT,P s
```

The GHR is an N-bit shift register of the last N branch outcomes (Taken→1, Not Taken→0). `Index = PC[N-1:0] XOR GHR[N-1:0]`; the XOR gives a unique index per (PC, history) pair, capturing correlations such as "if the last two branches were taken, this one is usually not taken." Typical accuracy 93–95% on SPEC; GHR width 10–18 bits; PHT 2¹⁰–2¹⁸ entries.

### 4.6 Tournament Predictor

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk", "nodeSpacing": 60, "rankSpacing": 60, "htmlLabels": false}}}%%
flowchart TD
    PC["PC"] --> CH["Chooser / meta<br/>2-bit counter per entry"]
    CH --> L["Local predictor<br/>per-branch history"]
    CH --> G["Global predictor<br/>gshare-style shared history"]
    L --> Fr["Final prediction"]
    G --> Fr
    classDef s fill:#dbeafe,stroke:#1d4ed8,color:#000
    classDef c fill:#fde68a,stroke:#b45309,color:#000
    class PC,L,G,Fr s
    class CH c
```

The chooser selects local or global. On each outcome: if local was right and global wrong, bias the chooser toward local (and vice-versa); if both agree, no change. Local history suits branches whose pattern is tied to their own history (loops); global history suits correlated branches (if-else chains). Tournament accuracy is 95–97% on SPEC (the Alpha 21264 used this).

### 4.7 Prediction Accuracy Comparison

```text
| Predictor           | Accuracy (SPEC) | Hardware Cost   | Used In           |
|---------------------|-----------------|-----------------|-------------------|
| Always Not Taken    | 30-40%          | None            | -                 |
| Always Taken        | 60-70%          | None            | -                 |
| BTFNT               | 65-75%          | Minimal         | Simple embedded   |
| 1-bit BHT (1K)      | 80-85%          | 1KB             | -                 |
| 2-bit BHT (4K)      | 85-90%          | 8KB             | Early RISC        |
| Gshare (14-bit)     | 93-95%          | 32KB            | AMD K6            |
| Tournament          | 95-97%          | 64KB            | Alpha 21264       |
| TAGE                | 97-99%          | 64-256KB        | Intel Haswell+    |
| Perceptron          | 95-97%          | 64KB            | AMD Zen           |
```

---

## 5. Tomasulo's Algorithm -- Step-by-Step Execution

### 5.1 Architecture Recap

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk", "nodeSpacing": 60, "rankSpacing": 60, "htmlLabels": false}}}%%
flowchart TD
    IQ["Instruction queue"] --> ISS["Issue (in-order)"]
    ISS --> RSA["Reservation stations<br/>Add1 / Add2 / Add3"]
    ISS --> RSM["Reservation stations<br/>Mul1 / Mul2 / Mul3"]
    RSA --> ADD["Adder"]
    RSM --> MUL["Multiplier"]
    ADD --> CDB["Common Data Bus (CDB)"]
    MUL --> CDB
    CDB --> RF["Register file + ROB"]
    classDef q fill:#dbeafe,stroke:#1d4ed8,color:#000
    classDef rs fill:#fde68a,stroke:#b45309,color:#000
    classDef fu fill:#dcfce7,stroke:#15803d,color:#000
    class IQ,ISS,RF q
    class RSA,RSM rs
    class ADD,MUL,CDB fu
```

### 5.2 Step-by-Step Example

**Code sequence:**

```text
I1: LD    F6, 34(R2)       # Load F6 from memory[R2+34]
I2: LD    F2, 45(R3)       # Load F2 from memory[R3+45]
I3: MULTD F0, F2, F4       # F0   = F2 * F4 (depends on I2)
I4: SUBD  F8, F6, F2       # F8   = F6 - F2 (depends on I1, I2)
I5: DIVD  F10, F0, F6      # F10  = F0 / F6 (depends on I3, I1)
I6: ADDD  F6, F8, F2       # F6   = F8 + F2 (depends on I4, I2)
```

**Execution trace (cycle by cycle):**

```text
Cycle 1: Issue I1 (LD F6, 34(R2))
  Load1 RS: Op=LD, Addr=R2+34, Dest=F6
  Register status: F6 -> Load1

Cycle 2: Issue I2 (LD F2, 45(R3))
  Load2 RS: Op=LD, Addr=R3+45, Dest=F2
  Register status: F2 -> Load2

Cycle 3: Issue I3 (MULTD F0, F2, F4)
  Mul1 RS: Op=MULT, Vj=?, Qj=Load2 (F2 not ready), Vk=F4 value, Qk=0
  Register status: F0 -> Mul1
  I1 executes (LD completes, CDB broadcasts: Load1 -> F6 value)
  Load1 result written to F6 in register file

Cycle 4: Issue I4 (SUBD F8, F6, F2)
  Add1 RS: Op = SUB, Vj=F6 value (now available from I1), Qj=0,
           Vk = ?, Qk=Load2 (F2 still loading)
  Register status: F8 -> Add1
  I2 may execute this cycle

Cycle 5: Issue I5 (DIVD F10, F0, F6)
  Mul2 RS: Op=DIV, Vj=?, Qj=Mul1 (F0 not ready), Vk=F6 value, Qk=0
  Register status: F10 -> Mul2
  I2 completes (CDB broadcasts: Load2 -> F2 value)
  Mul1 captures F2 value (Qj = Load2 matches, now Vj=F2 value, Qj=0)
  Add1 captures F2 value (Qk = Load2 matches, now Vk=F2 value, Qk=0)

Cycle 6: Issue I6 (ADDD F6, F8, F2)
  Add2 RS: Op=ADD, Vj=?, Qj=Add1 (F8 not ready), Vk=F2 value, Qk=0
  Register status: F6 -> Add2  (overwrite! WAW with I1 resolved by renaming)
  I3 begins execution (Mul1 has both operands)
  I4 begins execution (Add1 has both operands)

Cycle 7:
  I4 completes (Add1 result on CDB: F8  = F6 - F2)
  Add2 captures (Qj                     = Add1 matches, now Vj=F8 value)

Cycle 8:
  I6 begins execution (Add2 has both operands)

Cycle 9:
  I6 completes (Add2 result on CDB: F6 = F8 + F2)

Cycle 10 (or later, multiplier takes multiple cycles):
  I3 completes (Mul1 result on CDB: F0  = F2 * F4)
  Mul2 captures (Qj                     = Mul1 matches, now Vj=F0 value)

Cycle 11+:
  I5 begins execution (Mul2 has both operands, division is slow)

Cycle 40+ (division takes ~30 cycles):
  I5 completes (Mul2 result on CDB: F10 = F0 / F6)
```

### 5.3 How RAW, WAR, WAW Are Handled

**RAW (I3 depends on I2 for F2):**
   - I3 issues with Qj=Load2 (tag, not value). When I2 completes and broadcasts
   - on CDB, Mul1 snoops and captures the value. True data flow respected.

**WAR (I6 writes F6, which I5 reads):**
   - At issue time of I5, F6's value is already captured into Mul2's Vk.
   - When I6 later writes F6 (updating the register file), Mul2 already has
   - the old value. No conflict.

**WAW (I1 writes F6, I6 also writes F6):**
   - At I6's issue time, register status for F6 is updated from "Load1" to "Add2".
   - I1's result was already consumed. I6's result becomes the final F6 value.
   - Even if I1 completed after I6 (impossible in this example, but in general
   - with ROB), the ROB ensures in-order commit.

---

## 6. Superscalar and Register Renaming

### 6.1 Superscalar Pipeline

```text
2-wide superscalar: Issue up to 2 instructions per cycle

  Fetch:  [I0, I1]  [I2, I3]  [I4, I5]  ...  (fetch 2 per cycle)
  Decode: [I0, I1]  [I2, I3]  [I4, I5]  ...  (decode 2 per cycle)
  Rename: [I0->P5, I1->P6]  [I2->P7, I3->P8]  ... (rename 2 per cycle)
  Issue:  Check dependencies, issue to reservation stations
  Execute: Multiple FUs (2 ALUs, 1 FPU, 1 Load, 1 Store)
  Commit: ROB retires 2 per cycle (in program order)

Challenges:
  - Dependency checking: O(W^2) comparisons for W-wide issue
  - Register renaming: need W rename ports per cycle
  - Multiple FUs: 2+ ALUs, 1+ FPU, 1+ load/store unit
  - Branch prediction accuracy must be very high (deep speculation)
```

### 6.2 Register Renaming with Physical Register File

```text
Architectural registers: R0-R31 (ISA visible, 32 registers)
Physical registers: P0-P127 (128 physical registers, not ISA visible)

Register Alias Table (RAT):
  R0 -> P0 (R0 is always 0, hardwired)
  R1 -> P15
  R2 -> P42
  R3 -> P7
  ...

Free list: {P48, P51, P63, P72, ...}  (available physical registers)

Instruction: ADD R1, R2, R3

Rename process:
  1. Read source mappings: R2 -> P42, R3 -> P7 (from RAT)
  2. Allocate destination: R1 gets P48 (from free list)
  3. Update RAT: R1 -> P48 (old mapping P15 saved in ROB)
  4. Issue: ADD P48, P42, P7 (no architectural registers, all physical)

On commit:
  Free the OLD physical register (P15, the previous mapping of R1)
  P15 returns to the free list

On flush (misprediction):
  Restore RAT to committed state (from ROB snapshots)
  Free all speculative physical registers
```

### 6.3 Reorder Buffer (ROB) Management

ROB is a circular buffer — **Head** points to the oldest instruction (next to commit), **Tail** to the newest (most recently issued).

| # | Instr | Dest | Value | Done? | Exception | Notes |
|---|---|---|---|---|---|---|
| 0 | ADD | P48 | 42 | Yes | No | ← Head (ready to commit) |
| 1 | LW | P51 | 100 | Yes | No | |
| 2 | BEQ | – | – | Yes | No | ← branch: check prediction |
| 3 | MUL | P63 | – | No | – | ← not done yet |
| 4 | ADD | P72 | – | No | – | ← Tail |

Commit rules: (1) check the head entry; (2) if Done and no exception, commit — write to arch state and free the old physical reg; (3) if Done with exception, flush all entries from this one to the tail; (4) if not Done, wait — later entries cannot commit out of order; (5) advance head. Commit width is typically 2–8 per cycle (matching issue width).

---

## 7. Memory Hierarchy -- AMAT and Cache Design

### 7.1 AMAT (Average Memory Access Time)

```verilog
AMAT = Hit_Time + Miss_Rate * Miss_Penalty

For multi-level cache:
AMAT = L1_Hit_Time + L1_Miss_Rate * (L2_Hit_Time + L2_Miss_Rate * (L3_Hit_Time + L3_Miss_Rate * Mem_Latency))

Typical modern values:
  L1 I-Cache: 32-64 KB, 4-way, 1-4 cycles hit, ~5% miss rate
  L1 D-Cache: 32-64 KB, 8-way, 1-4 cycles hit, ~10% miss rate
  L2 Cache:   256KB-1MB, 8-way, 10-20 cycles hit, ~2% miss rate (of L1 misses)
  L3 Cache:   4-32 MB, 16-way, 30-50 cycles hit, ~20% miss rate (of L2 misses)
  Main Memory (DDR4): 100-300 cycles

Numerical AMAT example:
  AMAT = 4 + 0.10 * (15 + 0.02 * (40 + 0.20 * 200))
       = 4 + 0.10 * (15 + 0.02 * (40 + 40))
       = 4 + 0.10 * (15 + 0.02 * 80)
       = 4 + 0.10 * (15 + 1.6)
       = 4 + 0.10 * 16.6
       = 4 + 1.66
       = 5.66 cycles

  Without L2/L3:
  AMAT = 4 + 0.10 * 200 = 24 cycles  (4.2x worse!)
```

### 7.2 Cache Organization

```verilog
Cache parameters:
  Cache size:    S = 32 KB
  Block size:    B = 64 bytes (cache line size)
  Associativity: A = 8-way set-associative
  Number of sets: N = S / (B * A) = 32768 / (64 * 8) = 64 sets

Address breakdown (32-bit address, 64B block, 64 sets):
  [Tag (20 bits) | Set Index (6 bits) | Block Offset (6 bits)]

  Block Offset: log2(64) = 6 bits (byte within cache line)
  Set Index:    log2(64) = 6 bits (which set)
  Tag:          32 - 6 - 6 = 20 bits (unique identifier)
```

### 7.3 Cache Line Size Trade-offs

```verilog
Larger cache lines (e.g., 128B vs 64B):
  Pros:
    - Better spatial locality (more nearby data prefetched)
    - Fewer tag bits per byte (less overhead)
    - Higher bandwidth utilization per DDR burst
  Cons:
    - Higher miss penalty (more bytes to transfer per miss)
    - More wasted bandwidth on partial-line usage
    - Fewer total cache lines for same cache size (worse temporal locality)
    - False sharing in multi-core (larger granularity of coherence)

Typical cache line sizes:
  L1: 64 bytes (most common in x86, ARM)
  L2/L3: 64-128 bytes
  Memory bus: optimized for 64B bursts (matches L1 line size)
```

### 7.4 Write Policies

```verilog
Write-Through:
  Every write updates BOTH cache AND main memory
  Simple, memory always consistent
  High memory traffic (every store goes to memory)
  Often used with write buffer to reduce stalls

Write-Back:
  Writes only update cache, mark line "dirty"
  Dirty lines written to memory only on eviction
  Lower memory traffic (most writes stay in cache)
  More complex (dirty bit, write-back on eviction)
  Used in all modern high-performance caches

Write-Allocate (fetch on write miss):
  On write miss: fetch the block, then write into it
  Usually paired with write-back

No-Write-Allocate (write-around):
  On write miss: write directly to memory, don't fetch block
  Usually paired with write-through
```

### 7.5 Miss Types (3 C's)

```verilog
Compulsory (Cold): First access to a block. Unavoidable (except with prefetching).
Capacity: Cache is too small to hold all needed blocks. Reduce by increasing cache size.
Conflict: Two blocks map to the same set. Reduce by increasing associativity.

+ Coherence misses (4th C): Invalidation by another core in multi-core.

Reducing each:
  Compulsory: Prefetching (hardware or software)
  Capacity:   Larger cache, better replacement policy
  Conflict:   Higher associativity, victim cache
  Coherence:  Better data placement, private vs shared data separation
```

### 7.6 Store Buffer and Store-to-Load Forwarding

The store buffer sits between the execution units and the L1 data cache. When a
store instruction executes, its address and data are written to the store buffer,
not directly to the cache. The store commits (writes to the D-cache) only when it
is the oldest completed instruction at the head of the ROB -- this is in-order
retirement.

**Why stores are buffered:**

1. **Speculation** -- stores from a mispredicted branch must never reach the
   cache. Buffering lets the core discard them on a pipeline flush.
2. **Out-of-order execution** -- stores can execute as soon as the address and
   data are ready, without waiting for older instructions to retire.
3. **Port decoupling** -- the D-cache write port may be busy (e.g., a line fill
   from an earlier miss). The store buffer absorbs the write and drains to the
   cache when the port is free.

**Store-to-load forwarding (SLF):** When a subsequent load executes and its
address matches a pending store in the store buffer, the load can obtain the data
directly from the buffer instead of waiting for the store to commit to cache.

Store buffer (circular FIFO):

| Entry | Addr | Data |
|---|---|---|
| 0 (oldest) | 0x1000 | 42 |
| 1 | 0x1008 | 7 |
| 2 | 0x2000 | 99 |
| 3 (tail) | empty | — |

On `Load(addr=0x1008)`, a CAM search matches entry 1 and returns Data = 7 immediately — no D-cache access needed. This is store-to-load forwarding.

**Forwarding conditions:**

1. **Exact address match** -- the load's byte range must fall entirely within a
   single store's byte range. A 4-byte load at 0x1000 matches a store of 4
   bytes at 0x1000 exactly.
2. **Data ready** -- the store must have its data value available (the store
   address may be known before the data in some microarchitectures).
3. **Partial / overlapping matches** -- if the load spans bytes from multiple
   pending stores, the hardware must merge bytes from each store (and possibly
   the cache). Not all designs support this; some require a full stall until the
   stores commit.

**Store-forwarding hazard (SFX):** When a load depends on a recent store to the
same address but the forwarding check fails -- the store address is not yet
computed, or the overlap is partial -- the load must wait. This is a major
performance limiter. Intel's optimization manual reports SFX stalls account for
5-10% of cycles in typical integer workloads.

**Typical SFX pattern:**
   - STORE [rax], rcx          ; address = rax, data = rcx
   - LOAD  rdx, [rax]          ; same address -> must forward

Best case (address known, data ready): 1 extra cycle (forwarded)
Worst case (address unknown):         4-5 cycle stall on Intel
Partial overlap (load wider than store): ~15 cycle penalty (store must commit first)

**Store buffer sizing:** Modern CPUs typically have 30-60 entries.

| Microarchitecture | Store Buffer Entries |
|-------------------|----------------------|
| Apple M1          | ~40                  |
| Intel Raptor Lake | ~60                  |
| AMD Zen 4         | ~48                  |
| ARM Cortex-X2     | ~32                  |

When the store buffer is full, the processor cannot execute more stores and must
stall, which can become a bottleneck in write-heavy code (e.g., memcpy, memset).

### 7.7 Load-Store Queue (LSQ) -- Detailed Implementation

The LSQ is the structure that enforces memory ordering in an out-of-order core. It
is split into two sub-structures with distinct roles:

**Load Queue (LQ) -- 32-64 entries, CAM on address:**

Each LQ entry stores: `{valid, addr[63:0], data[63:0], rob_idx[6:0], completed, size[2:0], va_known}`.

Loads enter the LQ at dispatch time. When the AGU computes the load's virtual
address, `va_known` is set and the address is broadcast for CAM matching against
all older SQ entries. The LQ is searched by address on every store commit to detect
violations (a younger load observed a value before an older store to the same address
wrote it).

**Store Queue (SQ) -- 24-48 entries, CAM on address:**

Each SQ entry stores: `{valid, addr[63:0], data[63:0], rob_idx[6:0], committed, size[2:0], addr_known}`.

Stores enter the SQ at dispatch. Address computation sets `addr_known`. Data may arrive
later (out-of-order address/data). Stores write to the D-cache only at ROB commit
(`committed` flag), ensuring precise exceptions.

**Store coloring / store sets for disambiguation:**

To avoid comparing every load against every store (O(N) per load), many designs use
store sets. Each load is associated with a "store set" -- a bitmask of stores it has
previously alias-conflicted with. On dispatch, if any store in the set is still in the
SQ without a known address, the load stalls. This reduces the number of comparisons
from O(SQ_size) to O(store_set_size).

An alternative is **store coloring**: a small hash of the store address partitions
stores into "colors." A load only checks stores of matching colors. This is less
accurate than store sets but cheaper to implement.

Store coloring works as follows. Each store is assigned a 2--4 bit "color" derived
from a hash of its virtual address (e.g., `color = addr[5:3]`). A small per-load
"color mask" tracks which colors the load has previously observed conflicts with.
On dispatch, the load's color mask is compared against all older stores in the SQ
whose addresses are not yet known. If any pending store has a color that is in the
load's conflict set, the load stalls. If no pending store matches, the load issues
speculatively. The color space is deliberately small (4--16 colors) so the mask
fits in a few bits per load and the comparison logic is a simple bitwise AND rather
than a full address CAM. The trade-off is over-conservatism: two addresses that hash
to the same color but do not actually alias will cause unnecessary stalls. In practice,
4-bit coloring (16 colors) achieves ~90% of the performance of a full store-set
predictor at roughly half the area cost, making it attractive for mid-range cores
where the store-set predictor's SSID table (typically 256--1024 entries x 8--16 bits)
is too expensive.

**Store-to-load forwarding with age-based priority:**

When a load address is computed, it is compared against all older SQ entries with
`addr_known=1`:

**Forwarding priority:**
   1. Find all SQ entries older than this load (lower SQ index) with matching address
2. Among matches, select the YOUNGEST (closest to the load in program order)
3. If multiple SQ entries overlap the load's byte range, merge bytes:
- Youngest store provides bytes it covers
   - Next-youngest covers remaining bytes, etc.
   - Any uncovered bytes come from D-cache

The "youngest older store" rule ensures the load sees the most recent write to each
byte. If the youngest older store covers the entire load, no D-cache access is needed
(pure forward). If no older store matches, the load reads from the D-cache.

**Age-priority forwarding in detail:**

Age-based priority is critical for correctness when multiple older stores overlap the
load's address range. Consider this scenario:

```verilog
SQ state (program order):
  SQ[0]: SW x5, addr=0x1000, size=8, data=0xAAAABBBBCCCCDDDD  (oldest)
  SQ[1]: SW x6, addr=0x1004, size=4, data=0x11223344          (younger)
  SQ[2]: SW x7, addr=0x1000, size=4, data=0x55667788          (youngest)

Load: LW x8, addr=0x1000, size=8
  Overlap analysis (little-endian, byte addresses 0x1000..0x1007):
    SQ[0] covers bytes 0x1000..0x1007 (full 8-byte store)
    SQ[1] covers bytes 0x1004..0x1007 (upper 4 bytes)
    SQ[2] covers bytes 0x1000..0x1003 (lower 4 bytes)

  Priority: youngest first
    Bytes 0x1000..0x1003: SQ[2] is youngest -> use 0x55667788
    Bytes 0x1004..0x1007: SQ[1] is youngest (SQ[2] doesn't cover these)
                          -> use 0x11223344
    Result: x8 = 0x1122334455667788
```

Hardware implementation: a priority encoder scans the SQ from the youngest entry
(highest index) to the oldest. For each byte lane of the load, the first SQ entry
that covers that byte provides the data. This requires a per-byte-byte multiplexer
tree, which is one reason the SQ data path is wide and power-hungry. In designs that
do not support partial forwarding, the load is simply stalled until all overlapping
stores have committed to the D-cache, trading ILP for hardware simplicity.

**Load bypass of earlier stores (when safe):**

A load may execute before an older store whose address is unknown, speculating that
they do not alias. This is called **load bypass** and is critical for ILP:

**Safe to bypass:**
   - Store address is known AND different from load address -> bypass (no alias)
   - Store address is unknown -> speculate bypass, check later

**Must wait:**
   - Store address is known AND matches load address -> forward (no bypass)
   - Store-set predictor says this load aliases the pending store -> stall

**Memory violation detection and recovery pipeline flush:**

If a load bypassed an older store speculatively, and that store later computes an
address that aliases the load, a **memory ordering violation** occurs:

**Violation detection:**
   1. Store computes address -> SQ entry gets addr_known=1
2. SQ address is broadcast to all younger LQ entries
3. CAM match: LQ entry has same address AND load already completed
4. Violation detected! The load read stale data.

**Recovery:**
   1. Identify the violating load's rob_idx
2. Flush the ROB from (load's rob_idx + 1) to tail
-- the load itself must be re-executed
3. Restore RAT from checkpoint at the load's rob_idx
4. Re-issue the load (this time it will forward from the store)
5. Penalty: typically 10-20 cycles (same as branch misprediction)

Violation frequency: ~0.1-1% of all loads on typical integer workloads. Store-set
predictors keep this rate low by preventing speculative bypass when history indicates
aliasing.

### 7.8 Detailed Reorder Buffer (ROB) Implementation

**Circular buffer mechanics:**

The ROB is implemented as a SRAM array with head and tail pointers that wrap modulo
$N_{ROB}$:

```python
ROB[0]  ROB[1]  ...  ROB[Head]  ROB[Head+1]  ...  ROB[Tail-1]  ROB[Tail]
                                    |                              |
                              oldest uncommitted            next free slot

Head pointer: points to the oldest uncommitted instruction
Tail pointer: points to the next free slot for dispatch

Allocate (dispatch W instructions):
  for i in 0..W-1:
    ROB[(tail + i) % N_ROB] = new_entry[i]
  tail = (tail + W) % N_ROB
  if (tail + 1) % N_ROB == head: ROB_FULL -> stall dispatch

Commit (retire up to C instructions):
  for i in 0..C-1:
    if ROB[(head + i) % N_ROB].completed && !ROB[(head + i) % N_ROB].exception:
      commit_entry(ROB[(head + i) % N_ROB])
    else:
      break
  head = (head + i) % N_ROB

Occupied count: N = (tail - head) % N_ROB
  (When tail >= head: N = tail - head. When tail < head: N = tail + N_ROB - head)
```

The modulo arithmetic is implemented with an adder and a mux: since $N_{ROB}$ is
always a power of 2, the wrap is a simple bit truncation (keep only $\log_2 N_{ROB}$
bits). Detecting "full" requires an extra comparison; detecting "empty" is
`head == tail`.

**Physical register recycling on commit:**

When instruction $I$ commits and its destination architectural register $rd$ maps to
physical register $P_{new}$, the *previous* physical register $P_{old}$ for that same
$rd$ is returned to the free list:

```verilog
At commit:
  P_old = ROB[head].phys_dst_old    // saved during rename
  arch_RAT[ROB[head].arch_dst] = ROB[head].phys_dst  // not strictly needed
                                                      // (commit RAT update)
  FreeList.push(P_old)
  head = (head + 1) % N_ROB
```

Why not free $P_{old}$ immediately when $P_{new}$ is written by the execution unit?
Because other in-flight instructions may still read $P_{old}$ as a source. Only when
$I$ commits (guaranteeing no earlier instruction will need $P_{old}$) is it safe to
recycle.

**Precise exception handling through ROB:**

The ROB enables precise exceptions by serializing all architectural state updates
through the in-order commit path:

```verilog
1. Exception detected at any execution stage (e.g., page fault at MEM)
2. Exception code written to ROB entry's exception field
3. ROB entry marked completed=1
4. Commit logic scans from head:
   - All entries before the faulting one commit normally
   - When the faulting entry reaches head:
     a. Flush all entries from (head+1) to tail
     b. Restore RAT to committed state (all prior commits are correct)
     c. Redirect PC to trap vector (mtvec/stvec)
5. Architectural state is now precise:
   - All instructions before the fault have updated arch state
   - No instruction after the fault has modified arch state
   - Memory unchanged (stores commit only at ROB head)
```

The key invariant: no store reaches the D-cache and no architectural register is
permanently updated until the instruction reaches the ROB head. This makes rollback
trivial -- simply discard everything after the faulting instruction.

**ROB walk-back for precise exceptions (step-by-step hardware sequence):**

When a faulting instruction $I_f$ at ROB index $B$ reaches the head, the following
sequence executes in 2--3 cycles:

```verilog
Cycle 1: Detection and flush initiation
  1. ROB commit logic reads head entry (index B)
  2. Exception field != 0 -> trigger exception pipeline
  3. Invalidate all ROB entries from B+1 to tail:
     for i = B+1 to tail-1:
       ROB[i].valid = 0
     tail = B + 1 (mod N_ROB)
  4. Invalidate all IQ entries (branch mask not needed; entire IQ flushed)
  5. Invalidate all LSQ entries after B (walk LQ and SQ, match rob_idx > B)

Cycle 2: Architectural state restoration
  1. Restore the frontend RAT from the commit RAT:
     for arch_reg = 0 to 31:
       frontend_RAT[arch_reg] = commit_RAT[arch_reg]
     This is the "snapback" -- the commit RAT reflects all instructions up to
     but NOT including the faulting instruction.
  2. Free all physical registers that were allocated by instructions B+1..tail:
     Walk the invalidated ROB entries and push each phys_dst back to the free list.
     (Some designs skip this by maintaining a speculative free list pointer that
     is simply restored.)
  3. Set exception CSRs:
     mepc = ROB[B].PC          (address of faulting instruction)
     mcause = ROB[B].exception  (exception code, e.g., 13 for load page fault)
     mtval = ROB[B].ldst_addr   (faulting virtual address or instruction encoding)

Cycle 3: Redirect pipeline
  1. PC = mtvec (trap vector base address)
  2. mstatus.MIE = 0 (disable interrupts in handler)
  3. Privilege mode -> M-mode (or S-mode if delegated)
  4. Frontend begins fetching from trap vector
```

The critical property: after the walk-back, the frontend RAT exactly matches the
commit RAT, which reflects only instructions 0 through B-1 (all those that committed
before the fault). The faulting instruction's physical destination register is NOT
freed -- it was allocated at rename but never committed, so it simply remains
allocated. On return from the handler (MRET), the instruction will be re-fetched
and re-executed, allocating a fresh physical register. The stale phys_dst from the
original (faulting) execution is returned to the free list during the walk-back.

### 7.9 Detailed Register Renaming Implementation

**RAM-based rename table vs CAM-based:**

| Design | Structure | Read | Update | Power | Latency |
|--------|-----------|------|--------|-------|---------|
| RAM-based RAT | SRAM array, 32 entries x 7-bit phys_tag | Direct index (1 cycle) | Write port per rename lane | Low | Fast (O(1) lookup) |
| CAM-based RAT | 128-entry CAM, match on arch_reg tag | Associative search | Update matched entry | High (every entry compares) | Slower (O(N) compare) |
| Fusion: RAM for common case, CAM for checkpoint restore | Hybrid | RAM indexed by arch_reg | RAM write + CAM for recovery | Medium | Fast lookup, 1-cycle restore |

RAM-based is universally used for the active (speculative) RAT because the 32
architectural registers make it a small, fast SRAM. CAM-based rename is used only
for checkpointing (see below).

**RAM-based RAT implementation details:**

```verilog
RAT SRAM array (32 entries x 7 bits = 224 bits total):

  Read: arch_reg[4:0] drives the SRAM word-line decoder.
        1-cycle access (often half-cycle with fast SRAM).
        For a 4-wide machine: 8 read ports (2 per instruction x 4 instructions).
        Multi-ported SRAM area scales as O(read_ports x write_ports).
        8-read, 4-write port SRAM for 32 entries is ~2-3 KB in 7 nm.

  Write: On rename, the new phys_tag is written to RAT[arch_dst].
         Bypass: if two instructions in the same rename group write the same
         arch register, the second write wins (the first's phys_dst is still
         recorded in its ROB entry for recycling).

  Intra-group forwarding: Within a single-cycle rename group of W instructions,
    instruction i+1 may read an arch register that instruction i just renamed.
    The rename logic must bypass the new phys_tag from instruction i directly
    to instruction i+1's source lookup, without waiting for the SRAM write.
    Implementation: a W x W bypass network (W=4 -> 16 comparisons).

  Port count analysis for 4-wide rename:
    Read ports:  2 sources * 4 instructions = 8 read ports
    Write ports: 1 destination * 4 instructions = 4 write ports
    Total: 12 ports on a 32-entry SRAM
    Area: ~3 KB in 7 nm (dominated by the crossbar for multi-porting)
```

**CAM-based RAT implementation details (for checkpoint recovery):**

```verilog
CAM-based rename table (128 entries x 7-bit phys_tag):

  Each entry stores: {arch_reg[4:0], phys_tag[6:0], valid}
  CAM match: incoming arch_reg[4:0] compared against all 128 entries in parallel.
  Only one entry per arch_reg should be valid at a time (invariant).

  Why use CAM at all? For checkpoint-based misprediction recovery:
    On branch: save a pointer to the current CAM state (or copy 32 valid entries)
    On misprediction: restore the CAM by re-validating the saved entries
    No need to walk the ROB backwards -- the CAM IS the history.

  Area cost: 128 entries x (5+7+1) bits x 2 (complement for dynamic CAM) = ~4.5 KB
  Power cost: every lookup compares against all 128 entries -> 128 * 13-bit comparisons
  Latency: ~2 FO4 for match-line discharge in dynamic CMOS

  Hybrid approach (most common in modern designs):
    Active RAT: RAM-based (fast, low power)
    Checkpoint storage: compact shadow copies of the 32-entry RAM RAT
    On recovery: copy shadow -> active RAT (1 cycle)
    No CAM needed for the main rename path
```

**Free list management -- implementation considerations:**

The free list must support W allocations per cycle (rename) and up to C deallocations
per cycle (commit). For a 4-wide machine, this means 4 pops and up to 8 pushes per
cycle (commit is typically wider than dispatch).

```text
Free list FIFO implementation (N = 128 physical registers):

  head_ptr[6:0]: index of next register to allocate
  tail_ptr[6:0]: index where next freed register is inserted
  count[7:0]:    number of free registers

  Allocate (rename, 4 per cycle):
    if count < 4: stall dispatch
    phys_dst[0] = freelist[head_ptr]
    phys_dst[1] = freelist[(head_ptr+1) % 128]
    phys_dst[2] = freelist[(head_ptr+2) % 128]
    phys_dst[3] = freelist[(head_ptr+3) % 128]
    head_ptr = (head_ptr + 4) % 128
    count -= 4

  Free (commit, up to 8 per cycle):
    for i = 0 to num_committed-1:
      freelist[tail_ptr] = ROB[head+i].phys_dst_old
      tail_ptr = (tail_ptr + 1) % 128
      count += 1

  Critical race: allocate and free in the same cycle.
    If count is near zero but commit will free registers this cycle:
      Option A: stall (conservative, simpler)
      Option B: forward freed registers directly from commit to rename (bypass)
                Used in Intel P-cores; saves ~2% dispatch stalls

  Storage: 128 entries x 7 bits = 896 bits = 112 bytes (trivial)
  The head/tail/count registers are the critical state that must be checkpointed
  along with the RAT on branch rename for misprediction recovery.
```

**Free list management:**

```verilog
Free list: circular FIFO of physical register tags
  head_ptr: next tag to allocate
  tail_ptr: next slot to return a freed tag
  count: number of free registers

Allocate (rename stage):
  if count < W: stall (not enough free regs for W instructions)
  for i in 0..W-1:
    phys_dst[i] = freelist[head_ptr]
    head_ptr = (head_ptr + 1) % N_FREELIST
    count -= 1

Free (commit stage):
  for i in 0..C-1:
    freelist[tail_ptr] = ROB[head].phys_dst_old
    tail_ptr = (tail_ptr + 1) % N_FREELIST
    count += 1
```

Critical invariant: `count >= 0` at all times. A free-list underflow means the core
dispatched more instructions than it has physical registers to support, which must
be prevented by stalling dispatch when `count < dispatch_width`.

**Checkpoint vs history buffer for recovery:**

| Mechanism | Storage | Recovery Time | Complexity |
|-----------|---------|---------------|------------|
| Checkpoint (copy-on-write shadow RAT) | 8-16 snapshots x 32 x 7 bits = ~3.5 KB | 1 cycle (activate shadow) | Moderate |
| History buffer (ROB stores old mapping) | 0 extra (old mapping in ROB entry) | O(N_ROB) cycles (walk back) | Low area |
| Hybrid: checkpoint for branches, ROB-walk for exceptions | 8-16 checkpoints + ROB walk | 1 cycle for branch, O(N) for exception | Moderate |

Checkpoint approach (used in Intel, AMD, ARM high-performance cores):
```verilog
On branch rename:
  Snapshot the active RAT into checkpoint[k]
  checkpoint[k].valid = 1

On branch misprediction:
  Restore active RAT from checkpoint[k]
  Invalidate all checkpoints younger than k
  Free all physical regs allocated after checkpoint[k]
```

History buffer approach (used in area-constrained designs):
```verilog
On any rename:
  ROB[tail].phys_dst_old = previous mapping (already stored)

On misprediction at ROB[B]:
  Walk from tail back to B:
    RAT[ROB[i].arch_dst] = ROB[i].phys_dst_old  // undo the rename
    FreeList.push(ROB[i].phys_dst)               // free speculative phys reg
  Takes (tail - B) cycles
```

**Architectural vs physical register mapping on commit:**

The processor maintains two RATs:

1. **Frontend (speculative) RAT** -- maps arch regs to the latest physical register,
   including uncommitted (speculative) instructions. Updated during rename. Used for
   source operand lookup.

2. **Commit (architectural) RAT** -- maps arch regs to the physical register of the
   last *committed* instruction. Updated only at commit. Used for exception recovery
   and precise state.

On commit, the commit RAT is updated: `commit_RAT[rd] = phys_dst`. The old physical
register `phys_dst_old` is freed. On a pipeline flush (exception or misprediction),
the frontend RAT is restored from the commit RAT -- all speculative mappings are
discarded.

### 7.10 Exception/Interrupt Pipeline -- Stage-by-Stage Detection

Each pipeline stage can detect different classes of exceptions. The following table
shows exactly where each exception type is detected and how it is handled:

| Pipeline Stage | Exception Type | Example (RISC-V) | Detection Mechanism | Handling |
|---|---|---|---|---|
| **Fetch (IF)** | Instruction page fault | Access unmapped page in I-TLB miss | I-TLB miss + page walk returns invalid PTE | Abort fetch, write exception to special ROB entry at decode |
| **Fetch (IF)** | Instruction access fault | Misaligned fetch, fetch from I/O region | PC alignment check + PTE permission check | Same as above |
| **Decode (ID)** | Illegal instruction | Undefined opcode, reserved encoding | Opcode decoder finds no valid decode | Write exception code to ROB entry at dispatch |
| **Rename** | None (typically) | -- | -- | Rename does not raise exceptions |
| **Execute (EX)** | Divide by zero | DIV by zero (RISC-V: no trap, result defined) | Dividend check on divisor operand | Architecture-dependent: some trap, others return sentinel |
| **Execute (EX)** | Integer overflow | ADD overflow (MIPS: trap; RISC-V: no trap) | ALU overflow flag | Architecture-dependent |
| **Execute (EX)** | ECALL / EBREAK | System call / breakpoint | Opcode recognized at decode, flagged in ROB | Handled at commit (no "real" execution needed) |
| **Execute (EX)** | Breakpoint (hardware) | Trigger match on PC or data address | Hardware debug comparator | Written to ROB at execute |
| **Memory (MEM)** | Load page fault | Load from unmapped/protected page | D-TLB miss + page walk returns invalid PTE | Exception code written to ROB entry |
| **Memory (MEM)** | Store page fault | Store to read-only/unmapped page | D-TLB miss + permission check | Same as load page fault |
| **Memory (MEM)** | Load access fault | Misaligned load, device error | Alignment checker + bus error signal | Same |
| **Memory (MEM)** | Store access fault | Misaligned store | Same | Same |
| **Writeback (WB)** | **None** | -- | -- | WB is a data movement stage; no new exceptions |

**How each stage's exception detection interacts with the ROB:**

The fundamental principle is: *any exception detected at any stage is recorded in
the instruction's ROB entry and deferred to commit time*. The instruction is marked
`completed=1` (the exception IS the "result"), but no architectural state is changed.
This lazy handling is what makes precise exceptions cheap -- no special fast-flush
logic is needed at the detection point.

Fetch-stage exception (instruction page fault):
Detection: I-TLB miss -> page walk -> PTE not present or no execute permission
Action at fetch: the faulting instruction cannot enter the pipeline normally.
Instead, the fetch unit creates a "poison" fetch packet containing the PC
and the exception code (CAUSE_INSN_PAGE_FAULT = 12). This packet travels
through decode and rename like a normal instruction. At dispatch, the ROB
entry is created with exception_code = 12 and completed = 1 immediately.
ROB interaction: when this entry reaches the head, the normal exception
handler fires (flush, restore RAT, redirect to mtvec). No special handling
needed -- the ROB doesn't care that the instruction never actually executed.

Decode-stage exception (illegal instruction):
Detection: opcode decoder cannot match the encoding to any valid instruction.
Action at decode: the instruction is still dispatched to the ROB, but its
exception field is set to CAUSE_ILLEGAL_INSTRUCTION = 2 and completed = 1.
The instruction does NOT proceed to issue or execute.
ROB interaction: identical to fetch-stage. The ROB entry arrives at commit
with exception already set. The handler sees mcause=2 and mtval contains
the faulting instruction encoding.

**Execute-stage exception (e.g., ECALL):**
   - Detection: the execution unit recognizes the opcode (ECALL is a "trap"
   - instruction, not a computational one). Some designs detect ECALL at decode
   - and mark the ROB entry immediately, bypassing execute entirely.
   - Action at execute: the functional unit writes exception_code = CAUSE_ECALL_U/S/M
   - (8, 9, or 10 depending on current privilege mode) to the ROB entry via the CDB.
   - ROB interaction: the entry is marked completed=1 with the exception code.
   - When it reaches the head, the pipeline flushes and redirects to mtvec/stvec.

Memory-stage exception (load/store page fault):
Detection: D-TLB miss triggers a hardware page table walk. The walk returns a
PTE that is not present, or does not permit the required access (e.g., read-
only page for a store). Alternatively, the PTE may be present but the
privilege check fails (user-mode access to supervisor-only page).
Action at memory: the AGU/LSQ pipeline writes exception_code = CAUSE_LOAD_PAGE_FAULT
(13) or CAUSE_STORE_PAGE_FAULT (15) to the ROB entry, along with the faulting
virtual address in the ldst_addr field.
ROB interaction: the store's SQ entry is NOT committed to the D-cache (it never
reaches the head). On exception handling, the SQ entry is simply discarded.
For loads, the LQ entry is discarded and any data loaded from the wrong page
is ignored (it was never written to the PRF because the exception prevented it).

The key insight: because the ROB commits in-order, an exception detected in the
memory stage of instruction $I_k$ cannot affect architectural state until $I_k$
reaches the head. All instructions before $I_k$ commit normally, and by the time
$I_k$ reaches the head, they have already updated the architectural register file
and D-cache. The handler therefore sees a precise state: everything before $I_k$ is
committed, nothing after $I_k$ has touched architectural state.

**Precise vs imprecise traps:**

- **Precise trap** (all modern OoO cores): the trap handler sees architectural state
  as if all instructions before the faulting one completed and none after executed.
  Guaranteed by the ROB's in-order commit.

- **Imprecise trap** (some older FP architectures, e.g., MIPS R2000 FP exceptions):
  the faulting instruction may have committed, or later instructions may have
  modified state. The handler cannot trivially resume. Avoided in modern designs.

**How the ROB enables precise exceptions -- detailed walkthrough:**

```verilog
Scenario: I3 causes a load page fault

ROB state at fault detection:
  [I1: ADD  | Done=1 | Exc=0]  <- head, can commit
  [I2: SUB  | Done=1 | Exc=0]  <- can commit
  [I3: LD   | Done=1 | Exc=13] <- load page fault detected in MEM stage
  [I4: ADD  | Done=1 | Exc=0]  <- executed out-of-order, already done
  [I5: MUL  | Done=0 | Exc=0]  <- still executing

Cycle-by-cycle:
  Cycle N:   Commit I1, I2 (both done, no exception). Head advances to I3.
  Cycle N+1: Head = I3, Exc=13 (page fault).
             -> Do NOT commit I3.
             -> Flush ROB entries I3..I5 (set valid=0, tail = I3 index)
             -> Restore RAT to commit_RAT state (I1, I2 committed; I3+ did not)
             -> Set mepc = I3.PC, mcause = 13, mtval = I3.load_addr
             -> Redirect PC to mtvec

Result: architectural state = state after I1 and I2 committed, nothing else.
        This is a precise exception.
```

### 7.11 Detailed Forwarding Network

**Bypass paths in a 5-stage pipeline:**

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk", "nodeSpacing": 60, "rankSpacing": 60, "htmlLabels": false}}}%%
flowchart TD
    EM["EX/MEM pipeline reg<br/>ALU_result"] --> MA["MUX · Forward_A=01"]
    MW["MEM/WB pipeline reg<br/>result (ALU or MEM data)"] --> MA2["MUX · Forward_A=10"]
    MA --> AA["ALU Input A"]
    MA2 --> AA
    EM --> MB["MUX · Forward_B=01"]
    MW --> MB2["MUX · Forward_B=10"]
    MB --> AB["ALU Input B"]
    MB2 --> AB
    R1["ID/EX.rs1_data"] --> AA
    R2["ID/EX.rs2_data"] --> AB
    classDef reg fill:#dbeafe,stroke:#1d4ed8,color:#000
    classDef mux fill:#fde68a,stroke:#b45309,color:#000
    classDef out fill:#dcfce7,stroke:#15803d,color:#000
    class EM,MW,R1,R2 reg
    class MA,MA2,MB,MB2 mux
    class AA,AB out
```

Bypass paths: **EX→EX** (Forward=01) — result available after EX, forwarded to the next instruction's EX; **MEM→EX** (Forward=10) — result available after MEM, forwarded to an instruction two cycles later; **WB→EX** — not needed in a classic 5-stage pipeline with a write-first register file (WB writes in the first half of the cycle, ID reads in the second, so the value is available same-cycle).

In pipelines deeper than 5 stages a WB→EX (late-stage-to-early-stage) bypass *is* needed — e.g., a 7-stage pipeline with WB in stage 6 and a dependent EX in stage 4 must forward back two stages. In an out-of-order core the Common Data Bus handles this: on write-back, the result tag is broadcast to the issue queue's CAM, waking dependent instructions regardless of pipeline distance — replacing all explicit bypass paths with one broadcast.

**Which forwarding is needed to avoid stalls:**

| Dependency Distance | Source Stage | Consumer Stage | Forward Path | Stall? |
|---|---|---|---|---|
| 1 cycle apart (back-to-back ALU) | EX/MEM | EX | EX->EX (Forward=01) | No stall |
| 2 cycles apart (ALU, NOP, ALU) | MEM/WB | EX | MEM->EX (Forward=10) | No stall |
| Same cycle (WB->ID) | WB | ID | Write-first RF | No stall |
| Load -> ALU (1 cycle apart) | MEM/WB | EX | MEM->EX after 1 stall | **1 stall cycle** |
| 3+ cycles apart (deeper pipeline) | WB (or later) | EX | WB->EX bypass (deeper pipelines) or CDB broadcast (OoO) | No stall |

**Worked pipeline timing diagram showing data hazard resolution:**

```verilog
Instruction sequence:
  I1: ADD  $1, $2, $3     (writes $1)
  I2: SUB  $4, $1, $5     (reads $1 -- RAW hazard, distance 1)
  I3: AND  $6, $1, $7     (reads $1 -- RAW hazard, distance 2)
  I4: OR   $8, $1, $9     (reads $1 -- RAW hazard, distance 3)

Without forwarding:
  Cycle:  1    2    3    4    5    6    7    8    9   10   11
  I1:    IF   ID   EX  MEM  WB
  I2:         IF   ID  stall stall EX  MEM  WB
  I3:              IF  stall stall ID   EX  MEM  WB
  I4:                  stall stall IF   ID   EX  MEM  WB
  Total: 11 cycles, 4 stall cycles

With forwarding:
  Cycle:  1    2    3    4    5    6    7    8
  I1:    IF   ID   EX  MEM  WB
  I2:         IF   ID   EX  MEM  WB       <- EX->EX forward (EX/MEM.ALU_result -> I2's EX)
  I3:              IF   ID   EX  MEM  WB  <- MEM->EX forward (MEM/WB.result -> I3's EX)
  I4:                   IF   ID   EX  MEM WB  <- No forward needed (WB writes, ID reads same cycle)
  Total: 8 cycles, 0 stall cycles

Forwarding detail for each cycle:

  Cycle 3: I1 in EX, I2 in ID
    I1's ALU computes $1 = $2 + $3
    Result available at end of cycle 3 (written to EX/MEM register)

  Cycle 4: I1 in MEM, I2 in EX, I3 in ID
    Forward A for I2: EX/MEM.Rd=$1 matches I2.Rs1=$1 -> ForwardA=01
    I2's ALU input A = EX/MEM.ALU_result (forwarded from I1)
    I2 computes $4 = $1(forwarded) - $5

  Cycle 5: I1 in WB, I2 in MEM, I3 in EX, I4 in ID
    Forward A for I3: MEM/WB.Rd=$1 matches I3.Rs1=$1 -> ForwardA=10
    I3's ALU input A = MEM/WB.ALU_result (forwarded from I1)
    I3 computes $6 = $1(forwarded) & $7
    I4 reads $1 from register file (I1 wrote $1 in WB first half of cycle 5,
    I4 reads in ID second half of cycle 5 -- write-first register file)

Load-use hazard (1 stall, unavoidable):
  I1: LW   $1, 0($2)
  I2: ADD  $4, $1, $5

  Cycle:  1    2    3    4    5    6    7
  I1:    IF   ID   EX  MEM  WB
  I2:         IF   ID  stall EX  MEM  WB        <- 1 stall cycle
  I3:              IF  stall ID   EX  MEM  WB

  Why 1 stall: I1's data available at end of MEM (cycle 4).
  I2 needs data at start of EX (cycle 4 originally).
  After 1 stall, I2's EX is in cycle 5, I1's MEM data is available via MEM->EX forward.

  Forward in cycle 5: MEM/WB.mem_data forwarded to I2's ALU input A.
```

### 7.12 Memory-Level Parallelism (MLP)

MLP measures how many outstanding cache misses a processor can tolerate
simultaneously. A processor with MLP $= N$ can have $N$ cache misses in flight
without stalling the core's independent execution.

**Relationship to AMAT.** With perfect overlap of $N$ independent cache misses,
each of latency $t_{\text{miss}}$:

$$
\text{AMAT}_{\text{effective}} = \frac{\text{Total latency}}{\text{Number of misses}} = \frac{t_{\text{miss}}}{N}
$$

Without overlap the same $N$ misses take $N \cdot t_{\text{miss}}$ cycles. MLP
converts serial latency into parallel throughput.

**Hardware support for MLP:**

1. **Non-blocking caches with MSHRs** -- a Miss Status Holding Register tracks
   each outstanding miss. The number of MSHR entries sets the maximum MLP.
   Typical sizes: 8-16 entries for L1, 32-64 for L2. Each entry records the
   missed address, the cache line state, and which load instructions are waiting.
2. **Out-of-order execution** -- the core continues past a cache miss, issuing
   independent loads that may also miss, naturally generating parallel misses.
3. **Hardware prefetcher** -- the dominant source of MLP in practice. A stride
   detector observes consecutive load addresses and generates prefetch requests
   ahead of the program, converting what would be serial compulsory misses into
   parallel prefetch hits.

**MLP vs ILP.** MLP is independent of instruction-level parallelism. A processor
can have low ILP (sequential dependency chain) but high MLP (strided access
pattern recognized by the prefetcher). The prefetcher detects a constant stride
$\Delta$ between successive load addresses and issues prefetches for addresses
$A + \Delta, A + 2\Delta, \ldots$ before the loads execute.

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk", "nodeSpacing": 60, "rankSpacing": 60, "htmlLabels": false}}}%%
flowchart TB
    subgraph serial["No prefetcher — serial misses"]
        direction LR
        s1["Miss 0x1000<br/>200 cyc"] --> s2["Miss 0x1040<br/>200 cyc"] --> s3["Miss 0x1080<br/>200 cyc"] --> st["Total ≈ 600 cyc"]
    end
    subgraph par["Stride prefetcher — MLP ≈ 3"]
        direction LR
        p0["Miss 0x1000"] --> pt["Total ≈ 200 cyc<br/>≈ 67 cyc effective per miss"]
        p1["Prefetch 0x1040"] --> pt
        p2["Prefetch 0x1080"] --> pt
    end
    classDef bad fill:#fee2e2,stroke:#b91c1c,color:#000
    classDef good fill:#dcfce7,stroke:#15803d,color:#000
    class s1,s2,s3,st bad
    class p0,p1,p2,pt good
```

**MSHR sizing trade-off.** More MSHRs enable higher MLP but increase area and
power (each entry needs address comparators and state tracking). An L1 with 16
MSHRs can track 16 concurrent misses -- enough for most prefetch streams but a
bottleneck for pointer-chasing workloads (graph traversal, sparse matrix) where
addresses are data-dependent and the prefetcher cannot help.

### 7.13 Decoupled Access/Execute Architecture

A decoupled architecture splits the processor into two semi-independent engines
connected by an address/data queue:

1. **Access (memory) engine** -- executes address-generation instructions (loads,
   stores, address arithmetic) and runs ahead of the main instruction stream,
   prefetching data into the cache.
2. **Execute (compute) engine** -- consumes data from the queue and performs
   arithmetic, logic, and control-flow operations.

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk", "nodeSpacing": 60, "rankSpacing": 60, "htmlLabels": false}}}%%
flowchart TD
    IS["Instruction stream"] --> AE["Access engine<br/>(runs ahead, prefetches)"]
    IS --> EE["Execute engine<br/>(consumes prefetched data)"]
    AE --> L1["L1 D-cache"] --> DQ["Data queue"] --> EE
    EE -.->|compute results| AE
    classDef a fill:#fde68a,stroke:#b45309,color:#000
    classDef e fill:#dbeafe,stroke:#1d4ed8,color:#000
    class AE,L1,DQ a
    class IS,EE e
```

The access engine uses address-generation instructions whose inputs are
independent of the compute engine's results, so it can run arbitrarily far ahead.
The execute engine then sees a cache hit on nearly every load because the access
engine has already fetched the data.

**Why it is not mainstream.** General-purpose CPUs (x86, ARM) achieve a similar
effect through out-of-order execution with a large load/store queue and hardware
prefetcher. The explicit split adds complexity and restrictive programming
constraints without a clear advantage over the OoO approach for irregular code.

**Where it is used.** Some DSPs and embedded processors (e.g., ADSP-2100,
TMS320C6000) employ decoupled access/execute pipelines where deterministic
memory timing is more valuable than general-purpose flexibility.

**Interview significance.** The access/execute split illustrates the fundamental
principle that memory latency and compute can be overlapped -- the same principle
behind GPU warp scheduling (one warp computes while another waits on memory) and
FlashAttention's tiled computation (load next tile while computing the current
tile).

---

## 8. Cache Coherence -- MESI with All Transitions

### 8.1 MESI States

| State     | Valid | Dirty | Exclusive | Meaning                                    |
|-----------|-------|-------|-----------|--------------------------------------------|
| Modified  | Yes   | Yes   | Yes       | Only copy, modified. Must write back.      |
| Exclusive | Yes   | No    | Yes       | Only copy, clean. Can modify silently.     |
| Shared    | Yes   | No    | No        | May be in other caches. Read-only.         |
| Invalid   | No    | -     | -         | Not valid. Must fetch on access.           |

### 8.2 All State Transitions (Processor Events and Bus Events)

**Processor Events (local CPU actions):**
   - PrRd:  Processor read
   - PrWr:  Processor write

**Bus Events (snooped from other CPUs):**
   - BusRd:    Another CPU reads this address (bus read)
   - BusRdX:   Another CPU writes this address (bus read exclusive / invalidate)
   - BusUpgr:  Another CPU upgrades from Shared to Modified (bus upgrade)

**Transitions from INVALID (I):**
   - PrRd  -> fetch from memory
   - If no other cache has it: I -> E (Exclusive)
   - If another cache has it:  I -> S (Shared)
   - PrWr  -> fetch from memory with invalidation
   - I -> M (Modified), invalidate all other copies
   - Bus transaction: BusRdX

**Transitions from EXCLUSIVE (E):**
   - PrRd  -> E -> E (hit, no bus transaction)
   - PrWr  -> E -> M (silent upgrade, no bus transaction! This is the E state advantage)
   - BusRd -> E -> S (another CPU reads, must share; supply data)
   - BusRdX-> E -> I (another CPU writes, must invalidate; supply data)

**Transitions from SHARED (S):**
   - PrRd  -> S -> S (hit, no bus transaction)
   - PrWr  -> S -> M (must invalidate other copies)
   - Bus transaction: BusUpgr (invalidation, no data transfer needed)
   - BusRd -> S -> S (another CPU reads, still shared)
   - BusRdX-> S -> I (another CPU writes, must invalidate our copy)
   - BusUpgr-> S -> I (another CPU upgrading, must invalidate)

**Transitions from MODIFIED (M):**
   - PrRd  -> M -> M (hit, no bus transaction)
   - PrWr  -> M -> M (hit, no bus transaction)
   - BusRd -> M -> S (another CPU reads; must write back to memory, then share)
   - This is the expensive transition: writeback + state change
   - BusRdX-> M -> I (another CPU writes; must write back, then invalidate)

### 8.3 Snooping Example with 4 Cores

```verilog
Initial: Address X is not in any cache.

Step 1: Core 0 reads X
  Cache 0: I -> E (exclusive, fetched from memory)
  Bus: BusRd, memory responds, no other cache has it

Step 2: Core 1 reads X
  Core 1: I -> S
  Core 0: E -> S (snoops BusRd, supplies data or memory does)
  Bus: BusRd, Core 0 snoops and transitions

Step 3: Core 2 reads X
  Core 2: I -> S
  Cores 0,1: S -> S (no change needed)
  Bus: BusRd

Step 4: Core 0 writes X
  Core 0: S -> M
  Cores 1,2: S -> I (invalidated by BusUpgr)
  Bus: BusUpgr (no data transfer, just invalidation)

Step 5: Core 3 reads X
  Core 3: I -> S (needs data)
  Core 0: M -> S (must write back dirty data first!)
  Bus: BusRd from Core 3, Core 0 snoops, intervenes:
    Core 0 writes back X to memory
    Core 0: M -> S
    Core 3 gets data: I -> S

Step 6: Core 1 writes X
  Core 1: I -> M (was invalidated in Step 4, must fetch)
  Cores 0,3: S -> I
  Bus: BusRdX (fetch for write + invalidate)
```

### 8.4 Scalability Problem and Directory-Based Coherence

```verilog
Snooping problem: Every bus transaction is broadcast to ALL caches.
  With N cores: bus bandwidth = O(N * traffic_per_core)
  Beyond 8-16 cores: bus becomes the bottleneck

Directory-based protocol:
  A directory (in memory controller or distributed) tracks which caches
  hold each cache line.

  Directory entry for address X:
    State: {Uncached, Shared, Exclusive/Modified}
    Sharers: bit vector [Core0, Core1, Core2, ..., CoreN]

  On a read miss:
    Core sends request to directory
    Directory checks sharers list
    If exclusive in another core: forward request to owner, owner supplies data
    If shared: memory supplies data
    Directory updates sharers list

  No broadcast! Point-to-point messages.
  Scales to 100+ cores (AMD EPYC, ARM CMN-700, Intel Xeon SP)

  Trade-off: Higher latency per request (directory lookup + forwarding)
             vs scalable bandwidth
```

### 8.5 MOESI Protocol Extension

**Adds the Owned (O) state:**

O = line is dirty (modified from memory), but other caches have Shared copies.
Owner is responsible for supplying data on snoop requests.

**MESI problem:**
   - When M transitions to S (on BusRd), it must write back to memory.
   - This consumes memory bandwidth even though only a cache-to-cache transfer is needed.

**MOESI solution:**
   - M + BusRd -> O (Owner), requester gets S
   - Data goes directly from owner to requester (cache-to-cache transfer)
   - NO memory write-back! Saves memory bandwidth.
   - Memory is "stale" -- owner will write back on eviction.

AMD Zen uses MOESI (modified version called MOESI-F with F=Forward state).

---

## 9. Virtual Memory and TLB

### 9.1 Four-Level Page Table Walk (x86-64, 48-bit VA)

```verilog
48-bit Virtual Address:
  [PML4 (9)] [PDPT (9)] [PD (9)] [PT (9)] [Offset (12)]
  bits 47:39  bits 38:30  bits 29:21  bits 20:12  bits 11:0

CR3 register holds physical address of PML4 table

Walk:
  Step 1: PML4_entry = Memory[CR3 + VA[47:39] * 8]
  Step 2: PDPT_entry = Memory[PML4_entry.addr + VA[38:30] * 8]
  Step 3: PD_entry   = Memory[PDPT_entry.addr + VA[29:21] * 8]
  Step 4: PT_entry   = Memory[PD_entry.addr + VA[20:12] * 8]
  Step 5: PA = PT_entry.PFN || VA[11:0]

  Each step is a memory access -> 4 memory accesses per translation!
  Without TLB: every load/store takes 5 memory accesses (4 walk + 1 data)
  With TLB: 1 cycle (TLB hit) + 1 data access = 2 cycles

Page Table Entry (PTE) format (simplified):
  [PFN (physical frame number)] [Flags]
  Flags: Present, Read/Write, User/Supervisor, Accessed, Dirty, PAT, Global
```

### 9.2 TLB Design

```verilog
Typical TLB hierarchy:
  L1 ITLB: 64 entries, fully associative, 1-cycle lookup
  L1 DTLB: 64 entries, fully associative, 1-cycle lookup
  L2 STLB: 512-2048 entries, 8-12 way, 6-8 cycle lookup

TLB entry:
  {VPN (virtual page number), ASID, PFN, permissions, valid}

Fully associative: Compare incoming VPN against ALL entries simultaneously.
  Cost: parallel comparators for all entries.
  Benefit: No conflict misses, best hit rate.

Set-associative: L2 TLB often 8-way for larger capacity.
```

### 9.3 TLB Miss Penalty

```verilog
TLB miss -> page table walk:
  Best case (page walk cache hits): 10-30 cycles
  Typical: 30-60 cycles (page walk entries in L2/L3 cache)
  Worst case (page walk entries in memory): 200-400 cycles
  Page fault (page not in physical memory): 1-10 MILLION cycles (disk I/O)

TLB miss rate depends heavily on working set size:
  Working set < TLB coverage: ~0% miss rate
  Working set > TLB coverage: miss rate increases
  
  TLB coverage = entries * page_size
  L1 DTLB: 64 * 4KB = 256 KB (small!)
  L2 STLB: 2048 * 4KB = 8 MB
  With 2MB huge pages: 64 * 2MB = 128 MB (L1 DTLB alone)
```

### 9.4 Huge Pages

```verilog
4KB pages: 12-bit offset, 9 bits per level -> 4-level walk
2MB pages: 21-bit offset, skip PT level -> 3-level walk
1GB pages: 30-bit offset, skip PT and PD -> 2-level walk

Benefits of huge pages:
  1. Larger TLB coverage (64 entries * 2MB = 128MB vs 256KB)
  2. Fewer page table walk steps (3 vs 4)
  3. Fewer TLB misses for large working sets
  4. Less page table memory (fewer PTE entries)

Drawbacks:
  1. Internal fragmentation (allocating 2MB for a 4KB need)
  2. Memory allocation difficulty (need contiguous 2MB physical pages)
  3. Longer page fault handling (must zero 2MB, not 4KB)
```

### 9.5 VIPT (Virtually Indexed, Physically Tagged) Cache

**The problem:**
   - Cache lookup needs the physical address (tag comparison).
   - TLB provides physical address, but takes 1 cycle.
   - If cache uses physical index too, we need PA before indexing.
   - Total: 1 cycle (TLB) + 1 cycle (cache) = 2 cycles minimum.

**VIPT solution:**
   - INDEX the cache with virtual address bits (available immediately)
   - TAG the cache with physical address bits (from TLB)
   - TLB lookup and cache index happen IN PARALLEL -> 1 cycle total!

**Constraint:**
   - The virtual index bits must equal the physical index bits.
   - For a 32KB, 8-way cache with 64B lines:
   - Sets = 32768 / (8 * 64) = 64 sets
   - Index = 6 bits (log2(64)) = bits [11:6]
   - Offset = 6 bits = bits [5:0]
   - Total bits used for index+offset = 12 bits

For 4KB pages: page offset = 12 bits [11:0]
Index bits [11:6] are WITHIN the page offset -> same in VA and PA!
VIPT works without aliasing! (as long as index+offset <= page_offset_bits)

If cache is too large: index bits extend beyond page offset → VIPT becomes problematic (aliasing) → Solutions: page coloring, or increase associativity

---

## 10. Performance Analysis

### 10.1 Performance Equations

```verilog
CPU Time = Instruction_Count * CPI * Clock_Period

CPI = CPI_ideal + Stall_cycles_per_instruction

Stall_cycles = Memory_stalls + Branch_stalls + Structural_stalls

Memory_stalls = Memory_accesses_per_instr * Miss_rate * Miss_penalty

Branch_stalls = Branch_frequency * Misprediction_rate * Penalty

IPC = 1 / CPI (Instructions Per Cycle)

Throughput = IPC * Frequency
```

### 10.2 Amdahl's Law

- **Speedup** = `1 / ((1 - f) + f/S)`

where:
f = fraction of execution time that is improved
S = speedup of the improved portion

**Example:**
   - If 80% of execution is parallelizable and you have 8 cores:
   - Speedup = 1 / ((1-0.8) + 0.8/8) = 1 / (0.2 + 0.1) = 1 / 0.3 = 3.33x

Maximum speedup with infinite cores:
Speedup_max = 1 / (1 - 0.8) = 5x

Even with infinite parallelism, the serial 20% limits you to 5x!

### 10.3 Numerical Performance Example

```verilog
Processor A: 2 GHz, CPI = 1.5
Processor B: 3 GHz, CPI = 2.5

For a program with 10 billion instructions:
  Time_A = 10e9 * 1.5 / 2e9 = 7.5 seconds
  Time_B = 10e9 * 2.5 / 3e9 = 8.33 seconds

Processor A is faster despite lower frequency!
  IPC_A = 1/1.5 = 0.67
  IPC_B = 1/2.5 = 0.40
  Throughput_A = 0.67 * 2G = 1.33 GIPS
  Throughput_B = 0.40 * 3G = 1.20 GIPS
```

---

## 11. Interview Questions & Answers

### Q1: Describe the 5 pipeline stages in detail. What happens at each stage?

**A:** **IF**: Send PC to I-cache, fetch instruction, predict next PC (BTB+BHT). **ID**: Decode opcode, read register file (rs1, rs2), generate immediate, detect hazards (load-use stall). **EX**: Execute ALU operation, compute branch target, resolve branch direction, compute load/store address. Forwarding MUXes select between register values and forwarded results. **MEM**: Access D-cache for loads/stores. Loads read data, stores write data. Cache misses stall the pipeline. **WB**: Write result (ALU or memory data) back to register file. Write-first design allows same-cycle read in ID.

### Q2: Show a detailed RAW hazard example with forwarding paths.

**A:** Sequence: `ADD $1,$2,$3` followed by `SUB $4,$1,$5`. ADD writes $1 in WB (cycle 5). SUB needs $1 in EX (cycle 4). Without forwarding: 2-cycle stall. With forwarding: ADD's ALU result is available at the EX/MEM register (end of cycle 3). The forwarding MUX routes EX/MEM.ALU_result to SUB's ALU input A. Control: EX/MEM.Rd=$1 matches ID/EX.Rs1=$1 and EX/MEM.RegWrite=1, so ForwardA=01 (EX/MEM forward). Zero stall cycles. For a 2-stage distance (e.g., `ADD` then `NOP` then `AND $6,$1,$7`), the MEM/WB register provides the value through the ForwardA=10 path.

### Q3: Explain the load-use hazard. Why can't forwarding solve it completely?

**A:** After `LW $1, 0($2)`, if the next instruction is `ADD $4, $1, $5`, there's a fundamental timing conflict. LW produces data at the END of MEM (cycle 4). ADD needs data at the BEGINNING of EX (cycle 3). The data doesn't exist yet when ADD needs it. Even with forwarding, we need a 1-cycle stall. After the stall, ADD's EX is in cycle 4, and LW's MEM result is available at MEM/WB for forwarding. The compiler can help by scheduling an independent instruction in the delay slot (load delay slot scheduling), converting the stall into useful work.

### Q4: Derive the CPI impact of branch misprediction. Why do deep pipelines suffer more?

**A:** CPI = 1 + branch_freq * mispredict_rate * penalty. For a 5-stage pipeline (penalty=2, branch in EX): with 20% branches and 5% misprediction: CPI = 1 + 0.2*0.05*2 = 1.02. For a 20-stage pipeline (penalty=15): CPI = 1 + 0.2*0.05*15 = 1.15 (15% slower). Deep pipelines have more stages between fetch and branch resolution, so more instructions are fetched speculatively and wasted on misprediction. This is why Intel Pentium 4 (31 stages, penalty=20) needed very aggressive branch prediction (trace cache, sophisticated predictors) to be competitive, and ultimately lost to shorter-pipeline designs.

### Q5: Explain gshare prediction. How does XOR improve accuracy?

**A:** Gshare indexes the Pattern History Table (PHT) with PC XOR Global History Register (GHR). The GHR is a shift register of the last N branch outcomes. XOR creates a unique index for each (PC, history) pair, capturing correlations between branches. Example: two branches at different PCs but with the same history pattern would collide in a simple PC-indexed table but get different PHT entries with XOR. This captures "if branches A,B were taken, branch C is usually not taken." Gshare achieves 93-95% accuracy on SPEC benchmarks with a 14-16 bit GHR and 16K-64K entry PHT. The main weakness is aliasing (different PC+history pairs mapping to the same entry), which can be reduced with larger tables or anti-aliasing techniques.

### Q6: Walk through Tomasulo's algorithm for a 3-instruction sequence.

**A:** Consider: `MUL F0,F2,F4` / `ADD F2,F0,F6` / `ADD F4,F0,F2`. Cycle 1: MUL issues to Mul1 RS {Op=MUL, Vj=F2, Vk=F4, Dest=F0}. Register status: F0->Mul1. Cycle 2: ADD issues to Add1 RS {Op=ADD, Vj=?, Qj=Mul1 (F0 not ready), Vk=F6}. F2->Add1. Cycle 3: Second ADD issues to Add2 RS {Op=ADD, Vj=?, Qj=Mul1 (F0), Vk=?, Qk=Add1 (F2)}. F4->Add2. Note: WAR on F2 (MUL reads F2, second ADD needs F2 from first ADD) is handled because MUL captured F2's value at issue time. WAW on F4 (original F4 vs Add2's new F4) is handled because Add2's tag replaces F4's register status.

### Q7: What is the purpose of the Reorder Buffer? How does it enable precise exceptions?

**A:** The ROB ensures instructions commit in program order, even though they execute out of order. Each issued instruction allocates an ROB entry at the tail. When execution completes, the result is written to the ROB entry (not to architectural state). Commit happens at the head: if the head entry is complete and has no exception, it commits (updates the register file/memory). If it has an exception, ALL entries from that point to the tail are flushed. This ensures: (1) Precise exceptions: at the exception point, all prior instructions have committed and all later ones haven't; (2) Speculative execution: on branch misprediction, flush ROB entries after the branch, restoring the RAT to the committed state. (3) In-order completion visible to software, despite out-of-order execution.

### Q8: Explain MESI with all state transitions. Why does the E state exist?

**A:** MESI has 4 states per cache line. Key transitions: I->E on read miss (sole copy, clean), I->M on write miss (invalidate others), E->M on write (silent upgrade, no bus traffic -- this is why E exists), E->S on snoop read (share), S->M on write (bus upgrade to invalidate others), M->S on snoop read (write back + share), M->I on snoop write (write back + invalidate). The E state's purpose: when a cache has the only copy, a subsequent write can transition E->M without any bus transaction (no invalidation needed since no other cache has it). Without E (like MSI), every write from S requires a bus invalidation even if no other cache has the line. E saves bus bandwidth for private data patterns, which are very common.

### Q9: Calculate AMAT for a 3-level cache hierarchy. What dominates?

**A:** Given L1: 4-cycle hit, 8% miss rate; L2: 15-cycle hit, 3% local miss rate; L3: 40-cycle hit, 25% local miss rate; Memory: 200 cycles. AMAT = 4 + 0.08*(15 + 0.03*(40 + 0.25*200)) = 4 + 0.08*(15 + 0.03*(40+50)) = 4 + 0.08*(15 + 2.7) = 4 + 0.08*17.7 = 4 + 1.416 = 5.42 cycles. L1 hit time dominates (4 out of 5.42). Reducing L1 hit time from 4 to 3 saves 1 cycle = 18% improvement. Reducing L3 miss rate from 25% to 20% saves 0.08*0.03*0.05*200 = 0.024 cycles = 0.4% improvement. Lesson: L1 hit time is by far the most important parameter.

### Q10: Explain VIPT caches. Why do they enable parallel TLB/cache access?

**A:** VIPT uses virtual address bits for the cache set index and physical address bits for the tag. Since the index comes from the virtual address (available immediately), and the tag comes from the TLB translation (takes 1 cycle), both lookups happen in parallel. When both complete (same cycle), the tag from TLB is compared against the tag stored in the indexed cache set. Key constraint: the index bits must be within the page offset (same in VA and PA). For 4KB pages, bits [11:0] are identical in VA and PA. A 32KB, 8-way cache has 64 sets, using bits [11:6] for index -- all within page offset. If the cache were direct-mapped 32KB (512 sets, 9-bit index using bits [14:6]), bits [14:12] differ between VA and PA, causing aliasing problems.

### Q11: Compare snooping and directory-based coherence. When do you use each?

**A:** Snooping broadcasts every coherence transaction on a shared bus. Every cache snoops and acts if it holds the address. Pros: low latency (one bus cycle), simple. Cons: bus bandwidth scales as O(N), practical limit 8-16 cores. Directory-based: a directory (in memory controller or distributed across nodes) tracks which caches hold each line. Coherence messages are point-to-point. Pros: scales to 100+ cores, no broadcast. Cons: 3-hop latency for interventions (requestor -> directory -> owner -> requestor), directory storage overhead. Modern systems: Intel Xeon uses a mix (snooping within a socket, directory across sockets). AMD EPYC uses directory-based (Infinity Fabric). ARM CMN-700 (used in server chips) is fully directory-based.

### Q12: Explain virtual memory page table walk for x86-64. How many memory accesses?

**A:** x86-64 uses 4-level page tables for 48-bit virtual addresses. CR3 holds the physical address of the PML4 table. Walk: (1) Read PML4 entry at CR3 + VA[47:39]*8; (2) Read PDPT entry at PML4.addr + VA[38:30]*8; (3) Read PD entry at PDPT.addr + VA[29:21]*8; (4) Read PT entry at PD.addr + VA[20:12]*8; (5) Physical address = PT.PFN concatenated with VA[11:0]. That's 4 sequential memory reads for one translation. Without TLB, every load/store becomes 5 memory accesses. With TLB hit rate of 99%, average = 0.99*1 + 0.01*5 = 1.05 accesses. Modern hardware has page walk caches that cache intermediate page table entries, reducing most walks to 1-2 memory accesses.

### Q13: What is register renaming? How does it eliminate WAR and WAW?

**A:** Register renaming maps architectural registers (R0-R31) to a larger physical register file (P0-P127) via a Register Alias Table (RAT). Each instruction's destination gets a fresh physical register from the free list. WAR example: I1 reads R2 (mapped to P12), I2 writes R2 (mapped to P47, new allocation). I2 can write P47 anytime without affecting I1 (which reads P12). WAW example: I1 writes R1 (P48), I2 writes R1 (P63). Both write to different physical registers. The RAT is updated to point R1->P63 after I2. When I1 commits, its old mapping (before I1) is freed. The physical register file must have enough entries to support all in-flight instructions simultaneously (typically 128-256 entries for a 4-wide OoO machine).

### Q14: How does a tournament predictor work? Why is it better than gshare alone?

**A:** A tournament predictor runs two predictors in parallel (typically a local per-branch predictor and a global correlating predictor) plus a meta-predictor (chooser) that selects which one to trust. The chooser is a 2-bit counter per entry. When the local predictor is correct and global is wrong, the chooser shifts toward local. When global is correct and local is wrong, it shifts toward global. This captures the best of both: local predictor excels at branches with self-correlated patterns (loops), global excels at branches correlated with other branches (if-else chains). Tournament achieves 95-97% accuracy vs gshare's 93-95%. The Alpha 21264 was the first commercial implementation. Modern TAGE (TAgged GEometric) predictors extend this idea with multiple tables of different history lengths, achieving 97-99%.

### Q15: Explain Amdahl's Law with a numerical example. What are the implications for multi-core?

**A:** Amdahl's Law: Speedup = 1/((1-f) + f/S). If 90% of a workload is parallelizable (f=0.9) with 16 cores (S=16): Speedup = 1/(0.1 + 0.9/16) = 1/(0.1 + 0.05625) = 6.4x. With 64 cores: 1/(0.1 + 0.9/64) = 8.77x. With infinite cores: 1/0.1 = 10x maximum. The 10% serial portion limits total speedup to 10x regardless of core count. Implications: (1) Reducing the serial fraction matters more than adding cores; (2) Diminishing returns set in quickly (16 cores gives 6.4x, 4x more cores gives only 1.37x more); (3) Heterogeneous multi-core makes sense: use a big core for serial portions and many small cores for parallel portions (big.LITTLE, Intel P/E cores).

### Q16: What is the difference between write-back and write-through caches?

**A:** Write-through: every store writes to both cache and memory immediately. Memory is always up-to-date. Simple but generates high memory traffic (every store becomes a memory write). Typically uses a write buffer to hide latency. Write-back: stores only update the cache, marking the line dirty. Memory is updated only when the dirty line is evicted. Much lower memory traffic (most stores stay in cache; only evictions trigger memory writes). More complex (dirty bit tracking, writeback on eviction, coherence implications). All modern L1/L2/L3 caches use write-back. Write-through is sometimes used for specialized buffers or I/O regions where memory must be immediately consistent.

### Q17: What are the 3 C's of cache misses? How do you reduce each?

**A:** Compulsory (first access ever to a block): reduce with hardware/software prefetching. Capacity (working set exceeds cache size): reduce by increasing cache size, improving data locality in software, or using cache-friendly algorithms. Conflict (multiple addresses map to the same set): reduce by increasing associativity (8-way eliminates most conflict misses), using victim caches (small fully-associative cache for recently evicted lines), or using randomized indexing (skewed-associative caches). A 4th C is Coherence misses in multi-core (invalidations by other cores): reduce by minimizing shared mutable data, using proper synchronization granularity, and padding to avoid false sharing.

### Q18: Explain the TLB and ASID. What happens during a context switch?

**A:** The TLB caches virtual-to-physical translations. Without ASID, a context switch requires flushing the entire TLB (all entries belong to the old process). After the switch, every access TLB-misses until the new process's translations populate the TLB (cold start penalty). With ASID (Address Space ID), each TLB entry is tagged with the process's ASID. On a context switch, the OS sets the new ASID. Old entries remain in the TLB but won't match (wrong ASID). New entries are added naturally. If the old process resumes, its entries may still be in the TLB. This eliminates the flush penalty. Typical ASID: 8-16 bits (256-65536 processes). When ASIDs wrap around, a global TLB flush is needed.

### Q19: How does superscalar differ from VLIW? Give examples of each.

**A:** Superscalar issues multiple instructions per cycle from a sequential instruction stream. Hardware dynamically detects dependencies, performs register renaming, and schedules out-of-order. Examples: Intel Core, AMD Zen (4-6 wide), ARM Cortex-A78 (4-wide). VLIW: the compiler statically schedules multiple operations into a single wide instruction word. Hardware is simple (no dynamic scheduling, no renaming). The compiler must find independent operations to fill all slots (NOP if not found). Examples: TI C6000 DSP (8-wide), Intel Itanium/IA-64 (6-wide, called EPIC). Superscalar advantages: binary compatibility across microarchitecture generations, adapts to runtime behavior. VLIW advantages: simpler hardware (lower power), deterministic timing (good for real-time DSP). Superscalar dominates general-purpose computing; VLIW is used in specialized DSPs.

### Q20: Design considerations for a 4-wide superscalar processor. What are the bottlenecks?

**A:** Key bottlenecks: (1) **Fetch bandwidth**: need I-cache with 4+ instruction fetch per cycle, must handle branches within the fetch group (branch in position 2 means positions 3-4 are potentially wrong-path). (2) **Decode**: 4 decoders in parallel, each handling different instruction formats. x86 is particularly complex (variable-length instructions). (3) **Rename**: 4 rename operations per cycle, each reading 2 source mappings and writing 1 destination mapping to the RAT. Need 8-read, 4-write port RAT. (4) **Issue**: check 4 new instructions against all in-flight instructions for dependencies (O(4*N) comparisons where N is the issue queue depth). (5) **Execution**: need multiple FUs (2-4 ALUs, 1-2 FPU, 2 load/store). (6) **Commit**: 4 entries from ROB head per cycle. Register file needs many ports: practical limit is 6-8 wide issue before the register file becomes the area/power bottleneck.

---

## 12. Fetch Unit Microarchitecture

### 12.1 Fetch Target Queue (FTQ)

The FTQ is a small FIFO structure sitting between the branch predictor and the
I-cache. It decouples prediction throughput from cache latency so the predictor
can run ahead of the cache without stalling.

```verilog
FTQ entry fields:
  { predicted_PC,          -- next fetch address (from BPU)
    fall_through_PC,       -- PC + fetch_width (sequential fallback)
    btb_hit,               -- was there a BTB hit for this prediction?
    bht_direction,         -- taken / not-taken from the 2-bit counter
    cond_branch_mask,      -- which instruction positions contain branches
    pred_target            -- predicted branch target address
  }
```

Each cycle the BPU pushes one FTQ entry; the I-cache pops entries at its own
rate. On a misprediction the FTQ is flushed and refilled from the corrected PC.

### 12.2 Fetch Pipeline Stages

The fetch pipeline typically spans 3--5 stages in a modern out-of-order core.
The key dataflow is:

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk", "nodeSpacing": 60, "rankSpacing": 60, "htmlLabels": false}}}%%
flowchart TD
    A[BPU predicts next PC] --> B[FTQ Entry Allocated]
    B --> C[I-Cache Tag + Data Lookup]
    C -->|Hit| D[Instruction Bytes Returned]
    C -->|Miss| E[Line-Fill Buffer Allocated]
    E --> F[Critical-Word-First Fill from L2]
    F --> D
    D --> G[Pre-Decoder: scan for branches]
    G --> H[Decode Queue -- feeds rename/decode]
    H --> I[Rename Stage]
```

Stage-by-stage breakdown:

| Stage | Latency | Action |
|-------|---------|--------|
| BPU + FTQ | 1 cycle | Generate predicted PC, enqueue in FTQ |
| I-cache lookup | 1--4 cycles | Tag compare + data RAM read in parallel |
| I-cache miss | 10--20 cycles | Line-fill from L2, stall fetch pipe |
| Pre-decode | 1 cycle | Identify branches, compute targets |
| Decode queue | buffer | Decouple fetch from rename bandwidth |

### 12.3 I-Cache Miss Handling

When the I-cache misses the fetch unit must wait for the line to arrive from
the next cache level. The key mechanisms are:

1. **Line-fill buffer** -- holds the incoming cache line while it is being
   written into the I-cache data array. One or two fill buffers allow
   overlapping misses (one being filled while another waits for L2).

2. **Critical-word-first** -- the first 16--32 bytes that contain the
   requested PC are forwarded to the pre-decoder immediately; the rest of the
   line arrives over subsequent bus beats. This reduces the visible stall by
   roughly $\lceil B/w \rceil - 1$ cycles, where $B$ is the cache-line size
   and $w$ is the bus width per beat.

3. **Fetch stall + FTQ drain** -- the FTQ continues to drain entries that hit
   in the I-cache, but no new FTQ entries are pushed until the miss resolves.
   In practice the BPU also stalls to avoid polluting the FTQ with
   predictions for addresses that cannot yet be fetched.

4. **Streaming / prefetch** -- some designs (e.g., Apple M-series) employ a
   next-line prefetcher that initiates an L2 request for the sequential line
   in parallel with the current I-cache access, hiding most of the miss
   penalty for straight-line code.

### 12.4 Fetch Bandwidth Constraints

Fetch bandwidth is measured in instructions per cycle (IPC$_{\text{fetch}}$):

$$
\text{IPC}_{\text{fetch}} = \min\!\left(\frac{\text{I-cache line size}}{\text{avg instruction width}},\;\text{decode width}\right)
$$

For a fixed-length ISA (RISC-V, ARM):
- I-cache line = 64 bytes, instruction = 4 bytes $\Rightarrow$ up to 16
  instructions per line.
- Practical fetch width limited to 4--8 instructions per cycle by the
  pre-decoder and branch-prediction check within the fetch group.

For a variable-length ISA (x86):
- Instructions range from 1--15 bytes.
- Pre-decode must find instruction boundaries, so 4--6 instructions per cycle
  is typical; Intel calls this the "MOP cache" problem and uses a decoded
  instruction cache (DSB / MOP cache) to bypass the pre-decoder.

Alignment penalty: if the fetch PC is not aligned to the start of a cache line,
only the remaining bytes in the line are usable. The fetch unit issues a second
I-cache access in the next cycle for the remainder, costing one cycle.

### 12.5 Pre-Decode

The pre-decoder scans raw instruction bytes for lightweight hints that are
used by later pipeline stages:

- **Branch detection** -- identify conditional and unconditional branch opcodes
  so the BTB and BHT can be updated early.
- **Branch target computation** -- for direct branches the target is encoded
  in the immediate field; the pre-decoder extracts it in parallel with decode.
- **Instruction length** -- for variable-length ISAs the pre-decoder marks byte
  boundaries, enabling the decoder to slice the fetch group into individual
  instructions.
- **Call/return identification** -- mark CALL and RET instructions so the
  Return Address Stack (RAS) can be updated speculatively at fetch time.

Pre-decode results are stored alongside each instruction in the decode queue
and forwarded to the BPU update path on commit, completing the prediction
feedback loop.

---

## 13. RISC-V Instruction Examples

### 13.1 RV64I Base Instruction Formats

RISC-V uses six base encoding formats, all fixed at 32 bits. The opcode always
occupies bits [6:0], which makes early decode straightforward.

| Format | Fields (bit ranges) |
|--------|---------------------|
| R-type | funct7[31:25] rs2[24:20] rs1[19:15] funct3[14:12] rd[11:7] opcode[6:0] |
| I-type | imm[31:20] rs1[19:15] funct3[14:12] rd[11:7] opcode[6:0] |
| S-type | imm[31:25] rs2[24:20] rs1[19:15] funct3[14:12] imm[4:0][11:7] opcode[6:0] |
| B-type | imm[12\|10:5] rs2[24:20] rs1[19:15] funct3[14:12] imm[4:1\|11][11:7] opcode[6:0] |
| U-type | imm[31:12] rd[11:7] opcode[6:0] |
| J-type | imm[20\|10:1\|11\|19:12] rd[11:7] opcode[6:0] |

### 13.2 R-Type Example: ADD x5, x6, x7

```verilog
ADD x5, x6, x7
  funct7 = 0000000
  rs2    = 00111   (x7)
  rs1    = 00110   (x6)
  funct3 = 000
  rd     = 00101   (x5)
  opcode = 0110011 (OP)

Machine code: 0000000_00111_00110_000_00101_0110011
Hex: 0x007302B3
```

### 13.3 I-Type Example: ADDI x5, x6, -1

```verilog
ADDI x5, x6, -1
  imm    = 111111111111  (-1, sign-extended 12-bit = 0xFFF)
  rs1    = 00110   (x6)
  funct3 = 000
  rd     = 00101   (x5)
  opcode = 0010011 (OP-IMM)

Machine code: 111111111111_00110_000_00101_0010011
Hex: 0xFFF30293
```

### 13.4 Load Example: LD x5, 8(x6)

```verilog
LD x5, 8(x6)
  imm    = 000000001000  (8)
  rs1    = 00110   (x6)
  funct3 = 011     (doubleword = 64-bit load)
  rd     = 00101   (x5)
  opcode = 0000011 (LOAD)

Machine code: 000000001000_00110_011_00101_0000011
Hex: 0x00831283
```

### 13.5 Store Example: SD x5, 8(x6)

```verilog
SD x5, 8(x6)
  imm[11:5] = 0000000   (upper immediate bits)
  rs2       = 00101     (x5, data to store)
  rs1       = 00110     (x6, base address)
  funct3    = 011       (doubleword = 64-bit store)
  imm[4:0]  = 01000     (lower immediate bits = 8)
  opcode    = 0100011   (STORE)

Machine code: 0000000_00101_00110_011_01000_0100011
Hex: 0x00531423
```

### 13.6 Branch Example: BEQ x5, x6, offset

The B-type immediate is the most complex encoding in RISC-V. For a branch
offset of +4 (i.e., skip one instruction forward), the immediate is 4:

```verilog
offset = +4 decimal = 0x004 = 0000000000100 binary (13-bit signed)

B-type immediate layout (scattered):
  imm[12]    = bit 31    = 0
  imm[10:5]  = bits 30:25 = 000000
  imm[4:1]   = bits 11:8  = 0010
  imm[11]    = bit 7      = 0

BEQ x5, x6, +4
  imm[12|10:5] = 0_000000
  rs2          = 00101  (x5)
  rs1          = 00110  (x6)
  funct3       = 000    (BEQ)
  imm[4:1|11]  = 0010_0
  opcode       = 1100011 (BRANCH)

Machine code: 0_000000_00101_00110_000_0010_0_1100011
Hex: 0x00531463
```

The scattered immediate was an intentional design choice: it keeps the
rs1/rs2/opcode fields in the same bit positions as other formats, simplifying
decode hardware. The reassembly logic is:

$$
\text{offset} = \text{sign-ext}\!\left(\text{imm}[12] \,\|\, \text{imm}[11] \,\|\, \text{imm}[10{:}5] \,\|\, \text{imm}[4{:}1] \,\|\, 0\right)
$$

### 13.7 MIPS vs. RISC-V Quick Comparison

| Feature | MIPS (MIPS32/64) | RISC-V (RV64I) |
|---------|-------------------|-----------------|
| Register naming | $0--$31 (dollar prefix) | x0--x31 (or abi names: zero, ra, sp, ...) |
| x0 / $0 | Always zero | Always zero |
| Instruction count (base ISA) | ~150 (MIPS32r5) | ~40 (RV64I) |
| Encoding length | Fixed 32-bit | Fixed 32-bit (base); RVC extension adds 16-bit |
| Immediate encoding | Contiguous bit field | Scattered in B/J-types (for decode regularity) |
| Branch delay slot | Yes (1 slot, mandatory) | No (speculative execution preferred) |
| Conditional move | MOVZ/MOVN (separate) | Not in base ISA; added via Zicond or Zicsr |
| Multiply / divide | Optional DSP or MSA | M extension (standardized) |
| Privilege levels | Kernel / User (2) | M / S / U (3 levels, matches modern OS needs) |
| Page table format | Fixed 2-level | Sv39 / Sv48 / Sv57 (configurable) |
| CSR access | CP0 registers (coproc 0) | Dedicated CSR instructions (CSRRW, CSRRS, CSRRC) |
| Atomic operations | LL/SC (load-linked / store-conditional) | LR/SC + AMO (load-reserved / store-conditional + atomic memory ops) |
| Endianness | Big-endian (original) / bi-endian | Little-endian (base), bi-endian extension available |
| Commercial license | Proprietary (was MIPS Technologies) | Open (no license fee, governed by RISC-V International) |

Key design philosophy difference: MIPS accumulated instructions over decades
(backward-compatible additions), while RISC-V was designed with a small
frozen base ISA and modular extensions (M, A, F, D, C, V, ...) to keep
hardware minimal.

---

## 14. SMT Resource Partitioning

### 14.1 Simultaneous Multithreading Overview

Simultaneous Multithreading (SMT), commercially known as Intel Hyper-Threading,
allows two (or more) hardware threads to share a single out-of-order core.
Each thread has its own architectural state but competes for the same physical
resources. The goal is to improve utilization: when one thread stalls on a
long-latency event (cache miss, branch misprediction), the other thread can
consume the otherwise-idle execution units.

SMT throughput is typically:

$$
\text{SMT}_{\text{speedup}} = 1 + \alpha \cdot (n - 1)
$$

where $n$ is the number of threads and $\alpha$ is the per-thread resource
overlap factor ($\alpha \approx 0.2$--$0.4$ in practice). A 2-thread SMT core
delivers 1.2--1.4x the throughput of a single-threaded core, not 2x.

### 14.2 Resource Partitioning Table

The table below classifies every major microarchitectural structure by its
sharing policy in a typical 2-thread SMT design:

| Structure | Policy | Rationale |
|-----------|--------|-----------|
| L1 I-Cache | Fully shared | Code often shares libraries; capacity benefits both threads |
| L1 D-Cache | Fully shared | Data working sets overlap less, but L1 is fast enough to interleave |
| L2 Cache | Fully shared | Large enough for two working sets; partitioning would waste capacity |
| ROB | Dynamically partitioned | Each thread gets a guaranteed minimum (e.g., 50/50 split of 224 entries) but can borrow unused slots |
| Issue Queue (IQ) | Dynamically partitioned | Thread ID tagged on each entry; partitions rebalance each cycle |
| Physical Register File (PRF) | Dynamically partitioned | Free-list reserves floor per thread; excess distributed on demand |
| Load/Store Queue (LSQ) | Dynamically partitioned | Each thread's addresses are independent; partition prevents one thread from monopolizing entries |
| Common Data Bus (CDB) | Fully shared, arbitrated | Both threads' results multiplex onto the bus; priority alternates to ensure fairness |
| Front-end (fetch unit) | Time-multiplexed | Fetch alternates between threads each cycle (round-robin) or fetches from the thread with fewer in-flight instructions |
| Register Alias Table (RAT) | Replicated (one per thread) | Each thread has its own mapping of arch regs to physical regs |
| Program Counter (PC) | Replicated | Independent fetch PCs |
| Return Address Stack (RAS) | Replicated | Call/return prediction is per-thread |
| Branch Predictor (BHT / BTB) | Shared or partitioned BHT | BTB is shared (tag includes thread ID); BHT may be statically partitioned to reduce cross-thread interference |
| Next-Page Predictor / ITLB | Shared | ASID tagging distinguishes threads |
| Reorder Buffer commit port | Shared, round-robin | Alternate which thread's head entry commits each cycle |

### 14.3 Partitioning Policies in Detail

**Statically partitioned** (fixed split):
- Each thread is guaranteed a fixed number of entries (e.g., 112 ROB entries
  each in a 224-entry ROB).
- Advantage: predictable per-thread performance; no starvation.
- Disadvantage: if one thread is stalled, its unused entries cannot be
  reallocated to the other thread, leaving resources idle.

**Dynamically shared** (competitive):
- Both threads draw from a common pool up to a per-thread cap.
- Example: ROB has 224 entries, each thread can use up to 180, but combined
  cannot exceed 224. When Thread A has only 44 entries in use, Thread B can
  consume the remaining 180.
- Advantage: higher aggregate throughput when one thread is bottleneck-bound.
- Disadvantage: one aggressive thread can starve the other; requires fairness
  counters.

**Flush-and-reallocate policy**:
- On a long-latency event (L2 miss, TLB miss) detected for Thread A, the core
  reduces Thread A's resource cap to its minimum and grants the excess to
  Thread B. When Thread A's miss returns, the caps are rebalanced.
- This is analogous to a "resource loan" and is used in Intel's microarchitectures
  since Skylake.

### 14.4 SMT Interaction with the Memory Hierarchy

When both threads generate cache misses simultaneously:

1. **L1 D-cache contention** -- each thread competes for MSHRs (Miss Status
   Holding Registers). A typical core has 10--16 MSHRs; with two threads the
   effective per-thread miss bandwidth is halved.

2. **L2 / L3 bandwidth** -- the prefetcher may service both threads' streams,
   potentially interfering. Some designs implement per-thread prefetcher state.

3. **Memory-level parallelism (MLP)** -- SMT can actually *improve* MLP because
   two threads' misses are naturally independent and can be serviced concurrently
   by the memory controller's open-page policy.

4. **Store forwarding** -- each thread's stores only forward to its own loads
   (checked via thread ID tag on the store queue), so no cross-thread
   forwarding hazards exist.

---

## 15. Exception Pipeline in Out-of-Order Execution

### 15.1 Precise Exception Guarantee

A precise exception means that when an exception is delivered to the handler,
the architectural state (register file, PC, memory) reflects exactly the state
as if all instructions before the faulting instruction had completed and no
instruction after it has modified state.

In an out-of-order core the ROB enforces this by committing results in program
order. Even though instruction execution may complete out of order, no
architectural state is updated until the instruction reaches the ROB head and
is committed.

$$
\text{Architectural State}_{\text{at exception}} = \text{State after committing ROB}[0..i-1] \;\;\text{where } i \text{ is the faulting instruction}
$$

### 15.2 Exception Flow in the OoO Pipeline

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk", "nodeSpacing": 60, "rankSpacing": 60, "htmlLabels": false}}}%%
flowchart TD
    A[Instruction executes on FU] --> B{Exception detected?}
    B -->|No| C[Write result to ROB entry, set<br/>Done=1]
    B -->|Yes| D[Write exception code to ROB entry,<br/>set Exception=1, Done=1]
    C --> E[ROB commit: check head entry]
    D --> E
    E --> F{Exception bit set?}
    F -->|No| G[Commit: update arch register file /<br/>D-cache]
    F -->|Yes| H[Flush all ROB entries from faulting<br/>entry to tail]
    H --> I[Save faulting PC to mepc / sepc]
    I --> J[Set mcause / scause to exception<br/>code]
    J --> K[Set mtval / stval to faulting<br/>address or instruction]
    K --> L[PC = mtvec / stvec -- jump to trap<br/>vector]
    L --> M[Trap handler executes in M-mode or<br/>S-mode]
```

Step-by-step walk of the flow:

1. **Execute stage** -- the functional unit detects the exception (e.g., page
   fault on a load address). Instead of writing a normal result, it writes an
   exception code (e.g., `CAUSE_LOAD_PAGE_FAULT = 0xD`) into the ROB entry's
   exception field and sets the exception bit.

2. **ROB commit** -- the commit logic examines the head entry every cycle. If
   the head entry has `Done=1` and `Exception=1`, the core raises the
   exception. No instruction after the faulting one has committed because the
   ROB is in-order, so architectural state is already precise.

3. **Pipeline flush** -- all ROB entries from the faulting instruction to the
   tail are discarded. The RAT is restored to the committed state (using ROB
   snapshots or a checkpoint RAT). The issue queue, load/store queue, and
   reservation stations are cleared.

4. **Trap vector delivery** -- the PC is set to the trap vector base address
   (stored in `mtvec` for machine mode or `stvec` for supervisor mode). The
   hardware sets:
   - `mepc` / `sepc` = PC of the faulting instruction
   - `mcause` / `scause` = exception code identifying the cause
   - `mtval` / `stval` = auxiliary information (faulting virtual address for
     page faults, or the faulting instruction encoding for illegal instruction)

5. **Return from handler** -- the software handler executes `MRET` or `SRET`,
   which restores PC from `mepc`/`sepc` and resumes execution.

### 15.3 Synchronous Exceptions

Synchronous exceptions are caused directly by an instruction and are detected
during the execute or memory stage:

| Exception | Cause | Detection Point |
|-----------|-------|-----------------|
| ECALL | Environment call (system call) | Execute (recognized by opcode) |
| EBREAK | Software breakpoint | Execute (recognized by opcode) |
| Load page fault | Load accesses unmapped or protected page | Memory stage (after TLB / page table walk) |
| Store page fault | Store accesses unmapped or protected page | Memory stage |
| Load access fault | Misaligned load or device access error | Memory stage |
| Store access fault | Misaligned store or device access error | Memory stage |
| Illegal instruction | Opcode or encoding not supported | Decode stage (can be detected early, written to ROB) |
| Instruction page fault | Fetch from unmapped page | Fetch stage (handled before entering the OoO pipeline) |
| Instruction access fault | Misaligned fetch or fetch from I/O region | Fetch stage |
| Breakpoint | Trigger match (hardware breakpoint) | Execute stage |

Synchronous exceptions are *precise* in an OoO core because they are always
attributed to a specific instruction and the ROB ensures that instruction
appears to execute in program order.

### 15.4 Asynchronous Exceptions (Interrupts)

Asynchronous exceptions (interrupts) originate from external events and are not
tied to a specific instruction. They are **checked at commit boundaries**, not
at execute:

| Interrupt | Source | Timing |
|-----------|--------|--------|
| Timer interrupt |_mtime_ register exceeds _mtimecmp_ | Checked at commit boundary |
| External interrupt | Platform-level interrupt controller (PLIC) | Checked at commit boundary |
| Software interrupt | Inter-processor interrupt (IPI) via CLINT | Checked at commit boundary |
| Performance counter overflow | Hardware performance monitor | Checked at commit boundary |

**Why check at commit, not at execute?** Consider this scenario:

```ascii-graph
I1: ADD  x5, x6, x7     -- committed
I2: LD   x8, 0(x9)      -- executing, L2 miss in flight
I3: ADD  x10, x11, x12   -- executed out of order, result ready
IRQ --> arrives during cycle when I3 finishes
```

If the interrupt were taken immediately (at I3's completion), the handler
would see a state where I1 has committed but I2 and I3 have not. This is
correct. However, if I3 were *before* I2 in program order, the handler
must see a state where I2 has NOT committed (I2 is older). The ROB ensures
this by only servicing interrupts at the commit boundary, when the
architectural state is unambiguous.

### 15.5 Interrupt vs. Exception Timing Diagram

```verilog
Cycle:         1    2    3    4    5    6    7    8    9    10
I1 (ADD)      IF   ID   EX  MEM  WB(commit)                  <- commits at cycle 5
I2 (LD)            IF   ID   EX  MEM  MEM  MEM  MEM  MEM  WB <- L2 miss, 10-cycle MEM
I3 (ADD)                IF   ID   EX  MEM  WB(done)          <- finishes early (OoO)
I4 (SUB)                     IF   ID   EX  MEM  WB(done)     <- finishes early (OoO)

Scenario A: I2 causes a load page fault (synchronous exception)
  - Detected at cycle 6 (MEM stage of I2)
  - Exception code written to I2's ROB entry
  - I3 and I4 cannot commit until I2 commits (ROB is in-order)
  - At I2's commit (cycle 10): exception bit checked -> flush pipeline
  - Handler sees state as if only I1 executed

Scenario B: Timer interrupt arrives at cycle 7 (asynchronous)
  - Not serviced immediately -- checked only at commit boundary
  - I1 already committed; I2 is next to commit (but not done)
  - Interrupt is pending, waits for I2 to reach commit
  - If I2 finishes at cycle 10: interrupt is taken BEFORE I2 commits
  - Handler sees state where only I1 has committed
  - After handler returns, I2 re-executes (it was flushed)
```

### 15.6 Exception Priority and Delegation

When multiple exceptions are pending simultaneously, hardware resolves them
in a defined priority order (RISC-V privileged spec):

| Priority | Exception / Interrupt |
|----------|----------------------|
| 1 (highest) | Instruction access fault |
| 2 | Instruction page fault (fetch) |
| 3 | Illegal instruction |
| 4 | Breakpoint |
| 5 | Load access fault |
| 6 | Load page fault |
| 7 | Store access fault |
| 8 | Store page fault |
| 9 | ECALL from U-mode |
| 10 | ECALL from S-mode |
| 11 | ECALL from M-mode |
| 12 | -- (reserved) |
| 13 | Timer interrupt |
| 14 | External interrupt (PLIC) |

In RISC-V, exceptions can be *delegated* from M-mode to S-mode using the
`medeleg` and `mideleg` CSRs. For example, setting bit 13 in `mideleg` causes
timer interrupts to be delivered to the S-mode trap vector (`stvec`) instead of
the M-mode vector (`mtvec`). This allows the OS (running in S-mode) to handle
common exceptions without trapping into the firmware/hypervisor (M-mode),
reducing trap latency.

The delegation model is a key difference from MIPS, where all exceptions go to
a single exception level and the OS must determine the cause from the
exception code. RISC-V's separate trap vectors for M-mode and S-mode reduce
the number of privilege transitions on a typical syscall or page fault.

---

## 16. Memory Ordering and Consistency Models

### 16.1 Why Memory Ordering Matters

In a single-threaded in-order pipeline, memory operations appear to execute in
program order. Out-of-order execution and compiler optimizations break this
assumption: a load may execute before an older store to a *different* address,
and stores may be reordered in the store buffer. A **memory consistency model**
defines which reorderings are architecturally visible to software. It is a
contract between hardware and software -- the hardware promises not to violate
the model, and software must insert explicit synchronization when it needs
ordering stronger than the default.

### 16.2 Sequential Consistency (SC)

All memory operations from every thread appear to execute in a single global
order that respects each thread's program order. No reordering of any kind is
visible. Simplest model to reason about, worst performance: every load must wait
for all prior stores to become globally visible, and the store buffer cannot
hide store latency across threads.

No mainstream general-purpose CPU implements SC as its default model.

### 16.3 Total Store Order (TSO / x86)

TSO relaxes exactly one ordering: a load may be reordered before an older store
to a **different** address (store-load relaxation). All other program-order
pairs are preserved:

```ascii-graph
Allowed reorderings under TSO:
  Store X --> Load Y  (X != Y):  YES  (store-load relaxation)
  All other pairs:               NO

Prohibited:
  Load  --> Load     (load-load)
  Store --> Store    (store-store)
  Load  --> Store    (load-store)
```

The store buffer provides this relaxation naturally: when a store is buffered,
subsequent loads to different addresses can bypass it. Loads to the *same*
address are satisfied by store-to-load forwarding (Section 7.6).

**Practical consequence:** on x86, `X=1; print(Y)` can observe Y before X=1 is
visible to other cores. This is not a bug -- it is TSO. Software that needs X=1
to be visible before reading Y must use `MFENCE`, `LOCK XCHG`, or an atomic
instruction with implicit full barrier semantics.

### 16.4 Weak Ordering (ARM, RISC-V Default)

Loads and stores can be reordered freely except where constrained by explicit
fences or acquire/release annotations:

```ascii-graph
Allowed reorderings under weak ordering:
  Load  --> Load:   YES
  Load  --> Store:  YES
  Store --> Store:  YES
  Store --> Load:   YES
```

All reorderings are permitted because the hardware does not enforce any
ordering by default. Software must insert fences or use annotated operations
when ordering is required (e.g., between releasing a lock and subsequent
accesses to protected data).

**RISC-V fencing primitives:**

| Instruction | Semantics |
|-------------|-----------|
| `fence r,rw` | Order all loads before subsequent loads and stores (load-load + load-store fence) |
| `fence rw,rw` | Full fence: order all prior memory operations before all subsequent memory operations |
| `fence.i` | Instruction-cache synchronization: modifications to code memory become visible to fetch |
| `sfence.vma` | TLB synchronization: page-table updates become visible to address translation |

### 16.5 Acquire and Release Semantics

Finer-grained ordering primitives that avoid the cost of a full fence:

- **Acquire** (load-acquire, RISC-V `ld.aq`): prevents all subsequent memory
  operations from being reordered before this load. Used at the entry of a
  critical section (acquiring a lock).

- **Release** (store-release, RISC-V `st.rl`): prevents all prior memory
  operations from being reordered after this store. Used at the exit of a
  critical section (releasing a lock).

```python
Thread 0:                        Thread 1:
  data = 42                        ld.aq flag_val, flag
  st.rl flag, 1                    if flag_val == 1:
                                     print(data)  // guaranteed to see 42

Release on Thread 0 orders "data=42" before "flag=1".
Acquire on Thread 1 orders "load flag" before "load data".
Together they guarantee Thread 1 sees data=42 after observing flag==1.
```

### 16.6 Microarchitectural Implications for OoO

The Load/Store Queue (LSQ) enforces memory-ordering constraints in hardware:

- **On TSO (x86):** the store buffer already provides store-load relaxation.
  The LSQ must only prevent load-load, store-store, and load-store reordering.
  The store buffer naturally gives TSO compliance with minimal hardware cost.

- **On weak ordering (ARM/RISC-V):** the LSQ must track fence instructions as
  ordering barriers. When a `fence` is encountered, the LSQ stalls issue of
  subsequent memory operations until all prior ones have completed. `ld.aq` and
  `st.rl` are cheaper: they carry per-instruction ordering bits that the LSQ
  checks locally without stalling the entire pipeline.

- **Speculation:** both models allow the LSQ to speculate past unresolved
  stores. On TSO, a load may execute before confirming that no older store to
  the same address is pending (address match). On weak ordering, loads may
  speculate past anything unless a fence blocks them. Misprediction recovery
  requires replaying the load from the ROB.

---

## 17. Micro-op Fusion

### 17.1 Macro-Fusion

Macro-fusion merges two adjacent instructions in the decode stage into a single
internal micro-operation (uop). The fused pair occupies one ROB entry, consumes
one issue slot, and traverses the pipeline as a single operation.

**Common fused patterns:**

```ascii-graph
CMP reg1, reg2    }                          TEST reg1, reg2    }
JE  target        } --> fused-cmp-branch      JZ  target        } --> fused-test-branch

ADD reg1, 1       }                          SUB reg1, 1       }
JO  target        } --> fused-add-overflow    JS  target        } --> fused-sub-sign
```

The decoder recognizes these patterns by checking whether the first instruction
is a flag-producing ALU operation and the second is a conditional branch that
tests exactly those flags. If the condition code does not match the produced
flags (e.g., `ADD` then `JNE` which checks ZF set by a different operation),
fusion does not occur.

**Benefit:** the fused uop counts as one entry in the ROB, issue queue, and
reorder logic. This effectively increases ROB capacity and issue width at no
additional hardware cost. For a 4-wide machine that fuses 5% of dynamic
instructions, the effective ROB capacity increases by approximately 5%.

### 17.2 Micro-Fusion

Micro-fusion combines the micro-ops produced by decomposing a single complex
instruction so that they travel through most of the pipeline as one entry.

**Common pattern -- memory operands:**

```verilog
ADD rax, [rbx]      -- complex instruction with memory operand
```

This decomposes into two micro-ops: (1) a load uop that reads from [rbx], and
(2) an ALU uop that adds the loaded value to rax. With micro-fusion, these two
uops share a single ROB entry and issue-queue entry, splitting only at the
execution units (load port for uop 1, ALU port for uop 2).

**Other micro-fused patterns:**

| Instruction | Decomposed uops | Fused through |
|-------------|-----------------|---------------|
| `ADD rax, [rbx]` | load + add | ROB, issue queue |
| `CMP rax, [rbx]` | load + compare | ROB, issue queue |
| `MOV [rbx], rax` | store-addr + store-data | ROB, issue queue |

### 17.3 Quantitative Impact

- **Intel Golden Cove (Alder Lake P-core):** macro-fusion handles ~5--7% of
  dynamic instructions. Combined with micro-fusion, effective ROB capacity
  increases by ~8--10% beyond the nominal 512 entries.

- **Apple M1:** extensive macro-fusion (compare+branch, test+branch, and
  additional patterns) contributes to the M1's very high IPC despite a
  comparatively moderate ROB size (~600 entries reported).

- **Limitation:** fusion only applies to adjacent instructions in program order.
  Compiler scheduling can increase fusion opportunities by placing flag-setting
  instructions immediately before dependent branches, but this is constrained by
  register pressure and other dependencies.

### 17.4 Interview Significance

Micro-op fusion is relevant in interviews because it explains why a processor's
effective issue width and ROB capacity can exceed their nominal values. When
analyzing pipeline throughput, counting fused uops separately would
underestimate the machine's capability. It also illustrates the interplay
between ISA complexity (x86's memory-operand instructions generate more
micro-fusion opportunities) and microarchitectural efficiency.

---

## 18. Modern SMT Implementations

### 18.1 Intel Hyper-Threading (SMT-2)

Intel has shipped 2-thread SMT on most client and server cores since the
Pentium 4 Northwood (2002). Key implementation details from Golden Cove
(Alder Lake P-core, 2021):

| Structure | Sharing Model | Details |
|-----------|--------------|---------|
| ROB | Dynamically shared | 512 total entries, each thread capped at 384 |
| IQ (unified) | Dynamically shared | ~128 entries, thread-ID tagged |
| Physical Register File | Dynamically shared | ~280 INT, ~224 FP entries |
| L1 D-Cache | Fully shared | 48 KB, 12-way |
| L2 Cache | Fully shared | 1.25 MB (Golden Cove), 2 MB (Raptor Cove) |
| Rename width | Time-multiplexed | 6 uops/cycle total, alternating fetch groups |
| Execution units | Fully shared | 6 INT ALUs, 2 FP FMA units, 3 load AGUs |

Typical SMT-2 speedup on Golden Cove: 1.25--1.35x on server workloads
(SPEC CPU2017 multi-copy), diminishing to 1.10x on workloads with heavy L2
cache contention.

### 18.2 AMD Zen 4 SMT-2

AMD Zen 4 (2022) implements 2-thread SMT with a different resource partitioning
philosophy:

| Structure | Details |
|-----------|---------|
| ROB | 320 entries, per-thread floor of 160 |
| IQ | Distributed: 64 INT + 44 FP + 48 AGU/load + 24 store entries |
| PRF | 224 INT + 192 FP physical registers, shared with per-thread floor |
| Load/Store Queue | 128 combined entries, dynamically partitioned |
| L1 D-Cache | 32 KB, 8-way, fully shared |
| L2 Cache | 1 MB per core, private, fully shared by both threads |
| Execution units | 4 INT ALU + 2 AGU + 2 FPU (shared) |

Zen 4 SMT-2 speedup: 1.20--1.30x. AMD publishes per-thread IPC guarantees
for real-time use cases.

### 18.3 IBM POWER10 SMT-4 / SMT-8

IBM POWER10 supports up to 8 hardware threads per core (SMT-8), the widest
commercial SMT implementation:

- 8 threads share a 512-entry ROB and 8 execution slices.
- Each thread gets its own architected register file and program counter.
- SMT-4 mode: 4 threads, each with ~128 ROB entries guaranteed.
- SMT-8 mode: 8 threads, ~64 ROB entries per thread.
- SMT-8 throughput: 1.6--2.2x over single-thread, depending on memory
  intensity.

### 18.4 Apple: No SMT

Apple's M-series cores (Firestorm, Avalanche, Everest) do not implement SMT.
Instead, Apple uses very wide single-thread cores (8-wide decode, 600+ entry
ROB) and relies on chip-area efficiency: two non-SMT cores in the same area as
one SMT-2 core provide more predictable performance and avoid security
vulnerabilities inherent in shared microarchitectural state.

### 18.5 SMT Design Tradeoffs Summary

| Vendor | Core | SMT Width | ROB | Decode Width | SMT Speedup |
|--------|------|-----------|-----|-------------|-------------|
| Intel | Golden Cove | 2 | 512 | 6 | 1.25--1.35x |
| AMD | Zen 4 | 2 | 320 | 4 (6 uop) | 1.20--1.30x |
| IBM | POWER10 | 8 | 512 | 8 | 1.6--2.2x |
| Apple | Avalanche | 1 | ~630 | 8 | N/A |
| ARM | Cortex-X4 | 1 | ~320 | 5 | N/A |

---

## 19. Pipeline Security: Spectre, Meltdown, and Mitigations

### 19.1 The Attack Class

The Spectre (2018) and Meltdown (2018) attacks exploit the microarchitectural
side-effects of speculative execution. The core vulnerability: when the
processor speculatively executes instructions on a wrong path (or with wrong
privilege), the changes to microarchitectural state (cache contents, TLB
entries, branch predictor state) are not rolled back even though the
architectural state is restored. An attacker can use a **covert channel**
(typically the data cache) to exfiltrate secret data that was loaded during
the speculative window.

**Spectre Variant 1 (Bounds Check Bypass):**
```verilog
if (index < array_size)            // Trains branch predictor to "taken"
    temp = array2[array1[index]];  // Speculative access: index may be out-of-bounds
```
The attacker mistrains the branch predictor, causes speculative execution with
a malicious index, and observes which cache line was loaded via a timing
side-channel on array2.

**Spectre Variant 2 (Branch Target Injection):**
The attacker poisons the indirect branch predictor (BTB) so that an indirect
branch in the victim speculatively jumps to a gadget chosen by the attacker.

**Meltdown (Rogue Data Cache Load):**
On affected Intel processors (pre-2018), a user-mode load from a
supervisor-only page faults, but the data reaches the cache before the fault
is processed. The attacker reads the cached data via a covert channel.

### 19.2 Hardware Mitigations

#### Retpoline (Return Trampoline)

A software+hardware mitigation for Spectre Variant 2. Instead of executing an
indirect branch (which may be poisoned in the BTB), the compiler replaces it
with a `call` + `ret` sequence:

```verilog
; Original: jmp *%rax
; Retpoline:
  call retpoline_target
retpoline_target:
  lfence          ; stop speculation
  jmp retpoline_target  ; infinite loop -- never reaches here speculatively
```

The `call` pushes a return address onto the RAS; the `ret` pops it. Because the
RAS is not poisoned by the BTB attack, the speculative path is safe. The
performance cost is ~5--15% on indirect-heavy workloads.

#### IBRS (Indirect Branch Restricted Speculation)

An x86 MSR (Model-Specific Register) that, when set, prevents indirect branch
predictions from using branch predictor state that was populated at a lower
privilege level. This creates a "predictor fence" on privilege transitions.
Intel introduced IBRS in microcode updates (2018). Performance cost: 1--5%
on system-call-heavy workloads.

#### STIBP (Single Thread Indirect Branch Predictor)

Prevents one SMT thread from influencing the other thread's indirect branch
predictions. When enabled, each SMT thread has its own branch predictor state
(partitioned BTB). Performance cost on SMT-2: 1--10% depending on workload.

#### Microarchitectural Buffer Flushing

On return from a higher-privilege exception handler, the processor flushes
internal buffers that could leak data:

| Buffer | Mitigation |
|--------|-----------|
| Load port buffers | Cleared on VM exit / privilege change |
| Store buffer | Drained and verified before returning to less-privileged mode |
| Line fill buffer | Invalidated on context switch |
| L1 D-Cache | May require flush on core sibling entry (L1D flush on VM entry, some Intel) |

#### Speculative Store Bypass Disable (SSBD)

Controls whether speculative loads may bypass older stores to the same address
(before the store address is resolved). Disabling store bypass prevents a
variant where speculative loads read stale data. Performance cost: 0--3%.

### 19.3 RISC-V Security Considerations

RISC-V cores are not immune to speculative side-channels. Mitigations in
RISC-V designs include:

1. **Speculation barriers:** `FENCE` with appropriate predecessors/successors
   can act as a speculation barrier. The `Zicom` extension adds conditional
   select instructions that avoid branches entirely (eliminating BTB attack
   surface).

2. **Data-independent timing (Zkt):** The Zkt extension mandates that
   certain instructions (MUL, DIV, AES round) execute in data-independent time,
   preventing timing side-channels on secret-dependent values.

3. **PMP/PMA access checks before speculation:** In properly designed RISC-V
   cores, the Physical Memory Protection (PMP) and Physical Memory Attributes
   (PMA) checks are performed before speculative data is returned to the cache
   fill buffer, preventing Meltdown-class attacks by design.

4. **Platform-level security (TEE):** The RISC-V Trusted Execution
   Environment (TEE) proposal uses PMP to isolate a secure monitor from the
   rich OS, preventing the OS from accessing secure-memory regions even via
   speculative execution.

### 19.4 Performance Impact Summary

| Mitigation | Mechanism | Typical Cost |
|-----------|-----------|-------------|
| Retpoline | Software (compiler) | 5--15% (indirect-heavy code) |
| IBRS | MSR (microcode) | 1--5% (syscall-heavy) |
| STIBP | MSR (partition BTB) | 1--10% (SMT workloads) |
| SSBD | MSR (block store bypass) | 0--3% |
| L1D flush on VM entry | Microcode | 1--3% per VM entry |
| KPTI (Kernel Page Table Isolation) | Software (OS) | 5--10% (syscall-heavy) |
| Full mitigation stack | All combined | 10--30% worst case |

KPTI is a software mitigation for Meltdown: the kernel runs with a separate
page table that maps only a small trampoline, forcing a TLB flush on every
syscall entry/exit. This is not needed on processors with hardware Meltdown
mitigations (AMD Zen, ARM Cortex-A76+, post-2018 Intel).
