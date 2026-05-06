# Formal Verification — Senior Engineer Deep Dive

## Table of Contents
1. Why Formal Verification
2. Mathematical Foundations (SAT, BDD)
3. Equivalence Checking (LEC)
4. Model Checking / Property Verification
5. Assertion-Based Formal Verification
6. CDC Formal Verification
7. Connectivity and X-Propagation Formal
8. Formal in the ASIC Design Flow
9. Tools Comparison
10. Interview Q&A (25+ Questions)

---

## 1. Why Formal Verification

### The Simulation Coverage Gap

Simulation tests specific input sequences — it's inherently incomplete. A 32-bit adder has
2^64 possible input combinations. At 1 GHz simulation speed, exhaustively testing all inputs
would take:

```
2^64 / 10^9 = 1.84 × 10^10 seconds ≈ 584 years
```

Even a "simple" 8-bit FSM with 256 states and 8-bit input has 256 × 256 = 65,536
state-input combinations per cycle — manageable for simulation, but add a 32-bit datapath
and the space explodes.

**Formal verification** mathematically proves properties hold for ALL possible input sequences,
ALL reachable states, for ALL time — not just the cases you thought to test.

### Where Formal Excels

```
Control logic:     FSMs, arbiters, schedulers    (small state space, complex protocols)
Protocol checkers: AXI, AHB handshakes           (temporal properties are natural fits)
CDC verification:  Synchronizer presence/correctness
FIFOs:            Never overflow/underflow, data integrity
Connectivity:     SoC-level signal routing verification
Deadlock/livelock: Prove absence with fairness constraints
```

### Where Formal Struggles

```
Datapaths:    32-bit multipliers → state space > 2^64 → "state explosion"
Memories:     Large RAMs create enormous state spaces
Floating point: Complex algorithms with wide operands
Deep pipelines: Unrolling depth required > 100 cycles
```

**Key insight:** Formal and simulation are complementary. Use formal for control, simulation
for datapath. The best verification plans combine both.

---

## 2. Mathematical Foundations

### 2.1 Boolean Satisfiability (SAT)

**SAT Problem:** Given a Boolean formula in Conjunctive Normal Form (CNF), find an assignment
of variables that makes the formula TRUE, or prove no such assignment exists (UNSAT).

**CNF (Conjunctive Normal Form):**
```
Formula = (clause1) ∧ (clause2) ∧ ... ∧ (clauseN)
Each clause = (literal1 ∨ literal2 ∨ ... ∨ literalK)
Each literal = variable or its negation

Example:
  F = (a ∨ ¬b ∨ c) ∧ (¬a ∨ b) ∧ (b ∨ ¬c)

  Satisfying assignment: a=1, b=1, c=0 → F = (1∨1∨0) ∧ (0∨1) ∧ (1∨1) = 1
```

**DPLL Algorithm (1962):**
```
function DPLL(formula, assignment):
    // Unit propagation: if a clause has only one unassigned literal, assign it
    formula = unit_propagate(formula, assignment)

    if formula is empty:           return SAT     // all clauses satisfied
    if formula contains empty clause: return UNSAT  // contradiction

    // Choose an unassigned variable
    var = pick_variable(formula)

    // Try var = TRUE
    if DPLL(formula ∧ var, assignment ∪ {var=TRUE}) == SAT:
        return SAT

    // Backtrack, try var = FALSE
    return DPLL(formula ∧ ¬var, assignment ∪ {var=FALSE})
```

**Modern CDCL Solvers (Conflict-Driven Clause Learning):**

CDCL extends DPLL with:
1. **Conflict analysis:** When a contradiction is found, analyze WHY it occurred
2. **Learned clauses:** Add new clauses that prevent the same conflict
3. **Non-chronological backtracking:** Jump back to the decision level that caused the conflict
4. **Variable activity heuristics (VSIDS):** Prioritize variables involved in recent conflicts

```
Performance:
  DPLL:  Exponential backtracking, no learning
  CDCL:  Can solve problems with millions of variables
         (used in industrial verification tools since ~2000)
```

**Why SAT matters for formal verification:**
Any bounded verification problem can be encoded as SAT:
- "Can the design reach a bad state within K clock cycles?"
- Encode the transition relation for K cycles as CNF
- Add the negation of the property
- If SAT → counterexample found (property violated)
- If UNSAT → property holds for K cycles

### 2.2 Binary Decision Diagrams (BDD)

A BDD is a directed acyclic graph (DAG) that represents a Boolean function.

**Example: F = (a ∧ b) ∨ c**

```
Truth table:            BDD (after reduction):
a b c | F
0 0 0 | 0                     a
0 0 1 | 1                    / \
0 1 0 | 0                 0/   \1
0 1 1 | 1                 b     1 (terminal)
1 0 0 | 0                / \
1 0 1 | 1             0/   \1
1 1 0 | 1              c     1
1 1 1 | 1             / \
                    0/   \1
                    0     1

  Dashed edges = variable=0, Solid edges = variable=1
  (Reduced: nodes with identical children are merged)
```

