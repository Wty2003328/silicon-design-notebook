# 11 — Condensed Cram Sheet: High-Frequency Recall

The last-mile recall page: the questions and concepts that recur in almost every digital-IC front-end / architecture interview, each with a one- or two-line answer. Weighted toward **logic design, RTL, timing/CDC, and computer architecture** — the topics that dominate real loops. Every row cross-links (→) to the deep page or the full Q&A bank when you need more than a reminder.

Use it the night before, not the month before. If an answer here is the first time you are seeing the idea, follow the link and study it properly.

Companion pages: [08 RTL Coding](08_RTL_Coding_Questions.md) (full SystemVerilog) · [09 Numeric Drills](09_Hardware_Interview_Questions.md) · [10 Real-Company Bank](10_Digital_IC_Frontend_Interview_Bank.md) · [Folder Index](00_Index.md).

---

## 1. Timing & STA — the most-asked domain

| Concept | Snap answer |
|---|---|
| **Setup** | Data must be stable *before* the capture edge. Constraint: `T_clk ≥ t_cq + t_comb(max) + t_setup (+ skew)`. Sets **max frequency**. |
| **Hold** | Data must stay stable *after* the capture edge. Constraint: `t_cq + t_comb(min) ≥ t_hold (+ skew)`. **Independent of clock period.** |
| **Slack** | `slack = required − arrival`. Positive = passes. Setup slack `= T_clk − (t_cq+t_comb+t_setup)`. |
| **Fmax** | `1 / (t_cq + t_comb(max) + t_setup)` on the critical path. |
| **Fix a setup violation** | Shorten logic: pipeline/retime, faster cells, restructure, useful skew — or slow the clock. |
| **Fix a hold violation** | Add delay (buffers) on the short data path, or fix skew. **Slowing the clock does *not* help.** |
| **Why hold is scary** | Cannot be fixed by frequency; a hold failure is a *functional* bug that ships. Caught/fixed in the backend by buffer insertion. |
| **Clock skew** | Skew toward the capture flop *helps* setup but *hurts* hold (and vice-versa). "Useful skew" trades one for the other deliberately. |
| **input_delay / output_delay** | Model the off-chip launch/capture time so the tool budgets the on-chip path correctly. |
| **Critical path** | Longest register-to-register combinational delay; it caps Fmax. |
| **Multicycle path (MCP)** | Signal allowed >1 cycle to settle; relax setup by N cycles, but constrain hold correctly. |
| **False path** | Never sampled / not real (e.g., two async clocks, config regs). `set_false_path`. |
| **Recovery / removal** | The "setup/hold" of *reset deassertion* relative to the clock edge. |
| **OCV / CPPR** | On-chip variation derates paths for local variation; CPPR credits back the *common* clock segment so you don't double-count. |
| **Post-tapeout violation** | Setup: sometimes rescued by lowering frequency/raising voltage. Hold: generally a **respin**. |

Deep dives: [06 STA Questions](06_Signoff_Questions.md) · [10 §2](10_Digital_IC_Frontend_Interview_Bank.md).

---

## 2. Reset — "asked by nearly every company"

| Q | Snap answer |
|---|---|
| **Sync vs async reset** | Async: acts instantly, works with a dead clock, but *release* can violate recovery/removal → metastability. Sync: clean timing, filters glitches, but needs a live clock and adds to the data path. |
| **Best of both** | **Async-assert, sync-deassert.** Assert immediately; release on one clean edge. |
| **The circuit** | Feed `1'b1` through a 2-FF synchronizer whose *async clears* are the raw reset. Assertion is immediate; deassertion walks through both flops so the whole domain leaves reset together. |
| **Can the release still go metastable?** | The first flop can, but the second resolves it — same MTBF argument as any 2-FF synchronizer. |

→ [RTL Methodology §5](../03_Frontend_RTL_and_Verification/01_RTL_Design_Methodology.md) · [Async & CDC §9](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md).

---

## 3. Metastability & CDC

