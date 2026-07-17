# RTL Coding Questions — Classic Problems with Full Solutions

> The whiteboard-RTL canon. Each problem: statement → key insight → SystemVerilog solution → follow-ups interviewers actually ask. Theory backup: [Logic_Building_Blocks](../00_Fundamentals/02_Logic_Building_Blocks.md), [Clock_Division_and_Switching](../03_Frontend_RTL_and_Verification/04_Clock_Division_and_Switching.md), [Async_Design_and_CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) (CDC, async FIFO).

---

## 0. Why this page exists

Hardware interviews almost always include live RTL (Register-Transfer Level). The problem set is remarkably stable — arbiters, FIFOs (first-in first-out), dividers, detectors, CDC (clock domain crossing) cells — because each one tests a specific habit: clean sequential/combinational separation, reset discipline, full-case thinking, and "what does this synthesize to." This page is the drill set: canonical solutions with the reasoning, plus the standard follow-up twists. Write every one of these from memory at least once.

**House rules assumed everywhere below:** synchronous active-low reset shown (swap trivially); `always_ff` for state, `always_comb` for logic; no latches; one driver per signal.

---

## 1. Edge detector

**Q:** Detect a rising edge on slow asynchronous-ish input `din` (already synchronized); output a 1-cycle pulse.

```verilog
module edge_det (input logic clk, rst_n, din,
                 output logic rise, fall, toggle);
  logic d_q;
  always_ff @(posedge clk or negedge rst_n)
    if (!rst_n) d_q <= 1'b0;
    else        d_q <= din;
  assign rise   =  din & ~d_q;
  assign fall   = ~din &  d_q;
  assign toggle =  din ^  d_q;
endmodule
```

**Follow-ups:** If `din` is truly asynchronous → 2-FF (two flip-flop) synchronizer *before* this (and the edge pulse is then 1 cycle of the local clock). Why not `posedge din` as a clock? — gated/data clocks break STA (Static Timing Analysis) and CTS (Clock Tree Synthesis); never clock on data.

---

## 2. Divide-by-2 / divide-by-3 with 50% duty

**Div-2:** `q <= ~q` — trivially 50%.

**Div-3, 50% duty (the classic trap):** counter alone gives 33/66%. Insight: **OR two waveforms offset by half an input period** — one flopped on posedge, one on negedge.

```verilog
module div3_50 (input logic clk, rst_n, output logic clk_o);
  logic [1:0] cnt;
  logic       p_q, n_q;
  always_ff @(posedge clk or negedge rst_n)
    if (!rst_n) cnt <= 0;
    else        cnt <= (cnt == 2) ? 0 : cnt + 1;
  always_ff @(posedge clk or negedge rst_n)
    if (!rst_n) p_q <= 0;
    else        p_q <= (cnt == 0);
  always_ff @(negedge clk or negedge rst_n)
    if (!rst_n) n_q <= 0;
    else        n_q <= (cnt == 0);
  assign clk_o = p_q | n_q;        // high 1.5 input periods of every 3
endmodule
```

**Follow-ups:** generalize to any odd N (same trick: two phases, OR); is `clk_o` a "real" clock? — only via CTS-aware implementation; prefer clock generators/ICG (integrated clock gating) cells in production ([Clock_Division_and_Switching](../03_Frontend_RTL_and_Verification/04_Clock_Division_and_Switching.md) has the full zoo). Even N → counter + toggle at N/2.

---

## 3. Gray code counter

**Insight:** `gray = bin ^ (bin >> 1)`. Keep a binary counter as state; output gray. (Direct gray-state increment is also fine: `bin = gray2bin(gray); bin++; gray = bin2gray(bin)`.)

```verilog
module gray_cnt #(parameter W=4)
  (input logic clk, rst_n, en, output logic [W-1:0] gray);
  logic [W-1:0] bin;
  always_ff @(posedge clk or negedge rst_n)
    if (!rst_n)  bin <= '0;
    else if (en) bin <= bin + 1'b1;
  assign gray = bin ^ (bin >> 1);
endmodule
// gray→bin:  for (i=W-1;i>=0;i--) bin[i] = ^gray[W-1:i];
```