**Reduced Ordered BDD (ROBDD):**
- **Ordered:** Variables appear in same order on all paths (e.g., a before b before c)
- **Reduced:** No two nodes represent the same function; no redundant tests
- **Canonical:** For a given variable ordering, the ROBDD is UNIQUE

**Variable ordering MATTERS enormously:**
```
Function: x1*y1 + x2*y2 + x3*y3 + ... + xn*yn

Good ordering (x1, y1, x2, y2, ...):  BDD size = O(n)     — linear!
Bad ordering  (x1, x2, ..., xn, y1, y2, ..., yn): BDD size = O(2^n) — exponential!
```

This is why BDD-based model checking can fail for certain designs — the BDD "blows up"
depending on the function structure and variable ordering.

**BDD Operations (polynomial time for fixed BDDs):**
```
Operation          | Complexity
AND(f, g)          | O(|f| × |g|)     — product of BDD sizes
OR(f, g)           | O(|f| × |g|)
NOT(f)             | O(|f|)            — swap terminal nodes
Restrict(f, xi=v)  | O(|f|)            — cofactor
Compose(f, xi, g)  | O(|f|^2 × |g|^2)
Satisfying count   | O(|f|)            — traverse, accumulate
Equivalence check  | O(1)              — just compare root pointers!
```

**Symbolic Model Checking with BDDs:**

Represent the set of reachable states as a BDD. Compute the fixed point:

```
Reachable = Initial_States
repeat:
    Reachable_new = Reachable ∪ Image(Reachable, Transition_Relation)
until Reachable_new == Reachable    // fixed point reached

If Bad_States ∩ Reachable == ∅:   Property HOLDS
Otherwise:                          Property FAILS (extract counterexample from BDD)
```

The Image computation uses BDD operations:
```
Image(S, T) = ∃x . (S(x) ∧ T(x, x'))
           // existentially quantify current-state vars
           // leaving next-state vars
```

### 2.3 SAT-Based Bounded Model Checking (BMC)

Instead of BDD-based fixed-point, unroll the transition relation for K steps and encode
as a SAT problem:

```
BMC formula for property P, bound K:

  I(s_0) ∧ T(s_0, s_1) ∧ T(s_1, s_2) ∧ ... ∧ T(s_{K-1}, s_K) ∧ ¬P(s_K)

  I(s_0):           Initial state constraints
  T(s_i, s_{i+1}):  Transition relation (one clock cycle)
  ¬P(s_K):          Negation of property at step K

  If SAT:   The satisfying assignment is a counterexample (K-cycle trace)
  If UNSAT: Property holds for all traces up to K cycles
```

**Increasing K:** Start with K=0, increment. First K where SAT is found gives the
shortest counterexample.

**Completeness:** How do you know when to stop increasing K?
- Completeness threshold: the smallest K beyond which BMC is complete
- For safety properties: K = diameter of the state graph (longest shortest path)
- In practice: use k-induction or IC3/PDR for unbounded proofs

### 2.4 k-Induction

A technique for proving properties hold for ALL time (unbounded):

```
Base case (depth K):
  I(s_0) ∧ T(s_0,s_1) ∧ ... ∧ T(s_{K-1},s_K) ∧ ¬P(s_K)  must be UNSAT

Inductive step:
  P(s_0) ∧ P(s_1) ∧ ... ∧ P(s_{K-1})
  ∧ T(s_0,s_1) ∧ ... ∧ T(s_{K-1},s_K)
  ∧ ¬P(s_K)
  must be UNSAT

If both are UNSAT → P holds for ALL reachable states
```

**Intuition:** If the property has held for K consecutive states, it must hold for the next
state. Combined with the base case (it holds for the first K states from initial), this
proves it holds forever — mathematical induction over time.

**Strengthening:** Sometimes simple 1-induction fails (inductive step is SAT because
unreachable states provide a "fake" counterexample). Solutions:
- Increase K (try 2-induction, 3-induction, ...)
- Add auxiliary lemmas (strengthening invariants) that eliminate unreachable states

### 2.5 IC3/PDR (Property-Directed Reachability)

The state-of-the-art algorithm for unbounded model checking (2011, Aaron Bradley):

```
Key idea: Incrementally build an over-approximation of reachable states
           using clause learning (similar to SAT solver's CDCL)

Maintain a sequence of "frames" F_0, F_1, ..., F_N:
  - F_0 = Initial states
  - F_i ⊇ states reachable in exactly i steps
  - F_i ⊇ F_{i+1}  (monotone: earlier frames are weaker)
  - Each F_i is a conjunction of clauses (over-approximation)

Algorithm:
  1. Check if ¬P ∧ F_N is SAT
     - If SAT: found a potential bad state. Try to prove it's reachable
       by backward-tracing through frames (generalize and block)
     - If a trace reaches F_0: real counterexample found → FAIL
  2. Block: add clauses to frames to exclude unreachable states
  3. Propagate: push clauses forward through frames
  4. If F_i == F_{i+1} for some i: fixed point reached → PROVEN

IC3 advantages:
  - Often proves properties much faster than k-induction
  - Works well for control-heavy designs
  - Integrated into JasperGold, VC Formal, and other tools
```