| Concept | Snap answer |
|---|---|
| **Metastability** | A flop sampled with a setup/hold violation hangs near `V_dd/2` for an unbounded time before resolving randomly. Cannot be eliminated — only made *improbable*. |
| **MTBF** | `MTBF = e^(t_r/τ) / (T0 · f_clk · f_data)`. More resolution time `t_r` (extra flops) → exponentially larger MTBF. |
| **2-FF synchronizer** | Two flops in the destination clock; the first may go metastable, the second samples it after ~1 cycle of settle. Timing: the first flop's output must resolve within one destination period. |
| **Single-bit CDC** | 2-FF synchronizer. |
| **Multi-bit CDC** | Never sync bits independently (they can skew). Use **gray code** (one bit changes), a **handshake**, or an **async FIFO**. |
| **Slow → fast** | A 2-FF synchronizer suffices; the fast side can't miss a wide pulse. |
| **Fast → slow** | The pulse may be too narrow to sample. Widen it: **toggle-synchronizer** or a req/ack **handshake**. |
| **No FIFO/handshake allowed** | Stretch the pulse (toggle flop) into the slow domain, then edge-detect. |

→ [Async & CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) · [03 Frontend Questions](03_Frontend_RTL_and_Verification_Questions.md) · [10 §3](10_Digital_IC_Frontend_Interview_Bank.md).

---

## 4. Async FIFO

| Q | Snap answer |
|---|---|
| **Principle** | Dual-port RAM + a write pointer (write clock) and read pointer (read clock); each pointer is gray-coded and synchronized into the other domain. |
| **Why gray code** | Only one bit changes per increment, so the cross-domain sample can only be the old or new value — never a corrupt intermediate. |
| **Empty** | Synchronized read pointer `==` write pointer (in the read domain). |
| **Full** | Write pointer has lapped the read pointer by one. Pointers are 1 bit wider than the address; with **gray** pointers, full = **top two bits differ, lower bits equal** (binary pointers would just invert the MSB). |
| **Depth math** | Writing a burst of `B` at `f_w` while draining at `f_r`: `depth ≥ ⌈B · (1 − f_r/f_w)⌉`. |
| **Gray → power of two?** | Standard binary-reflected gray needs a power-of-two depth; non-power-of-two needs extra pointer-correction logic. |
| **Constraining the pointer sync** | `set_max_delay`/`set_false_path` per the design; the two flops just need to be placed close and timed as a CDC path. |

Full RTL: [08 §6](08_RTL_Coding_Questions.md) · → [Async & CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md).

---

## 5. Clock dividers & gating

| Q | Snap answer |
|---|---|
| **Divide by 2** | Toggle a flop every clock (`q <= ~q`); output is half-frequency, 50% duty. |
| **Divide by even N** | Count to N/2 and toggle. |
| **Divide by 3, 50% duty** | Two counters — one on posedge, one on negedge — OR their outputs to fill the half-cycle; a single-edge ÷3 gives a 33/66 duty. |
| **Programmable N (1..255)** | Loadable down-counter reloading N−1; fractional needs dual-modulus / accumulator (delta-sigma). |
| **How glitches arise** | Combinational logic with unequal input arrival times momentarily produces a wrong value before settling. |
| **Why not gate a clock with a plain AND** | The enable can change mid-high-phase → a runt/partial clock pulse. |
| **ICG cell** | A **latch (transparent on the low phase) that samples the enable, feeding the AND** — the enable can only change while the clock is low, so no glitch. |
| **Glitch-free clock mux** | Cross-coupled enables so the new clock turns on only after the old is proven off (break-before-make), each enable synchronized to its own clock. |
| **On an FPGA** | Use dedicated clock buffers / `BUFGCE`, not logic-gated clocks. |

→ [Clock Division & Switching](../03_Frontend_RTL_and_Verification/04_Clock_Division_and_Switching.md) · full ÷ RTL in [08 §2](08_RTL_Coding_Questions.md).

---

## 6. RTL semantics — the language traps

| Concept | Snap answer |
|---|---|
| **Blocking `=`** | Evaluates and updates immediately, in order. Use in **combinational** (`always_comb`). |
| **Non-blocking `<=`** | RHS sampled now, LHS updated at the end of the time step (all in parallel). Use in **sequential** (`always_ff`). Models real flop behavior. |
| **Mixing them** | Blocking in clocked logic → races/sim-synth mismatch. Rule: `<=` for flops, `=` for comb. |
| **Inferred latch** | A combinational `always` that doesn't assign every output on every path → synthesis infers a latch. Fix: default assignments / full `if-else`. |
| **Latch vs flip-flop** | Latch is level-sensitive (transparent while enabled); flop is edge-triggered. Flops dominate synchronous design. |
| **`wire` vs `reg`** | `reg` is a procedural variable, not necessarily a register; `wire` is a net. In SV prefer `logic`. |
| **`always_ff/comb/latch`** | SV intent keywords that let the tool *check* you built what you meant. |