**Why interviewers care:** exactly-one-bit-changes property → safe to synchronize a multi-bit pointer across clock domains (async FIFO, §6). Follow-up: prove the property — incrementing binary flips a suffix `0111→1000`; XOR (exclusive-OR) with shift collapses it to one flip.

---

## 4. Parameterized round-robin arbiter

**Q:** N requesters, grant one per cycle, rotating priority (last granted → lowest priority).

**Insight:** keep a `mask` of "requests at or above the pointer." Grant from masked requests if any, else from unmasked (wrap). Fixed-priority encode both in parallel — no loops in the critical path. The double-`{req,req}` trick:

```verilog
module rr_arb #(parameter N=4)
 (input  logic clk, rst_n,
  input  logic [N-1:0] req,
  output logic [N-1:0] gnt);
  logic [N-1:0] ptr;                       // one-hot: highest priority position
  logic [2*N-1:0] dreq, dgnt;
  // double-vector fixed-priority-from-ptr:
  assign dreq = {req, req};
  assign dgnt = dreq & ~(dreq - {{N{1'b0}}, ptr});  // isolate first 1 at/above ptr
  assign gnt  = dgnt[N-1:0] | dgnt[2*N-1:N];
  always_ff @(posedge clk or negedge rst_n)
    if (!rst_n)      ptr <= 'b1;
    else if (|gnt)   ptr <= {gnt[N-2:0], gnt[N-1]};  // next after winner
endmodule
```

`x & ~(x-p)` isolates the least-significant 1 of `x` at or above one-hot `p` (subtract borrows through the low zeros). **Follow-ups:** fairness definition (work-conserving, bounded waiting ≤ N−1 grants); weighted RR (per-source credit counters); matrix arbiter alternative; how this becomes the SA (switch allocation) stage of a NoC (Network-on-Chip) router ([Network_on_Chip](../01_Architecture_and_PPA/04_Interconnect/02_Network_on_Chip/01_Network_on_Chip.md)).

---

## 5. Synchronous FIFO + depth math

```verilog
module sfifo #(parameter W=32, D=16, A=$clog2(D))
 (input  logic clk, rst_n, push, pop,
  input  logic [W-1:0] din,
  output logic [W-1:0] dout,
  output logic full, empty);
  logic [W-1:0] mem [D];
  logic [A:0]   wp, rp;                    // extra MSB disambiguates full/empty
  assign empty = (wp == rp);
  assign full  = (wp == {~rp[A], rp[A-1:0]});
  always_ff @(posedge clk or negedge rst_n)
    if (!rst_n) begin wp <= '0; rp <= '0; end
    else begin
      if (push && !full) begin mem[wp[A-1:0]] <= din; wp <= wp + 1'b1; end
      if (pop  && !empty) rp <= rp + 1'b1;
    end
  assign dout = mem[rp[A-1:0]];
endmodule
```

**The depth question (always asked):** writer bursts B words per T_w window at f_w; reader drains continuously at f_r. Required depth = max backlog:

$$
D \ge B - \left\lfloor B \cdot \frac{f_r \cdot t_{\text{burst}}}{B} \right\rfloor \quad\text{practically: } D \ge B\left(1 - \frac{f_r}{f_{w,\text{eff}}}\right)
$$

Worked: B=120 writes @ 200 MHz back-to-back, reads @ 80 MHz. Burst time = 120/200 MHz = 600 ns; reads during burst = 600 ns × 80 MHz = 48. Depth ≥ 120 − 48 = **72**. Add margin for read stalls; if the reader can stall S cycles mid-burst, add S×(read rate share). Always state assumptions (alignment of burst start, sustained vs single burst — for repeated bursts require average write rate ≤ read rate or no finite depth works).

---

## 6. Asynchronous FIFO (the canonical CDC question)