---

## 3. Equivalence Checking (LEC)

### 3.1 What LEC Proves

LEC mathematically proves that two representations of a design produce identical outputs
for ALL possible input sequences. It does NOT require simulation vectors.

### 3.2 Where LEC is Used in the ASIC Flow

```
                    RTL
                     |
                  Synthesis
                     |
              Gate-level netlist (pre-layout)
                     |
               LEC #1: RTL vs post-synthesis netlist
                     |
              Place & Route
                     |
              Gate-level netlist (post-layout)
                     |
               LEC #2: pre-layout vs post-layout netlist
                     |
              Timing ECO / Functional ECO
                     |
               LEC #3: pre-ECO vs post-ECO netlist
                     |
                  Signoff
```

### 3.3 LEC Flow (Formality / Conformal)

```
Step 1: READ DESIGNS
  // Formality
  read_verilog -r reference.v        // Golden (reference)
  read_verilog -i implementation.v   // Revised (implementation)
  read_db -r library.db

  // Conformal LEC
  read design -golden reference.v
  read design -revised implementation.v
  read library library.lib

Step 2: SETUP
  // Set scan mode, constants, don't-verify points
  set_constant -golden scan_enable 0
  set_constant -revised scan_enable 0
  // Black-box memories or IPs if needed
  set_black_box mem_inst

Step 3: MATCH (Mapping)
  // Tool automatically matches compare points between designs
  match

  // Compare points = {primary outputs, register data inputs,
  //                    black-box inputs, primary inputs}

  // Match types:
  //   By name: FF "reg_a" in golden matches "reg_a" in implementation
  //   By signature: Logic cone analysis when names differ
  //   Manual: User-guided for complex transformations

Step 4: VERIFY
  verify
  // For each compare point:
  //   Extract logic cone (combinational fan-in)
  //   Prove golden_cone ≡ implementation_cone using BDD or SAT

Step 5: DEBUG (if non-equivalent points exist)
  // Analyze failing points
  report_failing_points
  // Visualize logic cones
  analyze_points -failing
```

### 3.4 Key Points and Compare Points

```
KEY POINTS (internal reference nodes):
  - Registers (D-input of flip-flops)
  - Primary inputs / outputs
  - Black-box boundaries

COMPARE POINTS (what gets verified):
  - Each compare point has a "logic cone" in both designs
  - The logic cone = all combinational logic feeding that compare point
  - LEC proves: for all inputs, golden_cone(inputs) == impl_cone(inputs)

  Example:
    Golden:   a → AND → OR → FF.D     (compare point = FF.D)
    Impl:     a → NAND → INV → FF.D   (same function, different gates)
    LEC:      proves AND-OR ≡ NAND-INV  ✓
```

### 3.5 Common LEC Failures and Root Causes

| Failure Type | Root Cause | Solution |
|---|---|---|
| Unmapped points | Naming changes during synthesis | Use name mapping file, set_dont_verify |
| Retimed registers | Synthesis moved registers | Enable retiming-aware LEC, set_dp_option |
| Register merging | Two FFs merged into one multibit FF | Setup multibit handling in LEC |
| Boundary optimization | Constant ports removed, tied-off logic | set_constant, guidance from synthesis log |
| Clock gating | ICG cells added, enable logic changed | Setup clock-gating style, map ICG cells |
| Scan insertion | MUX added to FF input | Set scan_enable = 0 in both designs |
| State encoding change | FSM re-encoded during synthesis | Map state registers, or set_dont_verify FSM |
| Datapath optimization | Carry-save, Booth encoding | Enable DP-aware verification |
| Sequential optimization | Constant registers removed | Read synthesis change log, verify manually |
| Phase inversion | Inverter pushed through register | Enable phase mapping |

### 3.6 Debugging Methodology

```
1. Start with clean LEC (no optimizations)
   - Disable retiming, boundary opt in synthesis
   - If passes: incrementally enable optimizations

2. Incremental LEC
   - Verify at each synthesis step:
     RTL → elaborate → generic → mapped → optimized
   - Narrows down which transformation broke equivalence

3. Use synthesis log
   - Synthesis tools report what transformations were applied
   - e.g., "Information: register 'cnt_reg[3]' merged with 'cnt_reg[7]'"
   - Use this info to set up LEC correctly

4. Hierarchical vs Flat LEC
   Hierarchical: Verify each module separately (faster, but misses cross-boundary issues)
   Flat: Verify entire design at once (slower, but catches everything)
   
   Practical: Use hierarchical for blocks, flat for top-level integration
```

### 3.7 Sequential Equivalence Checking

Standard LEC is combinational — it compares logic cones between registers. But some
transformations change the register structure (retiming, FSM re-encoding).

**Sequential Equivalence Checking (SEC)** handles this:
```
- Does NOT require 1-to-1 register mapping
- Proves: for same input sequences, outputs are identical cycle-by-cycle
- Much more computationally expensive (SAT-based bounded unrolling)
- Used for: retiming verification, C-to-RTL equivalence
- Tools: Synopsys HECTOR, Cadence JasperGold SEQ
```