→ [Data Types & Basics](../03_Frontend_RTL_and_Verification/02_Data_Types_and_Basics.md) · [10 §7](10_Digital_IC_Frontend_Interview_Bank.md).

---

## 7. FSMs

| Q | Snap answer |
|---|---|
| **Moore vs Mealy** | Moore output depends on **state only** (glitch-free, one cycle later). Mealy output depends on **state + input** (faster, but combinational on inputs). |
| **1 / 2 / 3-block style** | 1-block: everything in one clocked `always`. **2-block: state reg + next-state/output comb (preferred).** 3-block: separate registered output. |
| **Next-state logic** | Combinational (`always_comb`), fully specified with a `default`. State register is the only clocked part. |
| **When to use an FSM** | Control that depends on history/sequence — protocols, handshakes, sequencers. |
| **Sequence detector** | Draw states = "how much of the pattern matched so far"; overlapping means a mismatch can restart mid-pattern, not to idle. |
| **One-hot vs binary encoding** | One-hot: fast, more flops, easy decode (FPGA-friendly). Binary: fewer flops (ASIC-friendly). |

Full RTL: [08 §7](08_RTL_Coding_Questions.md) · → [Logic Building Blocks §6](../00_Fundamentals/02_Logic_Building_Blocks.md).

---

## 8. CMOS & power — "asked constantly"

| Concept | Snap answer |
|---|---|
| **Dynamic power** | `P_dyn = α · C · V² · f` (switching) + short-circuit. Voltage is the biggest lever (squared). |
| **Static power** | Leakage: subthreshold + gate + junction. Grows with temperature and thinner oxides; dominates when idle. |
| **Cut dynamic power** | Clock gating (kill `f` on idle logic), lower `V`/`f` (DVFS), reduce activity α, smaller C. |
| **Cut static power** | Power gating (MTCMOS), multi-Vt (high-Vt on non-critical paths), body bias, lower temperature. |
| **Clock gating vs power gating** | Gating the clock stops switching (dynamic); gating the supply stops leakage (static) but loses state → needs retention/isolation. |
| **DVFS** | Scale V and f together with the workload; `P ∝ V²f` and V∝f, so power scales ~cubically with performance. |
| **CMOS inverter** | PMOS pull-up + NMOS pull-down; one conducts at a time → ~zero static current, full-rail swing. |
| **Stack effect** | Series (stacked) transistors leak less than a single device — used to save standby power. |

→ [Power Fundamentals](../02_Power_and_Low_Power/01_Power_Fundamentals.md) · [Power Reduction](../02_Power_and_Low_Power/04_Power_Reduction_Techniques.md) · [CMOS Fundamentals](../00_Fundamentals/01_CMOS_Fundamentals.md).

---

## 9. Computer architecture

| Concept | Snap answer |
|---|---|
| **5-stage pipeline** | IF, ID, EX, MEM, WB. Hazards: **structural, data, control.** |
| **Data hazard fix** | Forwarding/bypass; stall (bubble) only when forwarding can't supply in time (load-use). |
| **Control hazard fix** | Branch prediction + flush on mispredict; delay slots (legacy). |
| **Branch prediction** | 2-bit saturating counters, BHT/BTB, global/local history, gshare, TAGE. Mispredict flushes the pipe (~10–20 cycle penalty). |
| **Cache mapping** | Direct-mapped, set-associative, fully-associative. Index selects set; tag confirms; offset picks the byte. |
| **AMAT** | `hit_time + miss_rate × miss_penalty` (recurse for multi-level). |
| **3 C's of misses** | Compulsory, Capacity, Conflict. |
| **Write-back vs write-through** | WB: write to cache, mark dirty, flush on eviction (less bandwidth). WT: write both (simpler coherence, more traffic). Pair WB with **write-allocate**, WT with no-write-allocate. |
| **Cache coherence** | Keep multiple copies consistent. **MESI**: Modified, Exclusive, Shared, Invalid — snoop or directory. |
| **Virtual memory** | Per-process virtual→physical mapping via page tables; the **MMU** walks them, the **TLB** caches translations. |
| **TLB miss** | Hardware/software page-table walk; on page fault the OS brings the page in. |
| **Ping-pong buffering** | Two buffers: compute on one while DMA fills/drains the other; hides latency. |
| **Amdahl's law** | `speedup = 1 / ((1−p) + p/s)` — the serial fraction caps you. |
| **Little's law** | `L = λ · W` (occupancy = throughput × latency); sizes queues/MSHRs/buffers. |
| **OoO basics** | Rename (kill WAR/WAW) → schedule when operands ready (Tomasulo/RS) → execute → **ROB retires in order** for precise state. |

