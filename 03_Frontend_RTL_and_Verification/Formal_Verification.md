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
10. Numbers to Memorize
11. Interview Q&A (25+ Questions)

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

### 2.1.1 CDCL Worked Example

Walk through CDCL on a concrete problem to see conflict analysis, clause learning,
non-chronological backtracking, and pruning in action.

**Problem (5 variables, 6 clauses):**
```
C1: ( x1 ∨  x2 ∨  x3)
C2: ( x1 ∨ ¬x2 ∨  x4)
C3: (¬x1 ∨  x3 ∨  x5)
C4: (¬x3 ∨ ¬x4)
C5: (¬x3 ∨ ¬x5)
C6: ( x2 ∨  x4 ∨  x5)
```

We track: current assignment, decision level (DL), implication graph (each literal
annotated with the clause that forced it during BCP, or "d" for decision).

---

**DL 0 — No unit clauses at the start, so no BCP possible.**

---

**DL 1 — Decision: x1 = 0**
```
BCP (unit propagation from x1=0):
  C1: (x1 ∨ x2 ∨ x3)   → only x2, x3 remain. Not unit yet.
  C2: (x1 ∨ ¬x2 ∨ x4)  → only ¬x2, x4 remain. Not unit yet.
  C3: (¬x1 ∨ x3 ∨ x5)  → clause satisfied (¬x1 = 1). Remove from watch.

  No forced assignments. Assignment so far: {¬x1}
```

**DL 2 — Decision: x3 = 1**
```
BCP from x3=1:
  C4: (¬x3 ∨ ¬x4)  → only ¬x4 remains → unit! Force x4 = 0   [reason: C4]
  C5: (¬x3 ∨ ¬x5)  → only ¬x5 remains → unit! Force x5 = 0   [reason: C5]
  C6: (x2 ∨ x4 ∨ x5) → with x4=0, x5=0, only x2 remains → unit! Force x2 = 1  [reason: C6]

  Assignment: {¬x1, x3, ¬x4, ¬x5, x2}
  All variables assigned. Check all clauses:
    C1: (0 ∨ 1 ∨ 1) = 1 ✓    C2: (0 ∨ 0 ∨ 0) = 0 ✗ CONFLICT!
```

**Conflict on C2 under assignment {¬x1, x3, ¬x4, ¬x5, x2}:**

Build the implication graph. Nodes are literals with their reason clauses:
```
  ¬x1 [d, DL1]
        \
  x3 [d, DL2] ──→ ¬x4 [C4, DL2] ──→ ¬x2 must be true for C2 ──┐
     │                                                           │
     ├──→ ¬x5 [C5, DL2] ──────────────────────────┐              │
     │                                             │              │
     └──→ x2 [C6, DL2] ← (x4=0, x5=0 forced x2)  ▼              ▼
                                    CONFLICT at C2: (x1 ∨ ¬x2 ∨ x4)
                                    needs 1 but ¬x1, x2, ¬x4 all contradict
```

**Conflict analysis (1-UIP scheme):**

Starting from the conflicting clause C2 = (x1 ∨ ¬x2 ∨ x4), resolve backwards.
The Unique Implication Point (UIP) is the closest decision-made literal on the
conflict side that "dominates" the conflict in the implication graph.

```
Resolve C2 with the reason for x2 (= C6):
  C2:  (x1 ∨ ¬x2 ∨  x4)
  C6:  (x2 ∨  x4 ∨  x5)    ← reason x2 was forced (x4=0, x5=0 left x2 unit)
  Resolvent: (x1 ∨ x4 ∨ x5)

This resolvent only involves literals from DL ≤ 2, and x4, x5 were implied at DL2.
The 1-UIP is x3 (the decision at DL2 that caused the implications).

Resolve further — the resolvent (x1 ∨ x4 ∨ x5) already has no DL2 decision literal
(x3) in it, but all of x4, x5 were implied from x3. The asserting clause is:

  LEARNED CLAUSE: (x1 ∨ x4 ∨ x5)
  This says: if x1=0 and (x4=0 ∨ x5=0) is forced, the solver will never
  need to explore this branch again.
```

A more precise analysis (resolving until we reach the asserting literal):
```
Actually, (x1 ∨ x4 ∨ x5) still has two implied literals from DL2.
Continue resolving to simplify:

  Resolve (x1 ∨ x4 ∨ x5) with reason for ¬x4 (= C4):
    (x1 ∨  x4 ∨ x5)
    (¬x3 ∨ ¬x4)        ← reason ¬x4 was forced by x3=1
    Resolvent: (x1 ∨ x5 ∨ ¬x3)

  Resolve (x1 ∨ x5 ∨ ¬x3) with reason for ¬x5 (= C5):
    (x1 ∨  x5 ∨ ¬x3)
    (¬x3 ∨ ¬x5)        ← reason ¬x5 was forced by x3=1
    Resolvent: (x1 ∨ ¬x3)

  LEARNED CLAUSE (refined): (x1 ∨ ¬x3)
  This is the asserting clause — it is unit at DL1 (x1=0 is the DL1 decision).
```