---

## 4. Model Checking / Property Verification

### 4.1 Temporal Logic

Properties express behavior over time. Two main logics:

**LTL (Linear Temporal Logic) — reasoning over single execution paths:**
```
G p      : Globally p holds (always)
F p      : Finally p holds (eventually)
X p      : neXt cycle p holds
p U q    : p holds Until q becomes true
p R q    : q holds unless and until p Releases it

Examples:
  G(req → F ack)             // Every request is eventually acknowledged
  G(¬(grant_a ∧ grant_b))   // Mutual exclusion always holds
  G(req → X(X ack))          // Ack comes exactly 2 cycles after req
```

**CTL (Computation Tree Logic) — reasoning over branching execution trees:**
```
Path quantifiers: A (for All paths), E (Exists a path)
Combined with temporal operators:

  AG p   : On All paths, Globally p         (p is an invariant)
  EF p   : there Exists a path where Finally p  (p is reachable)
  AF p   : on All paths, Finally p           (p is inevitable)
  AX p   : on All paths, neXt cycle p
  A[p U q]: on All paths, p Until q

Examples:
  AG(¬deadlock)              // No deadlock on any execution path
  AG(req → AF ack)           // Every request always eventually gets ack
  EF(state == DONE)          // It's possible to reach DONE state
  AG(EF(state == IDLE))      // From any state, IDLE is always reachable
```

**SVA to Temporal Logic Mapping:**
```
SVA:   @(posedge clk) req |-> ##[1:3] ack;
LTL:   G(req → (X ack ∨ X X ack ∨ X X X ack))

SVA:   @(posedge clk) req |-> s_eventually ack;
LTL:   G(req → F ack)

SVA:   @(posedge clk) $rose(req) |=> ##1 ack ##1 !ack;
LTL:   G(rose(req) → X(X(ack ∧ X(¬ack))))
```

### 4.2 How Model Checking Works

```
Inputs:
  1. Design (FSM: states, transitions, initial state)
  2. Property (temporal logic formula)

Output:
  - PROVEN: Property holds for all reachable states and all time
  - FAILED: Counterexample trace (sequence of states violating property)
  - UNDETERMINED: Tool ran out of resources (time/memory)

Algorithm choices:
  - BDD-based symbolic model checking (good for small/medium designs)
  - SAT-based BMC (good for finding bugs quickly)
  - IC3/PDR (good for unbounded proofs)
  - k-induction (good when property is "almost" inductive)
  - Combination of all above (modern tools try multiple engines)
```

### 4.3 Convergence and Proof Depth

```
Formal tool reports:
  "Property PROVEN at depth 42"   → Unbounded proof achieved
  "Property PROVEN at bound 100"  → Only proven for 100 cycles (bounded)
  "Property FIRED at cycle 17"    → Counterexample found
  "Property UNDETERMINED"         → Could not prove or disprove

Bounded vs Unbounded:
  Bounded proof:   Safe for K cycles. Bug might exist at cycle K+1.
  Unbounded proof: Safe FOREVER. Mathematical guarantee.

  Always aim for unbounded proofs. If tool can't converge:
    1. Simplify the design (abstract datapaths)
    2. Add helper assertions (strengthening lemmas)
    3. Break into smaller sub-properties
    4. Use assume-guarantee reasoning
```

---

## 5. Assertion-Based Formal Verification

### 5.1 Formal vs Simulation Assertions

```
In simulation:
  - Assertions are MONITORS — they watch for violations in tested scenarios
  - Coverage depends on testbench quality
  - assert (fifo_count <= DEPTH);  // only fires if simulation exercises this

In formal:
  - Assertions are PROOFS — tool exhaustively checks ALL reachable states
  - 100% coverage for the property (if proven)
  - assert property (@(posedge clk) fifo_count <= DEPTH);  // proven for ALL inputs
```

### 5.2 Assumptions vs Assertions

```
ASSUMPTIONS (assume):
  - Constrain the input space (tell the tool what inputs are legal)
  - If you over-constrain: proof is vacuous (true but meaningless)
  - If you under-constrain: false failures (tool finds "impossible" scenarios)

ASSERTIONS (assert):
  - Properties to prove about the design
  - Must hold under ALL legal inputs (those satisfying assumptions)

COVER PROPERTIES (cover):
  - Reachability checks
  - CRITICAL for catching over-constrained environments
  - If a cover can't be hit → your assumptions may be too restrictive
```

**Example: AXI Write Channel Formal Verification**
```systemverilog
// ASSUMPTIONS — constrain AXI master behavior (legal inputs)
assume property (@(posedge clk) disable iff (!rst_n)
    awvalid && !awready |=> awvalid);          // AW must stay valid until ready

assume property (@(posedge clk) disable iff (!rst_n)
    wvalid && !wready |=> wvalid);             // W must stay valid until ready

assume property (@(posedge clk) disable iff (!rst_n)
    awvalid |-> awlen inside {[0:255]});       // Legal burst length

// ASSERTIONS — prove about the DUT (AXI slave)
assert property (@(posedge clk) disable iff (!rst_n)
    bvalid && !bready |=> bvalid);             // B must stay valid until ready

assert property (@(posedge clk) disable iff (!rst_n)
    bvalid |-> bresp inside {OKAY, SLVERR});   // Legal response codes only

assert property (@(posedge clk) disable iff (!rst_n)
    awvalid && awready |-> ##[1:MAX_LAT] bvalid);  // Response within bounded time

// COVER — check environment is not over-constrained
cover property (@(posedge clk)
    awvalid && awready && awlen == 8'hFF);     // Can we see max-length bursts?

cover property (@(posedge clk)
    bvalid && bready && bresp == SLVERR);      // Can we see error responses?
```