**Insight stack:** (1) pointers cross domains → gray-code them; (2) 2-FF synchronize the *gray* pointers; (3) compare synchronized pointer vs local pointer → conservative (pessimistic) full/empty — safe, never wrong-direction; (4) full: synced read-gray equals write-gray with top **two** bits inverted.

```verilog
module afifo #(parameter W=32, A=4)
 (input  logic wclk, wrst_n, push, input  logic [W-1:0] din,  output logic full,
  input  logic rclk, rrst_n, pop,  output logic [W-1:0] dout, output logic empty);
  logic [W-1:0] mem [1<<A];
  logic [A:0] wb, wg, rb, rg;             // binary + gray, N+1 bits
  logic [A:0] wg_r1, wg_r2, rg_w1, rg_w2; // synchronizers
  // write domain
  wire [A:0] wb_n = wb + (push & ~full);
  wire [A:0] wg_n = wb_n ^ (wb_n >> 1);
  always_ff @(posedge wclk or negedge wrst_n)
    if (!wrst_n) {wb,wg} <= '0;
    else begin
      if (push & ~full) mem[wb[A-1:0]] <= din;
      wb <= wb_n;  wg <= wg_n;
    end
  always_ff @(posedge wclk or negedge wrst_n)   // sync read ptr into wclk
    if (!wrst_n) {rg_w2,rg_w1} <= '0;
    else         {rg_w2,rg_w1} <= {rg_w1, rg};
  assign full = (wg_n == {~rg_w2[A:A-1], rg_w2[A-2:0]});
  // read domain (mirror image)
  wire [A:0] rb_n = rb + (pop & ~empty);
  wire [A:0] rg_n = rb_n ^ (rb_n >> 1);
  always_ff @(posedge rclk or negedge rrst_n)
    if (!rrst_n) {rb,rg} <= '0;
    else begin rb <= rb_n; rg <= rg_n; end
  always_ff @(posedge rclk or negedge rrst_n)
    if (!rrst_n) {wg_r2,wg_r1} <= '0;
    else         {wg_r2,wg_r1} <= {wg_r1, wg};
  assign empty = (rg_n == wg_r2);
  assign dout  = mem[rb[A-1:0]];
endmodule
```

**Follow-ups (know all):** why gray (one bit flips → synchronizer sees old or new value, both valid pointers — never a phantom); why pessimism is safe (synced pointer lags → full asserts early, empty asserts early — lose bandwidth, never corrupt); depth must be 2^A for gray wrap; resets must be coordinated (both sides reset before traffic); DFT/STA: set false-path/max-delay (skew) constraints on the gray buses. Full derivation in [Memory](../01_Architecture_and_PPA/03_Memory/04_Memory_Technologies/01_Memory_Arrays_and_Technologies.md) / [Async_Design_and_CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) (CDC, async FIFO).

---

## 7. Sequence detector FSM (e.g., detect `1011`, overlapping)

```verilog
module seq1011 (input logic clk, rst_n, din, output logic hit);
  typedef enum logic [2:0] {S0,S1,S10,S101} st_t;
  st_t st;
  always_ff @(posedge clk or negedge rst_n)
    if (!rst_n) st <= S0;
    else unique case (st)
      S0:   st <= din ? S1   : S0;
      S1:   st <= din ? S1   : S10;
      S10:  st <= din ? S101 : S0;
      S101: st <= din ? S1   : S10;   // overlap: ...1011|011 reuses suffix
    endcase
  assign hit = (st == S101) && din;   // Mealy: fires on the final 1
endmodule
```

**Follow-ups:** Mealy (output on edge into accept, 1 cycle earlier, glitch-prone if outputs decode combinationally off inputs) vs Moore (registered state only, 1 cycle later, clean); overlapping vs not (non-overlapping: accept → S0); derive states = longest proper suffix of seen-input matching a prefix of pattern (KMP (Knuth-Morris-Pratt) failure function — name-drop it); alternative for long patterns: shift register + compare.

---

## 8. Debouncer + 2-FF synchronizer