Deep dives: [Cache Microarchitecture](../01_Architecture_and_PPA/01_CPU_Architecture/04_Cache_Hierarchy/01_Cache_Microarchitecture.md) · [Cache Coherence](../01_Architecture_and_PPA/01_CPU_Architecture/06_Coherence_and_Consistency/01_Cache_Coherence.md) · [TLB & Virtual Memory](../01_Architecture_and_PPA/01_CPU_Architecture/05_Virtual_Memory/01_TLB_and_Virtual_Memory.md) · [01 Arch Questions](01_Architecture_and_PPA_Questions.md).

---

## 10. Number systems & arithmetic

| Concept | Snap answer |
|---|---|
| **Two's complement** | Negate = invert + 1. One zero, asymmetric range `[−2^(n−1), 2^(n−1)−1]`, adds/subtracts with the same hardware. |
| **Signed overflow** | Two same-sign operands producing the opposite sign; equivalently carry-in ≠ carry-out of the MSB. |
| **Ripple-carry adder** | `O(N)` delay — carry chains bit to bit. Area-minimal. |
| **Carry-lookahead / prefix** | Compute generate/propagate to get carries in `O(log N)`; prefix trees (Kogge-Stone, Brent-Kung) trade wires for depth. |
| **Booth recoding** | Encodes runs of 1s to halve multiplier partial products. |
| **Wallace / Dadda** | Carry-save 3:2 compressor trees reduce partial products in `O(log N)` before one final CPA. |
| **IEEE-754 float** | `(−1)^s · 1.mantissa · 2^(exp−bias)`. Specials: ±0, ±∞, NaN, subnormals. |
| **Fixed vs floating** | Fixed: cheap, constant resolution. Float: huge dynamic range, costlier, rounding. |

→ [Adders & Multipliers](../00_Fundamentals/03_Adders_and_Multipliers.md) · [Floating Point](../00_Fundamentals/04_Floating_Point.md) · [09 Numeric Drills](09_Hardware_Interview_Questions.md).

---

## 11. Bus & serial protocols (quick reference)

| Protocol | Snap answer |
|---|---|
| **APB** | Simple, low-bandwidth peripheral bus; 2-cycle access; `PREADY` extends. No bursts, no pipelining. |
| **AHB** | Single-channel, pipelined, bursts; one outstanding transfer; shared read/write. |
| **AXI** | Five independent channels (AW, W, B, AR, R) → concurrent read/write, out-of-order via IDs, **outstanding** transactions, bursts up to 256 (AXI4). |
| **AXI "outstanding"** | Issue new addresses before earlier data returns — hides latency. |
| **Fixed vs round-robin arbiter** | Fixed: lowest index always wins (starvation risk). Round-robin: rotate priority for fairness. |
| **I²C** | 2-wire (SDA/SCL), open-drain, addressed, start/stop + ACK; multi-master with arbitration. |
| **SPI** | 4-wire (SCLK/MOSI/MISO/SS), full-duplex, 4 modes (CPOL/CPHA), fast; one SS per slave. |
| **UART** | Async serial, no clock — start/stop bits + agreed baud; optional parity. |

→ [01 Arch Questions](01_Architecture_and_PPA_Questions.md) · [10 §9–10](10_Digital_IC_Frontend_Interview_Bank.md).

---

## 12. RTL coding canon (condensed)

Minimal skeletons + the idea graders look for. **Full, commented SystemVerilog with testbench notes is in [08 RTL Coding Questions](08_RTL_Coding_Questions.md).**

**Rising-edge detector** — register and compare.

```systemverilog
always_ff @(posedge clk) d <= sig;
assign rising = sig & ~d;   // falling = ~sig & d
```

**Divide-by-2** — toggle.

```systemverilog
always_ff @(posedge clk or negedge rst_n)
  if (!rst_n) clk_div <= 0; else clk_div <= ~clk_div;
```

**Divide-by-3, 50% duty** — combine a posedge and a negedge counter.

```systemverilog
// A single-edge div-by-3 gives 33%/66% duty.
// For 50%: make a div-by-3 on posedge and a half-cycle-shifted
// copy on negedge, then OR them so the output is high 1.5 of 3 cycles.
assign clk_div3 = q_pos | q_neg;
```