**Non-chronological backtrack to DL1 and apply learned clause:**
```
Backtrack to DL1 (where x1 = 0 was decided).
Learned clause (x1 ∨ ¬x3) is now unit: x1=0 forces ¬x3.
  → x3 = 0   [reason: learned clause, DL1]

Assignment: {¬x1, ¬x3}
```

**BCP after learning:**
```
C1: (x1 ∨ x2 ∨ x3)  → with ¬x1, ¬x3: only x2 remains → unit! Force x2 = 1  [C1]
C6: (x2 ∨ x4 ∨ x5)  → satisfied by x2=1.

Assignment: {¬x1, ¬x3, x2}
Check remaining open clauses:
  C2: (x1 ∨ ¬x2 ∨ x4) = (0 ∨ 0 ∨ x4) → only x4 → unit! Force x4 = 1  [C2]
  C3: (¬x1 ∨ x3 ∨ x5) = (1 ∨ ...) → satisfied by ¬x1.

Assignment: {¬x1, ¬x3, x2, x4}
C4: (¬x3 ∨ ¬x4) = (1 ∨ 0) = 1 ✓ (satisfied by ¬x3)
C5: (¬x3 ∨ ¬x5) = (1 ∨ ...) ✓

x5 is unassigned. All clauses containing x5:
  C3: satisfied, C5: satisfied, C6: satisfied.
  x5 can be either value. Set x5 = 0 (arbitrary).

FINAL ASSIGNMENT: {¬x1, ¬x3, x2, x4, ¬x5}
Verify: C1=✓ C2=✓ C3=✓ C4=✓ C5=✓ C6=✓  → SAT
```

**How the learned clause (x1 ∨ ¬x3) prunes future search:**
```
Without the learned clause, if the solver later tried:
  DL1: x1=0, DL2: x3=1 → same conflict → wasted work.

With the learned clause:
  DL1: x1=0 → learned clause (x1 ∨ ¬x3) becomes unit → ¬x3 forced immediately.
  The solver never explores x1=0 ∧ x3=1 again.

In industrial problems with millions of clauses, CDCL learns tens of thousands
of clauses. Each one prunes a subtree of the search space that would otherwise
cause repeated conflicts. This is why CDCL solves problems in seconds that would
take DPLL longer than the age of the universe.
```

---

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

### 2.2.1 BDD Worked Example — Building, Ordering, and Conjunction

**Function: f(a,b,c) = (a ∧ b) ∨ (¬a ∧ c)**

Truth table:
```
a b c | a∧b | ¬a∧c | f
  0 0 0|  0  |  0   | 0
  0 0 1|  0  |  1   | 1
  0 1 0|  0  |  0   | 0
  0 1 1|  0  |  1   | 1
  1 0 0|  0  |  0   | 0
  1 0 1|  0  |  0   | 0
  1 1 0|  1  |  0   | 1
  1 1 1|  1  |  0   | 1
```

**Building the ROBDD with ordering a < b < c:**

Start with a full binary tree, then apply reduction rules:
(R1) Eliminate redundant nodes where both children are identical.
(R2) Merge identical subtrees (share nodes).

```
Unreduced tree:                  After reduction (ROBDD):

        a                              a
       / \                            / \
      b   b                          b   b
     / \ / \                        / \ / \
    c  c c  c                      c  1 0  c
   /\ /\ /\ /\                    / \      / \
  0 1 0 1 0 0 1 1                0  1     0   1

Applying R1: b-node with c→0, c→0 → merge. c-nodes with same children → merge.
```