```verilog
module debounce #(parameter N=16) // ~N clk of stability required
 (input logic clk, rst_n, din_async, output logic q);
  logic s1, s2;                       // 2-FF synchronizer
  logic [$clog2(N):0] cnt;
  always_ff @(posedge clk or negedge rst_n)
    if (!rst_n) {s2,s1} <= '0; else {s2,s1} <= {s1, din_async};
  always_ff @(posedge clk or negedge rst_n)
    if (!rst_n)            begin cnt <= '0; q <= 1'b0; end
    else if (s2 == q)      cnt <= '0;
    else if (cnt == N-1)   begin q <= s2; cnt <= '0; end
    else                   cnt <= cnt + 1'b1;
endmodule
```

**Follow-up — MTBF (Mean Time Between Failures) math** (memorize the form):

$$
\text{MTBF} = \frac{e^{t_r/\tau}}{T_0 \cdot f_{clk} \cdot f_{data}}
$$

Here $\tau$ is the metastability time constant of the flop, $T_0$ and $f_{data}$ characterize the asynchronous input, and $f_{clk}$ is the sampling-clock frequency. Resolution time $t_r$ ≈ one clock period minus setup; adding the second FF adds a full period to $t_r$ → exponential MTBF improvement. That's *why two flops*: not filtering, but metastability resolution time.

---

## 9. Slow-to-fast and fast-to-slow pulse transfer

**Fast→slow pulse (toggle handshake):**

```verilog
// fast domain: convert pulse to level toggle
always_ff @(posedge fclk) if (pulse_f) tgl <= ~tgl;
// slow domain: 2FF sync + edge detect
always_ff @(posedge sclk) {t2,t1} <= {t1,tgl};
always_ff @(posedge sclk) t3 <= t2;
assign pulse_s = t2 ^ t3;
```

**Why:** a 1-cycle fast pulse can fall entirely between slow edges; a toggle *level* cannot be missed. **Constraint:** source pulses must be spaced > 2–3 slow periods, else toggles merge — add a busy/ack handshake (`req` toggle one way, `ack` toggle back) for arbitrary rates. This toggle-handshake is the universal single-event CDC primitive; multi-bit data → async FIFO or req/ack + qualifier ([Async_Design_and_CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md)).

---

## 10. Priority encoder / leading-zero count, parameterized

```verilog
module penc #(parameter W=8)
 (input logic [W-1:0] in, output logic [$clog2(W)-1:0] idx, output logic vld);
  always_comb begin
    idx = '0; vld = 1'b0;
    for (int i = W-1; i >= 0; i--)     // MSB priority
      if (in[i]) begin idx = i[$clog2(W)-1:0]; vld = 1'b1; break; end
  end
endmodule
```

**Follow-ups:** what does the loop synthesize to — a priority chain (O(W) depth from naive mapping; synthesis restructures to ~O(log W) mux tree); isolate-lowest-set-bit one-liner `in & (~in + 1)`; LZC (leading-zero count) via binary partition (check upper half nonzero → bit of result, recurse) for explicitly logarithmic depth — write it if asked for "fast."

---

## 11. Gearbox / width converter (e.g., 8→32 pack, 32→8 unpack)

```verilog
module pack8to32 (input logic clk, rst_n, in_vld, input logic [7:0] in,
                  output logic out_vld, output logic [31:0] out);
  logic [1:0]  cnt;
  logic [23:0] sh;
  always_ff @(posedge clk or negedge rst_n)
    if (!rst_n) begin cnt <= 0; out_vld <= 0; end
    else begin
      out_vld <= 1'b0;
      if (in_vld) begin
        sh <= {in, sh[23:8]};            // little-endian pack: first byte → LSB
        cnt <= cnt + 1'b1;
        if (cnt == 3) begin out <= {in, sh}; out_vld <= 1'b1; end
      end
    end
endmodule
```

**Follow-ups:** endianness (state your convention first — interviewers dock for silent assumptions); backpressure (add `ready`, stall upstream when downstream stalls — becomes a 2-deep skid buffer question); non-integer ratios (e.g., 10→8: accumulating residue, the real "gearbox" — keep a fill counter and barrel-shift).