### 5.3 Common Formal Verification Targets

**FIFO Formal Properties:**
```systemverilog
// Data integrity: what goes in comes out in order
// (requires auxiliary modeling — shadow FIFO or scoreboard)
logic [WIDTH-1:0] shadow_fifo [0:DEPTH-1];
int shadow_wr_ptr, shadow_rd_ptr, shadow_count;

assert property (@(posedge clk) disable iff (!rst_n)
    rd_en && !empty |-> rd_data == shadow_fifo[shadow_rd_ptr]);

// No overflow
assert property (@(posedge clk) disable iff (!rst_n)
    !(wr_en && full && !rd_en));   // Can't write when full (unless reading)

// No underflow
assert property (@(posedge clk) disable iff (!rst_n)
    !(rd_en && empty));

// Count consistency
assert property (@(posedge clk) disable iff (!rst_n)
    count == shadow_count);

// Eventually not full (liveness — needs fairness constraint)
assert property (@(posedge clk) disable iff (!rst_n)
    full |-> s_eventually !full);
```

**Arbiter Formal Properties:**
```systemverilog
// Mutual exclusion: at most one grant at a time
assert property (@(posedge clk) disable iff (!rst_n)
    $onehot0(grant));

// No spurious grants: grant only when requested
assert property (@(posedge clk) disable iff (!rst_n)
    |grant |-> |(grant & req));

// No starvation (with fairness): every persistent request eventually granted
assert property (@(posedge clk) disable iff (!rst_n)
    req[0] && !grant[0] |-> s_eventually grant[0]);

// Priority correctness (for fixed-priority arbiter)
assert property (@(posedge clk) disable iff (!rst_n)
    grant[1] |-> !req[0]);   // grant[1] only if higher-priority req[0] is inactive
```

### 5.4 Complexity Management

```
Problem: Formal tools hit state-space explosion on large designs

Solutions:

1. BLACK-BOXING
   - Replace large sub-blocks (memories, datapaths) with abstract models
   - Memory: assume reads return any valid value
   - Datapath: assume output is within legal range
   - Tradeoff: may miss bugs in black-boxed logic

2. CUTPOINTS
   - Break internal signals free from their logic — treat as free variables
   - Reduces cone of influence
   - May cause false failures if the signal has constraints

3. CASE SPLITTING
   - Verify property under specific configurations separately
   - e.g., verify read path and write path independently
   - set_case_analysis mode_pin 0  // analyze read mode only

4. ABSTRACTION
   - Reduce data width: prove with 4-bit data, assume holds for 32-bit
   - Counter abstraction: model counter as "zero / nonzero / max"
   - Symmetry reduction: if design is symmetric, verify one instance

5. ASSUME-GUARANTEE
   - Decompose: verify module A assuming B behaves correctly
   - Then verify B assuming A behaves correctly
   - Circular: needs careful soundness argument
```

---

## 6. CDC Formal Verification

### 6.1 CDC Formal vs CDC Lint

```
CDC Lint (structural):
  - Identifies CDC paths by tracing clock domain boundaries
  - Checks: synchronizer present? Correct topology?
  - Fast, handles million-gate designs
  - Misses: functional correctness of synchronization schemes

CDC Formal (functional):
  - PROVES that the synchronization scheme is correct
  - Example: DMUX scheme — formal proves select signal is stable
    during the entire data transfer window
  - Slower, may need design partitioning
  - Catches bugs that lint cannot
```

### 6.2 What CDC Formal Checks

```
1. Synchronizer verification:
   - 2-FF synchronizer present on every CDC path
   - Synchronizer output used correctly (no combinational logic between FFs)

2. Multi-bit CDC:
   - Gray code correctness: only 1 bit changes per transition
   - DMUX/recirculation MUX: data stable when enable is asserted
   - Handshake protocols: REQ-ACK timing is correct

3. Reconvergence:
   - Multiple CDC signals that reconverge in destination domain
   - Risk: if they resolve metastability differently → inconsistent values
   - Formal proves signals are properly synchronized and consistent

4. Reset domain crossing:
   - Reset synchronizer present
   - Async assert, sync de-assert verified
   - No glitch on synchronized reset

5. FIFO pointer CDC:
   - Gray encoding of FIFO pointers verified
   - No multi-bit transition possible
   - Full/empty generation is safe
```

### 6.3 CDC Formal Tools

