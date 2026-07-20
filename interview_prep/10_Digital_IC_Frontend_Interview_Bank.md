# 10 — Digital-IC Frontend Interview Bank

Real questions compiled from a Chinese digital-IC frontend/verification campus-recruit 面经 (interview-recap) collection — T-Head, Goodix, MediaTek, Huawei/HiSilicon, DJI, OPPO, AMD, Cambricon, Zhaoxin, Innosilicon, GigaDevice, Allwinner and more — organized by topic, one concise answer each.
It complements [08 RTL Coding Questions](08_RTL_Coding_Questions.md) (whiteboard RTL) and [09 Hardware Interview Questions](09_Hardware_Interview_Questions.md) (numeric drills): this bank is the *spoken* short-answer set, every entry cross-linked (→) to the notebook section that develops it in full.

---

## 1. Reset

**Q (asked by nearly every company):** Synchronous vs asynchronous reset — differences, pros and cons.
**A:** An asynchronous reset forces the flops the instant it asserts, independent of the clock (works even with a dead clock), but its *release* is asynchronous and can cause recovery/removal violations and metastability. A synchronous reset is sampled by the clock — clean timing, glitch-filtering, no special static timing analysis (STA) — but needs a running clock and adds load to the data path. Most system-on-chip (SoC) designs use async-assert/sync-deassert to get both. → [RTL_Design_Methodology](../03_Frontend_RTL_and_Verification/01_RTL_Design_Methodology.md) §5.

**Q (HiSilicon):** Async-reset / sync-release — why, and the circuit.
**A:** Tie the async reset to the clear ports of a 2-flip-flop (FF) synchronizer whose data input is `1'b1`: assertion is immediate (asynchronous), but deassertion propagates through both flops so the whole domain leaves reset on one clean clock edge. This prevents different flops exiting reset on different cycles and removes the recovery/removal timing hazard on release. → [Async_Design_and_CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) §9.

**Q (T-Head):** In async-reset/sync-release, can the second flop still go metastable on release?
**A:** The release edge is asynchronous to the clock, so the *first* synchronizer flop can go metastable if release coincides with a clock edge. The second flop exists to give the first roughly one full period to resolve, driving the probability of propagated metastability down to the mean-time-between-failures (MTBF) bound — extremely small, but never exactly zero. → [Async_Design_and_CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) §9.

---

## 2. Setup / Hold and STA

**Q (Goodix):** Define setup and hold, and how they are checked.
**A:** Setup requires data stable for `t_su` *before* the active clock edge; hold requires data stable for `t_h` *after* it. Setup bounds the maximum path delay and sets the frequency ceiling (`t_cq + t_logic + t_su ≤ T + t_skew`); hold bounds the minimum path delay (`t_cq,min + t_logic,min ≥ t_h + t_skew`) and is frequency-independent. → [STA](../06_Signoff/01_STA.md) §3.

**Q (AMD):** What are best-case/worst-case corners and on-chip variation (OCV)?
**A:** The worst-case (slow process, max delay) corner stresses setup; the best-case (fast, min delay) corner stresses hold — you must sign off both. OCV then adds intra-die derating on top of the corner, slowing the launch path and speeding the capture path (or vice-versa) to bound local device mismatch. → [STA](../06_Signoff/01_STA.md) §5.

**Q (MediaTek):** Chip is already taped out and a setup/hold violation is found — fixable?
**A:** A setup fail is frequency-dependent, so you may salvage the part by lowering the clock (or raising voltage for faster cells) if the spec allows. A hold fail is frequency-*independent* — slowing the clock does nothing — so it needs a metal-layer engineering change order (ECO) or a respin; hold fails are silicon-fatal. → [STA](../06_Signoff/01_STA.md) §3.

**Q (Zhaoxin):** How do you fix a hold violation in the backend?
**A:** Add delay on the too-short data path — insert buffers/delay cells or reroute — so `t_cq,min + t_logic,min ≥ t_h + t_skew`. Never slow the clock; hold is frequency-independent, so the fix is always delay insertion on the data, not the clock. → [Physical_Design](../05_Backend_Physical_Design/01_Physical_Design.md) §6.

**Q (Zhaoxin):** How do you fix a setup violation?
**A:** Reduce path delay: upsize or swap to lower-threshold cells, restructure or retime to cut logic depth, improve placement, apply useful skew, or raise supply voltage. Relaxing the clock period also fixes setup. → [STA](../06_Signoff/01_STA.md) §3.

**Q (Innosilicon):** Explain input_delay, output_delay, and critical path.
**A:** `set_input_delay` models how late external logic launches data at an input port; `set_output_delay` models the external setup the output must satisfy; both frame the block's I/O against an external clock. The critical path is the worst-slack timing path — the one that sets the maximum frequency. → [Constraints_SDC](../04_Synthesis/02_Constraints_SDC.md) §3.

**Q (Huawei):** What is STA — path types, OCV, and CPPR?
**A:** STA checks every timing path against setup and hold across corners without input vectors. Path groups are input-to-reg, reg-to-reg, reg-to-output, and input-to-output. OCV derates launch and capture for local variation, and clock-path/reconvergence pessimism removal (CPPR/CRPR) adds back the pessimism that OCV double-counts on the shared common segment of the clock tree. → [STA](../06_Signoff/01_STA.md) §4.

**Q (Innosilicon):** Source vs capture clock?
**A:** The launch (source) clock edge launches data from the start flop; the capture clock edge clocks it into the end flop. Setup/hold are measured between these two edges, and skew = capture arrival − launch arrival. → [STA](../06_Signoff/01_STA.md) §3.

**Q (Cambricon):** Multiple multipliers create a timing problem — where do you place registers (retiming/pipelining)?
**A:** Break the long combinational multiplier chain by inserting pipeline registers so each stage fits the period; retiming then moves registers across logic to balance the stage delays. Place them to equalize stage depth — trading added latency for higher frequency/throughput. → [Synthesis_and_Optimization](../04_Synthesis/01_Synthesis_and_Optimization.md) §6.

**Q (Goodix):** Draw the setup/hold waveform with the delays labeled.
**A:** Draw the clock with its active edge; the data must be valid `t_su` before that edge and stay valid `t_h` after it. The launch-to-data arrow (`t_cq + t_logic`) must land inside `T − t_su`, while the min-delay arrow must exceed `t_h`. → [STA](../06_Signoff/01_STA.md) §3.