---

## 12. Skid buffer (ready/valid pipeline register)

**Q:** Register a valid/ready stream for timing without losing throughput.

**Insight:** when you flop `ready`, upstream may send one more beat after you stall → need a 1-deep side buffer ("skid") to catch it.

```verilog
module skid #(parameter W=32)
 (input  logic clk, rst_n,
  input  logic         s_vld,  output logic s_rdy,  input  logic [W-1:0] s_dat,
  output logic         m_vld,  input  logic m_rdy,  output logic [W-1:0] m_dat);
  logic         skid_vld;
  logic [W-1:0] skid_dat;
  assign s_rdy = !skid_vld;                       // registered, cuts s_rdy path
  always_ff @(posedge clk or negedge rst_n)
    if (!rst_n) begin m_vld <= 0; skid_vld <= 0; end
    else begin
      if (s_vld && s_rdy && (!m_vld || m_rdy))      // straight through
        begin m_dat <= s_dat; m_vld <= 1; end
      else if (s_vld && s_rdy)                      // downstream stalled: skid
        begin skid_dat <= s_dat; skid_vld <= 1; end
      else if (m_rdy && skid_vld)                   // drain skid
        begin m_dat <= skid_dat; m_vld <= 1; skid_vld <= 0; end
      else if (m_rdy)
        m_vld <= 0;
    end
endmodule
```

**Why it's asked:** it is the atom of every AXI (Advanced eXtensible Interface) register slice ([AHB_AXI_APB](../01_Architecture_and_PPA/04_Interconnect/01_Protocols/01_AHB_AXI_APB.md)) and elastic pipeline. Follow-up: full-throughput proof (accepts every cycle downstream is ready; skid absorbs exactly the one in-flight beat) and the half-bandwidth naive alternative (deassert ready whenever output valid — 50% duty under stall).

---

## 13. Parallel CRC (concept + method)

**Q:** CRC-8/CRC-32 processing W bits per cycle.

**Method (what they want to hear):** CRC (Cyclic Redundancy Check) is linear over GF(2) (the two-element Galois field): next_state = M·state ⊕ G·data for constant 0/1 matrices. Derive each next-state bit as XOR of current-state bits and data bits — by symbolically unrolling the serial LFSR (Linear-Feedback Shift Register) W steps (script/table, not by hand at the board). Sketch the serial LFSR, state the linearity argument, write 2–3 unrolled equations, mention generator tools. Depth grows ~log(taps×W) XOR levels; wide-W CRC becomes a timing question → pipeline by splitting the message or using two interleaved CRCs combined at the end (CRC of concatenation via matrix powers).

---

## 14. Divisible-by-3 detector (serial, MSB-first)

**Q:** Bits of an unsigned integer arrive serially, **MSB (most significant bit) first**, one per cycle. Assert `out` whenever the value received *so far* is divisible by 3.

**Insight:** appending bit `b` to the low end of a binary number means `value' = 2·value + b`, so the remainder mod 3 follows `rem' = (2·rem + b) mod 3`. Only the remainder matters → 3 states `{0,1,2}`. Moore output `out = (rem == 0)`. (General rule: **divisible-by-N** needs exactly N states; this is unrelated to the divide-by-3 *clock* problem in §2, which is about waveforms, not arithmetic.)

Transition table (derive each entry from `(2·rem + b) mod 3`):

| state (rem) | b=0 → | b=1 → |
|-------------|-------|-------|
| S0 (0)      | S0    | S1    |
| S1 (1)      | S2    | S0    |
| S2 (2)      | S1    | S2    |

```verilog
module div3_detect (input logic clk, rst_n, vld, b, output logic out);
  typedef enum logic [1:0] {S0, S1, S2} state_t;
  state_t st, nxt;
  always_comb unique case (st)
    S0:      nxt = b ? S1 : S0;
    S1:      nxt = b ? S0 : S2;
    S2:      nxt = b ? S2 : S1;
    default: nxt = S0;
  endcase
  always_ff @(posedge clk or negedge rst_n)
    if (!rst_n)    st <= S0;          // empty string ≡ 0, divisible by 3
    else if (vld)  st <= nxt;
  assign out = (st == S0);            // Moore: value-so-far ≡ 0 (mod 3)
endmodule
```