```
| Tool                    | Vendor    | Type          |
|-------------------------|-----------|---------------|
| SpyGlass CDC            | Synopsys  | Lint + Formal |
| VC Formal CDC           | Synopsys  | Formal        |
| Meridian CDC            | Cadence   | Lint          |
| JasperGold CDC          | Cadence   | Formal        |
| Questa CDC              | Siemens   | Lint + Formal |
```

---

## 7. Connectivity and X-Propagation Formal

### 7.1 Formal Connectivity Verification

For large SoCs, manually verifying that thousands of signals are correctly connected
between subsystems is error-prone. Formal connectivity checking automates this:

```
Specification (CSV or spreadsheet):
  Source                  | Destination            | Transform
  top.cpu.irq_out[0]     | top.intc.irq_in[3]    | direct
  top.cfg.base_addr[31:0]| top.dma.base_reg[31:0]| direct
  top.pll.clk_out        | top.cpu.clk_in         | direct
  top.gpio.pin[7:0]      | top.iomux.gpio_in[7:0] | mux (sel=iomux_cfg)

Tool generates formal properties automatically:
  assert property (top.intc.irq_in[3] == top.cpu.irq_out[0]);
  // ... hundreds of such properties

Benefit: Catches integration bugs immediately, before simulation
```

### 7.2 X-Propagation Formal

**The X problem:**
```
Simulation has two X semantics:
  X-optimism (Verilog default): X in condition → both branches explored
    if (x_signal) a = 1; else a = 0;
    // Simulation: a could be 0 OR 1 — but just picks one!

  X-pessimism: X in condition → output is X
    // More conservative but causes X-explosion

Neither is correct — real hardware will resolve to 0 or 1 (we just don't know which)
```

**Formal X-prop verification proves:**
```
1. After reset, all FFs have known values (no X)
   - Formal proves: after N reset cycles, every FF ∈ {0, 1}
   - Catches: FFs not in reset list, incomplete reset sequencing

2. X cannot propagate to critical outputs
   - Formal proves: if any input goes X, the output remains deterministic
   - Catches: uninitialized registers affecting control logic

3. X-safety of control signals
   - FSM state register: formal proves no X can reach state bits
   - Memory write enable: formal proves wr_en is never X
```

---

## 8. Formal in the ASIC Design Flow

### 8.1 Where Formal Fits

```
Spec → Architecture → RTL → Synthesis → PnR → Signoff
         |                  |            |        |
    Property          LEC #1      LEC #2   LEC #3
    Verification    (RTL vs     (pre vs   (pre vs
    (model check)   netlist)   post-PnR)  post-ECO)
         |
    CDC Formal
         |
    Connectivity
    Formal
```

### 8.2 Formal Sign-off Criteria

```
All assertions:     PROVEN (unbounded) or BOUNDED to sufficient depth
All covers:         COVERED (reachable) — no vacuous proofs
No assumptions:     That weaken the property unintentionally
Complexity:         All properties converged (no UNDETERMINED)
Review:             Formal plan reviewed, assumptions justified
```

### 8.3 Formal Regression

```
Nightly runs:
  - Re-run all formal properties on latest RTL
  - Report: new failures, convergence changes, cover hits
  - Track: proof depth trends (deeper = design growing more complex)

Property management:
  - Version-controlled property files (.sv, .tcl)
  - Ownership: each property has an author and reviewer
  - Status tracking: proven/bounded/failing/waived
```

---

## 9. Tools Comparison

### 9.1 Equivalence Checking Tools

| Feature | Formality (Synopsys) | Conformal LEC (Cadence) |
|---|---|---|
| Engine | BDD + SAT hybrid | BDD + SAT hybrid |
| Retiming support | Via setup options | Via mapping guidance |
| Multibit handling | Automatic | Automatic |
| Hierarchical LEC | Supported | Supported |
| Datapath verification | DP option (extra license) | Built-in |
| Typical capacity | 50M+ gates | 50M+ gates |
| Integration | Works best with DC/Fusion | Works best with Genus/Innovus |

### 9.2 Property Verification Tools

| Feature | VC Formal (Synopsys) | JasperGold (Cadence) |
|---|---|---|
| Engines | BMC, k-ind, IC3, BDD | BMC, k-ind, IC3, BDD, hybrid |
| Formal Apps | CDC, connectivity, X-prop, DPV, SEQ | CDC, connectivity, X-prop, SEC, FPV |
| Assertion language | SVA, PSL | SVA, PSL |
| Abstraction support | Auto-abstraction | Auto-abstraction + user-guided |
| Debug | Waveform, schematic | Waveform, schematic, source-level |
| Scripting | Tcl | Tcl |
| Strength | Tight integration with VCS | Strongest formal app ecosystem |

### 9.3 CDC Tools

| Feature | SpyGlass CDC (Synopsys) | JasperGold CDC (Cadence) | Questa CDC (Siemens) |
|---|---|---|---|
| Lint | ✓ | ✓ | ✓ |
| Formal | Via VC Formal | Built-in | Built-in |
| Metastability injection | ✓ | ✓ | ✓ |
| Protocol verification | ✓ | ✓ | ✓ |
| Reconvergence analysis | ✓ | ✓ | ✓ |
| Integration | Best with Synopsys flow | Best with Cadence flow | Standalone |