**Q (AMD):** Uses of set_false_path beyond async crossings?
**A:** Also static/quasi-static configuration registers, scan/test-only paths, reset distribution, and logically un-sensitizable paths — anything that is never functionally timed in normal operation. It removes the path from analysis entirely, so misapplying it to a real path is a silent silicon bug. → [STA](../06_Signoff/01_STA.md) §6.

**Q (MediaTek):** What is a multicycle path?
**A:** A path the design guarantees is captured only every N cycles (e.g., a slow multiplier result). `set_multicycle_path N -setup` relaxes the setup check to edge N, and you must add `-hold N-1` to pull the hold check back to the launch edge — forgetting the hold move explodes hold fixing. → [Constraints_SDC](../04_Synthesis/02_Constraints_SDC.md) §4.

**Q (GigaDevice):** What is a virtual clock?
**A:** A clock defined with a waveform but no source pin in the design, used purely as the timing reference for `set_input_delay`/`set_output_delay` when the off-chip device's clock is not part of the netlist. → [Constraints_SDC](../04_Synthesis/02_Constraints_SDC.md) §3.

---

## 3. Metastability and CDC

**Q (Allwinner):** Synchronous vs asynchronous clocks?
**A:** Synchronous signals share one clock with a fixed phase relationship, so every path is statically timeable; asynchronous signals come from unrelated clocks whose phase drifts, so any capture can violate setup/hold and go metastable. Every asynchronous boundary needs a synchronizer. → [Async_Design_and_CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) §1.

**Q (GigaDevice):** Why synchronous design?
**A:** A single-clock synchronous methodology makes every path statically timeable, glitch-tolerant (only edge-sampled values matter), and scan-testable. It trades some clock power and latency for closure, verifiability, and predictability versus asynchronous or latch-based styles. → [RTL_Design_Methodology](../03_Frontend_RTL_and_Verification/01_RTL_Design_Methodology.md) §2.

**Q (Allwinner):** Synchronous vs asynchronous circuit — pros and cons?
**A:** Synchronous: easy STA, scan design-for-test (DFT), deterministic — at the cost of clock-tree power and skew management. Self-timed asynchronous handshake logic: no clock tree, potentially lower power and electromagnetic interference — but very hard to design, verify, and support in tools, so industry is overwhelmingly synchronous. → [RTL_Design_Methodology](../03_Frontend_RTL_and_Verification/01_RTL_Design_Methodology.md) §2.

**Q (Cambricon):** Why must metastability be solved, and what are its hazards?
**A:** When data violates a flop's setup/hold, its output can hover between 0 and 1 for an unbounded time; different downstream gates may read it as different values, corrupting state. A synchronizer gives the node time to resolve, pushing the failure probability out to an astronomically large MTBF. → [Async_Design_and_CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) §2.

**Q (OPPO):** Single-bit crossing, slow to fast domain?
**A:** A single control bit going slow→fast needs only a 2-FF synchronizer; the fast clock samples the slow, wide, stable signal and cannot miss it. Latency is the only cost. → [Async_Design_and_CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) §3.

**Q (Huawei):** Single-bit vs multi-bit CDC methods?
**A:** Single-bit control: a 2-FF synchronizer, or a pulse/toggle synchronizer. Multi-bit must never use per-bit synchronizers (bits resolve independently and can form codes that never existed); use gray-coded pointers, a data-hold/MUX scheme with a synchronized qualifier, a req/ack handshake, or an asynchronous FIFO. → [Async_Design_and_CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) §4.

**Q (OPPO):** Why does a two-flop synchronizer work?
**A:** The first flop may go metastable but gets roughly one full destination period to resolve before the second flop samples it, so the second flop sees a settled value. MTBF grows exponentially with the resolution time, so one extra flop buys many orders of magnitude. → [Async_Design_and_CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) §3.

**Q (T-Head):** Timing condition for a 2-FF synchronizer?
**A:** The intra-domain path FF1/Q → FF2/D must meet normal setup/hold in the destination clock, so the two flops are placed close (short wire) to maximize resolution time. The source→FF1 path is a true CDC crossing and is declared false/asynchronous in the constraints. → [Async_Design_and_CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) §3.

**Q (MediaTek):** 2-FF synchronizer vs handshake — when each?
**A:** Use a 2-FF for a single-bit level or pulse where occasional latency is fine. Use a req/ack handshake for multi-bit data that must transfer coherently with guaranteed delivery and backpressure, paying a full round-trip latency per transfer. → [Async_Design_and_CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) §6.

**Q (Innosilicon):** Fast to slow domain — how do you synchronize a pulse?
**A:** A pulse narrower than the slow clock period can be missed. Convert it to a level/toggle in the fast domain, synchronize the toggle with a 2-FF, then edge-detect (XOR of two samples) in the slow domain — or stretch/hold the pulse until it is acknowledged. → [Async_Design_and_CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) §8.

**Q (Goodix):** 3 MHz → 20 MHz, 8-bit bus, no handshake and no async FIFO allowed — how?
**A:** Use a data-hold (MUX-recirculation) scheme: hold the 8-bit bus stable in the 3 MHz domain and synchronize only a 1-bit data-valid/toggle into the 20 MHz domain with a 2-FF; when the synchronized qualifier fires, capture the already-stable bus directly, so the data lines themselves never pass through a synchronizer. Since 20/3 ≈ 6.7, the bus is stable for several fast clocks, so it is safe. → [Async_Design_and_CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) §6.

**Q (T-Head):** Async FIFO 100 MHz write / 10 MHz read — the slow side samples a multi-bit gray pointer; still a problem?
**A:** No. Gray coding changes only one bit per increment, so at any sampling instant the write-pointer bus holds a valid gray value with at most one bit transitioning; the 2-FF resolves each bit to old-or-new, giving a valid — possibly one-count-stale — pointer, never a code that was never emitted. A stale-but-valid pointer only makes empty detection conservative, and the FIFO RAM still holds every written word, so nothing is lost. → [Async_Design_and_CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) §5.

---

## 4. Asynchronous FIFO