**Gray counter** — count in binary, convert.

```systemverilog
assign gray = bin ^ (bin >> 1);     // bin -> gray
// gray -> bin: b[i] = ^(gray >> i)
```

**2-FF synchronizer** — the CDC workhorse.

```systemverilog
always_ff @(posedge dclk) {q2,q1} <= {q1, async_in};
// q2 is the safe, synchronized signal
```

**Debouncer** — synchronize, then require the input stable for N counts before accepting.

**Synchronous FIFO** — one clock; `count` tracks occupancy; `full = count==DEPTH`, `empty = count==0`. Depth ≥ ⌈B·(1 − f_r/f_w)⌉ for a burst B.

**Async FIFO** — gray read/write pointers (1 bit wider than the address), each 2-FF-synced into the other domain; empty = pointers equal, full = top two bits differ with the rest equal. *The canonical CDC question.*

**Sequence detector (overlapping, e.g. `1011`)** — Moore/Mealy FSM whose states track the longest matched prefix; on mismatch fall back to the longest suffix that is still a valid prefix (not always idle).

**Round-robin arbiter** — rotate the priority base by the last grant; mask requests below the pointer, pick lowest, wrap.

```systemverilog
// grant = lowest set bit of (req & ~(mask-1)) rotated by last winner
```

**Priority encoder / leading-zero count** — `req & (~req + 1)` isolates the lowest set bit; parameterized reduction tree for LZC.

**Skid buffer (ready/valid pipeline register)** — one-slot buffer that captures data when downstream stalls, so `ready` can be registered without dropping a beat. Key to full-throughput AXI-style pipelines.

**Pulse cross fast↔slow** — slow→fast: 2-FF sync. fast→slow: toggle-flop in the fast domain, sync the level, edge-detect in the slow domain.

---

## 13. C / bit-tricks (recur in HW interviews)

| Task | One-liner |
|---|---|
| **Is power of two** | `x && !(x & (x-1))` |
| **Isolate lowest set bit** | `x & (-x)` |
| **Clear lowest set bit** | `x & (x-1)` |
| **Count set bits (popcount)** | Brian Kernighan: `while(x){x &= x-1; c++;}` — loops once per set bit. |
| **Count trailing zeros** | popcount of `(x & -x) - 1`, or `__builtin_ctz`. |
| **Reverse bits** | Swap by masks: 1s↔, 2s↔, nibbles↔, bytes↔ (log₂N passes). |
| **Swap without temp** | `a ^= b; b ^= a; a ^= b;` |
| **Endianness swap (32b)** | `((x>>24)&0xFF) | ((x>>8)&0xFF00) | ((x<<8)&0xFF0000) | (x<<24)` |
| **Absolute value, branchless** | `m = x >> 31; (x + m) ^ m` |
| **Detect signed add overflow** | `((a^sum) & (b^sum)) < 0` — both operands differ in sign from the result. |
| **Round up to power of two** | Smear bits down (`x |= x>>1…`) then `+1`. |

→ Worked versions in [09 Numeric Drills](09_Hardware_Interview_Questions.md).

---

## 14. Killer one-liners & gotchas

The traps that separate a clean answer from a shaky one:

- **Hold violations do not care about clock frequency** — you cannot slow your way out of one.
- **Metastability is never "solved," only made improbable** — quote MTBF, add resolution flops.
- **Never synchronize a multi-bit bus bit-by-bit** — gray code, handshake, or async FIFO.
- **Never gate a clock with a bare AND gate** — use an ICG (latch-based) cell.
- **`<=` for sequential, `=` for combinational** — mixing them causes sim/synth mismatch.
- **Fully specify combinational `always` blocks** — a missing branch infers a latch.
- **Async-assert / sync-deassert reset** — the default correct reset strategy.
- **Fast→slow pulse can be missed** — widen it (toggle + edge-detect), don't just 2-FF it.
- **Setup is a speed limit; hold is a functional bug** — different severities, different fixes.
- **Write-back needs dirty bits and eviction flushes** — the coherence cost you must mention.
- **Amdahl caps you at `1/(1−p)`** — the serial fraction, not the parallel part, is the ceiling.
- **Round-robin, not fixed priority, when fairness/starvation matters.**

---

⬅ prev [10 · Real-Company Bank](10_Digital_IC_Frontend_Interview_Bank.md) · [Folder Index](00_Index.md) · [Root Index](../Index.md)
