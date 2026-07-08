# Adders — From First Principles to Silicon

## Half Adder — The Foundation

Two single-bit inputs A, B produce Sum and Carry:

```text
Sum   = A XOR B
Carry = A AND B
```

| A | B | Sum | Carry |
|---|---|-----|-------|
| 0 | 0 |  0  |   0   |
| 0 | 1 |  1  |   0   |
| 1 | 0 |  1  |   0   |
| 1 | 1 |  0  |   1   |

Gate count: 1 XOR + 1 AND = 2 gates, 6 transistors in CMOS.

---

## Full Adder — Deriving Generate and Propagate

### Truth Table and Equations

| A | B | Cin | Sum | Cout |
|---|---|-----|-----|------|
| 0 | 0 |  0  |  0  |  0   |
| 0 | 0 |  1  |  1  |  0   |
| 0 | 1 |  0  |  1  |  0   |
| 0 | 1 |  1  |  0  |  1   |
| 1 | 0 |  0  |  1  |  0   |
| 1 | 0 |  1  |  0  |  1   |
| 1 | 1 |  0  |  0  |  1   |
| 1 | 1 |  1  |  1  |  1   |

From the truth table, using K-map or algebraic simplification:
```text
Sum  = A XOR B XOR Cin
Cout = A*B + Cin*(A XOR B)
```

### Derivation of G and P

Let's define:
```text
G_i = A_i AND B_i        (Generate: carry is produced regardless of incoming carry)
P_i = A_i XOR B_i        (Propagate: incoming carry passes through)
```

**Why these definitions?** Look at Cout:
```text
Cout = A*B + Cin*(A XOR B)
     = G_i + P_i * Cin
```

This is the **fundamental carry recurrence:**
```text
C_{i+1} = G_i + P_i * C_i
```

And Sum:
```text
S_i = A_i XOR B_i XOR C_i = P_i XOR C_i
```

**Physical meaning:**
- **Generate (G=1):** Both inputs are 1, so a carry is ALWAYS produced (like a battery generating voltage)
- **Propagate (P=1):** Exactly one input is 1, so a carry from below passes through (like a wire conducting)
- **Kill (G=0, P=0):** Both inputs are 0, carry is absorbed (blocked)

### Gate Delay Model

For the rest of this document, we use the following gate delay model (representative of a 28nm standard-cell library, in units of "gate delays" ≈ FO4 inverter delay ≈ 15-25 ps):

| Gate     | Delay (gate delays) | Notes                                  |
|----------|--------------------|-----------------------------------------|
| INV      | 0.5                | Simplest, fastest gate                  |
| NAND2    | 1.0                | Reference gate                          |
| NOR2     | 1.0                | Slightly slower than NAND in CMOS       |
| AND2     | 1.0                | Often implemented as NAND+INV           |
| OR2      | 1.0                | Often implemented as NOR+INV            |
| XOR2     | 2.0                | Typically 2 NAND delays (complex gate)  |
| AOI21    | 1.0                | AND-OR-Invert: fast compound gate       |
| OAI21    | 1.0                | OR-AND-Invert: fast compound gate       |
| MUX2     | 1.0-1.5            | Transmission gate or AOI implementation |
| FA carry | 1.5                | Using AOI/OAI: G+P*Cin in one compound gate |
| FA sum   | 2.0                | XOR of P and Cin                        |

---

## Subtractor and Overflow Detection

### Subtractor

To compute A - B using an adder, exploit 2's complement: A - B = A + (~B) + 1.

**Implementation:** Invert all bits of B (NOT gate per bit) and set the carry-in of the LSB full adder to 1. The carry-in = 1 completes the 2's complement negation of B.

```text
    A[N-1:0]   ~B[N-1:0]
       |           |
     [  Adder, Cin = 1  ]
       |
     S[N-1:0] = A - B
```

A combined adder/subtractor uses a MUX on each B bit (select B or ~B) and sets Cin accordingly:

```verilog
wire [N-1:0] b_effective = sub ? ~b : b;
wire         cin_effective = sub;
assign {cout, sum} = a + b_effective + cin_effective;
```

### Overflow Detection for 2's Complement Signed Addition

Overflow occurs when two same-sign numbers produce a different-sign result.

**Method 1 — Carry XOR:**
```text
Overflow = carry_in[MSB] XOR carry_out[MSB]
```

Why: carry_in[MSB] and carry_out[MSB] disagree exactly when the sign of the result is wrong. If both are 1, the MSB propagated a carry correctly. If both are 0, no carry was involved. Only when they differ did a carry appear or disappear in a way that flipped the sign unexpectedly.

**Method 2 — Sign check:**
```text
Overflow = (A[MSB] == B[MSB]) AND (Sum[MSB] != A[MSB])
```

This directly encodes the definition: same-sign inputs, different-sign output.