**Q (Cambricon):** Async FIFO principle and the internal CDC.
**A:** A dual-clock RAM with a write pointer in the write domain and a read pointer in the read domain; each pointer is gray-coded and 2-FF-synchronized into the opposite domain to compute full/empty. Only the pointers cross domains (safely, via gray code) — the data itself never does. → [Async_Design_and_CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) §7.

**Q (Huawei):** How are empty and full detected?
**A:** Pointers carry one extra most-significant bit (MSB) for wrap detection. Empty (read side): synchronized write-gray equals local read-gray. Full (write side): local write-gray equals the synchronized read-gray with the top two MSBs inverted (the pointer wrapped and caught up). Both flags are conservative — they can be falsely asserted briefly but never falsely de-asserted. → [Async_Design_and_CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) §7.

**Q (GigaDevice):** Why gray code?
**A:** Consecutive gray values differ by exactly one bit, so a pointer sampled mid-transition resolves to either the old or the new value — always legal — avoiding the multi-bit incoherency a binary counter would produce across a clock boundary. → [Async_Design_and_CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) §5.

**Q (Zhaoxin):** Does gray code force FIFO depth to a power of two?
**A:** Effectively yes for a plain binary-reflected gray counter: only a power-of-two count wraps with a single-bit change at the boundary. Non-power-of-two depths break the one-bit-change guarantee at wrap and need special gray encodings, so designs round up to a power of two. → [Async_Design_and_CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) §5.

**Q (MediaTek):** How do you calculate FIFO depth?
**A:** For a write burst of B words at `f_wr` drained continuously at `f_rd`, depth ≥ B·(1 − f_rd/f_wr); add about 4–6 entries for the 2-FF pointer latency, then round up to a power of two. → [Async_Design_and_CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) §7.

**Q (Cambricon):** Read and write two words per clock — what changes?
**A:** Widen the RAM and step the pointers by 2 each clock; keep the intra-group select bit local (it never crosses) and gray-encode/synchronize only the coarse pointer. Keep the depth even so the by-2 pointer wraps cleanly. → [Async_Design_and_CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) §7.

**Q (OPPO):** Turn a bursty input into a steady output stream.
**A:** A FIFO is an elastic buffer: it absorbs the bursty writer and lets the reader drain at a steady rate. Size the depth to the burst length minus what drains during the burst. → [Logic_Building_Blocks](../00_Fundamentals/02_Logic_Building_Blocks.md) §9.

**Q (Allwinner):** What are the read and write pointers?
**A:** Binary counters used to address the RAM, gray-encoded before crossing domains; the write pointer advances on writes, the read pointer on reads, and their synchronized gray-coded comparison yields full/empty. → [Async_Design_and_CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) §7.

**Q (T-Head):** Hand-write the async-FIFO control logic.
**A:** Binary + gray write/read counters, gray registers, a 2-FF synchronizer each way, and the empty/full comparators above; the datapath is a dual-port RAM. See the full worked RTL in the coding bank. → [08 RTL Coding Questions](../interview_prep/08_RTL_Coding_Questions.md).

**Q (DJI, on async constraints):** How do you constrain the pointer-synchronizer paths — and do you actually need `set_max_delay` on them?
**A:** The gray pointer feeding the first synchronizer flop is a true asynchronous crossing, so it must be cut from setup/hold STA with `set_false_path` (or `set_clock_groups -asynchronous`) — that part is mandatory. Teams often *also* add `set_max_delay` as a skew bound so the multi-bit gray bus can't spread by more than about one destination cycle. But on the full-flag path specifically it is usually **not** required: the flags are *conservative* — a late-synchronized read pointer can only make "full" assert slightly early, and you never write when full — so a pointer that arrives a bit late causes only a harmless throughput delay, never a functional error. Hence `set_false_path` is required; the `set_max_delay` there is belt-and-suspenders, not a correctness constraint. → [Async_Design_and_CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) §7, [Constraints_SDC](../04_Synthesis/02_Constraints_SDC.md).

---

## 5. Clock dividers

**Q (Allwinner):** Even, odd, and fractional division?
**A:** Even ÷N: a mod-N counter toggling at N/2 gives 50% duty. Odd ÷N at 50%: OR a posedge-derived and a negedge-derived divided clock, whose half-period offset fills the asymmetry. Fractional: alternate two integer ratios (dual-modulus) for the correct average — inherently jittery, so used inside a phase-locked loop (PLL). → [Clock_Division_and_Switching](../03_Frontend_RTL_and_Verification/04_Clock_Division_and_Switching.md) §4.

**Q (GigaDevice):** Divide-by-3 and how to improve it.
**A:** A mod-3 counter alone gives 33/66% duty; to get 50% duty, OR a posedge ÷3 with a negedge ÷3. To make the ratio programmable, replace the fixed terminal count with a register. → [Clock_Division_and_Switching](../03_Frontend_RTL_and_Verification/04_Clock_Division_and_Switching.md) §4.

**Q (Innosilicon):** Programmable divider, N = 1..255.
**A:** Use an 8-bit down-counter loaded with N−1 to produce a clean divided clock; latch the new N at terminal count (`cnt == 0`) so ratio changes take effect at a period boundary (hitless). N = 1 bypasses the counter and passes the clock through. → [Clock_Division_and_Switching](../03_Frontend_RTL_and_Verification/04_Clock_Division_and_Switching.md) §2.

---

## 6. Clock gating and glitches

**Q (Goodix):** How do glitches arise?
**A:** Combinational races — unequal path delays into a gate (e.g., `a + a'`) produce a transient wrong value before the logic settles. On a clock net that transient becomes a false edge that clocks flops. → [Logic_Building_Blocks](../00_Fundamentals/02_Logic_Building_Blocks.md) §8.

**Q (HiSilicon):** Draw the integrated clock-gating (ICG) circuit.
**A:** An ICG cell is a low-phase-transparent latch on the enable followed by an AND with the clock; the latch captures the enable while the clock is low, so the gated clock cannot glitch when the enable changes near the active edge. → [Clock_Division_and_Switching](../03_Frontend_RTL_and_Verification/04_Clock_Division_and_Switching.md) §8.

**Q (MediaTek):** Why can't you casually gate a clock with an AND gate?
**A:** Raw `clk & en` glitches if the enable changes while the clock is high, injecting a runt edge; and a bare combinational gate on the clock is untimed and untestable. Use a characterized ICG cell instead — glitch-free, scan/test-enable-aware, and modeled for STA. → [Clock_Division_and_Switching](../03_Frontend_RTL_and_Verification/04_Clock_Division_and_Switching.md) §8.