**Follow-ups:** generalize to divisible-by-N (N states, $rem' = (2\cdot rem+b) \bmod N$); LSB-first (least-significant-bit-first) instead needs tracking `2^k mod N` (powers cycle) — harder, usually you reverse to MSB-first. Mealy vs Moore: Moore output (shown) is glitch-free and registered-clean.

---

## 15. Pipeline operand-alignment pitfall (chained comparators)

**Q:** Build `min3(A,B,C)` from two pipelined `min(x,y)` units, each with **1-cycle latency**. Naive: stage-1 `m = min(A,B)`, stage-2 `min(m, C)`. What's wrong?

**The bug — data misalignment:** `m` emerges from stage-1 one cycle *after* `A,B` were presented, but `C` is wired straight into stage-2 **undelayed**. So in any given cycle stage-2 compares `m = min(A,B)` computed from the inputs of cycle *t−1* against `C` from cycle *t* — two operands from **different** input vectors → wrong `min3`.

```verilog
cycle:        t          t+1
stage-1 in:   A,B(t)     A,B(t+1)
stage-1 out:  --         m=min(A,B(t))      <- m is 1 cycle late
stage-2 in:               m(from t),  C(t+1)   <- MISMATCH: C should be C(t)
```

**Fix:** delay `C` by one register stage so it lines up with `m` (balance the two paths to equal latency):

```verilog
// stage 1
always_ff @(posedge clk) begin
  m_q <= min(A, B);     // 1-cycle latency
  c_q <= C;             // <-- matching delay line keeps C aligned with m
end
// stage 2
always_ff @(posedge clk) min3_q <= min(m_q, c_q);   // both from the SAME input cycle
```

**General rule:** every operand entering a pipeline stage must originate from the **same source cycle**. Whenever one path through a pipe is deeper than another, insert matching FF delays (a delay line / "skid") on the shorter path so latencies balance. This "comparator data-alignment" point generalizes to any multi-operand pipelined datapath (a tree of adders, MAC (multiply-accumulate) arrays, sorting networks).

---

## 16. The grading rubric behind all of these

What senior interviewers actually score:

1. **Reset and init discipline** — every flop known after reset; no `x` leakage.
2. **Comb/seq separation** — no accidental latches (`always_comb` with complete assignments / default-first idiom).
3. **CDC instincts** — anything crossing domains gets a named technique (2FF / toggle / gray / FIFO), not vibes.
4. **Synthesis awareness** — "this loop is a priority chain," "this compare is W-bit XOR tree," "this divider won't close at 2 GHz."
5. **Stated assumptions** — burst alignment, endianness, backpressure policy, overflow behavior — said out loud before coding.
6. **Verification reflex** — after writing, volunteer the 3 worst-case tests (full+push, empty+pop, simultaneous, wrap, reset-mid-burst).

---

## Cross-references

- Building blocks: [Logic_Building_Blocks](../00_Fundamentals/02_Logic_Building_Blocks.md), [Adders_and_Multipliers](../00_Fundamentals/03_Adders_and_Multipliers.md).
- Deep dives behind problems: [Memory](../01_Architecture_and_PPA/03_Memory/04_Memory_Technologies/01_Memory_Arrays_and_Technologies.md) (FIFOs/SRAM), [Clock_Division_and_Switching](../03_Frontend_RTL_and_Verification/04_Clock_Division_and_Switching.md), [Async_Design_and_CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) (CDC), [Network_on_Chip](../01_Architecture_and_PPA/04_Interconnect/02_Network_on_Chip/01_Network_on_Chip.md) (arbiter in context).
- Companion: [Hardware_Interview_Questions](09_Hardware_Interview_Questions.md) (concept Q&A + timing/power math); the AI-systems counterpart (*System Design Interview*) lives in the companion AI-infra notebook.