**Step-by-step reduction:**
```
Level c (leaf parents):
  Node c₀: children (0, 1) — from b-left, c-left
  Node c₁: children (0, 1) — from b-left, c-right   → same as c₀, MERGE
  Node c₂: children (0, 0) — from b-right, c-left   → REDUNDANT (both 0), eliminate → terminal 0
  Node c₃: children (1, 1) — from b-right, c-right  → REDUNDANT (both 1), eliminate → terminal 1

After c-level reduction:
  Only two distinct c-nodes: c₀ (0/1 children) and terminals 0, 1.

Level b:
  b-left:  0-child → c₀,  1-child → c₀   → REDUNDANT? No — wait, b-left's
           0-branch and 1-branch both go to c₀? Let me recount.

Actually, from the truth table with ordering a < b < c:

  When a=0: f = c (independent of b). So b-node on the a=0 branch is REDUNDANT.
  When a=1: f = b (independent of c). So c-node on the a=1 branch is REDUNDANT.

Final ROBDD (ordering a < b < c):

          a
         / \
        /   \
       c     b       ← a=0 skips b (b is irrelevant), a=1 skips c
      / \   / \
     0   1 0   1

  Size: 3 non-terminal nodes (a, c, b). Very compact!
```

**Impact of variable ordering — swap to ordering b < a < c:**

```
With b < a < c, the function f = (a∧b) ∨ (¬a∧c):

When b=0: f = (a∧0) ∨ (¬a∧c) = ¬a∧c = f depends on a and c
When b=1: f = (a∧1) ∨ (¬a∧c) = a ∨ (¬a∧c) = a ∨ c

        b
       / \
      a    a          ← can these be merged? No — different subfunctions.
     / \  / \
    c  c  1  c        ← a-left(b=0,a=0) and a-right(b=1) share c-node? Partially.
   /\  /\    /\
  0  1 1 0  0  1

After reduction: still need separate a-nodes because their 0-children differ.
Final size: 4 non-terminal nodes (b, a₀, a₁, c).

Worse than the a < b < c ordering (3 nodes vs 4 nodes).
```

**Scaling example — why ordering matters for large functions:**
```
Function: f = (x₁∧y₁) ∨ (x₂∧y₂) ∨ (x₃∧y₃)

Ordering 1: x₁, y₁, x₂, y₂, x₃, y₃
  Each xi-yi pair is local. ROBDD "remembers" only the current pair.
  Size: O(n) — 2n + constant nodes for n pairs.

Ordering 2: x₁, x₂, x₃, y₁, y₂, y₃
  After deciding x₁, x₂, x₃, the BDD must "remember" ALL x values
  because each xi needs its corresponding yi. The number of distinct
  subfunctions that need to be represented DOUBLES with each new x.
  Size: O(2ⁿ) — exponential blowup.

For n=20:  Ordering 1 → ~40 nodes.  Ordering 2 → ~1,048,576 nodes.
```

**BDD Conjunction — combining two BDDs:**

Given f(a,b) = a ∧ b and g(a,b) = ¬a ∨ b, compute h = f ∧ g = (a∧b) ∧ (¬a∨b):

```
BDD for f = a∧b (ordering a < b):       BDD for g = ¬a∨b (ordering a < b):
        a                                      a
       / \                                    / \
      0   b                                  1   b
         / \                                    / \
        0   1                                  0   1

BDD AND operation (Apply algorithm):
  Start at roots of both BDDs. For each pair (f_node, g_node):
    If both terminals: return terminal(f_val AND g_val)
    If same variable:  recurse on (f_lo, g_lo) and (f_hi, g_hi)
    If different vars: recurse on the higher-variable node with both branches
                       of the lower-variable node.

  Pair (a_f, a_g): same variable
    (a=0 branch): f→0, g→1 → terminal 0∧1 = 0
    (a=1 branch): f→b_f, g→b_g
      (b=0): f→0, g→0 → 0∧0 = 0
      (b=1): f→1, g→1 → 1∧1 = 1

  Result BDD for h:
        a
       / \
      0   b
         / \
        0   1

  h = a ∧ b, which makes sense: (a∧b) ∧ (¬a∨b) = a∧b∧¬a ∨ a∧b∧b = 0 ∨ a∧b = a∧b.

  Complexity: O(|f| × |g|) node pairs visited. For large BDDs, this product
  can be expensive — but with good variable ordering, individual BDD sizes
  stay small and the product remains tractable.
```

**When to prefer BDDs over SAT:**
```
Use BDDs when:
  - Circuit is small-to-medium (< 10K state variables)
  - You need equivalence checking (BDD: O(1) pointer comparison!)
  - You need to COUNT solutions (SAT only gives one or proves none)
  - The function has good variable ordering (compact BDD)
  - You need an explicit representation of the state set (not just SAT/UNSAT)

Use SAT when:
  - Problem is large (> 100K variables)
  - BDD blows up regardless of ordering (many interdependent variables)
  - You only need one counterexample (not the full state set)
  - Bounded model checking (unroll K cycles → solve once)
  - The problem structure has "local" conflicts that CDCL exploits well
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

### 6.3 What CDC Formal Actually Proves

CDC formal verification addresses three fundamental questions for every signal
crossing a clock domain boundary:

```
1. STABILITY: Is the data signal stable for long enough in the source domain
   that the destination domain can sample a consistent value?
   - For a 2-FF synchronizer: formal proves the signal is stable for at least
     1.5 destination clock cycles (covers both edges of the source clock).
   - For a MUX-based synchronizer: proves the select signal transitions
     only when data is stable.