**Q (AMD):** Design a glitch-free clock mux.
**A:** Never a bare MUX — it produces runt pulses when select changes while the clocks differ. Use per-source enable flops synchronized to each clock with cross-coupled feedback, so one source is fully disabled before the other is enabled (break-before-make), guaranteeing no glitch on the output. → [Clock_Division_and_Switching](../03_Frontend_RTL_and_Verification/04_Clock_Division_and_Switching.md) §6.

**Q (Allwinner):** Porting clock gating to an FPGA?
**A:** FPGAs have limited clock resources and dislike gated clocks on general logic; replace ICG gating with clock-enable flops (route the enable to the flop's CE pin) or use a dedicated global clock buffer such as BUFGCE. → [Clock_Division_and_Switching](../03_Frontend_RTL_and_Verification/04_Clock_Division_and_Switching.md) §8.

---

## 7. Blocking vs non-blocking

**Q (asked very frequently):** Blocking vs non-blocking assignments — definition, difference, resulting hardware.
**A:** Blocking (`=`) executes immediately and in order within a block, modeling combinational logic; non-blocking (`<=`) samples all right-hand sides then updates all left-hand sides at the end of the time step, modeling the parallel register update of sequential logic. The rule is `<=` in clocked `always_ff` and `=` in `always_comb`; mixing them causes simulation races and simulation/synthesis mismatch. → [Procedural_Processes_and_IPC](../03_Frontend_RTL_and_Verification/03_Procedural_Processes_and_IPC.md) §3.

---

## 8. Finite state machines

**Q (HiSilicon):** One-, two-, and three-block FSM styles — differences?
**A:** One-block: state, next-state, and outputs in one clocked `always` — compact but outputs are registered and the code tangles. Two-block: a clocked state register plus a combinational next-state/output block. Three-block: separate clocked state, combinational next-state, and separate output logic — the most readable and generally recommended style. → [RTL_Design_Methodology](../03_Frontend_RTL_and_Verification/01_RTL_Design_Methodology.md) §6.

**Q (OPPO):** When do you use an FSM?
**A:** When control is sequential and protocol-driven — behavior depends on history and steps through defined states (handshakes, arbiters, bus controllers). Pure datapath/arithmetic needs no FSM. → [Logic_Building_Blocks](../00_Fundamentals/02_Logic_Building_Blocks.md) §6.

**Q (GigaDevice):** Sequence detector — detect `11010`; also "find the first 1".
**A:** Overlapping sequence detection uses a Mealy or Moore FSM with one state per matched prefix (five states for `11010`), advancing on a matching bit and falling back to the longest still-valid prefix otherwise. "Find the first 1" is not sequential — it is a leading-one detector / priority encoder. → [Logic_Building_Blocks](../00_Fundamentals/02_Logic_Building_Blocks.md) §6.

---

## 9. Bus protocols

**Q (Huawei):** AXI/AHB/APB differences and use cases.
**A:** APB (Advanced Peripheral Bus): one channel, 2-cycle SETUP/ACCESS, no burst or outstanding — for low-bandwidth peripherals and config registers. AHB (Advanced High-performance Bus): a single pipelined shared bus (address/data overlap, ~1 transfer/cycle) — legacy mid-bandwidth. AXI (Advanced eXtensible Interface): five independent channels, bursts, out-of-order by ID — for high-throughput masters (CPU/GPU/DDR). → [AHB_AXI_APB](../01_Architecture_and_PPA/04_SoC_and_Chiplet_Architecture/03_Transaction_Protocols/01_AHB_AXI_APB.md) §6.

**Q (MediaTek):** AXI's five independent channels — why is write independent, and can AHB read and write at once?
**A:** AXI has AW, W, B, AR, and R, separating address from data and read from write so all can proceed concurrently and out of order. AHB cannot: it has one shared, arbitrated address phase with time-multiplexed data, so a master does one read *or* write at a time, never both simultaneously. → [AHB_AXI_APB](../01_Architecture_and_PPA/04_SoC_and_Chiplet_Architecture/03_Transaction_Protocols/01_AHB_AXI_APB.md) §3.

**Q (Goodix):** APB write timing diagram and the default value of PREADY.
**A:** A write is a SETUP cycle (PSEL=1, PENABLE=0, PWRITE=1, PADDR/PWDATA driven) then an ACCESS cycle (PENABLE=1); the slave completes when PREADY=1 and inserts wait states by holding PREADY low. A simple slave that never waits ties PREADY high, so every transfer is exactly two cycles. → [AHB_AXI_APB](../01_Architecture_and_PPA/04_SoC_and_Chiplet_Architecture/03_Transaction_Protocols/01_AHB_AXI_APB.md) §6.

**Q (AMD):** AXI read channels/timing, and the burst length of 255/256.
**A:** A read uses AR (address phase) then R (data beats, with RLAST on the final beat). AxLEN is 8 bits in AXI4, so burst length = AxLEN + 1 = 1..256 beats (AXI3 uses 4 bits, 1..16). → [AHB_AXI_APB](../01_Architecture_and_PPA/04_SoC_and_Chiplet_Architecture/03_Transaction_Protocols/01_AHB_AXI_APB.md) §5.

**Q (T-Head):** What is AXI "outstanding"?
**A:** A master may issue multiple addresses before earlier data returns, tracking each by ID; the depth is sized to the bandwidth-delay product, `N_out = ceil(L/t) + 1` (Little's law), to hide slave latency. → [AHB_AXI_APB](../01_Architecture_and_PPA/04_SoC_and_Chiplet_Architecture/03_Transaction_Protocols/01_AHB_AXI_APB.md) §4.

**Q (Cambricon):** 2 masters, 4 slaves, master ID = 3 bits — how many ID bits does a slave see?
**A:** The fabric prepends `ceil(log2 M)` master-index bits so responses route back, giving slave-side ID = 3 + ceil(log2 2) = 4 bits. The number of slaves does not affect ID width — it drives address decode, not response routing. → [AHB_AXI_APB](../01_Architecture_and_PPA/04_SoC_and_Chiplet_Architecture/03_Transaction_Protocols/01_AHB_AXI_APB.md) §7.

**Q (HiSilicon):** What is an AMBA bridge?
**A:** A protocol converter (e.g., AXI-to-APB) that terminates the high-performance transaction and re-issues it as the simpler protocol's phases behind a valid/ready handshake, letting many low-speed peripherals hang off one bridge. → [AHB_AXI_APB](../01_Architecture_and_PPA/04_SoC_and_Chiplet_Architecture/03_Transaction_Protocols/01_AHB_AXI_APB.md) §8.

**Q (Zhaoxin):** Arbiter schemes — fixed priority vs round-robin.
**A:** Fixed priority always grants the highest-priority requester (simple, but can starve the lowest); round-robin rotates which requester holds top priority for fairness, with a worst-case wait of (k−1)·B cycles for k masters and B-beat bursts. → [AHB_AXI_APB](../01_Architecture_and_PPA/04_SoC_and_Chiplet_Architecture/03_Transaction_Protocols/01_AHB_AXI_APB.md) §7.

---

## 10. Serial peripheral protocols

**Q (Goodix):** Describe the I2C bus protocol.
**A:** Two open-drain wires — SCL (clock) and SDA (data) with pull-ups — shared by multiple masters/slaves. A transfer is START (SDA falls while SCL high), a 7-/10-bit address + read/write bit, a per-byte ACK, data bytes, then STOP; SDA may change only while SCL is low. Standard/fast/fast-plus speeds are 100 k/400 k/1 M, with clock-stretching and wired-AND arbitration. → [AHB_AXI_APB](../01_Architecture_and_PPA/04_SoC_and_Chiplet_Architecture/03_Transaction_Protocols/01_AHB_AXI_APB.md) §13.

**Q (Allwinner):** SPI principle, its four modes, and one-to-many via chip-select.
**A:** SPI (Serial Peripheral Interface) is full-duplex synchronous serial — SCLK, MOSI, MISO, and one active-low slave-select (SS/CS) per slave — shifting a byte out while shifting one in on the same edges. The four modes come from CPOL (idle clock level) × CPHA (sample on leading vs trailing edge). One master fans out to many slaves by giving each its own chip-select. → [AHB_AXI_APB](../01_Architecture_and_PPA/04_SoC_and_Chiplet_Architecture/03_Transaction_Protocols/01_AHB_AXI_APB.md) §13.

**Q (GigaDevice):** How does a UART work?
**A:** A UART (Universal Asynchronous Receiver/Transmitter) is a point-to-point link with no shared clock — both ends preset the same baud rate. A frame is a START bit (0), 5–9 data bits LSB-first, an optional parity bit, and 1–2 STOP bits (1); the receiver oversamples (typically 16×) to find the start edge and sample each bit center, so a baud mismatch beyond ~2–3% corrupts the frame. → [AHB_AXI_APB](../01_Architecture_and_PPA/04_SoC_and_Chiplet_Architecture/03_Transaction_Protocols/01_AHB_AXI_APB.md) §13.

---

## 11. Low power

**Q (asked constantly):** What low-power techniques do you know?
**A:** Clock gating (cut switching activity), data/operand gating, dynamic voltage-and-frequency scaling (DVFS), power gating / multi-threshold CMOS (MTCMOS) to cut idle leakage, multi-threshold (multi-Vt) cells and body bias to trade leakage against speed, and multi-voltage domains. Dynamic power dominates when busy, leakage when idle. → [Power_Reduction_Techniques](../02_Power_and_Low_Power/04_Power_Reduction_Techniques.md) §1.

**Q (OPPO):** Static vs dynamic power — origin and how to reduce each.
**A:** Dynamic power is `α·C·V²·f` switching plus short-circuit current; static power is leakage (subthreshold + gate) that flows whenever the block is powered. Cut dynamic with clock/data gating and lower V/f; cut static with power gating, higher-Vt cells, and body bias. → [Power_Fundamentals](../02_Power_and_Low_Power/01_Power_Fundamentals.md) §2.

**Q (HiSilicon):** How specifically do you cut dynamic power?
**A:** Lower the voltage (quadratic effect) via DVFS, reduce activity α with clock/operand gating and by avoiding redundant toggling, reduce switched capacitance C (smaller cells, bus encoding), and lower frequency where throughput allows. → [Power_Reduction_Techniques](../02_Power_and_Low_Power/04_Power_Reduction_Techniques.md) §2.

---

## 12. CPU architecture

**Q (Huawei):** Pipelining — purpose, drawbacks, hazards.
**A:** Pipelining overlaps instruction stages to raise throughput toward one instruction/cycle without shortening any single instruction; the costs are added latency, pipeline registers, and hazards. The three hazard classes are structural (resource conflict), data (RAW/WAR/WAW dependence), and control (branches). → [CPU_Architecture](../01_Architecture_and_PPA/01_CPU_Architecture/01_Core_Foundations/01_CPU_Architecture.md) §2.

**Q (T-Head):** The classic 5-stage pipeline and its three hazards.
**A:** IF, ID, EX, MEM, WB. Data hazards are handled by forwarding/bypass plus a load-use stall, control hazards by branch prediction and flush, and structural hazards by duplicating resources (e.g., separate instruction and data memory). → [CPU_Architecture](../01_Architecture_and_PPA/01_CPU_Architecture/01_Core_Foundations/01_CPU_Architecture.md) §1.

**Q (AMD):** Single-issue vs 4-wide issue?
**A:** A scalar (1-wide) core retires at most one instruction per cycle; a 4-wide superscalar fetches, decodes, and issues up to four per cycle for higher instructions-per-cycle, at the cost of wide fetch, more rename and register-file ports, and dependency-check logic that grows roughly quadratically with issue width. → [CPU_Architecture](../01_Architecture_and_PPA/01_CPU_Architecture/01_Core_Foundations/01_CPU_Architecture.md) §5.

**Q (T-Head):** RISC-V ISA basics.
**A:** RISC-V is a small fixed-width load/store instruction-set architecture (ISA) base (RV32I/RV64I) with orthogonal optional extensions — M (mul/div), A (atomics), F/D (float), C (compressed), V (vector). It is open, modular, and easy to decode. → [RISC_V_ISA](../01_Architecture_and_PPA/01_CPU_Architecture/01_Core_Foundations/02_RISC_V_ISA.md) §1.

**Q (Cambricon):** Virtual-memory mapping and the MMU.
**A:** The memory management unit (MMU) translates virtual to physical addresses one page at a time via page tables, cached in the translation lookaside buffer (TLB); a TLB miss triggers a hardware or software page walk. Virtual memory provides process isolation, relocation, and demand paging. → [TLB_and_Virtual_Memory](../01_Architecture_and_PPA/01_CPU_Architecture/05_Virtual_Memory/01_TLB_and_Virtual_Memory.md) §5.

**Q (Zhaoxin):** The three cache mapping schemes.
**A:** Direct-mapped (one line per set — cheap, conflict-prone), fully-associative (any line — flexible but expensive content-addressable lookup), and set-associative (N lines per set — the practical middle ground). More ways cut conflict misses at higher hit-path cost. → [Cache_Microarchitecture](../01_Architecture_and_PPA/01_CPU_Architecture/04_Cache_Hierarchy/01_Cache_Microarchitecture.md) §2.

**Q (Huawei):** Cache coherence.
**A:** When several caches hold private copies of a line, a coherence protocol (MESI/MOESI, snoop- or directory-based) enforces the single-writer/multiple-reader invariant — invalidating or updating other copies on a write so all cores see a consistent value. → [Cache_Microarchitecture](../01_Architecture_and_PPA/01_CPU_Architecture/04_Cache_Hierarchy/01_Cache_Microarchitecture.md) §8.

**Q (Zhaoxin):** Write-back vs write-through.
**A:** Write-through updates both the cache and the next level on every store — simple and always coherent, but more traffic. Write-back updates only the cache and marks the line dirty, flushing on eviction — less traffic, but it needs dirty bits and careful coherence handling. → [Cache_Microarchitecture](../01_Architecture_and_PPA/01_CPU_Architecture/04_Cache_Hierarchy/01_Cache_Microarchitecture.md) §4.

**Q (Cambricon):** Ping-pong (double) buffering.
**A:** Two buffers alternate roles — the producer fills one while the consumer drains the other, then they swap — decoupling producer and consumer and hiding latency at the cost of 2× storage. → [CPU_Architecture](../01_Architecture_and_PPA/01_CPU_Architecture/01_Core_Foundations/01_CPU_Architecture.md) §6.

**Q (T-Head):** How does datapath control change across instructions?
**A:** The control unit decodes each instruction into the control signals (ALU op, mux selects, register-write, memory read/write) that steer the shared datapath cycle by cycle; different instructions light up different control-signal patterns over the same hardware. → [CPU_Architecture](../01_Architecture_and_PPA/01_CPU_Architecture/01_Core_Foundations/01_CPU_Architecture.md) §1.

---

## 13. Verification

**Q (asked by every verification role):** What is UVM?
**A:** UVM (Universal Verification Methodology) is a SystemVerilog class library and methodology for building reusable, layered testbenches — driver/monitor/agent/env/scoreboard components, sequences for stimulus, factory and config database for reuse, and phases for ordering. → [UVM_Methodology](../03_Frontend_RTL_and_Verification/10_UVM_Methodology.md) §2.

**Q (MediaTek):** What language do you write the reference model in, and how do you model a FIFO?
**A:** The golden reference model can be SystemVerilog, C/C++ via the direct programming interface (DPI), or SystemC; a FIFO reference model is just a queue — push on write, pop on read, flag full/empty — that the scoreboard compares against the DUT. → [UVM_Methodology](../03_Frontend_RTL_and_Verification/10_UVM_Methodology.md) §2.

**Q (Huawei):** How is functional coverage collected and merged, and what coverage types exist?
**A:** Covergroups sample coverpoints and crosses at defined events; each test's database is merged (unioned) into a cumulative score. Types: code coverage (line/toggle/branch/FSM/condition) — automatic — and functional coverage (covergroups plus assertion cover) — written by hand from the verification plan. → [Assertions_and_Coverage](../03_Frontend_RTL_and_Verification/09_Assertions_and_Coverage.md) §4.

**Q (Cambricon):** Register model — front-door vs back-door access.
**A:** The UVM register abstraction layer (RAL) models registers by name. Front-door access drives real bus transactions (exercises the path but consumes cycles); back-door access reads/writes the DUT storage directly through the hierarchical path (instant, bypasses the bus — ideal for setup and checking). → [UVM_Methodology](../03_Frontend_RTL_and_Verification/10_UVM_Methodology.md) §8.

**Q (AMD):** The UVM phase mechanism and why it exists.
**A:** Phases impose a global order across all components: build (top-down) → connect (bottom-up) → run (time-consuming, concurrent) → extract/check/report; the run phase has twelve sub-phases (reset/config/main/shutdown). The ordering is forced by dependencies — you must build components before you can connect them. → [UVM_Methodology](../03_Frontend_RTL_and_Verification/10_UVM_Methodology.md) §4.

**Q (T-Head):** fork-join vs join_any vs join_none.
**A:** `fork…join` waits for all spawned processes; `join_any` resumes after the first completes (the rest keep running); `join_none` resumes immediately, leaving all children running in the background. → [Procedural_Processes_and_IPC](../03_Frontend_RTL_and_Verification/03_Procedural_Processes_and_IPC.md) §6.

**Q (OPPO):** What is a virtual interface?
**A:** A handle (pointer) to a physical interface instance, letting dynamic class-based testbench components — which cannot bind to static module interfaces directly — drive and sample real DUT pins; it is passed in through the config database. → [UVM_Methodology](../03_Frontend_RTL_and_Verification/10_UVM_Methodology.md) §6.

**Q (Goodix):** What is SVA, and when do you assert?
**A:** SystemVerilog Assertions (SVA) are temporal properties checked continuously. Assert protocol rules and design invariants — handshakes, one-hot, no overflow, X-checks — at interfaces and key internal points, and pair each assertion with a cover to avoid vacuous passes. → [Assertions_and_Coverage](../03_Frontend_RTL_and_Verification/09_Assertions_and_Coverage.md) §2.

**Q (Huawei):** How do you know verification is complete?
**A:** Passing tests alone never means "done" — completeness is measured by the verification plan mapped to coverage: 100% code and functional coverage, all assertions exercised (covers hit), and a flattening bug-discovery rate. → [Verification_Planning_and_Coverage_Closure](../03_Frontend_RTL_and_Verification/11_Verification_Planning_and_Coverage_Closure.md) §5.

**Q (Cambricon):** Stress testing / generating huge numbers of cases.
**A:** Constrained-random generation samples the vast legal input space directed tests cannot cover: randomize transactions under constraints, run many seeds, and let coverage feedback steer new constraints toward the remaining holes. → [OOP_and_Randomization](../03_Frontend_RTL_and_Verification/08_OOP_and_Randomization.md) §2.

**Q (MediaTek):** HLS vs RTL.
**A:** High-level synthesis (HLS) compiles untimed C/C++/SystemC into RTL, exploring schedules and pipelines automatically — faster to write and retarget, but with less control over microarchitecture and quality-of-results than hand RTL; verify HLS output against the C model via equivalence checking. → [Synthesis_and_Optimization](../04_Synthesis/01_Synthesis_and_Optimization.md) §1.

**Q (Zhaoxin):** The VCS flow and its post-compile files.
**A:** VCS (a Synopsys simulator) compiles SystemVerilog/RTL into a `simv` executable in three steps — analyze → elaborate → build `simv`; artifacts include `simv` plus `simv.daidir`, the `csrc/` intermediate C, and logs. Running `simv` simulates and dumps VCD/FSDB waveforms. → [Gate_Level_Sim_and_Emulation](../03_Frontend_RTL_and_Verification/13_Gate_Level_Sim_and_Emulation.md) §8.

**Q (Huawei):** Inputs to gate-level (post-)simulation (GLS).
**A:** GLS needs the synthesized or post-route netlist, the standard-cell simulation models, an SDF (Standard Delay Format) file for back-annotated delays, and the testbench — running with timing to catch X-propagation, reset, and DFT issues that static analysis cannot see. → [Gate_Level_Sim_and_Emulation](../03_Frontend_RTL_and_Verification/13_Gate_Level_Sim_and_Emulation.md) §3.

**Q (AMD):** What is equivalence checking?
**A:** Logic equivalence checking (LEC) formally proves the gate netlist is functionally identical to the RTL (or across an ECO) by mapping compare points and proving each combinational logic cone equal — no test vectors needed. → [Formal_Verification](../03_Frontend_RTL_and_Verification/12_Formal_Verification.md) §5.

---

## 14. Synthesis

**Q (Innosilicon):** What are constraints?
**A:** SDC (Synopsys Design Constraints) tell synthesis and STA the clocks (`create_clock`), the I/O timing (`set_input_delay`/`set_output_delay`), and the exceptions (false and multicycle paths); they define the "when" against which the RTL's "what" is optimized. → [Constraints_SDC](../04_Synthesis/02_Constraints_SDC.md) §1.

**Q (Zhaoxin):** Is `===` synthesizable?
**A:** No — `===`/`!==` are 4-state (X/Z) case-equality operators with no hardware meaning, so synthesis rejects them. Use `==` in RTL and reserve `===` for testbench X-checks. → [RTL_Design_Methodology](../03_Frontend_RTL_and_Verification/01_RTL_Design_Methodology.md) §4.

**Q (GigaDevice):** Write Verilog that infers a D flip-flop (DFF) versus an unintended latch (`if(a) q<=d`).
**A:** A clocked `always` with a complete assignment infers a flip-flop; a combinational `always` that fails to assign an output on every path (e.g., `if(a) q<=d` with no `else`) infers a *latch* to hold the old value. Avoid it by assigning defaults or covering all branches. → [RTL_Design_Methodology](../03_Frontend_RTL_and_Verification/01_RTL_Design_Methodology.md) §2.

**Q (Allwinner):** Can the module name differ from the file name?
**A:** Yes — the language does not require them to match; the compiler finds modules by parsing, not by filename. But one-module-per-file with matching names is a near-universal convention that many tool and lint flows assume. → [RTL_Design_Methodology](../03_Frontend_RTL_and_Verification/01_RTL_Design_Methodology.md) §3.

---

## 15. Numeric and scripting drills

**Q (Goodix):** Represent decimal −6.25 in 8 bits (two's-complement fixed-point).
**A:** Choose Q4.4 (4 integer bits including sign, 4 fraction bits): value = raw · 2⁻⁴, so raw = −6.25 · 16 = −100; −100 in 8-bit two's complement = 256 − 100 = 156 = `8'b1001_1100` (0x9C). → [09 Hardware Interview Questions](../interview_prep/09_Hardware_Interview_Questions.md).

**Q (Zhaoxin):** Signed 5-bit `a` — put −a into `b`.
**A:** `b = ~a + 1` (two's-complement negate). Watch the edge case `a = −16` (`5'b10000`): its negation +16 does not fit in 5 bits and wraps back to −16, so you need a 6-bit `b` to hold it. → [Adders_and_Multipliers](../00_Fundamentals/03_Adders_and_Multipliers.md) §9.

**Q (Cambricon):** Compute `b = a³ + a² + 2a` — LUT vs multiplier.
**A:** A lookup table (LUT) precomputes `b` for every `a` and indexes it — constant one-cycle time, but memory grows as 2^width · (output bits), so it is only viable for small `a`. A multiplier factors it as `a·((a+1)·a + 2)` (Horner form) and evaluates with a couple of multiplies/adds — less area for wide `a`, at more latency. → [Adders_and_Multipliers](../00_Fundamentals/03_Adders_and_Multipliers.md) §7.

**Q (AMD):** Show why 2nd-largest + 3rd-largest of a,b,c,d = `min(max(a,b),max(c,d)) + max(min(a,b),min(c,d))`.
**A:** Pair (a,b) and (c,d): the two pair-maxes are the winners — the larger is the global max, so the *smaller* winner, `min(max,max)`, is exactly the 2nd largest. The two pair-mins are the losers — the smaller is the global min, so the *larger* loser, `max(min,min)`, is the 3rd largest; summing gives 2nd + 3rd. → [09 Hardware Interview Questions](../interview_prep/09_Hardware_Interview_Questions.md).

**Q (HiSilicon):** Divide, especially by an odd constant.
**A:** Constant division becomes a fixed-point multiply by a reciprocal "magic number": `x/N ≈ (x · ceil(2^k/N)) >> k`, choosing k for exactness — no divider needed. Variable division uses iterative restoring/non-restoring or SRT algorithms. → [Adders_and_Multipliers](../00_Fundamentals/03_Adders_and_Multipliers.md) §9.

**Q (GigaDevice):** Boolean simplification.
**A:** Apply the algebra or a Karnaugh map, e.g. `a + a'b = a + b` (absorption) and `ab + ab' = a` (combining), to reach the fewest literals/gates before technology mapping. → [Logic_Building_Blocks](../00_Fundamentals/02_Logic_Building_Blocks.md) §2.

**Q (T-Head):** Tcl — accumulate/factorial 1..100.
**A:** `set s 0; for {set i 1} {$i<=100} {incr i} {incr s $i}` yields 5050; a factorial loop `set f [expr {$f*$i}]` works directly because Tcl (8.5+) integers are arbitrary-precision. → [09 Hardware Interview Questions](../interview_prep/09_Hardware_Interview_Questions.md).

**Q (Innosilicon):** Perl — `@ARGV` and `$!`.
**A:** `@ARGV` is the array of command-line arguments (the program name is `$0`, not in `@ARGV`); `$!` is the last system/errno error string, e.g. printed after a failed `open`. → [09 Hardware Interview Questions](../interview_prep/09_Hardware_Interview_Questions.md).

**Q (OPPO):** Grep a keyword in shell, Tcl, and Python.
**A:** Shell: `grep pattern file`; Tcl: read the file line by line and test `regexp $pat $line`; Python: iterate the opened file and test `re.search(pat, line)`. All three are "scan lines, match a pattern." → [09 Hardware Interview Questions](../interview_prep/09_Hardware_Interview_Questions.md).

**Q (Allwinner):** Explain a Makefile.
**A:** A rule is `target: prerequisites` then a TAB-indented recipe; `make` rebuilds a target when a prerequisite is newer. Automatic variables: `$@` (target), `$<` (first prerequisite), `$^` (all prerequisites). → [09 Hardware Interview Questions](../interview_prep/09_Hardware_Interview_Questions.md).

---

## 16. Devices (fundamentals)

**Q (Goodix):** NMOS and PMOS structure.
**A:** An NMOS is two n+ source/drain diffusions in a p-substrate under a gate; it conducts electrons when Vgs > Vth (>0) and passes a strong 0. A PMOS is p+ diffusions in an n-well, conducts holes when Vgs < Vth (<0), and passes a strong 1. → [CMOS_Fundamentals](../00_Fundamentals/01_CMOS_Fundamentals.md) §1.

**Q (GigaDevice):** How does a CMOS inverter operate?
**A:** A CMOS inverter is a PMOS pull-up to VDD and an NMOS pull-down to ground with gates tied to the input: input low → PMOS on, NMOS off → output high; input high → NMOS on, PMOS off → output low. The complementary action ideally draws no static current. → [CMOS_Fundamentals](../00_Fundamentals/01_CMOS_Fundamentals.md) §2.

**Q (Allwinner):** PMOS enhancement vs depletion mode.
**A:** An enhancement-mode device is OFF at Vgs = 0 and must be turned on by |Vgs| > |Vth| to form a channel (standard CMOS). A depletion-mode device has an implanted channel and conducts at Vgs = 0, turning off only when the gate depletes that channel. → [CMOS_Fundamentals](../00_Fundamentals/01_CMOS_Fundamentals.md) §1.

**Q (HiSilicon):** Gate-level operation of a D flip-flop.
**A:** A master-slave DFF is two gated latches in series on opposite clock phases: while the clock is low the master is transparent and tracks D (the slave holds); on the rising edge the master locks and the slave becomes transparent, passing the captured D to Q — so Q updates exactly once per rising edge. → [Logic_Building_Blocks](../00_Fundamentals/02_Logic_Building_Blocks.md) §4.

---

## 17. Out of scope for this notebook

A few topics recur at analog- or FPGA-leaning shops but fall outside this ASIC-frontend notebook. Digital signal processing (DSP) filters — finite impulse response (FIR) and cascaded integrator-comb (CIC) — and the sampling theorem, oversampling, and signal-to-noise ratio (SNR) belong to a DSP text (Oppenheim & Schafer, or Proakis). FPGA-specific IO-bank planning, LVDS (low-voltage differential signaling) and other IO electricals belong to the FPGA vendor's user guides (Xilinx/AMD or Intel/Altera). The one adjacent point this notebook does touch is clock-jitter-limited ADC SNR, developed in the PLL/clock material — see [09 Hardware Interview Questions](09_Hardware_Interview_Questions.md) and [Clock_Division_and_Switching](../03_Frontend_RTL_and_Verification/04_Clock_Division_and_Switching.md).

---

## 18. Behavioral prompts (brief)

The recurring non-technical prompts, with one line of advice each:

- **Self-introduction** — 60–90 seconds, funnel from background to the one project that fits this role.
- **Explain your project (modular, top-down)** — start with the block diagram and your module's interface, then drill into the hard problem you solved.
- **Strengths / weaknesses** — one real weakness plus the concrete step you took to improve it.
- **Career plan** — show a direction (frontend design vs verification) that this role advances.
- **Why this company / city** — tie to their products and your location constraints; be specific, not flattering.
- **Offers in hand** — be honest and neutral; signal genuine interest without leveraging aggressively.
- **Teamwork conflict and resolution** — a concrete disagreement, the data/experiment that settled it, the outcome.
- **Biggest setback** — a real failure, what you changed, and the measurable result afterward.
- **Reverse questions (ask them)** — always have two: team/tapeout scope and how new-grads ramp up.

---

## Coverage note

Every technical topic above is developed in full on the linked notebook pages; the only content gap this bank surfaced was serial peripheral protocols (I2C/SPI/UART), now added as the serial-protocols section (§13) of the [SoC transaction-protocols page](../01_Architecture_and_PPA/04_SoC_and_Chiplet_Architecture/03_Transaction_Protocols/01_AHB_AXI_APB.md).

---

⬅ [Folder Index](00_Index.md) · next ➡ [11 · Condensed Cram Sheet](11_Condensed_Cram_Sheet.md) · companions: [08 RTL Coding](08_RTL_Coding_Questions.md) · [09 Hardware Interview](09_Hardware_Interview_Questions.md)