**Worked example (4-bit):** 0111 (7) + 0011 (3) = 1010 (-6 in 4-bit 2's complement).
- carry_in[MSB] = 1 (carry propagated into bit 3)
- carry_out[MSB] = 0 (no carry out of bit 3)
- Overflow = 1 XOR 0 = 1 -- overflow detected correctly.

### Magnitude Comparator

To compare A and B: compute A - B and examine the carry-out of the subtractor.

- If cout = 1: no borrow occurred, so A >= B.
- If cout = 0: borrow occurred, so A < B.
- Equality: A == B iff the subtractor result is zero (use a NOR tree on all sum bits).

---

## Ripple Carry Adder (RCA)

### Architecture

Chain N full adders, Cout[i] → Cin[i+1]:

```ascii-graph
    A[0] B[0]    A[1] B[1]    A[2] B[2]        A[N-1] B[N-1]
      |   |        |   |        |   |              |     |
     [FA_0]-------[FA_1]-------[FA_2]--- ... ---[FA_{N-1}]
      |   |        |   |        |   |              |     |
     S[0] Cout→Cin S[1] Cout→Cin S[2]           S[N-1] Cout
```

### Exact Delay Analysis

**Critical path:** Carry from bit 0 to bit N-1, then final sum.

```text
T_RCA = T_PG_generation + (N-1) * T_carry_propagation + T_final_sum
      = T_XOR + (N-1) * T_AOI + T_XOR
```

Using our delay model:
```text
T_PG = 2.0 (XOR for P_i)
T_carry = 1.5 (AOI/OAI compound gate for G + P*C)
T_final_sum = 2.0 (XOR for P XOR C)

T_RCA(N) = 2.0 + (N-1) * 1.5 + 2.0 = 4.0 + 1.5*(N-1)
```

**Numerical examples:**

| Bit width | Delay (gate delays) | @28nm FO4=20ps | @7nm FO4=8ps |
|-----------|--------------------|--------------------|------------------|
| 4-bit     | 8.5                | 170 ps             | 68 ps            |
| 8-bit     | 14.5               | 290 ps             | 116 ps           |
| 16-bit    | 26.5               | 530 ps             | 212 ps           |
| 32-bit    | 50.5               | 1010 ps            | 404 ps           |
| 64-bit    | 98.5               | 1970 ps            | 788 ps           |

A 32-bit RCA at 28nm takes ~1 ns — far too slow for a 2 GHz design (500 ps clock period). This motivates all the fast adder architectures.

```verilog
module ripple_carry_adder #(parameter N = 32) (
    input  [N-1:0] a, b,
    input           cin,
    output [N-1:0] sum,
    output          cout
);
    wire [N:0] c;
    assign c[0] = cin;

    genvar i;
    generate
        for (i = 0; i < N; i = i + 1) begin : fa
            assign sum[i] = a[i] ^ b[i] ^ c[i];
            assign c[i+1] = (a[i] & b[i]) | (c[i] & (a[i] ^ b[i]));
        end
    endgenerate

    assign cout = c[N];
endmodule
```

---

## Carry Look-Ahead Adder (CLA) — Full Derivation

### Expanding the Carry Recurrence

Starting from `C_{i+1} = G_i + P_i * C_i`:

```text
C1 = G0 + P0 * C0                                              ... (1)

C2 = G1 + P1 * C1
   = G1 + P1 * (G0 + P0 * C0)                    [substitute (1)]
   = G1 + P1*G0 + P1*P0*C0                                     ... (2)

C3 = G2 + P2 * C2
   = G2 + P2 * (G1 + P1*G0 + P1*P0*C0)           [substitute (2)]
   = G2 + P2*G1 + P2*P1*G0 + P2*P1*P0*C0                      ... (3)

C4 = G3 + P3 * C3
   = G3 + P3*G2 + P3*P2*G1 + P3*P2*P1*G0 + P3*P2*P1*P0*C0   ... (4)
```

**Observation:** Each carry expression is a sum-of-products with increasing fan-in:
- C1: 2-input AND + 2-input OR
- C2: 3-input AND + 3-input OR
- C3: 4-input AND + 4-input OR
- C4: 5-input AND + 5-input OR

### Fan-In Explosion Problem

For a flat (non-hierarchical) 16-bit CLA:
```text
C16 = G15 + P15*G14 + P15*P14*G13 + ... + P15*P14*...*P0*C0
```

This requires a **17-input AND gate** and a **17-input OR gate**. In CMOS:
- Maximum practical fan-in is 4-5 (beyond this, the series NMOS/PMOS stack resistance causes excessive delay)
- A 17-input AND must be decomposed into a tree: log2(17) ≈ 5 levels of 2-input ANDs

This defeats the purpose of O(1) look-ahead!

### Hierarchical CLA Solution

Group 4-bit CLA blocks, then apply look-ahead again at the group level.

**Group Generate and Propagate for bits i:j (e.g., bits 3:0):**
```text
G_{3:0} = G3 + P3*G2 + P3*P2*G1 + P3*P2*P1*G0
P_{3:0} = P3 * P2 * P1 * P0
```

These are computed within each 4-bit CLA block (fan-in ≤ 4, feasible).

**Second-level CLA computes group carries:**
```text
C4  = G_{3:0}  + P_{3:0}  * C0
C8  = G_{7:4}  + P_{7:4}  * C4
C12 = G_{11:8} + P_{11:8} * C8
C16 = G_{15:12}+ P_{15:12}* C12
```

Or, expanded:
```text
C8  = G_{7:4} + P_{7:4}*G_{3:0} + P_{7:4}*P_{3:0}*C0
C12 = G_{11:8} + P_{11:8}*G_{7:4} + P_{11:8}*P_{7:4}*G_{3:0} + ...
C16 = ... (still only 5-input fan-in at the second level)
```

### Delay of Hierarchical CLA

1. **PG generation** (XOR, AND)               = 2.0 gate delays
2. **4-bit CLA block** (2 gate levels: AND-OR) = 2.0 gate delays
3. **Second-level CLA** (group carries)        = 2.0 gate delays
4. **Carry back into blocks + final sum      = 2.0 gate delays**

- **Total for 16-bit** — ~8 gate delays

For 64-bit (3 levels of hierarchy):
```text
Total for 64-bit: ~12 gate delays  (vs. 98.5 for RCA)
```

**Comparison: 32-bit adders**

| Architecture       | Delay (gate delays) | Area (relative) |
|--------------------|--------------------|--------------------|
| RCA                | 50.5               | 1.0x               |
| Hierarchical CLA   | 10                  | ~2.5x              |
| Carry Select        | ~18                | ~2.0x              |
| Carry Skip          | ~20                | ~1.5x              |
| Kogge-Stone prefix  | ~9                 | ~4.0x              |
| Brent-Kung prefix   | ~14                | ~1.8x              |
| Han-Carlson prefix  | ~10                | ~2.5x              |

---

## Carry Select Adder (CSLA) — Optimal Block Sizing

### Basic Structure

Divide N-bit adder into blocks. For each block except the first, compute two sums in parallel (assuming Cin=0 and Cin=1), then select the correct one with a MUX when the actual carry arrives.

- **Block 0** — [0:k-1]    Compute normally (RCA)
- **Block 1** — [k:2k-1]   Two RCAs + MUX
- **Block 2** — [2k:3k-1]  Two RCAs + MUX
...
- **Block m** — [...:N-1]   Two RCAs + MUX

### Delay with Uniform Block Size k

```text
T_CSLA = T_RCA(k) + (N/k - 1) * T_MUX
```

Where $T_{RCA}(k) = 4.0 + 1.5(k-1)$ and $T_{MUX} \approx 1.0\text{–}1.5$.

### Optimal Block Size — Calculus Derivation

For uniform blocks:
```text
T(k) = T_RCA(k) + (N/k - 1) * T_MUX
     = [4.0 + 1.5(k-1)] + (N/k - 1) * 1.5
     = 2.5 + 1.5k + 1.5N/k - 1.5
     = 1.0 + 1.5k + 1.5N/k
```

Minimize by taking derivative and setting to zero:
```text
dT/dk = 1.5 - 1.5N/k^2 = 0
k^2 = N
k_opt = sqrt(N)
```

**Optimal delay:**
```text
T_opt = 1.0 + 1.5*sqrt(N) + 1.5*sqrt(N) = 1.0 + 3.0*sqrt(N)
```

This is **O(sqrt(N))**.

**Numerical example for N=32:**
```ascii-graph
k_opt = sqrt(32) ≈ 5.66 → use k=6 (or try 5 and 6)

k=5: T = 1.0 + 1.5*5 + 1.5*32/5 = 1.0 + 7.5 + 9.6 = 18.1
k=6: T = 1.0 + 1.5*6 + 1.5*32/6 = 1.0 + 9.0 + 8.0 = 18.0
```

### Square-Root Carry Select (Variable Block Sizes)

The insight: block 0 doesn't need to be fast because it has no MUX chain to feed. Later blocks can be larger because by the time their carry arrives, they've had more time to compute.

Use increasing block sizes: k1, k1+1, k1+2, ... such that:
```text
T_RCA(k_j) = T_RCA(k_1) + j * T_MUX
```

This balances the RCA delay of each block with the accumulated MUX chain delay.

```text
T_RCA(k_j) - T_RCA(k_1) = j * T_MUX
1.5 * (k_j - k_1) = j * 1.5
k_j = k_1 + j
```

So block sizes are: k1, k1+1, k1+2, ..., covering N total bits:
```text
sum = k1 + (k1+1) + (k1+2) + ... + (k1+m-1) = m*k1 + m*(m-1)/2 = N
```

For N=16, k1=2: blocks of 2, 3, 4, 5, 2 spare → total = 2+3+4+5 = 14 (need 2 more in last block = 7), or readjust: 2, 2, 3, 4, 5 = 16.

```text
T = T_RCA(2) + 4 * T_MUX = 5.5 + 6.0 = 11.5 gate delays
```

vs. uniform k=4: T = T_RCA(4) + 3*T_MUX = 8.5 + 4.5 = 13.0. The variable sizing saves 1.5 gate delays.

---

## Carry Skip (Bypass) Adder — Derivation

### Skip Condition

For a block of k bits (positions i to i+k-1):
```text
Block Propagate: BP = P_i * P_{i+1} * ... * P_{i+k-1}
```

If BP = 1, **every bit in the block propagates**, so:
```text
Cout_block = Cin_block (carry passes through unchanged)
```

If BP = 0, at least one bit generates or kills, and the block carry comes from the RCA within the block.

### Implementation

```text
Cout_block = (BP * Cin_block) | (~BP * Cout_RCA)
           = BP ? Cin_block : Cout_RCA
```

This is a 2:1 MUX (or AND-OR gate). The skip path bypasses the entire RCA delay.

### Delay Analysis

Critical path goes through:
1. First block: full RCA delay (no skip possible — no prior carry to skip)
2. Middle blocks: skip path (AND gate for BP + MUX)
3. Last block: full RCA delay (must compute final sum, can't skip)

```text
T_skip = T_RCA(k) + (N/k - 2) * T_skip_MUX + T_RCA(k)
       = 2 * T_RCA(k) + (N/k - 2) * T_skip
```

Using our model:
```text
T_skip = 2 * [4.0 + 1.5(k-1)] + (N/k - 2) * 1.5
       = 2 * (2.5 + 1.5k) + 1.5N/k - 3.0
       = 5.0 + 3.0k + 1.5N/k - 3.0
       = 2.0 + 3.0k + 1.5N/k
```

### Optimal Block Size — sqrt(N) Proof

```text
dT/dk = 3.0 - 1.5N/k^2 = 0
k^2 = N/2
k_opt = sqrt(N/2)
```

**Optimal delay:**
```text
T_opt = 2.0 + 3.0*sqrt(N/2) + 1.5*sqrt(2N)
      = 2.0 + 3.0*sqrt(N/2) + 3.0*sqrt(N/2)     [since 1.5*sqrt(2N) = 3.0*sqrt(N/2)]
      = 2.0 + 6.0*sqrt(N/2) = 2.0 + 3.0*sqrt(2N)
```

This is O(sqrt(N)), same asymptotic class as carry select, but with less area (no dual computation).

**For N=32:** k_opt = sqrt(16) = 4
```text
T = 2.0 + 3.0*4 + 1.5*32/4 = 2.0 + 12.0 + 12.0 = 26.0 gate delays
```

### Variable Block Sizing (Advanced)

Use smaller blocks at the beginning and end (where the critical path starts/ends) and larger blocks in the middle (where the skip path dominates). The optimal sizing has blocks growing as 1, 2, 3, ..., 3, 2, 1 (pyramid shape).

The optimal variable-block skip adder achieves delay proportional to O(sqrt(N)) with a smaller constant factor.

---

## Prefix Adders — Formal Treatment

### The Prefix Operator

Define the carry operator "o" on (G, P) pairs:
```text
(G_L, P_L) o (G_R, P_R) = (G_L + P_L * G_R, P_L * P_R)
```

**Physical meaning:** "o" combines two adjacent groups. The combined group generates a carry if the left group generates, OR the left propagates AND the right generates. The combined group propagates if BOTH propagate.

### Proof of Associativity

We must show: `[(G_a, P_a) o (G_b, P_b)] o (G_c, P_c) = (G_a, P_a) o [(G_b, P_b) o (G_c, P_c)]`

**Left side:**
```text
(G_a, P_a) o (G_b, P_b) = (G_a + P_a*G_b, P_a*P_b)

[(G_a + P_a*G_b, P_a*P_b)] o (G_c, P_c)
= ((G_a + P_a*G_b) + P_a*P_b*G_c, P_a*P_b*P_c)
= (G_a + P_a*G_b + P_a*P_b*G_c, P_a*P_b*P_c)
```

**Right side:**
```text
(G_b, P_b) o (G_c, P_c) = (G_b + P_b*G_c, P_b*P_c)

(G_a, P_a) o [(G_b + P_b*G_c, P_b*P_c)]
= (G_a + P_a*(G_b + P_b*G_c), P_a*P_b*P_c)
= (G_a + P_a*G_b + P_a*P_b*G_c, P_a*P_b*P_c)
```

Both sides equal `(G_a + P_a*G_b + P_a*P_b*G_c, P_a*P_b*P_c)`. **QED.**

Associativity means we can evaluate the prefix computation in ANY order — this is what enables tree-structured parallel computation.

**Note:** The operator is NOT commutative. `(G_a, P_a) o (G_b, P_b) ≠ (G_b, P_b) o (G_a, P_a)` in general. The left-right ordering matters (it corresponds to bit position ordering).

### The Parallel Prefix Problem

Given N individual (G_i, P_i) pairs, compute all PREFIX results:
```text
(G_{0:0}, P_{0:0}) = (G_0, P_0)
(G_{1:0}, P_{1:0}) = (G_1, P_1) o (G_0, P_0)
(G_{2:0}, P_{2:0}) = (G_2, P_2) o (G_1, P_1) o (G_0, P_0)
...
(G_{N-1:0}, P_{N-1:0}) = (G_{N-1}, P_{N-1}) o ... o (G_0, P_0)
```

Each (G_{i:0}, P_{i:0}) directly gives carry C_{i+1}:
```text
C_{i+1} = G_{i:0} + P_{i:0} * C_0
```

If C_0 = 0 (typical): `C_{i+1} = G_{i:0}`.

### 8-bit Kogge-Stone — Detailed Computation Graph

```ascii-graph
Bit position:    7      6      5      4      3      2      1      0
                 |      |      |      |      |      |      |      |
Level 0 (PG):  [7]    [6]    [5]    [4]    [3]    [2]    [1]    [0]
                 |      |      |      |      |      |      |      |
Level 1:       [7:6]  [6:5]  [5:4]  [4:3]  [3:2]  [2:1]  [1:0]  [0]
  (span 1)       \   / \   / \   / \   / \   / \   /
                  [o]   [o]   [o]   [o]   [o]   [o]    (7 nodes use result from 1 position left)
                 |      |      |      |      |      |      |      |
Level 2:       [7:4]  [6:3]  [5:2]  [4:1]  [3:0]  [2:0]  [1:0]  [0]
  (span 2)       \     |   / \     |   /     |      |
                  \    [o]    \    [o]        |      |     (combine with result from 2 positions left)
                 |      |      |      |      |      |      |      |
Level 3:       [7:0]  [6:0]  [5:0]  [4:0]  [3:0]  [2:0]  [1:0]  [0]
  (span 4)       \     |     |   /           |      |
                  \   [o]   [o]  /           |      |     (combine with result from 4 positions left)
                 |      |      |      |      |      |      |      |

Total prefix nodes: 7 + 5 + 3 = 15  (general: N*log2(N) - N + 1)
Depth: 3 levels (log2(8) = 3)
Fan-out: maximum 2 (each node feeds at most 2 nodes in the next level)
Wire tracks: at level k, wires cross 2^(k-1) bit positions → level 3 has wires spanning 4 positions
```

**Key property of Kogge-Stone:** Every prefix result is available at the same level (level 3 = log2(N)). No node waits for another node at the same level. Maximum parallelism.

**Key drawback:** Wiring congestion. At level 3, each node connects to a node 4 positions away. For a 64-bit Kogge-Stone, level 6 has wires spanning 32 bit positions. These long wires dominate the physical delay in advanced nodes. **In practice, Kogge-Stone is rarely used beyond 32 bits because wiring congestion kills the speed benefit.**

### 8-bit Brent-Kung — Detailed Computation Graph

```ascii-graph
Bit position:    7      6      5      4      3      2      1      0
                 |      |      |      |      |      |      |      |
Level 0 (PG):  [7]    [6]    [5]    [4]    [3]    [2]    [1]    [0]
                 |      |      |      |      |      |      |      |
Level 1:       [7:6]  [6]   [5:4]  [4]   [3:2]  [2]   [1:0]  [0]
  (span 1)     /  \          /  \          /  \          /  \
              [o]   skip   [o]   skip    [o]   skip   [o]   skip
                 |      |      |      |      |      |      |      |
Level 2:       [7:4]  [6]   [5:4]  [4]   [3:0]  [2]   [1:0]  [0]
  (span 2)     /  \                      /  \
              [o]   skip               [o]   skip
                 |      |      |      |      |      |      |      |
Level 3:       [7:0]  [6]   [5:4]  [4]   [3:0]  [2]   [1:0]  [0]
  (span 4)
         ─── now propagate results back DOWN ───
                 |      |      |      |      |      |      |      |
Level 4:       [7:0]  [6:0]  [5:4]  [4:0]  [3:0]  [2:0]  [1:0]  [0]
  (inverse)              \     |            \     |
                         [o]  skip          [o]  skip  (combine with left neighbor's prefix)
                 |      |      |      |      |      |      |      |
Level 5:       [7:0]  [6:0]  [5:0]  [4:0]  [3:0]  [2:0]  [1:0]  [0]
                         |
                        [o]  (combine [5:4] with [3:0])

Total prefix nodes: 4 + 2 + 1 + 2 + 1 = 10  (general: 2N - 2 - log2(N))
Depth: 5 levels (2*log2(N) - 1 = 5)
Fan-out: higher (level 3 node [7:0] feeds 3 nodes in inverse tree)
Wire tracks: much lower than Kogge-Stone
```

**Brent-Kung trade-off:** Nearly half the depth is "wasted" on the inverse tree (propagating prefix results from even to odd positions). But the total node count is O(N) instead of O(N*log N), making it dramatically cheaper in area and wiring.

### 8-bit Sklansky — Fan-Out Doubling Scheme

The Sklansky adder uses a fan-out doubling scheme: at level i, prefix results from row i-1 are broadcast to all higher rows.

```text
Bit position:    7      6      5      4      3      2      1      0
                 |      |      |      |      |      |      |      |
Level 0 (PG):  [7]    [6]    [5]    [4]    [3]    [2]    [1]    [0]
                 |      |      |      |      |      |      |      |
Level 1:       [7:6]  [6:5]  [5:4]  [4:3]  [3:2]  [2:1]  [1:0]  [0]
  (span 1)       [o]    [o]    [o]    [o]    [o]    [o]   (combine with 1 left)
                 |      |      |      |      |      |      |      |
Level 2:       [7:4]  [6:3]  [5:2]  [4:1]  [3:0]  [2:1]  [1:0]  [0]
  (span 2)       [o]    [o]    [o]    [o]
                 ↑ fan-out from level 1 = 2 per node
                 |      |      |      |      |      |      |      |
Level 3:       [7:0]  [6:3]  [5:2]  [4:1]  [3:0]  [2:1]  [1:0]  [0]
  (span 4)       [o]
                 ↑ fan-out from level 2 = 4 per node
```

**Key difference from Kogge-Stone:** Kogge-Stone replicates nodes to keep fan-out = 1 at every level. Sklansky accepts doubling fan-out instead: a node at level i drives 2^i downstream nodes. At the last level (level 3 for 8-bit), fan-out reaches N/2 = 4.

- Area: O(N log N) -- fewer nodes than Kogge-Stone.
- Wire: O(N log N) -- less wiring than Kogge-Stone.
- Fan-out: up to N/2 at the last level -- this is the weakness. High fan-out increases electrical delay (capacitive loading) at later levels.
- Delay: log2(N) prefix levels (same depth as Kogge-Stone), but the large fan-out degrades the electrical delay at later levels.

### Knowles [l,k,m] Parameterization

The Knowles parameterization generalizes prefix adders by specifying the maximum fan-out at each level of the prefix tree. The notation [l, k, m, ...] gives the fan-out limit at each level from the top down:

| Adder         | Knowles parameter | Fan-out per level        | Notes                        |
|---------------|--------------------|--------------------------|------------------------------|
| Kogge-Stone   | [1, 1, 1, ...]    | Fan-out 1 at every level | Maximum replication, minimum fan-out |
| Sklansky      | [N/2, N/4, ...]   | Maximum fan-out          | Minimum replication, maximum fan-out |
| Brent-Kung    | Intermediate       | Moderate fan-out         | Inverse tree for propagation |

This framework lets designers explore the trade-off between wiring complexity (Kogge-Stone) and fan-out loading (Sklansky) to find the optimal point for a given technology node and bit width.

### Prefix Adder Comparison — Quantitative

For N = 32:

| Property            | Kogge-Stone | Brent-Kung | Sklansky   | Han-Carlson  |
|---------------------|-------------|------------|------------|--------------|
| Depth (levels)      | 5           | 9          | 5          | 6            |
| Prefix nodes        | 129         | 57         | 80         | ~65          |
| Max wire span       | 16 positions| 2 positions| 16 positions| 8 positions |
| Max fan-out         | 2           | 16         | 16         | 2            |
| Delay (gate delays) | ~12         | ~20        | ~12*       | ~14          |
| Area (relative)     | 4.0x        | 1.8x       | 2.5x       | 2.5x        |

*Sklansky's 12 gate-delay estimate assumes ideal fan-out. With buffer insertion for the exponential fan-out, actual delay is 15-18 gate delays, comparable to Brent-Kung.

### Kogge-Stone 16-bit — Complete Synthesizable Verilog

```verilog
module kogge_stone_16 (
    input  [15:0] a, b,
    input          cin,
    output [15:0] sum,
    output         cout
);
    // Level 0: Initial PG generation
    wire [15:0] g0, p0;
    assign g0 = a & b;
    assign p0 = a ^ b;

    // Prefix tree — 4 levels for 16 bits
    // Level 1: span 1
    wire [15:0] g1, p1;
    assign g1[0]  = g0[0] | (p0[0] & cin);
    assign p1[0]  = p0[0];  // p1[0] not used for carry, but keep for regularity

    genvar i;
    generate
        for (i = 1; i < 16; i = i + 1) begin : L1
            assign g1[i] = g0[i] | (p0[i] & g0[i-1]);
            assign p1[i] = p0[i] & p0[i-1];
        end
    endgenerate

    // Level 2: span 2
    wire [15:0] g2, p2;
    assign g2[0] = g1[0]; assign p2[0] = p1[0];
    assign g2[1] = g1[1]; assign p2[1] = p1[1];

    generate
        for (i = 2; i < 16; i = i + 1) begin : L2
            assign g2[i] = g1[i] | (p1[i] & g1[i-2]);
            assign p2[i] = p1[i] & p1[i-2];
        end
    endgenerate

    // Level 3: span 4
    wire [15:0] g3, p3;
    generate
        for (i = 0; i < 4; i = i + 1) begin : L3_pass
            assign g3[i] = g2[i]; assign p3[i] = p2[i];
        end
        for (i = 4; i < 16; i = i + 1) begin : L3
            assign g3[i] = g2[i] | (p2[i] & g2[i-4]);
            assign p3[i] = p2[i] & p2[i-4];
        end
    endgenerate

    // Level 4: span 8
    wire [15:0] g4, p4;
    generate
        for (i = 0; i < 8; i = i + 1) begin : L4_pass
            assign g4[i] = g3[i]; assign p4[i] = p3[i];
        end
        for (i = 8; i < 16; i = i + 1) begin : L4
            assign g4[i] = g3[i] | (p3[i] & g3[i-8]);
            assign p4[i] = p3[i] & p3[i-8];
        end
    endgenerate

    // Carry vector: C[i+1] = G_{i:0} (since cin is folded into g1[0])
    // Sum: S[i] = P[i] ^ C[i]
    assign sum[0] = p0[0] ^ cin;
    generate
        for (i = 1; i < 16; i = i + 1) begin : SUM
            assign sum[i] = p0[i] ^ g4[i-1];
        end
    endgenerate

    assign cout = g4[15];
endmodule
```

---

## Carry-Save Adder (CSA)

### 3:2 CSA — A Full Adder Repurposed

A 3:2 CSA (carry-save adder) is simply a full adder used differently: it takes 3 input bits at the same bit position and produces 2 output bits (Sum and Carry). The critical property: **the carry is NOT propagated to the next column** -- it is "saved" and shifted left by 1 position (its weight is 2x).

```text
Inputs:  A[i], B[i], C_in[i]
Outputs: Sum[i], Carry[i-1]  (Carry has weight of position i+1)

Sum[i]   = A[i] XOR B[i] XOR C_in[i]
Carry[i] = (A[i] AND B[i]) OR (C_in[i] AND (A[i] XOR B[i]))
```

Because there is no carry chain, a 3:2 CSA has **O(1) delay regardless of bit width**.

### Multi-Operand Reduction

To reduce N operands to 2 operands (Sum vector + Carry vector), use a tree of 3:2 CSAs:

- Each CSA layer reduces the operand count by 1 (3 inputs -> 2 outputs, net reduction of 1).
- Total layers needed: N - 2.

The final Sum and Carry vectors must be added by a carry-propagate adder (CPA) to produce a single result.

### CSA vs CPA

| Property        | CSA (3:2)           | CPA (e.g., RCA, CLA)          |
|-----------------|----------------------|--------------------------------|
| Delay           | O(1) per layer       | O(N) or O(log N)              |
| Outputs         | 2 vectors (Sum, Carry) | 1 vector (result)           |
| Use case        | Multi-operand reduction | Final summation of 2 operands |
| Carry chain     | None                 | Yes (this is the bottleneck)  |

CSA trees are the backbone of multipliers (Wallace/Dadda trees), dot-product units, and FIR filter accumulators. The CPA at the end of the tree is the only stage with carry propagation.

---

## Booth Encoding for Multipliers

### The Problem

A straightforward N×N multiplier generates N partial products, each N bits wide. Reducing N partial products is expensive (N-1 levels of carry-save addition).

### Radix-2 Booth Encoding

Recode the multiplier B to use digits from {-1, 0, +1} instead of {0, 1}:

```text
Scanning from LSB, examine overlapping pairs (b_{i}, b_{i-1}) where b_{-1} = 0:
  b_i  b_{i-1}  |  Booth digit  |  Action
   0      0     |       0       |  No partial product (skip)
   0      1     |      +1       |  Add multiplicand
   1      0     |      -1       |  Subtract multiplicand
   1      1     |       0       |  No partial product (skip)
```

**Advantage:** Strings of consecutive 1s in the multiplier (like 0111110) produce only two non-zero digits (-1 at the start, +1 at the end), reducing the number of partial products.

**Disadvantage:** Worst case (alternating 01010101) still produces N non-zero digits — no reduction.

### Radix-4 (Modified Booth) Encoding

Scan the multiplier in overlapping groups of 3 bits: (b_{2i+1}, b_{2i}, b_{2i-1}):

```ascii-graph
b_{2i+1}  b_{2i}  b_{2i-1}  |  Booth digit  |  Partial product
   0        0        0       |       0        |  0
   0        0        1       |      +1        |  +A
   0        1        0       |      +1        |  +A
   0        1        1       |      +2        |  +2A (left shift A by 1)
   1        0        0       |      -2        |  -2A
   1        0        1       |      -1        |  -A
   1        1        0       |      -1        |  -A
   1        1        1       |       0        |  0
```

**Key benefit:** Reduces N partial products to N/2 — cutting the partial product tree depth roughly in half.

**Hardware for each Booth digit:**
- +A: pass through
- -A: bitwise invert + add 1 (2's complement)
- +2A: left shift by 1
- -2A: invert + shift + add 1
- 0: zero

**Cost:** Each partial product generator requires a small MUX network (select between 0, A, 2A, -A, -2A) plus a sign-extension scheme. But the reduction from N to N/2 partial products more than compensates.

**Radix-4 Booth is the standard in all commercial multiplier designs.** It is the default in synthesis tools (DesignWare, etc.).

### Signed vs Unsigned with Booth

Booth encoding naturally handles signed multiplication (2's complement) — negative partial products are generated by the encoding itself. For unsigned multiplication, prepend a 0-bit to both operands to prevent incorrect sign extension.

---

## Wallace Tree and Dadda Tree Multipliers

### The Partial Product Reduction Problem

After Booth encoding, an N-bit multiplier has N/2 partial products, each ~N+1 bits wide. We need to reduce these to two numbers (which are then added by a final CPA — carry-propagate adder).

### Wallace Tree

**Strategy:** At each level, group columns of bits into sets of 3, reduce each set with a full adder (3:2 compressor), and carry the sum/carry to the next level. Half adders (2:2 compressors) handle remaining pairs.

**For 8×8 multiplier (with Booth: 4 partial products):**

```ascii-graph
Level 0: Partial products (max column height = 4)
         4   4   4   4   4   4   3   2   1  (bit column heights, approximately)

Level 1: Apply FA to groups of 3, HA to pairs
         After reduction: max height 3

Level 2: Apply FA again
         After reduction: max height 2

Level 3: Final CPA (2 → 1)
```

**General:** Wallace tree reduces N partial products to 2 in ceil(log_{1.5}(N)) levels. For N=4: 3 levels. For N=16: 6 levels.

### Dadda Tree

**Strategy:** Reduce the minimum number of entries per column at each level to a predetermined sequence: ..., 6, 4, 3, 2. At each stage, only reduce columns that exceed the target height.

**Dadda sequence:** d_1 = 2, d_{j+1} = floor(1.5 * d_j)
```text
2, 3, 4, 6, 9, 13, 19, 28, 42, 63, ...
```

For N partial products, find the smallest d_j >= N, then reduce backward to d_1 = 2.

**Dadda vs Wallace comparison:**

| Property          | Wallace Tree      | Dadda Tree        |
|-------------------|-------------------|-------------------|
| Number of levels  | Same              | Same              |
| FAs used          | More (eager)      | Fewer (lazy)      |
| HAs used          | Fewer             | More              |
| Final CPA width   | Narrower          | Wider             |
| Total hardware    | Slightly more     | Slightly less     |
| Regularity        | Less regular      | More regular      |

In practice, the difference is small. Synthesis tools choose the optimal reduction tree automatically.

### 4:2 Compressor

A common building block that compresses 4 bits + carry-in into 2 bits + carry-out:

```text
Inputs:  x1, x2, x3, x4, cin
Outputs: sum, carry, cout

sum  = x1 XOR x2 XOR x3 XOR x4 XOR cin
carry = (x1 XOR x2 XOR x3 XOR x4) ? cin : x4
cout  = (x1 XOR x2) ? x3 : x1
```

**Key property:** cout does NOT depend on cin — it feeds the same column's next level, not the next column. This breaks the carry chain and allows parallel reduction.

A 4:2 compressor is equivalent to 2 cascaded full adders, but with the critical path optimized (3 XOR delays instead of 4).

---

## Sequential (Shift-Add) Multiplier

The Booth/Wallace/Dadda multipliers above are *combinational*: they build the entire partial-product tree in hardware and produce a result in one (deep) combinational path. The shift-add multiplier makes the opposite trade — it reuses **one** adder over ~N cycles instead of a tree, trading throughput and latency for a large area saving.

### Algorithm (N×N → 2N-bit)

Keep a single `2N`-bit *product* register; load the multiplier into its low half and clear the high half. The multiplicand sits in its own `N`-bit register. Each cycle:

1. If `product[0] == 1`, add the multiplicand to the **high half** of the product register.
2. **Shift the whole product register right by 1** (this also walks the multiplier bits down to `product[0]` one at a time).

After `N` cycles the multiplier bits are exhausted and the `2N`-bit product register holds `multiplicand × multiplier`.

```text
init:  P[2N-1:N] = 0 ;  P[N-1:0] = multiplier ;  M = multiplicand
repeat N times:
    if (P[0]) P[2N:N] = P[2N-1:N] + M     // extra bit catches the add carry
    P = P >> 1                            // shift right by 1
result: P[2N-1:0]
```

**Signed:** use an **arithmetic** right shift and sign-extend the partial sum, or Booth-encode the multiplier — radix-4 Booth also **halves the cycle count** (~N/2) while handling sign naturally (see [Booth Encoding for Multipliers](#booth-encoding-for-multipliers) above).

### Hardware

- One `N`-bit carry-propagate adder (CPA) — the only arithmetic unit, reused every cycle.
- A `(2N+1)`-bit shift register (the product reg plus a carry bit).
- AND-gating on the multiplicand (the conditional add = `M & {N{P[0]}}`).
- A small control FSM / down-counter with a `start` / `valid` (busy) / `done` handshake.

### Tradeoff vs Combinational Multipliers

| Multiplier        | Adders / area      | Throughput      | Latency        | Use when |
|-------------------|--------------------|-----------------|----------------|----------|
| Shift-add (this)  | **1 CPA**, tiny    | 1 result / ~N clk | ~N cycles    | area-critical, low multiply rate |
| Radix-4 Booth seq | 1 CPA + encoder    | 1 result / ~N/2 clk | ~N/2 cycles | area-critical, moderate rate |
| Array / Wallace (comb) | O(N²) FAs     | 1 result / clk (if pipelined) | 1 deep comb path (or P stages) | high throughput |

Rule of thumb: pick the architecture from the required multiply throughput. If you issue a multiply only occasionally (config math, address calc), the shift-add unit's near-zero area wins; if every cycle needs a product (datapath, MAC array), pay for the pipelined Wallace tree.

---

## Adder Selection Guide — Practical Wisdom

### When to Hand-Instantiate vs. Let the Tool Decide

In modern ASIC flows (Synopsys Design Compiler with DesignWare, Cadence Genus):

```text
// This is the RIGHT way for 99% of cases:
assign sum = a + b;
// The tool selects the optimal adder from its library
// based on timing constraints, area goals, and bit width.
```

**When to manually instantiate:**

1. **Specific pipeline structure:** If you need the adder split across pipeline stages (e.g., G/P generation in stage 1, prefix tree in stage 2, sum in stage 3), the tool cannot infer this from `a + b`.

2. **Compound operations:** For operations like `(a + b) == c` (add-compare), a manual implementation can merge the adder and comparator, saving hardware. The tool may not discover this.

3. **Critical path engineering:** If the tool's adder choice creates a critical path through one specific bit position, you might restructure the prefix tree to balance delays differently.

4. **DesignWare override:** You can hint to the tool:
```verilog
// Synopsys DesignWare pragmas:
// synopsys dc_script_begin
// set_implementation DW01_add/cla [find cell "add_instance"]
// synopsys dc_script_end
```

5. **FPGA carry chains:** On FPGAs, always use `+` — the dedicated carry chain hardware is faster than any custom LUT-based adder. Manually implementing a Kogge-Stone on FPGA would waste LUTs and be slower.

### Area-Delay Trade-off — Numerical Summary

For 32-bit adders in 28nm, post-synthesis:

| Architecture        | Area (um^2) | Delay (ps) | Power (uW@1GHz) |
|---------------------|-------------|------------|------------------|
| RCA                 | 280         | 980        | 12               |
| Carry Skip          | 410         | 480        | 18               |
| Carry Select        | 520         | 420        | 22               |
| Brent-Kung prefix   | 450         | 380        | 20               |
| Han-Carlson prefix  | 580         | 320        | 25               |
| Kogge-Stone prefix  | 820         | 290        | 32               |
| DesignWare auto     | ~500        | ~330       | ~23              |

These are representative numbers — actual values depend on the specific library, drive strengths, and tool optimization effort.

---

## Approximate Adders for AI and Neural Networks

### Motivation

Neural network inference is inherently error-tolerant — small arithmetic errors in individual additions are absorbed by the network's redundancy and activation functions. This creates an opportunity to trade arithmetic accuracy for speed, area, and power using **approximate adders**.

Error tolerance in neural networks:
- ReLU activation clips negative values to 0 → errors on small values don't propagate
- Sigmoid/tanh saturate for large values → errors on large values don't propagate
- Weight regularization (L2, dropout) already introduces noise → approximate arithmetic fits naturally
- Quantization to INT8/FP8 already loses 90%+ of precision → adder errors are secondary
- Typical accuracy impact: < 1% drop in top-1 accuracy for 8-bit approximate adders

### Truncated Carry Chain Adder

```ascii-graph
Idea: Only propagate carries for the lower K bits instead of all N bits.

  Standard N-bit RCA: carries propagate through all N bit positions
  Truncated (K-segment): only propagate carries within each K-bit block

  For N=8, K=4:
    Block 0: bits [3:0] — full carry chain within block
    Block 1: bits [7:4] — full carry chain within block
    Carry from block 0 to block 1 is IGNORED (assumed 0)

  Error rate: ~50% of inputs produce wrong results
  Error magnitude: up to 2^N - 2^(N-K) (missing carry can affect upper bits)
  But in practice, average error is small for random inputs

  Area savings: ~30-40% (eliminates inter-block carry logic)
  Delay savings: O(K) instead of O(N) → critical path is K bits, not N

  Hardware:
    N/K independent K-bit adders operating in parallel
    Each block: K-bit RCA or CLA
    No carry chain between blocks
```

### Error-Tolerant Adder (ETA)

```ascii-graph
The Error-Tolerant Adder splits the adder into accurate and approximate regions:

  For an N-bit adder, split at bit position N/2:
    Lower half (bits 0 to N/2-1): approximate addition
    Upper half (bits N/2 to N-1): accurate addition

  Lower half (approximate):
    Modified full adder where carry is NOT propagated to the next bit
    Instead, each bit computes: S_i = A_i XOR B_i (XOR instead of full add)
    This is equivalent to assuming Cin = 0 for every bit position
    No carry chain at all → O(1) delay for the lower half

  Upper half (accurate):
    Standard adder (CLA, RCA, etc.) for the upper N/2 bits
    Carry-in from lower half is estimated:
      If lower half would produce a carry-out > 50% of the time,
      set Cin to 1 (heuristic)

  Error characteristics:
    Lower half has at most 1-bit error per position (no carry propagation)
    Average error: ~2-5% of the result magnitude
    Maximum error: bounded by 2^(N/2) (about half the result)
```

### Lower-Part-OR Adder (LOA)

```ascii-graph
Simple approximate adder where the lower bits use OR instead of addition:

  N-bit adder split at position K:
    Upper bits [N-1:K]: standard accurate adder (e.g., Kogge-Stone)
    Lower bits [K-1:0]: bitwise OR → S_i = A_i | B_i

  Why OR? Truth table comparison:
    A_i  B_i  | Correct (A+B)  | OR  | Error
     0    0   |       0        |  0  |  none
     0    1   |       1        |  1  |  none
     1    0   |       1        |  1  |  none
     1    1   |      10        |  1  |  missing carry (error of +1 at position i)

  Only the (1,1) case produces an error (missing carry).
  Error probability per bit: P(A=1 AND B=1) = 1/4 for random inputs.
  Total error: bounded by K (if all K lower bits have missing carries).

  Carry from lower to upper: use AND of A[K-1] and B[K-1] as carry-in
  (captures the most significant carry that matters)

  Area savings: K XOR gates replaced by K OR gates (slightly simpler)
  Plus: no carry chain in lower K bits → significant delay and power savings
```

### Speculative Carry-Lookahead Approximate Adder

```ascii-graph
Combines speculative execution with approximation:

  Strategy:
    1. Divide N-bit addition into M blocks of size K = N/M
    2. Each block starts computing simultaneously
    3. For carry-in to each block (except the first), SPECULATE:
       - Compute two results: one assuming Cin=0, one assuming Cin=1
       - OR: predict carry-in using lower bits only (no waiting for actual carry)

  Speculative carry prediction:
    For block j, predict carry-out of block j-1:
      Predict_Cout = OR(A[j-1], B[j-1])  (overestimate: assumes propagation)
      If any bit in the previous block has both A=1 and B=1, carry is guaranteed
      If all bits have A=0 and B=0, carry is impossible
      Middle cases: prediction may be wrong

  Error recovery:
    In some implementations, compute actual carry in parallel
    If prediction was wrong, use a MUX to select correct result (1-cycle penalty)
    In approximate mode: skip the check, accept occasional errors

  Area: 2x the adder blocks (for dual computation) + MUX + prediction logic
  Delay: O(K) for K-bit blocks, computed in parallel → much faster than O(N)
  Error rate: depends on prediction accuracy, typically 1-5%
```

### Practical Application in AI Accelerators

**Where approximate adders are used:**

1. MAC (Multiply-Accumulate) units in neural network inference:
- Partial sums: approximate addition of accumulator and product
   - Accumulator width: typically 24-32 bits
   - Lower 8-12 bits can use approximation with minimal accuracy impact
   - Upper bits (which determine the output value) remain accurate

2. Softmax and attention mechanisms:
- Exponent subtraction for attention scores: approximate subtraction acceptable
   - Reduces critical path in attention computation by 30-50%

3. ReLU / activation functions:
- Only need to determine sign (positive or negative) for ReLU
   - Approximate addition sufficient for sign detection in most cases

4. Pooling layers:
- Average pooling: sum followed by division
   - Approximate sum has negligible effect after division

**Design considerations:**
   - Always keep the MSB (sign bit) computation accurate
   - Error should be unbiased (zero mean) to avoid systematic drift
   - Use accurate adders for the first and last layers of a network
   - (where errors have the most impact on final output)
   - Quantization-aware training (QAT) can compensate for approximate hardware
   - by training the network with injected adder errors

---

## Numbers to Memorize

| Quantity | Value |
|---|---|
| RCA delay (28nm) | 4.0 + 1.5(N-1) gate delays |
| CLA delay (28nm) | ~4 gate delays (any width) |
| Kogge-Stone delay | log2(N) prefix levels |
| Kogge-Stone area | O(N log^2 N) |
| Sklansky delay | log2(N) levels + fan-out penalty |
| Sklansky area | O(N log N) |
| Brent-Kung delay | 2 log2(N) - 1 levels |
| Carry-select optimal block | sqrt(N) |
| Carry-skip optimal block | sqrt(N/2) |
| 32-bit RCA delay (28nm) | ~50 gate delays |
| 32-bit CLA delay (28nm) | ~4 gate delays |
| 32-bit Kogge-Stone delay (28nm) | ~6 gate delays |
| FA area (28nm) | ~20-30 gate equivalents |
| 32-bit RCA area (28nm) | ~640-960 GE |
| 32-bit Kogge-Stone area (28nm) | ~3000-5000 GE |