2. NO DATA LOSS: Does the synchronization scheme guarantee that every
   meaningful data transition is captured by the destination domain?
   - For gray-code pointers: proves only one bit changes per increment,
     so the destination never reads a partially-updated value.
   - For handshake protocols: proves every REQ is eventually acknowledged
     before the source changes the data.

3. NO METASTABILITY PROPAGATION: If metastability occurs (which it will),
   is it contained to the synchronizer and never reaches functional logic?
   - Formal models metastability as an unknown (X) value at the first
     synchronizer flop output.
   - Proves that after the second flop, the value resolves deterministically
     for all legal input transitions.
   - Proves no downstream logic receives X.
```

### 6.4 CDC Formal vs Simulation-Based CDC Checks

```
                    | CDC Formal              | CDC Simulation (with metastability injection)
--------------------|--------------------------|-----------------------------------------------
Exhaustiveness      | Proves for ALL input    | Tests sampled sequences; may miss rare
                    | sequences and timings    | corner-case violations
                    |                          |
Coverage metric     | 100% of CDC paths,      | Depends on test plan coverage of clock
                    | 100% of property space   | ratio combinations and data patterns
                    |                          |
Metastability model | Mathematical: models     | Simulated: injects X or random values at
                    | all possible resolved    | specific injection points; limited to
                    | values simultaneously    | simulated scenarios
                    |                          |
Clock relationship  | Explores ALL phase       | Tests specific phase relationships;
                    | relationships between    | corner cases may be missed if not
                    | domains (ratio + skew)   | explicitly targeted
                    |                          |
Multi-cycle paths   | Proves stability window  | Checks stability in simulated cycles;
                    | is sufficient for ALL    | may miss the one cycle where data
                    | possible timing combos   | changes during the sampling window
                    |                          |
Reconvergence       | Proves all reconvergent  | Only catches if the injected metastable
                    | paths produce consistent  values happen to cause divergence
                    | values for any resolve   in the simulated test
                    |                          |
False paths         | Can prove a crossing is  | Cannot prove safety — absence of failure
                    | safe (no bug exists)     | in simulation ≠ proof of correctness
                    |                          |
Runtime             | Minutes to hours per     | Seconds per test; weeks for comprehensive
                    | block                    | regression
                    |                          |
Scalability         | State explosion for      | Handles full-chip with no state issues;
                    | large CDC path count     | but coverage gaps grow with complexity
```

### 6.5 Common CDC Patterns — Formal Verification Targets

**Pattern 1: 2-Flop Synchronizer (single-bit)**
```
Source Domain          Destination Domain
  ────→ FF1 ──→ FF2 ────→
        clk_a    clk_b

What formal proves:
  - FF1 and FF2 are clocked by the destination clock (clk_b), not source.
  - No combinational logic between FF1 and FF2 (direct connection only).
  - The output of FF2 is used consistently (not fed back without re-synchronization).
  - MTBF is acceptable given clock frequencies and metastability parameters.

