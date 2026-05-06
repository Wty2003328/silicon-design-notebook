# Floating Point Arithmetic — Deep Dive for IC Design

## IEEE 754 Format — Beyond the Basics

### Why Bias = 127 (Not 128)?

For 8-bit exponent, exponent values 0 and 255 are reserved (for denorms/zero and inf/NaN respectively). Usable exponent range: 1 to 254.

The actual exponent range is: `E_actual = E_stored - bias`, so:
```
E_actual_min = 1 - bias
E_actual_max = 254 - bias
```

We want the range to be roughly symmetric around 0. With bias = 127:
```
E_actual_min = 1 - 127 = -126
E_actual_max = 254 - 127 = +127
```

Range: [-126, +127] — nearly symmetric, with one more positive value.

With bias = 128:
```
E_actual_min = 1 - 128 = -127
E_actual_max = 254 - 128 = +126
```

Range: [-127, +126] — biased toward negative exponents.

**The choice bias = 2^(k-1) - 1 (not 2^(k-1))** ensures the maximum positive exponent magnitude is as large as the maximum negative, which is important because:
1. The reciprocal of the largest representable number should be representable (or at least close to it): 2^127 ≈ 1.7e38, and 2^(-126) ≈ 1.2e-38 — both in range.
2. With bias=128, 2^126 * 2^(-127) would result in 2^(-1) (fine), but 2^126 * 2^126 = 2^252 > 2^126 (overflow with bias=128's max of +126), while with bias=127, 2^127 * 2^(-127) = 1, and 2^127 * 2^127 overflows but that's expected.
3. Historical convention from the original IEEE 754-1985 committee deliberations.

### Actual Bit Patterns for Edge Cases (Single Precision)

**Positive zero: +0**
```
S=0, E=0000_0000, M=000_0000_0000_0000_0000_0000
Hex: 0x00000000
Value: +0
```

**Negative zero: -0**
```
S=1, E=0000_0000, M=000_0000_0000_0000_0000_0000
Hex: 0x80000000
Value: -0 (compares equal to +0 by IEEE rules, but sign bit differs)
```

**Smallest positive denormal:**
```
S=0, E=0000_0000, M=000_0000_0000_0000_0000_0001
Hex: 0x00000001
Value: 0.000...001 (binary) * 2^(-126) = 2^(-23) * 2^(-126) = 2^(-149)
     ≈ 1.401298e-45
```

**Largest denormal:**
```
S=0, E=0000_0000, M=111_1111_1111_1111_1111_1111
Hex: 0x007FFFFF
Value: 0.111...111 * 2^(-126) = (1 - 2^(-23)) * 2^(-126)
     ≈ 1.175494e-38
```

**Smallest positive normal:**
```
S=0, E=0000_0001, M=000_0000_0000_0000_0000_0000
Hex: 0x00800000
Value: 1.0 * 2^(1-127) = 2^(-126)
     ≈ 1.175494e-38
```

**Transition from denormal to normal:**
```
Largest denorm: 0.111...111 * 2^(-126) = (1 - 2^(-23)) * 2^(-126)
Smallest normal: 1.000...000 * 2^(-126) = 1.0 * 2^(-126)

Gap between them: 2^(-126) - (1 - 2^(-23)) * 2^(-126) = 2^(-23) * 2^(-126) = 2^(-149)
This equals exactly 1 ULP of the denormal range — CONTINUOUS! No gap.
```

This is the entire point of denormals: **gradual underflow**. Without denormals, there would be a gap from 0 to 2^(-126), and the property `a - b == 0 implies a == b` would break.

**Largest normal:**
```
S=0, E=1111_1110 (=254), M=111_1111_1111_1111_1111_1111
Hex: 0x7F7FFFFF
Value: 1.111...111 * 2^(254-127) = (2 - 2^(-23)) * 2^127
     ≈ 3.402823e+38
```

**Positive infinity:**
```
S=0, E=1111_1111, M=000_0000_0000_0000_0000_0000
Hex: 0x7F800000
```

**Quiet NaN (canonical):**
```
S=0, E=1111_1111, M=100_0000_0000_0000_0000_0000
Hex: 0x7FC00000
MSB of mantissa = 1 → quiet (does not trap)
```

**Signaling NaN:**
```
S=0, E=1111_1111, M=010_0000_0000_0000_0000_0000
Hex: 0x7FA00000
MSB of mantissa = 0, rest ≠ 0 → signaling (raises exception when used)
```

---

## Floating Point Addition — Detailed Pipeline

### Hardware Architecture: 5-Stage Pipeline

```
Stage 1: EXPONENT COMPARE + SWAP
  - Subtract exponents: d = E_A - E_B (8-bit subtractor)
  - If d < 0: swap operands (A becomes the larger-exponent operand)
  - Prepend implicit 1 (or 0 for denormals) to mantissas
  Hardware cost: 8-bit subtractor, 2:1 MUX for 24-bit mantissa swap, 8-bit MUX for exponent

Stage 2: ALIGNMENT SHIFT
  - Right-shift smaller mantissa by |d| positions
  - Capture shifted-out bits into Guard, Round, Sticky
  - Sticky = OR of all bits shifted beyond Round position
  Hardware cost: Barrel shifter (24+3 bit output, shift amount up to 24), sticky OR tree
  NOTE: This is the widest/most expensive stage — 24-bit barrel shifter

Stage 3: MANTISSA ADD/SUBTRACT
  - Add or subtract aligned mantissas (based on effective operation: signs match → add, differ → subtract)
  - Result width: 25 bits (carry-out for addition) or 24 bits (for subtraction)
  Hardware cost: 25-bit adder (or 28-bit including GRS)

Stage 4: NORMALIZATION
  - Leading-zero count (for subtraction results that may have leading zeros)
  - Left-shift mantissa by LZC amount, decrement exponent accordingly
  - OR: right-shift by 1 if addition overflow (mantissa >= 2.0), increment exponent
  Hardware cost: LZC tree (log N delay), barrel shifter (left), 8-bit exponent adder

Stage 5: ROUNDING + POST-NORMALIZATION
  - Apply rounding mode using GRS bits
  - If rounding causes mantissa overflow (1.111...1 + 1 = 10.0...0): right-shift by 1, increment exponent
  - Check for exponent overflow (→ infinity) or underflow (→ denorm or zero)
  Hardware cost: Incrementer (for round-up), exponent overflow/underflow detection
```

### The Dual-Path Adder (Why |d| <= 1 Is Special)

In a naive implementation, the critical path goes through: alignment shift → adder → LZC → normalization shift → rounding. This is 5 serial stages, and the LZC + normalization shift is expensive.

**Key observation:** When |d| <= 1 (exponent difference is 0 or 1), subtraction can cause massive cancellation — the result might have many leading zeros (up to 24 for single precision). This is the ONLY case where a large normalization shift is needed.

When |d| >= 2, the alignment shift discards many bits of the smaller operand. After subtraction, the result has at most 1 leading zero (addition: 0 zeros; subtraction with |d|>=2: at most 1).

**Dual-path architecture:**

```
FAR PATH (|d| >= 2):
  - Full alignment shift
  - Add/subtract
  - Normalize by 0 or 1 bit (trivial — just a MUX)
  - Round
  CRITICAL PATH: alignment shift → adder → 1-bit normalize → round

CLOSE PATH (|d| <= 1):
  - Align by 0 or 1 bit (trivial)
  - Subtract (always subtract in close path, since |d|<=1 and signs differ for cancellation)
  - LZA (Leading Zero Anticipator) runs IN PARALLEL with the subtractor
  - Normalize by LZA amount (may be 0 to 24 bits)
  - Round (simpler — fewer GRS bits matter after massive cancellation)
  CRITICAL PATH: subtractor + LZA-correction → normalize → round

SELECT: based on |d|, choose far or close path result
```

**Why this helps:** The far path avoids the expensive LZC + large normalization shift. The close path avoids the expensive alignment shift. Each path is individually faster than the combined path.

### Leading Zero Anticipator (LZA)

The LZA predicts the leading-zero count from the two mantissa inputs BEFORE the subtraction completes.

**Key insight:** For subtraction A - B (with |d| <= 1), define for each bit position i:
```
T_i = A_i XOR B_i       (transfer: bits differ)
G_i = A_i AND NOT B_i   (generate: A > B at this position)
Z_i = NOT A_i AND B_i   (zero: A < B at this position)
```

The leading zero pattern of A - B can be predicted from the T/G/Z pattern with at most 1 bit of error:

```
LZA pattern bit f_i = T_{i+1} XOR (G_i | Z_i)  (approximately)
```

The position of the leading 1 in the f vector gives the LZC, accurate to ±1. A 1-bit correction is applied after the actual subtraction result is available (just check if the MSB of the normalized result is 0, and if so, shift one more).

**Hardware:** The LZA is a parallel computation (just XOR/AND/OR gates across all bit positions), followed by a priority encoder. It adds O(log N) delay for the encoder but runs IN PARALLEL with the subtractor, so it doesn't add to the critical path.

### Compound Adder Trick

For the close path, we need both A - B and B - A (we don't know which is larger when |d| = 0). Instead of two full subtractors:

**Compound adder:** Computes both (A - B) and (B - A) using a single prefix tree.

The prefix tree computes carries for A + (~B) + 1 (which is A - B). Simultaneously, the complement sums give B - A = ~(A - B) + 1 (if A ≠ B). In practice, the compound adder computes (A - B - 1) and uses the carry-out to determine the sign, then adds 1 to the correct result. This reuses the same prefix tree for both directions.

---

## Rounding — GRS Bits Sufficiency Proof

### Claim

Three extra bits (Guard, Round, Sticky) are sufficient to correctly round the infinitely-precise result to the nearest representable floating-point number, for ALL four IEEE rounding modes.

### Proof Sketch

After alignment and addition, the infinitely-precise result is:
```
R = r_{n-1} r_{n-2} ... r_1 r_0 . g r s_{k} s_{k-1} ... s_0
                                    ^
                          rounding point (between bit 0 and guard bit)
```

We need to round R to n bits. The question is: does the exact truncated value `T = r_{n-1}...r_0` need to be incremented by 1 ULP?

**For round-to-nearest-even (default):**
- If the tail `g r s_k...s_0` < 0.1000...0 (i.e., < half ULP): truncate. Decision depends on g only: g=0 → truncate.
- If the tail > 0.1000...0 (> half ULP): round up. This happens when g=1 AND (r=1 OR s_k...s_0 ≠ 0). Since Sticky = OR(s_k...s_0), this is g=1 AND (r=1 OR sticky=1).
- If the tail = exactly 0.1000...0 (= half ULP, tie): round to even. This happens when g=1, r=0, sticky=0. Decision depends on LSB (r_0).

**All decisions can be made from {g, r, sticky, r_0}.** No additional bits beyond GRS are needed.

**For round-toward-+infinity (ceiling):**
- Round up if any bit in the tail is 1 AND the number is positive: `(g OR r OR sticky) AND sign==0`.

**For round-toward--infinity (floor):**
- Round up if any bit in the tail is 1 AND the number is negative.

**For round-toward-zero (truncate):**
- Always truncate. GRS not needed for the decision, but still needed for IEEE exception flags (inexact flag is raised if g OR r OR sticky is nonzero).

### Implementation of Round-to-Nearest-Even

```verilog
// Inputs: mantissa (n bits), guard, round, sticky, sign
// Output: whether to increment mantissa

wire lsb = mantissa[0];  // least significant bit of truncated result

wire round_up = guard & (round | sticky | lsb);
// Explanation:
//   guard=0 → truncate (tail < 0.5 ULP)
//   guard=1, round|sticky=1 → tail > 0.5 ULP → round up
//   guard=1, round=0, sticky=0 → tie → round to even → increment if lsb=1

wire [n-1:0] rounded_mantissa = mantissa + round_up;
```

**Post-normalization:** If the mantissa was 1.111...1 and we round up: result = 10.000...0. Must right-shift by 1 and increment exponent. This is a 1-bit shift (trivial) but must be checked.

---

## Division — SRT with Quotient Digit Selection Table

### Radix-4 SRT Division

Produces 2 quotient bits per iteration. Quotient digits: {-2, -1, 0, +1, +2}.

**Recurrence:**
```
w_{j+1} = 4 * w_j - q_{j+1} * D
```

where w is the partial remainder (initially = Dividend), D is the divisor, and q_{j+1} is the selected quotient digit.

**The digit selection function** maps (w_j, D) to q_{j+1}. For radix-4 SRT with redundancy:

The selection is based on a truncated estimate of w_j (typically top 7-8 bits) and a truncated D (top 3-4 bits). This forms a lookup table (PLA).

### Quotient Digit Selection Table (Radix-4 SRT, partial)

The table entries for specific (w_hat, D_hat) regions:

```
D range: [0.5, 1.0) (normalized divisor)

         D_hat →    0.5    0.625   0.75    0.875
w_hat ↓
 +6/8             +2      +2      +2      +2
 +5/8             +2      +2      +2      +2
 +4/8             +2      +2      +1      +1
 +3/8             +1      +1      +1      +1
 +2/8             +1      +1      +1      +1
 +1/8             +1      +1      +0      +0
  0                0       0       0       0
 -1/8              0       0       0      -1
 -2/8             -1      -1      -1      -1
 -3/8             -1      -1      -1      -1
 -4/8             -1      -2      -2      -2
 -5/8             -2      -2      -2      -2
 -6/8             -2      -2      -2      -2
```

**Key insight:** The regions overlap — multiple valid quotient digits exist for some (w, D) pairs. This redundancy makes the selection function simpler (it can tolerate truncation error in w_hat and D_hat). Without redundancy (non-restoring division), the selection must be exact, requiring more bits of w and D.

### The Pentium FDIV Bug

Intel's Pentium (1994) used radix-4 SRT division. The quotient digit selection table was implemented as a PLA with 1066 entries. Five entries were incorrectly programmed as 0 instead of +2. These entries corresponded to rare combinations of w and D that only occurred for specific operand values.

The bug caused errors starting at the 4th significant decimal digit in approximately 1 in 9 billion random single-precision divisions. It was discovered by a mathematician (Thomas Nicely) computing reciprocals of twin primes.

**Lesson for IC designers:** Memory/PLA content verification is as critical as logic verification. The SRT table should have been formally verified against the mathematical selection function.

### Newton-Raphson — Quadratic Convergence Derivation

**Goal:** Compute 1/B using the iteration X_{i+1} = X_i * (2 - B * X_i).

**Define the error:** e_i = 1/B - X_i (the difference from the true reciprocal).

```
X_{i+1} = X_i * (2 - B * X_i)
        = X_i * (2 - B * X_i)

Substitute X_i = 1/B - e_i:
X_{i+1} = (1/B - e_i) * (2 - B * (1/B - e_i))
        = (1/B - e_i) * (2 - 1 + B*e_i)
        = (1/B - e_i) * (1 + B*e_i)
        = 1/B + B*e_i/B - e_i - B*e_i^2
        = 1/B + e_i - e_i - B*e_i^2
        = 1/B - B*e_i^2

Therefore:
e_{i+1} = 1/B - X_{i+1} = B * e_i^2
```

**This is quadratic convergence:** the error is SQUARED each iteration (times B).

If the initial approximation has k correct bits (|e_0| < 2^{-k}):
```
After iteration 1: |e_1| < B * 2^{-2k} ≈ 2^{-2k+1}  (~2k bits correct)
After iteration 2: |e_2| < 2^{-4k+3}                    (~4k bits correct)
After iteration 3: |e_3| < 2^{-8k+7}                    (~8k bits correct)
```

**Practical convergence:**
| Starting accuracy | After 1 iter | After 2 iter | After 3 iter |
|-------------------|--------------|--------------|--------------|
| 8 bits (LUT)      | ~15 bits     | ~29 bits     | ~57 bits     |
| 10 bits (LUT)     | ~19 bits     | ~37 bits     | ~73 bits     |

For single precision (24 bits): 8-bit LUT + 2 iterations.
For double precision (53 bits): 10-bit LUT + 3 iterations.

Each iteration requires: 1 multiplication (B * X_i), 1 subtraction (2 - result), 1 multiplication (X_i * result). With a pipelined multiplier, this is ~3 multiplier latencies per iteration.

### Goldschmidt vs Newton-Raphson

**Goldschmidt iteration:**
```
N_0 = A, D_0 = B  (numerator, denominator)
F_i = 2 - D_i
N_{i+1} = N_i * F_i
D_{i+1} = D_i * F_i
```

As D → 1, N → A/B.

**Convergence:** Same as N-R (quadratic). Let e_i = 1 - D_i:
```
D_{i+1} = D_i * (2 - D_i) = D_i * (1 + (1 - D_i)) = D_i + D_i*(1-D_i)

1 - D_{i+1} = 1 - D_i*(2-D_i) = 1 - 2*D_i + D_i^2 = (1 - D_i)^2 = e_i^2
```

**Key advantage:** N_{i+1} = N_i * F_i and D_{i+1} = D_i * F_i are INDEPENDENT multiplications. With two multipliers, both can execute in parallel, halving the latency:
```
N-R per iteration: 3 serial multiplications
Goldschmidt per iteration: 1 mul for F + 2 parallel muls (N*F, D*F) = 2 serial muls
```

**Goldschmidt disadvantage:** Requires two multipliers (more area). Also, the final result is not self-correcting — rounding to IEEE precision requires careful analysis of error bounds. N-R can be made self-correcting by computing the residual A - Q*B and correcting.

**In practice:** High-performance FPUs (AMD, IBM) use Goldschmidt. Simpler FPUs (embedded) use N-R or SRT.

---

## Fused Multiply-Add (FMA)

### Why FMA Is Critical for Dot Products

Without FMA, computing a dot product a0*b0 + a1*b1 + ... + a_{n-1}*b_{n-1}:
```
temp0 = round(a0 * b0)        // 1 rounding
temp1 = round(temp0 + a1*b1)  // 2 roundings (1 for mul, 1 for add)... wait
```

Actually, each multiply-add pair incurs 2 roundings. For n terms: 2n roundings. Rounding errors accumulate.

With FMA: `fma(a_i, b_i, running_sum)` rounds only once per term. For n terms: n roundings. Error is reduced by ~sqrt(2) in the worst case.

More importantly, FMA enables **error-free transformations:** using Dekker's algorithm with FMA, you can compute the exact product a*b as (p, e) where p = round(a*b) and e = fma(a, b, -p) gives the exact error. This is the basis for compensated summation algorithms.

### The Wide Adder Problem

In an FMA computing A*B + C:
- A*B produces a product of 2p bits (48 bits for SP, 106 for DP)
- C can have an exponent much larger (or smaller) than A*B
- Alignment of C relative to the product can require shifting by up to 2*E_max positions

The adder in the FMA must handle the full product width PLUS the alignment range:
```
Adder width ≈ 2p + 2 (for product) + p (for C alignment overshoot)
            ≈ 3p + 2 bits

For single precision: ~74 bits
For double precision: ~161 bits
```

This is why FMA is area-expensive — the adder is much wider than a normal FP adder (which only handles p+3 bits).

### FMA for Division and Square Root

**Division via FMA:**
```
1. X_0 = LUT(B)                    // initial approximation of 1/B
2. X_1 = fma(-B, X_0, 2) * X_0    // N-R iteration (single rounding!)
   More precisely:
   r = fma(-B, X_0, 1)             // r = 1 - B*X_0 (exact with FMA)
   X_1 = fma(X_0, r, X_0)         // X_1 = X_0 + X_0*r = X_0*(1+r) ≈ X_0*(2-B*X_0)
3. Repeat until converged
4. Q = fma(A, X_final, 0)          // Q = A * (1/B) = A/B
```

The FMA ensures each N-R iteration loses NO precision from intermediate rounding.

**Square root via FMA:**
```
1. Y_0 = LUT(A)                    // initial approximation of 1/sqrt(A)
2. h = 0.5 * Y_0
   r = fma(-A, Y_0*Y_0, 1)        // r = 1 - A*Y_0^2
   Y_1 = fma(h, r, Y_0)           // Y_1 = Y_0 + 0.5*Y_0*r (N-R for reciprocal sqrt)
3. Repeat
4. sqrt(A) = A * Y_final
```

---

## Exception Handling

### NaN Propagation Rules

IEEE 754 specifies:
1. If ANY input to an operation is NaN, the result is NaN (with some exceptions)
2. If one input is QNaN and one is SNaN, the SNaN takes priority (raises exception, then converts to QNaN)
3. For binary operations with two NaN inputs: result is one of the input NaNs (implementation-defined which one). Common choice: return the first (destination) operand's NaN.
4. Operations that produce NaN from non-NaN inputs (0/0, inf-inf, 0*inf, sqrt(-1)): produce a canonical QNaN with payload = 0.

**NaN payload:** The mantissa bits (excluding the quiet/signaling bit) carry a "payload" — arbitrary data that can identify where the NaN was created. This is useful for debugging numerical code.

### Signed Zero Semantics

IEEE 754 has both +0 and -0. Key rules:
```
+0 == -0  is TRUE
+0 < -0   is FALSE (they compare equal)
1/(+0) = +inf
1/(-0) = -inf
log(+0) = -inf
log(-0) = NaN (negative input to log)

(-0) + (-0) = -0
(-0) + (+0) = +0 (in default rounding mode)
-(+0) = -0
-(-0) = +0
```

**Why signed zero matters:** In complex analysis and branch cuts. For example, `sqrt(-1 + 0i)` and `sqrt(-1 - 0i)` produce different results on different sides of the branch cut. The sign of zero determines which side.

In hardware: the sign logic must handle zero as a special case. After addition, if the result is zero, the sign depends on the rounding mode:
- Round-toward-negative-infinity: zero result is -0 if and only if both operands are -0 or the operation is subtraction of +0 from -0
- All other rounding modes: zero result is +0 unless both operands are -0 or the result is the difference of equal-magnitude, same-sign operands (which gives +0 by convention)

---

## Denormal Handling in Hardware

### The Performance Problem

Denormal numbers have exponent = 0 (minimum) and no implicit leading 1. This creates special cases throughout the FPU pipeline:

1. **No implicit 1:** Mantissa must be treated differently (prepend 0 instead of 1)
2. **Leading zeros:** Denormal mantissa can have many leading zeros, requiring large normalization shifts
3. **Exponent clamping:** After subtraction, if the exponent underflows below E_min, the result must be gradually denormalized (shift mantissa right, clamp exponent to E_min)
4. **Double rounding risk:** In pipelined designs, the denormalization shift and rounding must be done atomically to avoid double rounding

**Common implementation strategies:**
1. **Microcode trap:** Detect denormal input/output, trap to microcode handler. Fast common path, slow (100+ cycles) for denormals. Used in many x86 CPUs (including modern ones for some operations).
2. **Flush-to-zero (FTZ) + denormals-are-zero (DAZ):** Set both denormal results and inputs to zero. Breaks IEEE compliance but avoids all special-case hardware. Used in GPUs and DSPs.
3. **Full hardware support:** Handle denormals entirely in hardware with no performance penalty. Expensive (extra MUXes, wider shifters) but IEEE-compliant. Used in some server CPUs.

---

## Practical FP Addition Example — Full Walkthrough

### Compute 1.5 + 0.125 in IEEE 754 Single Precision

**Step 0: Encode operands**
```
A = 1.5 = 1.1 (binary) = 1.1 * 2^0
  S=0, E_stored = 0+127 = 127 = 0111_1111
  M = 10000...0 (23 bits, implicit leading 1)
  Hex: 0x3FC00000

B = 0.125 = 0.001 (binary) = 1.0 * 2^(-3)
  S=0, E_stored = -3+127 = 124 = 0111_1100
  M = 00000...0 (23 bits, implicit leading 1)
  Hex: 0x3E000000
```

**Step 1: Exponent comparison**
```
d = E_A - E_B = 127 - 124 = 3
A has the larger exponent. No swap needed.
```

**Step 2: Alignment shift**
```
M_A = 1.10000000000000000000000 (1 + 23 bits)
M_B = 1.00000000000000000000000

Right-shift M_B by 3:
M_B_aligned = 0.00100000000000000000000|0|0|0
                                          G R S (all shifted-out bits)
G = 0, R = 0, S = 0
```

**Step 3: Mantissa addition**
```
  1.10000000000000000000000
+ 0.00100000000000000000000
= 1.10100000000000000000000

No overflow (result < 2.0), no leading zeros.
```

**Step 4: Normalization**
```
Result is already in [1, 2) → no shift needed.
E_result = 127 (same as A)
```

**Step 5: Rounding**
```
GRS = 000 → truncate (no rounding needed)
```

**Step 6: Encode result**
```
S=0, E_stored=127=0111_1111, M=10100000000000000000000
Value = 1.101 * 2^0 = 1.625
Hex: 0x3FD00000
```

Verify: 1.5 + 0.125 = 1.625. Correct.

### Trickier Example: 1.0 - 0.9999999 (Near-Cancellation)

This demonstrates the close-path scenario:
```
A = 1.0     = 1.000...0 * 2^0  (E=127)
B = 0.99999 ≈ 1.111...1 * 2^(-1) (E=126)

d = 127 - 126 = 1 (close path, |d| <= 1)
```

Alignment: shift B right by 1:
```
M_A =      1.00000000000000000000000
M_B_aligned = 0.11111111111111111111111|1|0|0  (approximately, for 0.99999...)
```

Subtraction:
```
  1.00000000000000000000000
- 0.11111111111111111111111 1
= 0.00000000000000000000000 1...  (massive cancellation!)
```

The result has many leading zeros → the close-path LZA detects ~23 leading zeros → large left-shift → exponent decremented by ~23 → result is a very small number close to 2^(-24).

This is why the dual-path adder exists — only the close path handles this massive shift.

---

## Interview Q&A — IC Design Depth

### Q1: Why is the bias 127 instead of 128 for single precision?

**A:** The usable exponent range is [1, 254] (0 and 255 are reserved). Bias = 127 gives actual range [-126, +127], which is roughly symmetric and ensures the largest positive exponent (+127 → 2^127 ≈ 1.7e38) and smallest negative exponent (-126 → 2^(-126) ≈ 1.2e-38) cover comparable dynamic range. Bias = 128 would give [-127, +126], biased toward small numbers. The choice also ensures that the reciprocal of the largest representable number is approximately representable (2^(-127) is in the denormal range but close to 2^(-126)).

### Q2: Prove that GRS bits are sufficient for correct rounding.

**A:** For round-to-nearest-even: the decision depends on whether the truncated tail is <, =, or > 0.5 ULP. Guard bit tells us if >= 0.5 ULP (g=1). Round and Sticky together tell us if > 0.5 ULP (g=1 and (r=1 or s=1)). For the exact tie (g=1, r=0, s=0), we check LSB for the even rule. No additional bits can change these decisions — the Sticky bit captures the OR of ALL remaining bits, which is the only information we need (whether they're all zero or not). For directed rounding modes (toward +inf, -inf, zero): only the OR of (g, r, s) matters (are there any non-zero discarded bits?), which GRS fully determines.

### Q3: Explain the dual-path FP adder and why |d| <= 1 is the boundary.

**A:** When |d| >= 2, subtraction can produce at most 1 leading zero. Proof: if E_A - E_B >= 2, then A >= 4B (in terms of the implicit 1.xxx mantissas, A >= 2^2 * B_aligned). A - B >= A - A/4 = 3A/4, which is at least 3/4 — always in [0.5, 2), meaning 0 or 1 leading zeros. When |d| <= 1, A and B are nearly equal, and subtraction can cancel almost completely (up to p leading zeros). The far path (|d| >= 2) only needs a 1-bit normalizer. The close path (|d| <= 1) needs a full barrel shifter and LZA. Running these in parallel and selecting based on d gives better overall timing than a single path.

### Q4: How does the LZA work and why does it have +/-1 error?

**A:** The LZA examines corresponding bit pairs of A and B to predict the leading-zero position of A - B. For each position i, it computes a function f_i based on whether A_i > B_i (generate), A_i = B_i (transfer), or A_i < B_i (zero). The leading 1 of the f vector approximates the leading 1 of |A - B|. The +/-1 error arises because the LZA doesn't compute the actual borrows — there are edge cases where a borrow propagation shifts the leading 1 by one position. The correction is trivial: after the actual subtraction, check the MSB and conditionally shift by 1 more.

### Q5: Compare SRT, Newton-Raphson, and Goldschmidt division.

**A:** SRT (radix-4): digit-recurrence, 1 quotient digit (2 bits) per cycle, linear convergence, ~13 cycles for SP. Hardware: lookup table + adder + shifter. Moderate area. Newton-Raphson: multiplicative, quadratic convergence (bits double each iteration), 2-3 iterations for SP. Needs initial LUT + multiplier. Sequential dependency between iterations. Goldschmidt: also multiplicative/quadratic, but the two multiplications per iteration (N*F and D*F) are independent → parallelizable with 2 multipliers, roughly 2x throughput vs N-R. Both N-R and Goldschmidt are preferred for high-performance FPUs; SRT for area-constrained designs. The Pentium FDIV bug was in the SRT lookup table.

### Q6: Why is FMA important and what is the "wide adder problem"?

**A:** FMA computes round(A*B + C) with a single rounding, improving accuracy over separate multiply-then-add (which has 2 roundings). Hardware challenge: the product A*B is 2p bits wide (not rounded), and C must be aligned to it. The alignment shift can be up to 2*E_max positions, and the adder must be ~3p bits wide (161 bits for double precision). This makes the FMA significantly more expensive than a separate multiplier + adder. FMA is critical for: N-R division/sqrt iterations (no precision loss), dot products (reduced error accumulation), and polynomial evaluation (Horner's method).

### Q7: Explain signed zero and when it matters in hardware.

**A:** IEEE 754 has +0 and -0 which compare equal but have different sign bits. They produce different results under division (1/+0 = +inf, 1/-0 = -inf) and affect branch cuts in complex functions. In FPU hardware, the sign of a zero result must be computed specially: for addition, if the true result is zero, the sign depends on rounding mode (negative zero only if rounding toward -inf and the operands' signs differ, or if both operands are -0). This requires extra logic in the sign computation path. For multiplication and division, sign follows the XOR rule even for zero results.

### Q8: What is flush-to-zero (FTZ) and when is it acceptable?

**A:** FTZ replaces denormal results with zero, and DAZ (denormals-are-zero) treats denormal inputs as zero. This eliminates all denormal special-case handling, simplifying the FPU. It's non-IEEE-compliant but acceptable in: GPUs (graphics doesn't need denormal precision), DSP (signal values far from zero), ML inference (model accuracy barely affected). It's NOT acceptable in: scientific computing, financial calculations, or any IEEE-754-compliant implementation. Most x86 CPUs support both modes — IEEE-compliant by default, FTZ+DAZ via MXCSR register bits for performance-sensitive code.

### Q9: Walk through the hardware cost of each FP addition pipeline stage.

**A:** Stage 1 (exponent compare + swap): 8-bit subtractor (~50 gates), 2×24-bit MUXes (~100 gates), total ~200 gates. Stage 2 (alignment): 24-bit barrel shifter (~600 gates), sticky OR tree (~25 gates), total ~650 gates. Stage 3 (mantissa add): 28-bit adder (including GRS, ~400 gates). Stage 4 (normalize): 24-bit LZC (~200 gates), 24-bit barrel shifter (~600 gates), 8-bit exponent adder (~50 gates), total ~850 gates. Stage 5 (round): 24-bit incrementer (~100 gates), overflow detect (~20 gates), total ~150 gates. TOTAL: ~2250 gates for single-precision FP adder. Double precision roughly 2-3x.

### Q10: What happens when you multiply infinity by zero?

**A:** The result is NaN (quiet NaN). IEEE 754 specifies: inf * 0 = NaN, because the mathematical limit is indeterminate. Similarly: inf - inf = NaN, 0/0 = NaN, inf/inf = NaN. These are the "invalid operation" exceptions. The result is a canonical QNaN (exponent all-1s, MSB of mantissa = 1, rest = 0). In hardware, the exception flag is raised, and if the exception is not masked, the processor traps to an exception handler.

### Q11: How do modern CPUs handle denormals in practice?

**A:** Most x86 CPUs handle denormal inputs in microcode — when a denormal is detected, the hardware traps to a microcode sequence that performs the operation step by step (denormalize, compute, re-normalize). This adds 50-150 cycles of penalty. The detection is done in the initial pipeline stages by checking if the exponent field is 0 and mantissa is nonzero. Intel's Alder Lake (12th gen) and later handle many denormal cases in hardware without microcode traps, reducing the penalty to 0-5 cycles. ARM Cortex-A cores have had full hardware denormal support for many generations. GPUs universally use FTZ for performance.

### Q12: Explain quadratic convergence intuitively.

**A:** In Newton-Raphson division, the error after iteration i is e_{i+1} = B * e_i^2. Squaring the error means: if you have k correct bits, the next iteration gives ~2k correct bits. Starting with 8 correct bits (from a lookup table): iteration 1 gives 16, iteration 2 gives 32, iteration 3 gives 64. This is dramatically faster than linear convergence (which adds a fixed number of bits per step). The cost: each iteration requires a full-width multiplication, so the per-iteration cost is high. But the number of iterations is O(log(p)) where p is the target precision — only 2-3 iterations for practical FP formats.

---

## IEEE 754 Special Cases Deep Dive

### Subnormal (Denormalized) Numbers

**Why subnormals exist:** Without subnormals, the smallest positive normal number is 2^(-126) for single precision. Numbers smaller than this would flush to zero, creating a large gap between 0 and 2^(-126). This gap breaks the fundamental property:

```
a != b  implies  a - b != 0
```

Without subnormals, two distinct small normal numbers could subtract to zero (because the true result falls in the gap). This violates the mathematical expectation that subtraction of unequal values is nonzero.

**Gradual underflow:** Subnormals fill the gap [0, 2^(-126)] with evenly-spaced values:

```
Spacing in subnormal range = 2^(-149) (for single precision)
  = 2^(-126) * 2^(-23) = E_min * epsilon

Subnormal values: k * 2^(-149) for k = 1, 2, ..., 2^23 - 1
  Smallest: 1 * 2^(-149) ≈ 1.4e-45
  Largest:  (2^23 - 1) * 2^(-149) ≈ 1.175e-38

Normal values start at: 2^23 * 2^(-149) = 2^(-126) ≈ 1.175e-38
```

The transition is seamless — the largest subnormal is exactly 1 ULP below the smallest normal.

**Representation details:**

```
Subnormal format: (-1)^S * 0.M * 2^(1 - bias) = (-1)^S * 0.M * 2^(-126)
  - Exponent field is all zeros (stored exponent = 0)
  - No implicit leading 1 (mantissa starts with 0.)
  - Effective exponent is FIXED at E_min = -126 (not -127!)
  - This E_min = 1 - bias choice ensures continuity with normals

Normal format:    (-1)^S * 1.M * 2^(E_stored - bias)
  - Smallest normal: 1.0 * 2^(1 - 127) = 2^(-126)
```

**Hardware performance penalty:**

Subnormals cause problems in every FPU pipeline stage:

1. **Input detection:** Must check if exponent == 0 and mantissa != 0 (subnormal) vs exponent == 0 and mantissa == 0 (zero). This adds a zero-detector and AND gate to the input classification logic.

2. **Leading-zero detection changes:** Normal numbers always have an implicit 1, so the mantissa is 1.xxx. Subnormals have 0.xxx with variable leading zeros. The LZC (leading-zero count) circuit that is normally used only in the close-path subtraction must also handle subnormal inputs.

3. **Pre-normalization:** Before any operation, subnormal inputs must be pre-shifted to align their effective binary point. This requires an extra barrel shifter pass or a wider input shifter.

4. **Post-normalization with exponent clamping:** If the result exponent would go below E_min, the mantissa must be right-shifted and the exponent clamped to E_min. Each right-shift loses 1 bit of precision (gradual underflow). The right-shift amount depends on how far below E_min the exponent went.

5. **Double rounding hazard:** If a result is first rounded to normal precision, then denormalized by right-shifting, the second shift may discard bits that change the rounding decision. The result must be denormalized BEFORE rounding — requiring the FPU to compute extra precision internally.

**Implementation strategies (recap with detail):**

| Strategy | Latency (subnormal) | Area overhead | IEEE compliant |
|----------|---------------------|---------------|----------------|
| Full hardware | 0 extra cycles | +15-25% FPU area | Yes |
| Microcode assist | 50-150 extra cycles | Minimal | Yes |
| Flush-to-zero (FTZ) | 0 extra cycles | Minimal (simpler) | No |
| Hybrid (common cases HW) | 0-5 extra cycles | +10% | Yes |

### NaN Propagation Rules — Quiet NaN vs Signaling NaN

**Quiet NaN (QNaN):**
- Bit pattern: exponent = all 1s, mantissa MSB = 1, rest can be anything (payload)
- Propagates silently through operations without raising exceptions
- Generated by: operations that produce mathematically undefined results from non-NaN inputs (0/0, inf - inf, 0 * inf, sqrt(negative))
- Purpose: allows computation to continue, with the NaN "infecting" all downstream results

**Signaling NaN (SNaN):**
- Bit pattern: exponent = all 1s, mantissa MSB = 0, rest != 0
- Raises the "invalid operation" exception when used as an operand
- After raising the exception, the SNaN is converted to a QNaN (MSB of mantissa set to 1) and the QNaN propagates
- Purpose: used as a sentinel/trap value. Initialize memory with SNaN to detect use of uninitialized data. If the exception is masked, the operation continues with a QNaN.

**Propagation rules (IEEE 754-2008/2019):**

```
Rule 1: If any operand is SNaN → raise invalid exception → return QNaN
  The returned QNaN is typically the SNaN with its quiet bit set.

Rule 2: If any operand is QNaN (and no SNaN) → return QNaN, no exception
  If both operands are QNaN, return one of them (implementation-defined).
  Common convention: return the first operand's NaN (preserves payload).

Rule 3: If no operands are NaN, but the operation is invalid:
  0/0, inf/inf, inf - inf, 0 * inf, sqrt(negative) → return canonical QNaN
  Canonical QNaN: sign=0, exponent=all 1s, mantissa = 10...0

Rule 4: Comparison operations:
  NaN compared with ANYTHING (including itself) returns "unordered"
  NaN == NaN → false
  NaN != NaN → true
  NaN < x → false, NaN > x → false, NaN <= x → false, NaN >= x → false
  
  EXCEPTION: signaling comparisons (e.g., IEEE totalOrder) do not signal
  on QNaN. Only "signaling" comparison predicates (like `<` on SNaN) signal.
```

**Hardware NaN detection circuit:**

```verilog
// NaN detection for single precision
wire is_nan  = (exponent == 8'hFF) && (mantissa != 23'b0);
wire is_qnan = is_nan && mantissa[22];     // MSB of mantissa = 1
wire is_snan = is_nan && ~mantissa[22];    // MSB of mantissa = 0

// NaN propagation logic for binary operation (op A, B)
wire a_is_snan, b_is_snan, a_is_qnan, b_is_qnan;
wire any_snan = a_is_snan | b_is_snan;
wire any_nan  = a_is_snan | b_is_snan | a_is_qnan | b_is_qnan;

// Result selection
wire [31:0] nan_result;
assign nan_result = any_snan ? (a_is_snan ? {a[31], 8'hFF, 1'b1, a[21:0]}   // quieted A
                                          : {b[31], 8'hFF, 1'b1, b[21:0]})   // quieted B
                   : a_is_qnan ? a                                             // propagate A's QNaN
                   : b_is_qnan ? b                                             // propagate B's QNaN
                   : 32'h7FC00000;                                             // canonical QNaN

// Exception flag
wire invalid_op = any_snan | (/* operation-specific invalid conditions */);
```

### Infinity Arithmetic — Complete Rules

IEEE 754 defines precise results for all operations involving infinity:

**Addition and subtraction:**

```
(+inf) + (+inf) = +inf
(-inf) + (-inf) = -inf
(+inf) + (-inf) = NaN  (indeterminate)
(-inf) + (+inf) = NaN  (indeterminate)
(+inf) + x      = +inf  (for any finite x)
(-inf) + x      = -inf  (for any finite x)

(+inf) - (-inf) = +inf
(-inf) - (+inf) = -inf
(+inf) - (+inf) = NaN  (indeterminate)
(-inf) - (-inf) = NaN  (indeterminate)
```

**Multiplication:**

```
(+inf) * (+inf) = +inf
(+inf) * (-inf) = -inf
(-inf) * (-inf) = +inf
(+inf) * x      = +inf  (for finite x > 0)
(+inf) * x      = -inf  (for finite x < 0)
(+inf) * 0      = NaN   (indeterminate: 0 * inf)
(-inf) * 0      = NaN   (indeterminate)
```

**Division:**

```
(+inf) / (+inf) = NaN   (indeterminate: inf/inf)
x / (+inf)      = +0    (for finite x >= 0)
x / (-inf)      = -0    (for finite x >= 0)
x / (+0)        = +inf  (for finite x > 0, division by zero exception)
x / (-0)        = -inf  (for finite x > 0, division by zero exception)
(+0) / (+0)     = NaN   (indeterminate: 0/0)
(+inf) / x      = +inf  (for finite x > 0)
(+inf) / (+0)   = +inf
```

**Other operations:**

```
sqrt(+inf)      = +inf
sqrt(-inf)      = NaN   (invalid)
log(+inf)       = +inf
log(0)          = -inf  (division by zero exception)
log(-x)         = NaN   (invalid, for x > 0)
exp(+inf)       = +inf
exp(-inf)       = +0
```

**Hardware implementation note:** Infinity detection is a simple comparator (exponent == all 1s, mantissa == 0). The result-selection MUX at the output of the FPU must check for infinity inputs BEFORE the main datapath result is selected. The infinity/NaN result override has the highest priority in the result MUX.

### Signed Zero: +0 vs -0

**Complete signed zero rules:**

```
Arithmetic:
  (+0) + (+0)  = +0
  (-0) + (-0)  = -0
  (+0) + (-0)  = +0  (in RNE, round-toward-+inf, round-toward-zero modes)
  (+0) + (-0)  = -0  (in round-toward--inf mode ONLY)
  (+0) * (+0)  = +0
  (+0) * (-0)  = -0  (sign = XOR of operand signs)
  (-0) * (-0)  = +0
  -(+0)        = -0
  -(-0)        = +0
  abs(-0)      = +0
  
Division:
  1 / (+0)  = +inf  (with division-by-zero exception)
  1 / (-0)  = -inf  (with division-by-zero exception)
  -1 / (+0) = -inf
  -1 / (-0) = +inf
  (+0) / x  = +0  (for finite x > 0)
  (-0) / x  = -0  (for finite x > 0)

Comparison:
  (+0) == (-0)  → TRUE
  (+0) < (-0)   → FALSE
  (+0) > (-0)   → FALSE
```

**When signed zero matters in practice:**

1. **Complex arithmetic:** `sqrt(-1 + 0i)` vs `sqrt(-1 - 0i)` — the sign of the imaginary zero determines which side of the branch cut the result lands on, giving `+i` vs `-i`.

2. **Reciprocal of zero:** Code like `1.0 / x` where `x` may be a zero result from subtraction. The sign of zero determines whether you get +inf or -inf, which can propagate differently through subsequent computations.

3. **Serialization/comparison:** When storing/comparing floating-point values at the bit level (e.g., hash tables, memcmp), +0 and -0 have different bit patterns but compare equal under IEEE rules. This can cause subtle bugs in FP-keyed data structures.

**Hardware cost of signed zero:** The sign-computation logic at the output of an FP adder must special-case the zero result. After the mantissa adder produces zero, the hardware must determine the sign based on:
- Operand signs
- The rounding mode
- Whether this is addition or subtraction

This requires a small truth table (about 8 entries) encoded as combinational logic alongside the normal sign-computation path.

### Comparison Gotchas — NaN and Ordering

**IEEE 754 comparison semantics vs total ordering:**

The standard IEEE comparison returns one of four results: less, equal, greater, or **unordered** (when at least one operand is NaN). This creates non-intuitive behavior:

```
NaN == NaN  → false (unordered)
NaN != NaN  → true  (the ONLY value that is not equal to itself)
NaN < 1.0   → false
NaN > 1.0   → false
NaN <= 1.0  → false
NaN >= 1.0  → false

Consequence: if (x <= y || x > y) is NOT always true!
  When x or y is NaN, both conditions are false.
  
Consequence: sorting algorithms that assume trichotomy (a < b, a == b, or a > b)
  will malfunction if the data contains NaN. Quicksort can infinite-loop.
```

**totalOrder (IEEE 754-2008):** A total ordering that IS reflexive and handles NaN:

```
totalOrder places all values in a single chain:
  -NaN < -inf < -finite < -0 < +0 < +finite < +inf < +NaN
  
Within NaN: ordered by payload, with sign determining position.
  -NaN with larger payload < -NaN with smaller payload
  +NaN with smaller payload < +NaN with larger payload
  
totalOrder(-0, +0) = true (i.e., -0 < +0)
totalOrder(+0, -0) = false

This is useful for deterministic sorting, canonical forms, and serialization.
```

**Hardware comparison implementation:**

```verilog
// IEEE comparison (6 outcomes encoded)
// Returns: eq, lt, gt, unordered
wire a_is_nan = (a_exp == 8'hFF) && (a_mant != 0);
wire b_is_nan = (b_exp == 8'hFF) && (b_mant != 0);
wire unordered = a_is_nan | b_is_nan;

// For equal: must handle +0 == -0
wire both_zero = (a[30:0] == 31'b0) && (b[30:0] == 31'b0);
wire bit_equal = (a == b);
wire ieee_equal = ~unordered & (bit_equal | both_zero);

// For less-than: sign-magnitude comparison
wire a_neg = a[31];
wire b_neg = b[31];
wire mag_a_lt_b = (a[30:0] < b[30:0]);
wire ieee_lt = ~unordered & (
    (a_neg & ~b_neg & ~both_zero) |           // a negative, b positive (and not both zero)
    (a_neg & b_neg & ~mag_a_lt_b & ~bit_equal) | // both negative, |a| > |b|
    (~a_neg & ~b_neg & mag_a_lt_b)              // both positive, |a| < |b|
);
```

---

## Rounding Modes with Numerical Examples

### The Four IEEE 754 Rounding Modes

**Mode 1: Round-to-nearest-even (RNE) — the default "banker's rounding"**

Why "banker's rounding" prevents statistical bias: simple round-half-up always rounds 0.5 upward, introducing a systematic positive bias over many operations. RNE rounds ties to the nearest EVEN value, so half the ties round up and half round down (statistically). Over millions of operations, the accumulated rounding error averages to approximately zero.

```
Examples (decimal analogy):
  2.5 → rounds to 2 (nearest even)
  3.5 → rounds to 4 (nearest even)
  4.5 → rounds to 4 (nearest even)
  5.5 → rounds to 6 (nearest even)
  
  Average rounding direction: 2 up, 2 down → zero bias
```

**Mode 2: Round toward +infinity (ceiling)**

Always rounds toward positive infinity. Non-exact positive results round up; non-exact negative results round toward zero (which is toward +inf).

```
+1.1 → +2    (rounded up)
-1.1 → -1    (rounded toward +inf, i.e., toward zero)
+1.0 → +1    (exact, no rounding)
```

**Mode 3: Round toward -infinity (floor)**

Always rounds toward negative infinity. Non-exact positive results round toward zero (toward -inf); non-exact negative results round down (more negative).

```
+1.1 → +1    (rounded toward -inf, i.e., toward zero)
-1.1 → -2    (rounded down, more negative)
```

**Mode 4: Round toward zero (truncation)**

Always rounds toward zero — simply discard the fractional bits. Positive numbers round down, negative numbers round up (toward zero).

```
+1.9 → +1    (truncated)
-1.9 → -1    (truncated toward zero)
```

**Interval arithmetic application:** By computing the same operation in round-toward-+inf and round-toward--inf modes, you get guaranteed upper and lower bounds on the true result. This is the basis for IEEE 754's support of interval arithmetic in scientific computing.

### Worked Example: 1.0 + 2^(-24) in Single Precision, All 4 Modes

**Setup:**

```
A = 1.0  = 1.00000000000000000000000 * 2^0     (E=127)
B = 2^(-24) = 1.00000000000000000000000 * 2^(-24)  (E=103)

Exponent difference: d = 127 - 103 = 24
```

**Alignment shift (shift B right by 24):**

```
M_A = 1.00000000000000000000000  (24 bits: 1 implicit + 23 stored)
M_B before shift: 1.00000000000000000000000

Shift right by 24:
M_B_aligned = 0.00000000000000000000000 | 1 | 0 | 0
                                           G   R   S

The implicit 1 of B lands exactly in the Guard bit position.
G = 1, R = 0, S = 0
```

**Mantissa addition:**

```
  1.00000000000000000000000   (A)
+ 0.00000000000000000000000   (B aligned, truncated to 24 bits)
= 1.00000000000000000000000

The aligned B contributes 0 to the 24-bit mantissa. All its information is in GRS = 100.
```

**Infinitely precise result:**

```
True result = 1.00000000000000000000000 1  (binary)
                                         ^-- this is the 25th bit (Guard position)
= 1.0 + 2^(-24)  = 1 + 2^(-24)

The representable neighbors are:
  Lower: 1.00000000000000000000000 * 2^0  = 1.0
  Upper: 1.00000000000000000000001 * 2^0  = 1.0 + 2^(-23)

The true result is EXACTLY halfway between them:
  1.0 + 2^(-24) = 1.0 + (2^(-23))/2 = midpoint
```

**Rounding decisions:**

```
GRS = 100 (Guard=1, Round=0, Sticky=0) → exact tie (halfway case)
LSB of truncated mantissa = bit 0 = 0 (even)

Mode 1 — RNE:
  Tie → round to even → LSB is already 0 (even) → truncate (round down)
  Result: 1.00000000000000000000000 * 2^0 = 1.0
  The addition of 2^(-24) is LOST — the result is exactly 1.0.

Mode 2 — Round toward +inf:
  True result > representable result → round up (toward +inf)
  Result: 1.00000000000000000000001 * 2^0 = 1.0 + 2^(-23)
  Error = +2^(-24) (rounded up by half an ULP)

Mode 3 — Round toward -inf:
  True result > truncated result → but rounding toward -inf means truncate (for positive numbers)
  Result: 1.00000000000000000000000 * 2^0 = 1.0
  Same as RNE in this case.

Mode 4 — Round toward zero:
  Truncate (for positive numbers, toward zero = toward -inf)
  Result: 1.00000000000000000000000 * 2^0 = 1.0
  Same as mode 3.
```

**Key insight:** In RNE mode, adding 2^(-24) to 1.0 in single precision gives exactly 1.0 — the addition has no effect! This is because 2^(-24) is exactly 0.5 ULP of 1.0, and the tie-breaking rule rounds to the already-even LSB. However, adding 2^(-24) + 2^(-50) (anything slightly above half ULP) would round UP to 1.0 + 2^(-23).

### Another Worked Example: GRS in Action

**Compute 1.0 + 1.5 * 2^(-24) in single precision (RNE):**

```
A = 1.0 = 1.00000000000000000000000 * 2^0
B = 1.5 * 2^(-24) = 1.10000000000000000000000 * 2^(-24)

Alignment: shift B right by 24:
M_B_aligned = 0.00000000000000000000000 | 1 | 1 | 0
                                           G   R   S

G = 1, R = 1, S = 0

Mantissa sum (24-bit) = 1.00000000000000000000000 (same as before)

Rounding: GRS = 110
  G = 1 → tail >= 0.5 ULP
  R = 1 → tail > 0.5 ULP (not a tie!)
  → Round UP unconditionally

Result: 1.00000000000000000000001 * 2^0 = 1.0 + 2^(-23)
```

---

## FP Division Algorithms — Detailed

### SRT Division: Radix-2 and Radix-4

**Radix-2 SRT Division:**

The simplest form of SRT (Sweeney, Robertson, Tocher) division. Quotient digits are from {-1, 0, +1} — a redundant signed-digit representation.

```
Recurrence: w_{j+1} = 2 * w_j - q_{j+1} * D

Where:
  w_j = partial remainder (initially = dividend N)
  D = divisor (normalized: 0.5 <= D < 1)
  q_{j+1} = quotient digit selected from {-1, 0, +1}
```

Quotient digit selection for radix-2 SRT is trivial — based on the sign and magnitude of the partial remainder:

```
If w_j >= D/2:    q = +1  (subtract D from shifted remainder)
If -D/2 <= w_j < D/2:  q = 0   (no correction, just shift)
If w_j < -D/2:   q = -1  (add D to shifted remainder)
```

**Key advantage of redundancy:** The selection of q only depends on a few MSBs of w_j (typically 3-4 bits) because the regions overlap. Without redundancy (non-restoring division, q in {-1, +1}), the selection boundary is exactly at w=0, requiring full-precision comparison.

**Radix-4 SRT Division (as used in Pentium):**

Produces 2 bits per iteration. Quotient digits: {-2, -1, 0, +1, +2}.

```
Recurrence: w_{j+1} = 4 * w_j - q_{j+1} * D
```

The quotient digit selection function is more complex — it's a 2D function of (truncated w, truncated D). This is the P-D diagram (partial remainder vs divisor):

```
P-D Diagram for Radix-4 SRT:

  Partial            The diagram shows selection regions.
  Remainder          Overlap regions allow multiple valid digits.
  (w_hat)
    ^
    |  +2  +2  +2  +2
    |  /   /   /   /
    | +1  +1  +1  +1    ← Selection boundaries are
    | /   /   /   /       piecewise-linear functions of D
    | 0   0   0   0
    | \   \   \   \
    | -1  -1  -1  -1
    |  \   \   \   \
    |  -2  -2  -2  -2
    +------------------→ Divisor (D_hat)
       0.5   0.75   1.0
```

**Hardware implementation with carry-save adder:**

The partial remainder w is maintained in carry-save (redundant) form to avoid long carry propagation chains:

```
w = w_sum + w_carry  (two vectors, no carry propagation)

Each iteration:
  1. Truncate w_sum and w_carry to get w_hat (top 7-8 bits)
     w_hat = w_sum[MSBs] + w_carry[MSBs]  (short CPA, ~8 bits)
  2. Look up q_{j+1} in the selection table using (w_hat, D_hat)
  3. Compute: [w_sum_new, w_carry_new] = CSA(4*w_sum, 4*w_carry, -q*D)
     The 4x is a 2-bit left shift (free in hardware)
     -q*D is: 0 (q=0), -D (q=+1), +D (q=-1), -2D (q=+2), +2D (q=-2)
     2D is D left-shifted by 1 (also free)

Critical path per iteration: truncation CPA + table lookup + CSA = very short
Typically 1 clock cycle per iteration at high frequency.
```

**Quotient conversion (on-the-fly):**

The signed-digit quotient {-2,-1,0,+1,+2} must be converted to standard binary. This is done on-the-fly using two registers QP (positive running quotient) and QM (QP minus 1 ULP):

```
On each new digit q:
  If q >= 0: QP_new = QP_old << 2 | q;   QM_new = QP_old << 2 | (q-1)
  If q < 0:  QP_new = QM_old << 2 | (4+q); QM_new = QM_old << 2 | (4+q-1)

At the end: if final remainder >= 0, result = QP, else result = QM.
```

### Goldschmidt Division — Detailed

**Algorithm:**

```
Given: compute Q = N / D, where 0.5 <= D < 1 (normalized)

Step 0: Initial approximation
  F_0 = LUT(D)  ≈ 1/D  (8-12 bits of accuracy from lookup table)

Step 1-k: Iterate
  For each iteration i:
    N_{i+1} = N_i * F_i
    D_{i+1} = D_i * F_i
    F_{i+1} = 2 - D_{i+1}

  As D_i → 1, N_i → N/D = Q.
```

**Quadratic convergence proof:**

```
Let e_i = 1 - D_i  (error in D from the target value 1)

D_{i+1} = D_i * F_i = D_i * (2 - D_i)

e_{i+1} = 1 - D_{i+1} = 1 - D_i * (2 - D_i)
         = 1 - 2*D_i + D_i^2
         = (1 - D_i)^2
         = e_i^2

This is quadratic convergence: the error SQUARES each iteration.
If |e_0| < 2^(-8)  (8-bit initial approximation):
  |e_1| < 2^(-16)   (16 correct bits)
  |e_2| < 2^(-32)   (32 correct bits)
  |e_3| < 2^(-64)   (64 correct bits)
```

**Comparison with Newton-Raphson:**

| Aspect | Newton-Raphson | Goldschmidt |
|--------|---------------|-------------|
| Convergence | Quadratic (same) | Quadratic (same) |
| Operations/iteration | 3 serial multiplies | 1 mul (F) + 2 parallel muls |
| With 2 multipliers | 3 serial muls | 2 serial muls (N*F || D*F) |
| Latency/iteration | 3M | 2M (with 2 multipliers) |
| Total muls (SP, 8-bit LUT) | 6 (2 iterations * 3) | 4-6 (2 iterations, parallelized) |
| Self-correcting? | Yes (compute residual) | No (error analysis needed) |
| Hardware cost | 1 multiplier sufficient | Benefits from 2 multipliers |
| Used in | Simpler FPUs, embedded | IBM POWER, AMD |

**Goldschmidt's advantage** is that within each iteration, the N and D multiplications are independent and can execute simultaneously on two multiplier units. This nearly halves the latency compared to Newton-Raphson (where each multiply depends on the previous).

### Lookup Table Sizing for Initial Approximation

The initial reciprocal approximation comes from a ROM lookup table indexed by the top bits of the divisor:

```
Table parameters:
  Input:  top k bits of D (after normalization, D in [0.5, 1))
  Output: m bits approximating 1/D

Common configurations:
  8-bit input → 8-bit output:  256 entries * 8 bits = 256 bytes
    Accuracy: ~8 correct bits of 1/D
    Sufficient for SP with 2 N-R iterations

  10-bit input → 10-bit output: 1024 entries * 10 bits = 1.25 KB
    Accuracy: ~10 correct bits
    Sufficient for DP with 3 N-R iterations

  8-bit input → 16-bit output:  256 entries * 16 bits = 512 bytes
    Accuracy: ~15+ correct bits (lookup can exceed input precision
    by using fine-grained tabulation + linear interpolation)
    Can reduce DP to 2 iterations

Error bound for k-bit lookup:
  Maximum table error ≈ 2^(-(k+1)) (half an ULP of the table output)
  With linear interpolation between entries: error ≈ 2^(-2k)
```

**Bipartite table optimization:** Instead of a single large table, use two smaller tables whose outputs are added:

```
Table A: indexed by top k1 bits of D → coarse approximation
Table B: indexed by top k1 bits AND next k2 bits → correction term
Result = Table_A[D_high] + Table_B[D_high][D_mid]

Total entries: 2^k1 + 2^(k1+k2) (much less than 2^(k1+k2) for a single table)
Accuracy: comparable to a single (k1+k2)-bit table
```

This technique reduces table area by 50-70% for the same accuracy and is widely used in production FPU designs.

---

## FMA (Fused Multiply-Add) Micro-architecture

### Why FMA Exists

The FMA computes `round(a * b + c)` with a **single rounding** at the end, instead of the two roundings in separate multiply-then-add (`round(round(a*b) + c)`).

**Accuracy benefit:** The single rounding means the result is the correctly-rounded value of the infinitely-precise `a*b + c`. This is critical for:

1. **Dot products:** Each FMA accumulates one term with only 1 rounding error instead of 2. For an n-term dot product, error is reduced from O(n * epsilon^2) to O(n * epsilon).

2. **Polynomial evaluation (Horner's method):** `((a3*x + a2)*x + a1)*x + a0` — each step is an FMA, and single rounding per step minimizes error propagation.

3. **Error-free transformations:** With FMA, the exact rounding error of a multiplication can be computed: if `p = round(a*b)`, then `e = fma(a, b, -p)` gives the exact error such that `a*b = p + e` exactly (no rounding). This is the foundation of compensated algorithms.

4. **Newton-Raphson iterations:** Each iteration `x_{n+1} = x_n * (2 - b*x_n)` can be expressed as `fma(-b, x_n, 2) * x_n` — the FMA avoids intermediate rounding that would slow convergence.

### Hardware Architecture — Pipeline Stages

```
                      a (p bits)    b (p bits)    c (p bits, with exponent)
                      │              │              │
                ┌─────▼──────────────▼─────┐       │
 Stage 1:       │   p x p Multiplier Array │       │ Exponent compare
                │   (partial product tree) │       │ & alignment shift
                │   Result: 2p bits        │       │ calculation
                └──────────┬───────────────┘       │
                           │ (2p-bit product       │
                           │  in carry-save)       │
                ┌──────────▼───────────────────────▼──────┐
 Stage 2:       │  Alignment: shift c by (Ea+Eb-Ec)      │
                │  positions to align with product        │
                │  Adder width: ~3p bits                  │
                │  3-input carry-save adder (product_sum, │
                │  product_carry, aligned_c)              │
                └──────────┬──────────────────────────────┘
                           │
                ┌──────────▼──────────────────────────────┐
 Stage 3:       │  Carry-propagate addition (3p bits)     │
                │  Leading-zero anticipator (parallel)    │
                └──────────┬──────────────────────────────┘
                           │
                ┌──────────▼──────────────────────────────┐
 Stage 4:       │  Normalization shift (up to 2p+1 bits) │
                │  Exponent adjustment                    │
                └──────────┬──────────────────────────────┘
                           │
                ┌──────────▼──────────────────────────────┐
 Stage 5:       │  Rounding (single rounding!)            │
                │  Post-normalization (if round overflow)  │
                │  Special case handling (NaN, inf, zero)  │
                └──────────┬──────────────────────────────┘
                           │
                        result
```

**Key widths (double precision, p = 53):**

```
Multiplier array: 53 x 53 = 106-bit product (in carry-save: two 106-bit vectors)
Alignment shift: c can be shifted by up to 2*E_max ≈ 2048 positions
Adder: ~161 bits wide (106 for product + 53 for c alignment + guard bits)
Normalizer: barrel shifter up to 161 bits
```

**The (p_a + p_b) bit multiplier product:**

For p-bit mantissa inputs (including implicit 1), the exact product is (2p)-bit. No information is lost at this stage — the FMA preserves the full product. In a regular FP multiplier, this product would be immediately rounded to p bits. In the FMA, it feeds directly (unrounded) into the alignment/addition stage.

**Alignment shift of addend c:**

The addend c must be aligned so its binary point matches the product's binary point. The shift amount is:

```
shift = (E_a + E_b - bias) - E_c + p

Where E_a + E_b - bias is the product's exponent.
Shift can be from -(2p) to +(2*E_max):
  - Large positive shift: c is much smaller than product → shift c right (lose precision of c)
  - Large negative shift: c is much larger than product → product is effectively zero
  - Small shift: both contribute significantly → this is the hard case
```

### Latency and Critical Path Analysis

Typical FMA pipeline: **4-5 stages**, total latency **4-5 cycles** at 1-2 GHz in modern process nodes.

```
Stage 1 (Multiply): Booth-encoded partial product generation + Wallace tree reduction
  Latency: ~1.5 ns (in 7nm)
  This is the same as a standalone multiplier's critical path

Stage 2 (Align + 3:2 CSA): Barrel shifter for c alignment + carry-save addition
  Latency: ~0.8 ns
  The barrel shifter for alignment is on the critical path

Stage 3 (CPA + LZA): Full carry-propagate adder (161 bits!) + LZA in parallel
  Latency: ~1.0 ns
  The wide CPA is the dominant delay (161-bit Kogge-Stone or similar)

Stage 4 (Normalize): Wide barrel shifter
  Latency: ~0.8 ns

Stage 5 (Round): Incrementer + special-case MUX
  Latency: ~0.5 ns
```

**Critical path:** The overall critical path often goes through the wide CPA in stage 3, which is significantly wider than in a standalone adder. This is why FMA latency is typically 1-2 cycles longer than a standalone multiply.

**Throughput:** With full pipelining, the FMA can accept a new operation every cycle (throughput = 1 FMA/cycle), even though latency is 4-5 cycles.

### FMA for Division and Square Root — Detailed

**Division using FMA (Newton-Raphson):**

```
Goal: Q = A / B

Step 0: x0 = LUT(B)           // ~8-10 bit approximation of 1/B
Step 1: e0 = fma(-B, x0, 1.0) // e0 = 1 - B*x0 (exact with FMA!)
        x1 = fma(x0, e0, x0)  // x1 = x0 + x0*e0 = x0*(1+e0) ≈ 1/B
Step 2: e1 = fma(-B, x1, 1.0) // e1 = 1 - B*x1
        x2 = fma(x1, e1, x1)  // x2 = x1*(1+e1) ≈ 1/B (more accurate)
Step 3 (for DP): e2 = fma(-B, x2, 1.0)
                 x3 = fma(x2, e2, x2)
Final:  Q = fma(A, x3, 0.0)   // Q = A * (1/B)
        (or: Q = A * x3 with a separate multiply)

Residual check: r = fma(-B, Q, A)  // r = A - B*Q (exact residual!)
  If |r| > 0.5 ULP: Q_corrected = Q + sign(r) * ULP
```

The FMA is crucial here because `fma(-B, x_i, 1.0)` computes `1 - B*x_i` with no intermediate rounding. Without FMA, the multiplication `B*x_i` would be rounded, and the subtraction from 1 might lose significant bits due to cancellation.

**Square root using FMA:**

```
Goal: S = sqrt(A)

Strategy: compute 1/sqrt(A) first, then multiply by A.

Step 0: y0 = LUT(A)              // ~8-bit approximation of 1/sqrt(A)
Step 1: h0 = 0.5 * y0
        r0 = fma(-A, y0*y0, 1.0) // r0 = 1 - A*y0^2
        y1 = fma(h0, r0, y0)     // y1 = y0 + 0.5*y0*r0
Step 2: h1 = 0.5 * y1
        r1 = fma(-A, y1*y1, 1.0)
        y2 = fma(h1, r1, y1)
...
Final: S = fma(A, y_final, 0.0)  // S = A * (1/sqrt(A)) = sqrt(A)

Residual: r = fma(-S, S, A)      // r = A - S^2 (exact residual)
  Correction: S_corrected = fma(r, 0.5*y_final, S)
```

**Latency comparison (double precision):**

```
SRT radix-4 division:    ~27 cycles (53/2 = ~27 iterations)
Newton-Raphson with FMA: ~16-20 cycles (3 iterations * 2 FMA + setup)
Goldschmidt with FMA:    ~14-16 cycles (3 iterations, parallelized)
```

---

## FP in ASIC/FPGA Synthesis

### Synthesizable FP: DesignWare and Custom RTL

**Synopsys DesignWare FP library:**

DesignWare provides parameterizable, synthesizable floating-point operators:

```
DW_fp_add   — FP adder (configurable precision, pipeline stages)
DW_fp_mult  — FP multiplier
DW_fp_div   — FP divider (SRT or N-R selectable)
DW_fp_sqrt  — FP square root
DW_fp_fma   — Fused multiply-add
DW_fp_cmp   — FP comparator
DW_fp_i2flt — Integer to float conversion
DW_fp_flt2i — Float to integer conversion

Parameters:
  sig_width   — significand width (e.g., 23 for SP, 52 for DP)
  exp_width   — exponent width (e.g., 8 for SP, 11 for DP)
  ieee_compliance — 0: no denorm/NaN support (faster), 1: full IEEE
  num_stages  — pipeline depth (higher = faster clock, more latency)
```

**Example instantiation:**

```verilog
DW_fp_add #(
    .sig_width(23),
    .exp_width(8),
    .ieee_compliance(1)
) u_fp_add (
    .a(operand_a),     // [sig_width+exp_width:0] = [31:0]
    .b(operand_b),
    .rnd(3'b000),      // round-to-nearest-even
    .z(result),
    .status(status)    // {inexact, huge, tiny, invalid, zero, inf}
);
```

**Custom RTL considerations:**

When DesignWare is not available (open-source projects, FPGA with non-Synopsys tools), custom FP RTL must be written:

1. **Berkeley HardFloat:** Open-source IEEE 754 FP units in Chisel/Verilog, used in RISC-V cores (BOOM, Rocket). Well-tested, synthesizable, supports recoded FP format for efficient internal representation.

2. **FloPoCo:** Open-source FP core generator. Generates VHDL for arbitrary precision, optimized for FPGA (uses DSP slices for multipliers).

3. **Manual RTL:** Follow the pipeline stages described in this document. Key pitfall: getting rounding correct for all edge cases (denormals, overflow/underflow, exact ties). Extensive testing against a reference (e.g., MPFR library) is mandatory.

### Pipeline Depth vs Frequency Trade-off

The FP unit's pipeline depth directly affects achievable clock frequency and throughput:

```
Single-precision FP adder:
  1 stage (combinational): ~2-4 ns critical path → max 250-500 MHz
  3 stages: ~0.8-1.0 ns → 1.0-1.25 GHz
  5 stages: ~0.5-0.6 ns → 1.6-2.0 GHz

Single-precision FP multiplier:
  1 stage: ~3-5 ns → 200-333 MHz
  3 stages: ~1.0-1.5 ns → 667 MHz - 1.0 GHz
  4-5 stages: ~0.6-0.8 ns → 1.25-1.67 GHz

Double-precision FP multiplier:
  4 stages: ~1.0-1.2 ns → 833 MHz - 1.0 GHz
  6-7 stages: ~0.5-0.7 ns → 1.4-2.0 GHz
```

**Trade-off considerations:**

```
More pipeline stages:
  + Higher clock frequency (shorter critical path)
  + Better throughput (operations per second)
  - More registers → more area and power
  - Higher latency (more cycles from input to output)
  - More complex forwarding/hazard logic in the processor

Fewer pipeline stages:
  + Lower latency
  + Less area (fewer pipeline registers)
  - Lower clock frequency
  - May bottleneck the overall processor pipeline
```

**ASIC vs FPGA:** FPGA designs typically need deeper pipelines because routing delays are much larger than in ASIC. A 3-stage ASIC FP adder at 1 GHz might need 6-8 stages on FPGA to achieve 300 MHz.

### Area Comparison

Approximate gate counts and silicon area for key operations (in a modern 7nm-class process):

```
Operation          | Gate count (approx) | Relative area
-------------------+---------------------+--------------
INT32 add          | ~200                | 1x (baseline)
INT32 multiply     | ~3,000              | 15x
FP32 add           | ~2,500              | 12x
FP32 multiply      | ~8,000              | 40x
FP32 FMA           | ~15,000             | 75x
FP64 add           | ~5,000              | 25x
FP64 multiply      | ~25,000             | 125x
FP64 FMA           | ~50,000             | 250x
```

**FP32 multiply vs INT32 multiply — why 5-8x more area:**

The FP32 multiplier contains:
1. 24x24-bit mantissa multiplier (~4,000 gates) — larger than INT32 because of the implicit 1 bit
2. 8-bit exponent adder (~100 gates)
3. Normalization shifter (~600 gates)
4. Rounding logic (~300 gates)
5. Special-case detection (NaN, inf, zero, denormal) (~500 gates)
6. Sign logic (~50 gates)

The mantissa multiplier alone is ~1.3x the area of an INT32 multiplier (24-bit vs 32-bit mantissa, but the Booth encoding and Wallace tree are similar). The overhead comes from normalization, rounding, and special-case handling.

### AI-Specific Formats: bfloat16, FP16, TF32

Modern AI/ML workloads have driven the creation of reduced-precision FP formats:

```
Format    | Sign | Exponent | Mantissa | Total bits | Dynamic range | Precision
----------+------+----------+----------+------------+---------------+-----------
FP32      | 1    | 8        | 23       | 32         | ±3.4e38       | ~7.2 decimal digits
TF32      | 1    | 8        | 10       | 19         | ±3.4e38       | ~3.6 decimal digits
bfloat16  | 1    | 8        | 7        | 16         | ±3.4e38       | ~2.4 decimal digits
FP16      | 1    | 5        | 10       | 16         | ±6.5e4        | ~3.6 decimal digits
FP8 (E4M3)| 1    | 4        | 3        | 8          | ±240          | ~1.2 decimal digits
FP8 (E5M2)| 1    | 5        | 2        | 8          | ±5.7e4        | ~0.9 decimal digits
```

**bfloat16 (Brain Float 16):**
- Created by Google Brain for ML training
- Same exponent range as FP32 (8-bit exponent) → same dynamic range, no overflow issues during training
- Truncated mantissa (7 bits vs 23) → coarser precision, but sufficient for gradient descent
- Conversion to/from FP32 is trivial: just truncate or zero-extend the lower 16 mantissa bits
- Hardware: bfloat16 multiply is ~4x smaller than FP32 multiply (7x7 vs 24x24 mantissa multiplier)

**TF32 (TensorFloat-32, NVIDIA):**
- 19-bit format: 8-bit exponent + 10-bit mantissa
- Used internally in NVIDIA Ampere/Hopper tensor cores
- Input: read as FP32, truncate mantissa to 10 bits
- Multiply: 10x10 mantissa multiply (vs 24x24 for FP32) → ~5.7x less multiply area
- Accumulate: in FP32 for full precision of the sum
- Transparent to software — looks like FP32 operations but with reduced precision

**FP16 (IEEE half-precision):**
- Limited exponent range (5 bits → max ~65504) → prone to overflow in training
- Good mantissa precision (10 bits) → better than bfloat16 for inference
- Widely used in inference with mixed-precision (FP16 compute, FP32 master weights)

**Hardware savings in AI accelerators:**

```
Multiplier area scales roughly as O(mantissa_bits^2):
  FP32 multiply:    24 * 24 = 576 "multiply units"
  bfloat16 multiply: 8 * 8  = 64  "multiply units"  → ~9x smaller
  FP16 multiply:    11 * 11 = 121 "multiply units"  → ~4.8x smaller
  TF32 multiply:    11 * 11 = 121 "multiply units"  → ~4.8x smaller
  FP8 (E4M3):       4 * 4  = 16  "multiply units"  → ~36x smaller

In a fixed silicon area, you can fit:
  ~9x more bfloat16 MACs than FP32 MACs
  ~36x more FP8 MACs than FP32 MACs
  
This is why AI chips report massive TOPS/TFLOPS for lower precisions.
```

**Format selection guidance:**

| Use case | Recommended format |
|----------|-------------------|
| Training (large models) | bfloat16 compute, FP32 master weights |
| Training (mixed precision) | FP16 compute with loss scaling, FP32 accumulator |
| Inference (high accuracy) | FP16 or INT8 |
| Inference (throughput) | FP8 or INT4 (with calibration) |
| Scientific computing | FP64 (mandatory for convergence) |
| Graphics (shading) | FP16 or FP32 |

---

## Extended Interview Q&A — Advanced Topics

### Q13: Why does FMA give better accuracy than separate multiply-add?

**A:** FMA computes round(a*b + c) — the infinitely-precise product a*b is computed internally at full (2p-bit) width, added to c (after alignment) at full width, and only then is the single final result rounded to p bits. In contrast, separate multiply-then-add computes round(round(a*b) + c) — the product is rounded to p bits first, discarding up to 0.5 ULP of information, and then the addition introduces a second rounding of up to 0.5 ULP. The worst-case error of FMA is 0.5 ULP vs 1.0 ULP for separate ops. More importantly, FMA enables error-free transformations: given p = round(a*b), the error e = fma(a, b, -p) is EXACT — this is impossible without FMA.

### Q14: What is the area overhead of supporting subnormals in hardware?

**A:** Full subnormal support adds approximately 15-25% to the FPU datapath area. The main costs are: (1) a pre-normalization shifter at the input to shift subnormal mantissas to a canonical form (requires LZC + barrel shifter = ~800 gates for SP); (2) a denormalization path at the output that right-shifts the mantissa and clamps the exponent when the result underflows below E_min (~600 gates for SP); (3) wider internal datapaths to accommodate the extra precision needed to avoid double rounding during denormalization; (4) additional control logic for detecting subnormal inputs/outputs and steering the datapath accordingly (~200 gates). The latency impact is typically 0 cycles (if pipelined into existing stages) or 1 extra cycle (if the denormalization path is a separate stage). The area overhead is why many GPU and DSP designs use FTZ/DAZ mode.

### Q15: Explain the Pentium FDIV bug in detail — what went wrong and what's the lesson?

**A:** The Pentium (P5, 1994) implemented radix-4 SRT division. The quotient digit selection table was stored in a PLA with 1066 entries. Five entries (for specific combinations of truncated partial remainder and divisor) were incorrectly omitted — they should have had quotient digit +2 but were left as 0. This happened because the table was generated by a script that used an incorrect loop bound, excluding certain valid (w, D) pairs. The error only manifested when division encountered those specific remainder/divisor combinations, which occurred for approximately 1 in 9 billion random SP divisions. The lesson: (1) formal verification of lookup table contents against mathematical specifications is essential — the table should have been verified by checking that for EVERY entry, the selected quotient digit keeps the partial remainder bounded; (2) manufacturing test coverage must include functional tests for all table entries, not just random vectors; (3) the cost of a recall ($475M for Intel) far exceeds the cost of thorough verification.

### Q16: How does a bipartite lookup table reduce area for reciprocal approximation?

**A:** A direct lookup table for k-bit reciprocal approximation requires 2^k entries. A bipartite table splits the input into high bits (k1) and mid bits (k2) and uses two smaller tables: Table_A (2^k1 entries) stores the coarse approximation, and Table_B (2^(k1+k2) entries, but with narrow output) stores a correction. The result is Table_A[high] + Table_B[high][mid]. The key insight: the correction term has fewer significant bits than the full result (it only represents the difference from the coarse value), so Table_B outputs are narrower. Total storage is roughly 2^k1 * m + 2^(k1+k2) * (m/2), which for typical parameters is 50-70% less than the 2^(k1+k2) * m of a single table. Accuracy is comparable because the linear approximation (Table_A + correction) matches the curvature of 1/x well within small intervals.

### Q17: Compare FP32, bfloat16, and FP16 for neural network training.

**A:** FP32 (23-bit mantissa, 8-bit exponent): baseline accuracy, no overflow/underflow issues, but large area and memory bandwidth. bfloat16 (7-bit mantissa, 8-bit exponent): same dynamic range as FP32 so gradient values don't overflow, but coarse precision (2.4 decimal digits) may cause stagnation in fine-tuning. The truncated mantissa means some small gradient updates are lost, but this acts as implicit regularization. FP16 (10-bit mantissa, 5-bit exponent): better precision than bfloat16 but limited range (max 65504) — gradient values during backprop can easily overflow, requiring "loss scaling" (multiply loss by a large factor, compute gradients, then divide). Memory savings are identical for bfloat16 and FP16 (both 16 bits). Hardware: bfloat16 multiplier is ~2.3x smaller than FP16 (8x8 vs 11x11 mantissa). Industry consensus for LLM training: bfloat16 compute with FP32 accumulation and master weights.

### Q18: What is the "wide adder problem" in FMA and how is it mitigated?

**A:** The FMA must add the full 2p-bit product (unrounded) to the aligned addend c. The alignment shift can be up to 2*E_max positions. The resulting adder must be ~3p bits wide (161 bits for DP). This wide adder has O(log(3p)) delay — about 2x the delay of a normal FP adder's p-bit CPA. Mitigation strategies: (1) Use a fast parallel-prefix adder (Kogge-Stone) for the wide CPA, accepting the area cost. (2) Split the FMA into "close" and "far" paths analogous to a dual-path adder: when the product and addend exponents are similar, use the close path with full-width addition and LZA; when they differ greatly, the addend either dominates (just round c) or is negligible (just round the product), simplifying the adder. (3) Use carry-save representation through more of the pipeline, deferring the wide CPA. (4) Accept the longer critical path and add one more pipeline stage.

### Q19: How do you verify a floating-point unit?

**A:** FP verification requires multiple complementary approaches: (1) **Directed tests:** Cover all special cases — NaN propagation (QNaN/SNaN for both operands), infinity arithmetic (all combinations), signed zero (all sign/rounding-mode combinations), denormal inputs/outputs, exact tie-breaking in all rounding modes, exponent overflow/underflow boundaries. (2) **Random testing against reference:** Generate millions of random FP operands, compute results in hardware RTL simulation, and compare against a software reference (MPFR library gives arbitrary-precision results). Check both the result value and all status flags (inexact, overflow, underflow, invalid, divide-by-zero). (3) **Formal verification:** For the SRT quotient digit selection table, formally prove that every table entry maintains the partial remainder invariant. For rounding logic, prove equivalence to the mathematical rounding specification. (4) **Boundary testing:** Operand pairs near representable boundaries (largest normal, smallest normal, largest denormal, etc.) are most likely to trigger edge cases. (5) **IEEE 754 test suites:** Use standard test vectors (e.g., IBM FP test suite, TestFloat by Berkeley).

### Q20: Explain the concept of ULP (Unit in the Last Place) and why it matters.

**A:** ULP is the weight of the least significant bit of the mantissa for a given FP number. For a normal number x = 1.mmm...m * 2^e, ULP(x) = 2^e * 2^(-p+1) = 2^(e-p+1), where p is the precision (24 for SP, 53 for DP). ULP varies with the magnitude of x — it doubles each time x crosses a power of 2. For x in [1, 2): ULP = 2^(-23) for SP. For x in [2, 4): ULP = 2^(-22). The IEEE 754 guarantee is that correctly-rounded results are within 0.5 ULP of the true result. This means relative error is bounded by 0.5 * 2^(-p+1) = epsilon/2, where epsilon = machine epsilon = 2^(-p+1). ULP is the fundamental measure of FP accuracy — when we say an operation has "1 ULP error," we mean the result is off by at most 1 in the last mantissa bit, relative to the true value's magnitude.

### Q21: Why is Newton-Raphson for division not "self-correcting" like long division?

**A:** In digit-recurrence methods (SRT, long division), each iteration computes a residual (partial remainder) that exactly captures the remaining error. If an early quotient digit is slightly wrong (within the redundancy range), the residual naturally compensates in subsequent iterations — the method is self-correcting. In Newton-Raphson, we compute successive approximations x_i to 1/B, but we never compute the true residual A - Q*B during iteration. The final quotient Q = A * x_final inherits all accumulated rounding errors from the multiply. To get a correctly-rounded result, we must compute the residual AFTER the final iteration: r = fma(-B, Q, A). If |r| indicates Q is not the correctly-rounded value, we adjust by 1 ULP. This "residual + correction" step is mandatory for IEEE compliance and adds 1-2 extra cycles. Goldschmidt has the same issue.

### Q22: What is "catastrophic cancellation" and how does it relate to FP hardware?

**A:** Catastrophic cancellation occurs when subtracting two nearly-equal FP numbers. If A and B agree in their top k bits, then A - B loses those k leading bits, leaving only (p - k) significant bits in the result. Example: A = 1.00000001 and B = 1.00000000 (in some precision) → A - B = 0.00000001, which after normalization becomes 1.0 * 2^(-8), but only 1 significant bit remains. In FPU hardware, this manifests in the close-path of the dual-path adder: when |E_A - E_B| <= 1, subtraction can produce up to p leading zeros, requiring the LZA + full barrel shifter. The hardware cost of handling catastrophic cancellation (the entire close-path datapath) is roughly 40-50% of the total FP adder area. Algorithms that avoid catastrophic cancellation (e.g., Kahan summation, computing b^2 - 4ac as (b-2*sqrt(a*c))*(b+2*sqrt(a*c))) produce more accurate results and also happen to exercise only the far-path of the FPU.

### Q23: What is the role of GRS bits in FMA?

**A:** In a standalone adder, 3 bits (Guard, Round, Sticky) after the rounding point suffice because the alignment shift discards at most a few bits. In the FMA, the situation is more complex: the product is 2p bits wide, the alignment of c can place significant bits far from the rounding point, and the addition/subtraction can cause cancellation that shifts the rounding point. The FMA must internally maintain enough extra bits to compute the correct GRS values AFTER normalization. In practice, the FMA carries the full 3p-bit intermediate result through to normalization, and only then determines the guard, round, and sticky bits relative to the final rounding position. The sticky bit is the OR of ALL bits below the round position — for a 161-bit intermediate result, this can be a 100+ bit OR tree. Despite the wider internal datapath, only 3 bits (GRS) are needed for the final rounding decision — the sufficiency proof is the same as for the standalone adder.

### Q24: Explain the "double rounding" problem with subnormals.

**A:** Double rounding occurs when a result is rounded twice — first to normal precision, then denormalized by right-shifting (which is effectively a second rounding). Example: suppose the true result is 1.100...0100 * 2^(-127) (one below E_min for SP). The correct denormal result requires right-shifting by 1 to get 0.1100...010 * 2^(-126), then rounding. If the hardware first rounds to 24-bit normal precision (getting 1.100...01 * 2^(-127)), then right-shifts to denormalize (getting 0.1100...01 with a lost bit), the lost bit changes the rounding. The second rounding may round differently than a single rounding of the full-precision result. Fix: the FPU must detect potential underflow BEFORE rounding, compute the denormalized mantissa at full internal precision, and round only once. This requires the denormalization shift to be part of the same pipeline stage as rounding, or the internal result must carry enough extra bits to make the denormalization lossless until the single rounding step.

### Q25: What considerations apply when choosing FP precision for an ASIC?

**A:** The choice depends on the application's numerical requirements versus silicon budget: (1) **Dynamic range:** If the algorithm's values span many orders of magnitude, sufficient exponent bits are needed to avoid overflow/underflow. bfloat16 and FP32 both have 8-bit exponents (range ~1e-38 to ~3.4e38). FP16 has only 5-bit exponent (range ~6e-5 to ~65504) — inadequate for many training algorithms without loss scaling. (2) **Precision:** The mantissa width determines how many significant digits are preserved. For iterative algorithms (solvers, optimization), insufficient precision causes convergence failure or error accumulation. Rule of thumb: if the algorithm needs k decimal digits of accuracy, you need at least ceil(k / log10(2)) mantissa bits. (3) **Area and power:** FP multiplier area scales as O(p^2) where p is mantissa width. Power scales similarly. A bfloat16 MAC unit is ~9x smaller and uses ~9x less energy than FP32. (4) **Memory bandwidth:** Smaller formats reduce off-chip bandwidth requirements, which is often the bottleneck in data-intensive workloads (ML training, signal processing). (5) **Mixed precision:** Many designs use different precisions at different stages — e.g., FP8 for matrix multiply, FP32 for accumulation, FP16 for storage. This requires format conversion logic between stages.