---

## 10. Interview Q&A

**Q1: What is the difference between simulation and formal verification?**

Simulation applies specific input vectors and checks outputs — it's inherently incomplete
because it only tests the cases you thought of. Formal verification mathematically proves
that a property holds for ALL possible inputs and ALL reachable states. Simulation is good
for datapath validation and system-level testing. Formal excels at control logic, protocol
compliance, and proving absence of bugs (deadlock, starvation, CDC violations).

**Q2: Explain SAT-based bounded model checking.**

The design's transition relation is "unrolled" for K clock cycles. The initial state,
K transitions, and the negation of the target property at step K are encoded as a single
Boolean formula in CNF. A SAT solver determines satisfiability: if SAT, the satisfying
assignment is a counterexample trace; if UNSAT, the property holds for K cycles. The bound
K is incrementally increased to search for deeper bugs or achieve convergence.

**Q3: What is the difference between BDD-based and SAT-based model checking?**

BDD-based model checking computes the full set of reachable states as a BDD (fixed-point
computation). It gives unbounded proofs but can fail due to BDD size explosion.
SAT-based BMC unrolls the design for K steps — faster for finding bugs but only provides
bounded proofs. Modern tools combine both: use SAT-based BMC to find shallow bugs quickly,
then switch to BDD or IC3 for unbounded proofs.

**Q4: What is k-induction and why is it useful?**

k-induction extends mathematical induction to hardware verification. The base case checks
that the property holds for the first K steps from reset. The inductive step checks that
if the property holds for K consecutive arbitrary states, it holds for the next state.
If both checks pass (UNSAT), the property is proven for ALL time. It's useful because it
can prove properties without computing all reachable states, but may require strengthening
lemmas if the property is not directly inductive.

**Q5: What is IC3/PDR and why is it considered state-of-the-art?**

IC3 (Property-Directed Reachability) incrementally builds an over-approximation of
reachable states using clause learning, similar to how CDCL SAT solvers learn conflict
clauses. It maintains a sequence of "frames" that progressively tighten the reachable
state set. It often converges much faster than k-induction because it learns exactly
which states need to be excluded, rather than requiring uniform inductive depth.

**Q6: What happens during LEC? Walk through the flow.**

LEC reads two designs (reference and implementation), sets up constants and scan modes,
maps compare points (registers, IOs) between designs using name matching and signature
analysis, then for each compare point extracts the combinational logic cone and
mathematically proves (via BDD or SAT) that the two cones are functionally equivalent.
Failures are reported as non-equivalent points (NEPs) with debug information.

**Q7: Your LEC fails after synthesis retiming. What do you do?**

Retiming moves registers across combinational logic, changing the register structure.
Standard combinational LEC can't handle this because compare points don't match 1-to-1.
Solutions: (1) Enable retiming-aware LEC mode in the tool, which uses sequential equivalence
checking. (2) Use the synthesis tool's retiming information (mapping file) to guide LEC.
(3) If the tool still can't handle it, do incremental LEC — verify pre-retiming first,
then verify the retiming step separately using SEC.

**Q8: How do you handle clock gating in LEC?**