Typical property:
  // The synchronizer chain is exactly 2 flops with no combinational logic
  assert property (@(posedge clk_b)
      $changed(sync_in) |-> ##2 sync_out == $past(sync_in, 2));
```

**Pattern 2: MUX-Based Synchronizer (multi-bit, controlled crossing)**
```
Source Domain                Destination Domain
  data_bus[7:0]───→ MUX ────→ data_out[7:0]
                     ↑
  select ──→ 2-FF sync ──→ sel_sync

What formal proves:
  - data_bus is stable during the entire window when sel_sync transitions.
  - select is properly synchronized through a 2-FF chain before use.
  - Data is only sampled when sel_sync indicates a valid transfer.
  - No data change occurs between sel_sync assertion and consumption.
```

**Pattern 3: Asynchronous FIFO (multi-bit, gray-code pointers)**
```
Write Domain (clk_w)          Read Domain (clk_r)
  wr_ptr ──→ gray_enc ──→ 2-FF sync ──→ gray_dec ──→ rd_side_wr_ptr
  rd_ptr_gray ──→ 2-FF sync ──→ wr_side_rd_ptr
  full  = (wr_ptr_gray == {¬rd_ptr_gray[MSB], rd_ptr_gray[MSB-1:0]})
  empty = (rd_ptr_gray == wr_ptr_gray_synced)

What formal proves:
  - Gray encoding: wr_ptr and rd_ptr produce gray codes where exactly 1 bit
    changes per increment.
  - Full flag never deasserts falsely (safe pessimism): the synchronized
    read pointer may be stale, but the full calculation is conservative.
  - Empty flag never deasserts falsely: same argument for write pointer.
  - No data overwrite: write index is never equal to the (synchronized) read
    index when full, and read index is never equal to write when empty.
  - Data integrity: each read returns the value written to that location.
```

**Pattern 4: Handshake Synchronizer (req/ack)**
```
Source Domain              Destination Domain
  data ────────────────────→ data_out
  req ──→ 2-FF sync ──→ req_sync
  ack ←── 2-FF sync ←── ack_dom

What formal proves:
  - data is held stable from req assertion until ack is received.
  - req is not deasserted until ack_sync is seen in the source domain.
  - ack is not asserted until req_sync is seen in the destination domain.
  - No data is lost: every req eventually gets an ack (liveness with fairness).
  - No spurious ack: ack only follows req, never self-generated.
```

### 6.6 What Formal CDC Can Prove That Simulation Cannot

```
1. COVERAGE COMPLETENESS
   Simulation: "We tested with clk_a:clk_b ratios of 1:1, 2:1, 3:1, 1:2"
   Formal: "We proved correctness for ALL possible clock ratios and ALL
   phase alignments." No ratio is untested.

2. METASTABILITY CORRECTNESS
   Simulation: Injects X on one signal at a time; tests N scenarios.
   Formal: Simultaneously considers ALL possible metastable resolutions
   for ALL synchronized signals. Proves no resolution combination causes
   a protocol violation.

3. RECONVERGENCE SAFETY
   Simulation: Rarely catches the case where two synchronized versions of
   the same signal (taken through different synchronizer depths) cause
   functional errors — it depends on which metastable values were injected.
   Formal: Proves that for ALL possible metastable resolutions at ALL
   synchronizers, reconvergent logic produces consistent results.

4. ABSENCE OF BUGS
   Simulation: Can only show bugs ARE found, never that they are ABSENT.
   Formal: Proves no bug exists for the checked property. A PROVEN CDC
   property means the crossing is correct for all time, all inputs, all phases.
```

### 6.7 CDC Formal Tools

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

## 10. Numbers to Memorize

### 10.1 SAT Solver Capacity

```
Variables:         1M–100M+ variables per instance
Clauses:           5M–500M+ clauses per instance
Runtime:           Seconds to hours (varies wildly; industrial instances are hard to predict)
Memory:            1–50 GB for large instances
Learned clauses:   100K–10M during a single solve (clause deletion manages this)

MiniSat (open-source):    Handles ~1M variable instances in minutes
CryptoMiniSat:             ~10M variables with XOR reasoning and Gaussian elimination
CaDiCaL (used in ABC):     State-of-the-art competition winner, ~100M variables
```

### 10.2 LEC Runtime and Capacity

```
Design size:       100K–50M+ gates per block
Compare points:    10K–1M+ per design
Runtime scaling:   O(n log n) for well-structured designs (hierarchical LEC)
                    O(n²) worst-case for flat LEC with many optimization changes
Typical runtime:   Minutes (single block, clean synthesis)
                    Hours (full chip, many ECOs, retiming)
Typical pass rate:  >99% on first run with correct setup
Common failures:    Unmapped points, retiming, register merging
Tool capacity:      Formality and Conformal both handle 50M+ gates
```

### 10.3 Model Checking State Space

```
Practical limit:   ~10⁶–10⁸ reachable states for BDD-based model checking
                    (depends heavily on variable ordering and function structure)

BMC depth:         10–100 cycles typical for bug finding
                    1000+ cycles for deep pipelines (rare, requires abstraction)
                    Completeness threshold: design-specific, often 20–200 cycles

IC3/PDR:           No explicit state enumeration; handles designs where BDD fails
                    Practical limit: ~10⁴–10⁵ state variables for unbounded proofs

k-induction:       k = 1–20 typical (most properties are 1-inductive or need k < 10)
                    k > 50 usually indicates the property needs strengthening lemmas
```

### 10.4 Formal Verification Coverage and Properties

```
Coverage target:    100% of properties proven (unbounded preferred, bounded accepted)
                     Unlike simulation: no "percentage coverage" concept — either
                     proven for ALL cases or not.

Property count:     100–10,000 SVA assertions per block (depending on complexity)
                     Small block (FIFO, arbiter): 50–200 properties
                     Medium block (cache controller): 200–1,000 properties
                     Large block (CPU pipeline): 1,000–10,000 properties

Property categories:
  - Safety (no bad state): 70–80% of all properties
  - Liveness (good thing happens): 10–20%
  - Covers (reachability): 10–15%

Assumption count:   Typically 20–50% of assertion count
Vacuity check:      Every formal run must verify all covers are reachable
```

### 10.5 CDC Metastability MTBF

```
MTBF formula:  MTBF = (e^(t_r / τ)) / (W × f_clk_src × f_clk_dst × f_data)

Where:
  t_r         = Resolution time available (seconds)
                = destination clock period - setup time - clk-to-Q of FF1
  τ           = Metastability time constant (seconds, technology-dependent)
                Typically: 20–100 ps for modern processes
  W           = Metastability window (seconds)
                Typically: 10–100 ps (portion of clock period vulnerable to metastability)
  f_clk_src   = Source domain clock frequency (Hz)
  f_clk_dst   = Destination domain clock frequency (Hz)
  f_data      = Data toggle rate (Hz)

Typical values for 28nm process:
  τ  ≈ 50 ps
  W  ≈ 30 ps
  2-FF synchronizer with f_clk = 500 MHz, f_data = 100 MHz:
    t_r ≈ 2 ns (one full destination clock period for resolution)
    MTBF ≈ e^(2000/50) / (30e-12 × 500e6 × 500e6 × 100e6)
         ≈ e^40 / 7.5e-7 ≈ 2.35 × 10^17 / 7.5e-7 ≈ 3.1 × 10^23 seconds
         ≈ 10^16 years (effectively never fails)

  With 1-FF synchronizer (half the resolution time):
    t_r ≈ 1 ns
    MTBF ≈ e^(1000/50) / 7.5e-7 ≈ e^20 / 7.5e-7 ≈ 7.2 × 10^12 seconds
         ≈ 228,000 years (may not be sufficient for high-reliability)

Rules of thumb:
  - 2-FF synchronizer: MTBF > 10^10 years for most clock frequencies
  - 3-FF synchronizer: needed when f_clk > 500 MHz or t_r < 1 ns
  - Always use 2-FF minimum; never rely on 1-FF in production
```

### 10.6 Formal Tool Landscape

```
Property Verification (FPV):
  JasperGold (Cadence)   — Strongest formal app ecosystem; multiple engines
  VC Formal (Synopsys)   — Tight VCS integration; good for Synopsys flows
  OneSpin (Siemens)      — Strong assertion-based verification
  Ifram (Intel, internal) — Used within Intel for processor verification

Equivalence Checking (LEC):
  Formality (Synopsys)   — Paired with Design Compiler
  Conformal LEC (Cadence) — Paired with Genus/Innovus
  Hector (Synopsys)      — Sequential equivalence checking

CDC Verification:
  SpyGlass CDC (Synopsys) — Structural lint + formal
  JasperGold CDC (Cadence) — Full formal CDC analysis
  Meridian CDC (Cadence)   — Structural lint
  Questa CDC (Siemens)     — Lint + formal

Connectivity / X-Prop:
  JasperGold Connectivity App
  VC Formal Connectivity App
  SpyGlass CDC (also handles some connectivity)

Open-Source / Academic:
  ABC (UC Berkeley)       — Synthesis and verification framework (SAT-based)
  MiniSat                 — Minimalist SAT solver (educational baseline)
  Pono (Stanford)         — Modern open-source model checker
  AIGER format            — Standard for circuit verification benchmarks
```

### 10.7 Quick Reference — Key Numbers for Interviews

```
| Metric                              | Value / Range                    |
|-------------------------------------|----------------------------------|
| SAT solver capacity                 | 1M–100M+ variables               |
| SAT clauses per instance            | 5M–500M+                         |
| BDD practical state limit           | 10⁶–10⁸ reachable states         |
| BMC typical depth                   | 10–100 cycles                    |
| k-induction typical k               | 1–20                             |
| LEC design capacity                 | 50M+ gates                       |
| LEC runtime scaling                 | O(n log n) to O(n²)              |
| Property count per block            | 100–10,000 SVA assertions        |
| Formal coverage target              | 100% of properties proven        |
| 2-FF synchronizer MTBF (28nm)      | > 10^10 years                    |
| Metastability τ (28nm)             | 20–100 ps                        |
| Metastability window W (28nm)      | 10–100 ps                        |
| Resolution time for 2-FF (500 MHz) | ~2 ns                            |
| Compare points per LEC run          | 10K–1M+                          |
| CDCL learned clauses per solve      | 100K–10M                         |
```

---

## 11. Interview Q&A

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

---

## 12. AI Hardware Formal Verification

### 12.1 Tensor Core Correctness Verification

```
Challenge: Proving a GEMM (General Matrix Multiply) array produces correct results

  Formal verification target:
    Result = A × B + C  (matrix multiply-accumulate)
    Where A, B, C are NxN matrices (typically N=4 for a tensor core tile)

  Verification approach:
    1. Abstract datapath: reduce 32-bit to 4-bit for tractability
    2. Prove: systolic array output matches specification for ALL inputs
    3. Specification: golden reference model (simple loop implementation)

  SVA properties for tensor core:
    // After N cycles of valid input, output matches reference
    assert property (@(posedge clk) disable iff (!rst_n)
        output_valid |-> result_matrix == reference_matrix);

    // No data corruption in accumulation
    assert property (@(posedge clk) disable iff (!rst_n)
        accumulate_en |-> accumulator_next == accumulator_curr + product);

    // Pipeline integrity: no data lost between stages
    assert property (@(posedge clk) disable iff (!rst_n)
        stage_N_valid |-> stage_N_data == expected_stage_N_data);

  Challenge with wide datapaths:
    - 32-bit multiply: 2^64 input combinations
    - Must use data abstraction or word-level solvers
    - Some tools support bit-blasting with SAT for small widths
    - Alternative: word-level formal (SMT solvers, SMT-LIB QF_BV)

  Practical approach:
    - Prove correctness for 4×4 tiles with reduced precision (4-bit elements)
    - Use structural arguments to extend to full precision
    - Verify rounding / saturation / overflow behavior separately
    - Combine formal proof with constrained-random simulation for full-width
```

### 12.2 NoC (Network-on-Chip) Deadlock Freedom

```
NoC formal verification targets:

  1. Deadlock freedom:
     Prove: No cycle of dependencies among virtual channels can form

     Approach:
       - Model the routing function as a formal model
       - Prove: the channel dependency graph (CDG) is acyclic
       - This is a graph-theoretic property → naturally suited to formal

     Property:
       assert property (@(posedge clk) disable iff (!rst_n)
           !(packet_stuck && no_progress_for_N_cycles));
       // No packet can be indefinitely blocked

  2. Liveness (no starvation):
     Prove: Every injected packet eventually reaches its destination

     assert property (@(posedge clk) disable iff (!rst_n)
         packet_injected |-> s_eventually packet_received);

  3. In-order delivery (if required):
     Prove: Packets from same source to same destination arrive in order

     assert property (@(posedge clk) disable iff (!rst_n)
         packet_A_injected_before_packet_B |->
         packet_A_received_before_packet_B);

  4. Flow control correctness:
     Prove: credit-based flow control never overflows buffers

     assert property (@(posedge clk) disable iff (!rst_n)
         buffer_occupancy <= BUFFER_DEPTH);

  Abstraction strategies for NoC:
    - Model data payload abstractly (only headers matter for routing)
    - Reduce number of virtual channels in proof (prove structure holds)
    - Use symmetry: prove for one router, extend by structural argument
    - Topology-specific: mesh, torus, butterfly each have known proof methods
```

### 12.3 Dataflow Accelerator Verification

```
Dataflow accelerators (Google TPU, SambaNova, Groq):

  Verification challenge:
    Prove: the computation graph executes correctly on the dataflow fabric

  Formal targets:
    1. Schedule correctness:
       Prove: all operations execute in the correct order
       (respecting data dependencies)

       assert property (@(posedge clk) disable iff (!rst_n)
           consumer_en |-> producer_done);

    2. Buffer sizing correctness:
       Prove: no buffer overflow occurs for any valid schedule

       assert property (@(posedge clk) disable iff (!rst_n)
           fifo_count <= FIFO_DEPTH);

    3. Memory consistency:
       Prove: reads and writes to shared memory are correctly ordered

       assert property (@(posedge clk) disable iff (!rst_n)
           write_complete |-> ##1 read_sees_new_data);

    4. Data integrity through the pipeline:
       Prove: output matches the intended computation for all inputs
       (requires abstraction for large tensors)

  Approach:
    - Extract computation graph from compiler output
    - Generate formal properties from the graph structure
    - Verify hardware implementation satisfies graph execution requirements
    - Tools: custom property generators + JasperGold/VC Formal backends
```

### 12.4 RISC-V Formal Verification

```
RISC-V formal verification ecosystem:

  riscv-formal framework (open source, by Clifford Wolf):
    - Defines ISA-level formal properties for RISC-V processors
    - Covers: RV32I, RV64I, M, A, F, D extensions
    - Properties verify: instruction semantics, exception handling,
      privilege mode transitions, CSR behavior

  ISA formal specification:
    Prove: processor implementation matches ISA specification for ALL
    legal instruction sequences

    // Example: ADD instruction correctness
    assert property (@(posedge clk) disable iff (!rst_n)
        decode_ADD && execute_valid |->
        reg_file[rd] == reg_file_prev[rs1] + reg_file_prev[rs2]);

    // Example: No unintended side effects
    assert property (@(posedge clk) disable iff (!rst_n)
        decode_ADD && retire_valid && (rd != rs1) && (rd != rs2) |->
        reg_file[rs1] == reg_file_prev[rs1] &&
        reg_file[rs2] == reg_file_prev[rs2]);

  Verification levels:
    Level 1: ISA compliance (riscv-formal)
    Level 2: Microarchitecture-specific (pipeline hazards, forwarding)
    Level 3: Implementation-level (cache coherence, TLB correctness)

  Tools:
    riscv-formal:   Open-source ISA-level formal checks
    RVVI (RISC-V Verification Interface): Standardized formal interface
    Formal ASIC tools: JasperGold/VC Formal for custom properties
```

### 12.5 GPU Shader Verification

```
GPU shader core verification challenges:

  Formal targets:
    1. SIMT execution correctness:
       Prove: all threads in a warp execute the same instruction
       (divergence handled correctly by hardware)

       assert property (@(posedge clk) disable iff (!rst_n)
           warp_active && !divergence |->
           all_threads_same_pc);

    2. Warp scheduling fairness:
       Prove: no warp is starved indefinitely

       assert property (@(posedge clk) disable iff (!rst_n)
           warp_valid[i] && !warp_issued[i] |->
           s_eventually warp_issued[i]);

    3. Register file banking correctness:
       Prove: bank conflicts are detected and resolved

       assert property (@(posedge clk) disable iff (!rst_n)
           bank_conflict_detected |-> stall_or_arbitrate);

    4. Memory consistency (GPU memory model):
       Prove: memory operations obey the GPU memory model rules
       (e.g., CUDA memory model or OpenCL memory model)
       - This is extremely complex: requires reasoning about
         multiple memory scopes and ordering guarantees

  Why GPU shader verification is hard for formal:
    - Wide datapaths (32 threads × 32-bit = 1024-bit)
    - Deep pipelines (10-20 stages)
    - Complex memory hierarchy (L1, shared memory, L2, global)
    - Dynamic scheduling and dependency resolution
    - State space: exponentially larger than CPU cores

  Practical approach:
    - Verify control logic formally (scheduling, scoreboarding)
    - Verify datapath with simulation (wide arithmetic)
    - Use formal for memory ordering properties
    - Cache coherence protocol verification (formal is essential here)
```

### 12.6 Security Verification

```
Formal verification for hardware security:

  1. Side-channel resistance:
     - Prove: no data-dependent timing variation in crypto engines

       assert property (@(posedge clk) disable iff (!rst_n)
           crypto_operation_valid |->
           ##CONSTANT_LATENCY crypto_done);

     - Constant-time execution: prove number of cycles is input-independent
     - Power-equalized paths: formally prove no correlation between
       secret data and switching activity
     - Cache timing attacks: prove secret-dependent data does not
       influence cache access patterns

  2. Secure boot verification:
     - Prove: boot ROM code is immutable
     - Prove: chain of trust is maintained from ROM → bootloader → OS
     - Prove: no unauthorized code can execute before authentication

       assert property (@(posedge clk) disable iff (!rst_n)
           boot_phase == PHASE_ROM |->
           pc inside {ROM_START:ROM_END});

  3. Access control verification:
     - Prove: untrusted domain cannot access trusted resources
     - Prove: firewall/MPU rules are enforced for ALL access patterns

       assert property (@(posedge clk) disable iff (!rst_n)
           untrusted_master_access && target_requires_trusted |->
           access_denied);

  4. Information flow security (non-interference):
     - Prove: secret inputs do not influence observable outputs
     - Formalized as: for all secret states S1, S2 with same public state,
       the observable outputs are identical

     This is the strongest security guarantee — proving that
     no amount of observation of public outputs can reveal secret data.

  5. Glitch-free clock gating:
     - Prove: clock gating control cannot produce glitches
     - Glitches could be exploited for fault injection attacks

  Tools for security formal:
    - CW308/CW305 (ChipWhisperer): side-channel analysis (measurement)
    - Formal tools: prove absence of vulnerabilities (JasperGold, VC Formal)
    - Logic synthesis: verify no timing channels introduced during optimization
```