Clock gating adds ICG cells that modify the clock path and add enable logic. For LEC:
set scan_enable = 0 in both designs (so the scan MUX doesn't confuse mapping), and
set the clock-gating style so the LEC tool recognizes ICG cells and maps them correctly.
Some tools have specific commands like `set_clock_gating_style` for this.

**Q9: What is a vacuous proof in formal verification? Why is it dangerous?**

A vacuous proof occurs when the antecedent of a property can never be true, making the
implication trivially true regardless of the consequent. Example: if you assume `req`
is always 0, then `req |-> ack` is always true (req is never asserted, so the property
never fires). This gives a false sense of security. Cover properties are the defense:
if `cover property (req)` can never be hit, the assumptions are too restrictive.

**Q10: How do you manage formal complexity for a large design?**

Techniques: (1) Black-box large sub-blocks (memories, datapaths) and replace with abstract
models. (2) Use cutpoints to break internal signals free from their logic cones. (3) Case
splitting — verify under specific configurations separately. (4) Reduce data widths
(4-bit instead of 32-bit for structural properties). (5) Assume-guarantee decomposition
to verify modules independently.

**Q11: What is the difference between set_clock_groups -asynchronous and set_false_path in CDC formal?**

`set_clock_groups -asynchronous` tells the tool that clocks are unrelated — it drops all
timing paths between the groups. `set_false_path` drops specific paths. For CDC formal,
you typically DON'T want either — CDC formal specifically needs to analyze CDC paths.
Instead, you define clock domains and let the CDC tool identify and verify all crossings.

**Q12: How does CDC formal verify a DMUX (data MUX) synchronization scheme?**

In a DMUX scheme, multi-bit data crosses domains unsynchronized, but a control signal
(select/enable) is synchronized. CDC formal proves: (1) The data is stable for the entire
time the synchronized control is active. (2) The control signal is properly synchronized
with a 2-FF synchronizer. (3) The data is only sampled in the destination domain when
the control signal indicates validity.

**Q13: What is the difference between CTL and LTL?**

LTL reasons about single linear execution paths (one possible future). CTL reasons about
branching computation trees (all possible futures). LTL can express fairness naturally
(`GF grant` — infinitely often granted). CTL can express existential properties
(`EF done` — there exists a path reaching done). Neither subsumes the other — some
properties are expressible in one but not the other. CTL* is the superset of both.

**Q14: How do you know if a formal proof is trustworthy?**

Check: (1) All cover properties are reachable — proves assumptions aren't too restrictive.
(2) Proofs are unbounded, not just bounded to N cycles. (3) Assumptions are reviewed and
justified — no unnecessary constraints. (4) The formal testbench has been reviewed by
someone other than the author. (5) No UNDETERMINED properties (tool converged on everything).

**Q15: When would you use sequential equivalence checking instead of standard LEC?**

When the register structure has changed between the two designs. Standard LEC requires
1-to-1 register mapping. SEC handles: retiming, FSM re-encoding, pipeline balancing,
C/C++ to RTL equivalence (different abstraction levels). SEC is much more computationally
expensive — use it only when standard LEC can't handle the transformations.

**Q16: What is formal property synthesis?**

Automatically generating RTL monitors from formal properties (SVA assertions) that can
be synthesized into silicon for runtime verification. Used in safety-critical designs
(automotive, aerospace) where post-silicon monitoring is required.

**Q17: How does formal connectivity verification work?**

The user provides a specification (often a spreadsheet) listing source-destination signal
pairs and expected transformations (direct, inverted, muxed, etc.). The tool automatically
generates formal properties for each connection and proves them against the RTL. This is
invaluable for SoC integration where thousands of inter-block connections must be verified.

**Q18: What is the state-explosion problem and how do modern tools address it?**

The number of reachable states grows exponentially with the number of state variables
(registers). A design with N flip-flops has up to 2^N states. Modern tools address this
with: symbolic representations (BDDs represent state sets compactly), SAT-based techniques
(don't enumerate states, solve constraints), abstraction (reduce state variables),
compositional reasoning (verify modules independently), and cone-of-influence reduction
(only analyze logic relevant to the property).

**Q19: Explain assume-guarantee compositional verification.**

To verify a large system, decompose it into modules. Verify module A assuming module B
satisfies property P_B. Separately verify module B assuming module A satisfies property P_A.
If both proofs succeed and the assumption-guarantee pairs are consistent (no circular
dependency or proven sound with circular reasoning rules), the entire system is verified
without analyzing it monolithically. This is essential for million-gate designs.

**Q20: How do you debug a formal counterexample?**

The tool provides a trace (sequence of input values and state transitions) leading to the
property violation. Steps: (1) Examine the trace in waveform viewer. (2) Identify what
input sequence triggers the failure. (3) Determine if it's a real bug or a missing
assumption (is the input sequence legal?). (4) If real bug: fix RTL. (5) If missing
assumption: add assume property to constrain inputs and re-verify. (6) Always add a
cover for the scenario to ensure the fix is correct.

**Q21: What are formal "apps" in JasperGold/VC Formal?**

Pre-built formal verification solutions for common tasks: CDC app (clock domain crossing),
connectivity app (signal routing), X-propagation app (uninitialized register analysis),
sequential equivalence app, coverage app, and more. These apps provide specialized
property generation, abstraction, and debug capabilities tuned for their specific
verification task, reducing the effort to set up formal verification from scratch.

**Q22: Can formal verification replace simulation entirely?**

No. Formal is excellent for control logic and protocol verification but struggles with
datapath-heavy designs due to state explosion. System-level scenarios, performance
modeling, power analysis, and analog/mixed-signal verification all require simulation.
The best verification strategies combine: formal for control/protocol, simulation for
datapath/system-level, emulation for software-hardware co-verification.

**Q23: What is the role of formal in the post-silicon debug process?**

When a silicon bug is found, formal can quickly narrow down root causes. The failing
behavior is encoded as a property, and formal exhaustively searches the design for
conditions that produce it. This is often faster than simulation-based debug because
formal doesn't require reproducing the exact test conditions — it finds ALL conditions
that lead to the failure.

**Q24: How do you handle memories in formal verification?**

Large memories cause state explosion (a 4KB RAM has 2^32768 states). Common approaches:
(1) Black-box the memory — assume reads return arbitrary values, prove the surrounding
control logic is correct. (2) Abstract the memory — model as a small (2-4 entry) memory
with symbolic data. (3) Use memory-aware formal engines that handle memories natively
(some tools can track only the entries that are actually read/written).

**Q25: What is the difference between liveness and safety properties?**

Safety properties say "something bad never happens" (e.g., no deadlock, no overflow).
They can be disproven by a finite counterexample. Liveness properties say "something good
eventually happens" (e.g., every request eventually gets a response). They require fairness
constraints (assumptions about the environment not starving the system) and cannot be
disproven by a finite prefix — you'd need an infinite trace. Formal tools handle liveness
via k-liveness (bounded liveness checking) or fair-cycle detection.
