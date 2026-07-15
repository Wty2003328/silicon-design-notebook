# 03 — Frontend RTL and Verification: Interview Questions

Consolidated interview Q&A and worked problems from every page in `03_Frontend_RTL_and_Verification/`, moved here to keep topic pages focused on concepts. Each section links back to its source page for full context.

---

## SystemVerilog Assertions and Coverage -- Senior Engineer Deep Dive

*From [Assertions_and_Coverage.md](../03_Frontend_RTL_and_Verification/09_Assertions_and_Coverage.md)*

### Q1: What are sampled values in SVA and why do they matter?

**A:** Sampled values are signal values captured in the Preponed region (start of the time
slot, before any Active-region logic). Assertions evaluate in the Observed region using these
pre-captured values. This prevents race conditions: the assertion sees a consistent snapshot
of all signals from before any RTL (Register-Transfer Level) updates in the current cycle. Without sampled values,
assertions could see partially-updated combinational logic and produce non-deterministic results.

### Q2: Explain the difference between |-> and |=>.

**A:** `|->` is overlapping implication: the consequent starts evaluation in the SAME cycle
the antecedent completes. `|=>` is non-overlapping: the consequent starts ONE cycle AFTER the
antecedent completes. `A |=> B` is equivalent to `A |-> ##1 B`.

### Q3: What is the difference between [->N] and [=N]?

**A:** Both require the signal to be true N times non-consecutively. The difference is the
match point: `[->N]` (goto) ends at the Nth occurrence -- the sequence match point is the
cycle where the signal is true for the Nth time. `[=N]` (non-consecutive) requires N
occurrences but allows additional cycles after the Nth occurrence before the next part of the
sequence. Use `[->N]` when you want the sequence to advance immediately after the Nth hit.

### Q4: What is vacuous truth in SVA and why is it dangerous?

**A:** If the antecedent of an implication (`A |-> B`) never matches (A is never true), the
property is vacuously true -- it passes trivially. This is mathematically correct but
verification-dangerous: if the stimulus never triggers the assertion, bugs go undetected.
Mitigation: always pair important assertions with `cover property` on the antecedent to
verify it actually fires during simulation.

### Q5: Explain the bind construct and why it's critical for verification.

**A:** `bind` instantiates a checker module inside an RTL module without modifying the RTL
source. This is essential because: (1) RTL IP (Intellectual Property) may be encrypted; (2) RTL should stay clean for
synthesis; (3) assertions are verification artifacts that don't belong in design code; (4) bind
enables assertion libraries that can be reused across projects. The bound module has full
visibility into the target module's ports and internal signals.

### Q6: What is disable iff and what's the gotcha?

**A:** `disable iff (condition)` disables assertion checking when condition is true. The gotcha:
it's asynchronous -- checked continuously, not just at clock edges. A glitch on the disable
signal (e.g., reset going low for one delta cycle) can disable the assertion even though the
RTL didn't see a reset. For glitch-sensitive signals, consider a synchronous alternative:
`(!condition) || (property)`.

### Q7: How does cross coverage explosion happen and how do you manage it?

**A:** Cross coverage creates bins for every combination of the crossed coverpoints. With K
coverpoints of N bins each, you get N^K cross bins. This grows exponentially. Management
techniques: (1) `ignore_bins` to exclude irrelevant combinations; (2) `binsof...intersect`
to specify only interesting subsets; (3) reduce individual coverpoint bins; (4) split into
multiple targeted crosses instead of one big cross; (5) use `option.auto_bin_max` to limit
auto-generated bins.

### Q8: Write an assertion that checks an AXI-style handshake.

```verilog
property p_axi_handshake(valid, ready, data);
    @(posedge clk) disable iff (!rst_n)
    valid && !ready |=> valid && $stable(data);
endproperty
// Once VALID asserts, it must stay high and DATA must stay stable
// until READY handshake completes
```

### Q9: What is the difference between immediate and concurrent assertions?

**A:** Immediate assertions are procedural -- executed in the Active/Reactive region at the
point they appear in code, checked once, then done. Concurrent assertions are temporal --
sampled at clock edges, can span multiple cycles, evaluated in the Observed region using
sampled values. Use immediate for single-cycle checks (combinational invariants). Use concurrent
for multi-cycle protocols and temporal properties.

### Q10: Explain $past and its limitations.

**A:** `$past(signal, N)` returns the sampled value of signal from N clock edges ago. Default
N=1. Limitations: (1) at the start of simulation, past values are X (4-state) or 0 (2-state);
(2) it uses the assertion's clock, which may not be the signal's actual clock in multi-clock
designs; (3) it cannot look past reset (no built-in awareness of reset cycles); (4) it's based
on sampled values, so it shows what was stable before the clock edge, not mid-cycle values.

### Q11: When would you use assert vs assume vs cover?

**A:** `assert`: the DUT (Device Under Test) must satisfy this property -- failures indicate bugs. `assume`:
the environment/stimulus must satisfy this -- in simulation, acts like assert; in formal
verification, constrains the input space (tells the prover not to try illegal inputs). `cover`:
checks that a property CAN be satisfied -- verifies reachability. Use all three together: assume
legal inputs, assert DUT behavior, cover interesting scenarios.

### Q12: How do you check for X/Z in assertions?

```verilog
// $isunknown returns 1 if any bit is X or Z
property p_no_x;
    @(posedge clk) disable iff (!rst_n)
    !$isunknown(data_bus);
endproperty

// case equality for specific X checks
property p_specific;
    @(posedge clk) !(data === 'x);
endproperty
```

### Q13: Explain first_match and why it matters for performance.

**A:** When a sequence contains a range delay like `##[1:100]`, the evaluator creates 100
concurrent evaluation threads (one for each possible delay). `first_match` keeps only the
earliest matching thread and kills the rest. This matters for: (1) simulator performance --
fewer threads to track; (2) deterministic behavior -- without first_match, multiple matches
can cause unexpected assertion pass/fail patterns; (3) memory -- large ranges without
first_match can consume significant simulator memory.

### Q14: Write a coverage group for a cache controller.

```verilog
covergroup cg_cache @(posedge clk);
    cp_op: coverpoint cache_op {
        bins read     = {CACHE_READ};
        bins write    = {CACHE_WRITE};
        bins flush    = {CACHE_FLUSH};
        bins inv      = {CACHE_INVALIDATE};
    }

    cp_result: coverpoint cache_result {
        bins hit      = {CACHE_HIT};
        bins miss     = {CACHE_MISS};
        bins evict    = {CACHE_EVICT};
    }

    cp_state: coverpoint cache_state {
        bins states[] = {INVALID, SHARED, EXCLUSIVE, MODIFIED};
        bins mesi_transitions = (INVALID => EXCLUSIVE => MODIFIED => INVALID);
        bins share_path = (INVALID => SHARED => INVALID);
    }

    // Cross: every operation type should see every result type
    cx_op_result: cross cp_op, cp_result {
        ignore_bins flush_hit = binsof(cp_op) intersect {CACHE_FLUSH}
                             && binsof(cp_result) intersect {CACHE_HIT};
    }

    // Back-to-back operations
    cp_b2b: coverpoint {prev_op, cache_op} {
        bins read_after_write = {CACHE_WRITE, CACHE_READ};
        bins write_after_read = {CACHE_READ, CACHE_WRITE};
    }
endgroup
```

### Q15: What is the difference between assert #0 and assert final?

**A:** Both are deferred immediate assertions (avoid combinational glitches). `assert #0`
evaluates in the Observed region (same time slot, after Active/NBA (Non-Blocking Assignment) settle). `assert final`
evaluates in the Reactive region (after Observed assertions and program block scheduling).
Use `assert #0` for most cases. Use `assert final` when you need to check values after
concurrent assertion processing has completed.

### Q16: How do you measure assertion coverage?

**A:** Assertion coverage tracks: (1) how many times the antecedent matched (attempts); (2)
how many times the full property passed; (3) how many times it failed; (4) how many times
the antecedent didn't match (vacuous passes). Use `cover property` alongside `assert property`
to ensure non-trivial coverage. Most tools report assertion coverage in the same database as
functional coverage, allowing unified analysis.

---

## Asynchronous Circuit Design and Clock Domain Crossing -- The Complete CDC Bible

*From [Async_Design_and_CDC.md](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md)*

### Q1: Derive the MTBF formula for a 2-FF synchronizer. What determines MTBF?

**A:** Starting from the metastable state, the latch voltage evolves as V(t) = Vm + dV * exp(t/tau). Resolution occurs when V reaches a logic threshold. The probability of NOT resolving in time t_r is proportional to T_w = T_0 * exp(-t_r/tau). Failure rate = f_clk * f_data * T_w, so MTBF (Mean Time Between Failures) = exp(t_r/tau) / (f_clk * f_data * T_0). The key factors are: (1) tau -- technology-dependent, smaller is better (faster resolution); (2) t_r -- available resolution time = clock period - Tc2q - Tsetup; (3) f_clk and f_data -- higher rates increase failure probability. MTBF increases EXPONENTIALLY with t_r/tau, which is why one extra synchronizer stage (doubling t_r) increases MTBF by many orders of magnitude.

### Q2: Why 2 FFs and not 1? When do you need 3?

**A:** With 1 FF (flip-flop), the output goes directly to destination logic. If metastable, the metastable voltage propagates through combinational logic, potentially causing different gates to interpret it as different logic values simultaneously (one gate sees '1', another sees '0'). With 2 FFs, the first FF has a full clock period to resolve before the second FF samples it. The MTBF with 2 FFs is typically 10^15+ years. Use 3 FFs when: f_clk > 2GHz (reducing t_r per stage), safety-critical systems (ASIL-D, DO-254 Level A), or when technology has poor tau (high-Vt cells, older nodes). 3-FF adds 1 cycle latency but squares the MTBF exponent.

### Q3: Why can't you synchronize a multi-bit bus with individual 2-FF synchronizers?

**A:** Each synchronizer bit is independent and may resolve to the new or old value on any given clock cycle. For a bus transitioning from 0111 to 1000, the destination domain could see 1111, 0000, 1100, or any of 16 possible combinations of old/new bit values. These intermediate values never existed in the source domain and cause data corruption. Solutions: (1) Gray code for sequential counters (1-bit change guaranteed); (2) MUX-recirculation (synchronize control, capture stable data); (3) Handshake (req/ack protocol); (4) Async FIFO (First-In First-Out; best for streaming).

### Q4: Walk through the full/empty detection logic of an async FIFO.

**A:** Pointers are ADDR_W+1 bits (extra MSB (most significant bit) for wrap detection), Gray-coded before crossing domains. EMPTY (read domain): compare synchronized write Gray pointer with local read Gray pointer. If equal, empty (all entries read). FULL (write domain): compare local write Gray pointer with synchronized read Gray pointer. Full when the top 2 MSBs differ and the remaining bits are equal -- this means the write pointer has wrapped around and caught up to the read pointer. Both conditions are CONSERVATIVE: empty may be falsely asserted briefly (stale write pointer), and full may be falsely asserted briefly (stale read pointer). Neither condition can falsely indicate "not full" or "not empty," preventing overflow or underflow.

### Q5: Explain async-assert, sync-deassert reset. Why is it necessary?

**A:** The circuit uses a 2-FF synchronizer driven by the async reset. On reset assertion (async_rst_n goes low), the FFs are immediately forced to 0 via the async clear port -- no clock needed, works even with a dead clock. On deassertion, the 1'b1 value propagates through the synchronizer over 2 clock cycles, ensuring the reset release is aligned to a clean clock edge. Without sync deassert, different FFs in the domain might exit reset on different clock edges (some on cycle N, some on N+1), causing state machine corruption, bus contention, or initialization errors. Recovery/removal timing violations are also avoided because the deassertion is synchronous.

### Q6: How do you size an async FIFO for a given application?

**A:** FIFO depth depends on the rate mismatch and burst length. For bursty writes at rate f_wr with maximum burst length B, and continuous reads at rate f_rd: Depth >= B * (1 - f_rd/f_wr). Also add margin for synchronization latency (2-3 cycles on each pointer path effectively reduce usable depth by 4-6 entries). For back-to-back bursts, consider the burst gap (how much the FIFO drains between bursts). Example: f_wr=200MHz burst of 100 words, f_rd=150MHz continuous. Depth >= 100 * (1 - 150/200) = 25 entries. Add 6 for sync latency = 31. Round up to 32 (power of 2 for Gray code). Verify with simulation under worst-case burst patterns.

### Q7: What is a pulse synchronizer and when does it fail?

**A:** A pulse synchronizer converts a source-domain pulse to a toggle, synchronizes the toggle with a 2-FF synchronizer, then detects edges (XOR, i.e. exclusive-OR, of two consecutive samples) to regenerate a pulse in the destination domain. It fails when source pulses arrive faster than the synchronizer can process them -- specifically, if a second pulse arrives before the first toggle has been synchronized (less than ~2-3 destination clock cycles apart). The second pulse toggles the signal back, and if the synchronizer hasn't captured the first transition, it sees no net change and the pulse is lost. For high-rate pulse transfer, use an async FIFO or handshake with backpressure.

### Q8: What CDC verification checks does SpyGlass perform?

**A:** SpyGlass CDC (Clock Domain Crossing) performs: (1) **Structural checks**: missing synchronizers, combinational logic on CDC paths, incorrect synchronizer depth; (2) **Multi-bit CDC**: buses crossing without FIFO/handshake/Gray code; (3) **Reconvergence**: multiple synchronized versions of the same signal meeting at combinational logic; (4) **Protocol checks**: handshake protocol compliance, FIFO pointer Gray code correctness; (5) **Data stability**: ensuring data is stable when sampled by MUX-based synchronizers; (6) **Clock domain identification**: automatically determining clock domains from the netlist; (7) **Quasi-static checks**: signals marked as static that may actually change. Results are errors (must fix), warnings (review), and info (informational). Formal CDC mode uses model checking to exhaustively prove properties.

### Q9: Explain level shifter ordering when both voltage and clock domains change.

**A:** When a signal crosses both a voltage boundary and a clock boundary, the level shifter must come FIRST, then the synchronizer. Reason: the synchronizer FFs must operate at the correct voltage level to function properly (meet setup/hold, resolve metastability reliably). If the synchronizer were powered by the destination supply but received a signal at the source voltage level, the FF input stage might not correctly interpret the logic levels, or the signal might forward-bias protection diodes. Order: Source FF (VDD_low) -> Level Shifter (LH, powered by VDD_high) -> 2-FF Synchronizer (powered by VDD_high, clocked by dst_clk) -> Destination logic.

### Q10: What is reconvergence and why is it dangerous?

**A:** Reconvergence occurs when a CDC signal is synchronized through two independent synchronizer paths and then both paths feed into common downstream logic. Even though both paths synchronize the same source signal, each synchronizer independently resolves metastability and may settle to different values on the same clock cycle (one resolves to the new value, the other to the old value, or they resolve on different cycles). When these inconsistent values combine at downstream logic, the result can be incorrect for 1-2 cycles. Fix: use a single synchronizer and fan out its output, or use a FIFO/handshake that guarantees atomic transfer of all related signals.

### Q11: How does a MUX-recirculation synchronizer guarantee data stability?

**A:** The source domain holds data in a register (data_hold) and toggles a control signal. The control signal is synchronized via a 2-FF synchronizer to the destination domain. By the time the synchronized control signal arrives (after 2 destination clock cycles), the data_hold register has been stable for at least 2 destination clock cycles (assuming source clock is not dramatically faster). The destination domain uses the synchronized control to capture data_hold directly (no synchronizer on the data bus). The data is safe because it hasn't changed since the toggle was generated. The next data update can only happen after the toggle synchronization round-trip completes.

### Q12: What are the timing constraints for signals inside a synchronizer?

**A:** Within a 2-FF synchronizer, the path from FF1/Q to FF2/D must meet setup and hold timing in the destination clock domain. This is a NORMAL intra-domain timing check -- STA (Static Timing Analysis) analyzes it as setup: Tc2q_FF1 + T_wire < T_period - T_setup_FF2, and hold: Tc2q_min_FF1 > T_hold_FF2. The path from the source domain to FF1/D is a CDC crossing -- declared as false_path in SDC (no timing check possible across async domains). The critical requirement is that FF1 and FF2 are placed close together (short wire, minimal delay) to maximize the resolution time. Some tools have special `set_max_delay` constraints on synchronizer paths to enforce close placement.

### Q13: Describe a scenario where a 2-FF synchronizer is NOT sufficient.

**A:** (1) Multi-bit bus: individual 2-FF synchronizers per bit cause data coherency issues. Use FIFO or handshake instead. (2) Pulse shorter than destination clock period in fast-to-slow crossing: the pulse may be entirely missed. Use toggle-based pulse synchronizer. (3) Safety-critical system at very high frequency where MTBF with 2 stages is below 1000 years: use 3-FF synchronizer. (4) Signal with combinational logic fan-in: if two source-domain signals are combined through a gate before the synchronizer, a glitch at the gate output can be captured as a false transition. Solution: register the combined signal in the source domain first.

### Q14: How do you handle CDC for a signal that is used both as data and as a clock?

**A:** This is a very dangerous pattern. A signal that is used as a gated clock in the destination domain must be treated as a clock signal, not a data signal. Metastability on a clock can cause double-clocking or clock glitches that corrupt many FFs simultaneously. Solution: (1) Never directly use a CDC signal as a clock; (2) Synchronize it as data, then feed the synchronized version to an ICG (Integrated Clock Gating) cell to gate the local clock; (3) Ensure the ICG setup/hold timing is met by the synchronized signal. Alternatively, use a clock multiplexer with glitch-free switching logic.

### Q15: What happens to a FIFO during reset if the write domain deasserts reset before the read domain?

**A:** After the write domain deasserts reset, it may start writing data while the read domain is still in reset (rd_ptr = 0, empty flag in an indeterminate state). This is safe because: (1) The write side sees rd_gray_sync = 0 (the reset value), so full detection works correctly against the static rd pointer; (2) The read side is in reset, so it won't attempt to read; (3) When the read side deasserts reset, it sees wr_gray_sync = the Gray-coded write pointer (after synchronization), correctly reflecting the number of entries written. The FIFO operates correctly through asymmetric reset release. The only requirement is that both pointers reset to the same value (0).

### Q16: What is the relationship between FIFO depth and synchronization latency?

**A:** The 2-FF synchronizer introduces 2 clock cycles of latency on each pointer path. This means the full flag sees a read pointer that is 2 write-clock cycles OLD (possibly indicating the FIFO is more full than it actually is), and the empty flag sees a write pointer that is 2 read-clock cycles OLD (possibly indicating the FIFO is more empty than it actually is). This effectively reduces the usable depth by up to 4 entries total (2 on each side). For a 16-entry FIFO, the effective usable depth under worst-case timing is ~12 entries. For almost-full/almost-empty thresholds, account for this latency to avoid unnecessary backpressure or stalls.

### Q17: How do you verify CDC correctness in simulation?

**A:** (1) Run CDC-aware simulation with constrained-random clock frequencies and phase offsets (not integer ratios -- use irrational ratios like 3:7 to maximize edge alignment diversity); (2) Add protocol monitors/checkers at every CDC crossing (verify synchronizer output eventually matches source, verify no data corruption); (3) Inject metastability models: artificially randomize the synchronizer output for 1 cycle after the input changes (simulates non-deterministic resolution); (4) Run long simulations (millions of cycles) to exercise rare timing alignments; (5) Use formal CDC tools to complement simulation (formal is exhaustive, simulation is not). Neither alone is sufficient -- use both.

### Q18: Explain the difference between set_false_path and set_clock_groups for CDC in SDC.

**A:** `set_false_path -from clk_a -to clk_b` removes timing checks in ONE direction only. You need a second command for the reverse direction. It applies specifically to the named clocks and does not automatically include generated clocks. `set_clock_groups -asynchronous -group {clk_a clk_a_div2} -group {clk_b}` removes timing in BOTH directions and automatically includes all clocks in each group. It also prevents inter-group clock uncertainty computation. `set_clock_groups` is preferred because it's bidirectional, group-aware, and less error-prone. However, the synchronizer FFs (FF1 to FF2 within the destination domain) must still have normal timing checks -- these are intra-domain paths, not cross-domain.

### Q19: What is a "gray code pointer overflow" bug and how do you prevent it?

**A:** This occurs when the pointer width or FIFO depth doesn't match the Gray code assumptions. Specifically: (1) If FIFO depth is not a power of 2, the Gray code wraps incorrectly (multiple bits change at the non-power-of-2 boundary); (2) If the pointer width is too narrow and wraps around more than once before the other side catches up, the full/empty detection breaks. Prevention: (1) Always use power-of-2 depth; (2) Pointer width = log2(depth) + 1 (the +1 bit distinguishes "full" from "empty" when both pointers point to the same address); (3) For non-power-of-2 needs, round up to the next power of 2.

### Q20: How do you handle CDC for a reset signal that must reach multiple clock domains simultaneously?

**A:** Use a single async reset source feeding independent reset synchronizers in each clock domain. Each synchronizer uses the async-assert/sync-deassert pattern with the local clock. Assert is simultaneous (async -- bypasses the clock), but deassert may differ by a few cycles between domains. If ordering matters (e.g., producer domain must exit reset before consumer), chain the synchronizers: async_rst -> sync(clk_producer) -> rst_producer -> sync(clk_consumer) -> rst_consumer. This guarantees the producer is out of reset before the consumer. Always verify recovery/removal timing on the FF async reset pins in STA.

---

---

## Clock Division Techniques — Senior Engineer Level

*From [Clock_Division_and_Switching.md](../03_Frontend_RTL_and_Verification/04_Clock_Division_and_Switching.md)*

### Q1: How do you generate an odd-division clock with 50% duty cycle? Prove it works.

**A:** Generate two divided clocks — one triggered on rising edges, one on falling edges — each with duty cycle (N-1)/(2N). OR them. The negedge clock is shifted by T/2 relative to the posedge clock. The OR extends the high time by T/2 at each end, giving total HIGH = (N-1)/2*T + T/2 = N/2*T. Since the period is N*T, duty cycle = (N/2*T)/(N*T) = 50%. This works for any odd N. The proof relies on the half-cycle offset between posedge and negedge filling in the asymmetry.

### Q2: Why can't you use a simple MUX for clock switching?

**A:** A combinational MUX (multiplexer) `sel ? clk_b : clk_a` produces runt pulses when sel transitions while the two clocks are at different levels. Example: if sel goes from 0→1 while clk_a=HIGH and clk_b=LOW, the output drops from 1 to 0 creating a pulse shorter than a valid half-period. This runt pulse causes setup/hold violations in all downstream flip-flops. Additionally, clock tree buffers may filter the runt differently at different fanout points, causing some FFs to see an edge and others not — a catastrophic functional failure.

### Q3: Explain the glitch-free clock mux feedback mechanism.

**A:** Each clock domain has a 2-FF synchronizer for the select signal, with the output of the OTHER domain's synchronizer fed back as a gating condition. Specifically: en_a = sync_to_clk_a(sel_n AND ~en_b). This means en_a cannot go high until en_b is confirmed low (and vice versa). The feedback creates a mandatory dead-time during switching where both enables are 0, and clk_out = 0 (constant, no glitch). The dead-time is 2-4 cycles of each clock. This is a "break-before-make" switching strategy.

### Q4: What is the jitter of a dual-modulus fractional divider?

**A:** For divide-by-N.5 (alternating N and N+1): peak-to-peak jitter = T (one input clock period). For divide-by-N + K/F (general fractional): Jpp = T always (periods differ by 1 input cycle). The RMS (root mean square) jitter depends on the sequence pattern. A first-order sigma-delta modulator produces Jrms ≈ T/sqrt(12) for the divider output. Higher-order modulators shape the noise spectrum so that low-frequency jitter is reduced at the expense of high-frequency jitter, which the PLL (Phase-Locked Loop) loop filter attenuates. After PLL filtering, the output jitter can be sub-picosecond.

### Q5: What is the difference between clock gating and clock division?

**A:** Clock gating selectively enables/disables an existing clock using an ICG cell (latch + AND). It preserves the original frequency when active and eliminates toggling when gated. Used for power reduction of idle blocks. Clock division creates a new, lower-frequency clock signal. Used to generate clocks for slower domains (peripherals, IO). Key difference: gating saves power in the clock tree fanout when a block is idle; division reduces frequency for blocks that inherently need a slower clock. In an SoC (System-on-Chip), clock gating saves 30-60% of dynamic power.

### Q6: How do you handle the case where one clock stops during switching?

**A:** If the source clock stops, the synchronizer FFs in that domain can't update → the enable for the new clock can't assert → system is stuck with no output clock. Solutions: (1) Guaranteed running reference clock that monitors both sources and forces enable transitions via an asynchronous mechanism. (2) Timeout counter on a separate always-running clock — if no edges are detected for N cycles, force the enable state. (3) In practice, SoC clock controllers enforce a protocol: ensure the target clock is stable (PLL locked, crystal running) before asserting the switch. Software reads a "clock stable" status register before switching. The hardware itself may include a lock-detect output from the PLL.

### Q7: Why do ASIC designs use ICG cells instead of direct AND gating?

**A:** A raw `clk & en` glitches if en transitions while clk is HIGH — the AND output produces a narrow pulse. The ICG cell contains a negative-phase latch that captures `en` when clk is LOW (inactive phase). When clk goes HIGH, the latched enable is stable, so the AND output is clean. Additionally, ICG cells are: (1) characterized by the foundry with accurate timing models (Tsetup for enable, Tclk-to-Q), enabling correct STA; (2) DFT-aware (Design-for-Test) with a test-enable (TE) port for scan testing; (3) physically optimized for low insertion delay and balanced rise/fall times. Using raw AND gates for clock gating is a DRC violation in most ASIC (Application-Specific Integrated Circuit) methodologies.

### Q8: Can you implement a divide-by-1.5 clock?

**A:** Yes, using dual-modulus: alternate between divide-by-1 and divide-by-2. Average period = 1.5T. But the jitter is catastrophic: Jpp = T = 67% of the average period. This is unusable for any clocked logic. In practice, divide-by-1.5 (or any fractional ratio < 2) must use a PLL: set feedback divider to 2 and reference divider to 3 → f_out = f_ref * 3/2 = 1.5 * f_ref. The PLL produces a clean clock with sub-ps jitter.

### Q9: What happens if div_ratio is changed while the counter is mid-count?

**A:** If the new ratio is smaller than the current count value, the counter will never reach the (now-lowered) terminal count. It will count up to its maximum value, wrap around, and eventually reach the new terminal count — producing one very long output period (up to 2^N cycles). If the new ratio is larger, the counter reaches the old terminal count, rolls over, and the next period uses the new ratio — producing one short period. Both cases create frequency transients. Fix: latch the new ratio at counter rollover (cnt==0), ensuring it takes effect at a period boundary. Or hold the divider in reset during reconfiguration.

### Q10: Describe the clock mux design used in real SoC power management.

**A:** In a typical SoC (e.g., ARM-based mobile chip), the clock controller has: (1) Multiple clock sources: crystal oscillator (24-32 MHz), ring oscillator (low-power, inaccurate), PLL outputs (high-speed). (2) A tree of glitch-free clock MUXes selecting among sources. (3) Clock dividers downstream of each MUX to generate domain-specific frequencies. (4) ICG cells for fine-grained gating. During DVFS transitions: software programs the new PLL frequency → waits for PLL lock → switches the MUX from the old source to the new PLL → adjusts dividers. The MUX switching takes 2-5 us (including synchronizer latency and dead-time). During this time, the processor may run on a temporary clock (ring oscillator) to stay alive while the PLL relocks.

### Q11: A common tapeout bug related to clock dividers — what is it?

**A:** Forgetting to constrain the generated clock from a divider in SDC. If you write a divide-by-4 and don't tell the STA tool about it:
```tcl
# Missing this constraint:
create_generated_clock -name clk_div4 -source [get_ports clk] \
    -divide_by 4 [get_pins div4_inst/clk_out]
```
The tool treats the divider output as a regular signal, not a clock. Paths from this "clock" to flip-flops are treated as data paths and may be incorrectly timed (too relaxed or too tight). The chip may pass STA but fail in silicon because the actual clock period is 4x shorter than the tool assumed. This is one of the most common clock-domain bugs caught during signoff timing review.

*Q12–Q14: PLL/DLL material — from [PLL_DLL_and_Clock_Distribution.md](../03_Frontend_RTL_and_Verification/05_PLL_DLL_and_Clock_Distribution.md)*

### Q12: Draw the PLL block diagram and explain each component's function.

**A:** PFD (Phase-Frequency Detector): compares reference and feedback clock edges, outputs UP/DOWN error signals indicating whether VCO is too slow or too fast. Charge Pump: converts UP/DOWN pulses to current that charges/discharges the loop filter capacitor. UP → source current → V_ctrl increases → VCO speeds up. DOWN → sink current → V_ctrl decreases → VCO slows down. Loop Filter: integrates charge pump current into a smooth control voltage. Contains R-C network; the resistor provides a stabilizing zero. Determines bandwidth, stability, and lock time. VCO: converts control voltage to output frequency. Ring oscillator (digital) or LC (inductor-capacitor) tank (analog). Feedback Divider (÷N): divides VCO output frequency by N, producing f_fb = f_out/N. When locked, f_fb = f_ref, so f_out = N * f_ref.

### Q13: What are reference spurs and how do you minimize them?

**A:** Reference spurs are spectral peaks at f_out ± k*f_ref caused by periodic disturbances at the reference frequency. The primary source is charge pump mismatch — when I_up != I_dn, a net charge packet is injected every reference cycle, modulating the VCO control voltage at f_ref. This creates FM sidebands (spurs) on the output spectrum. Minimizing spurs: (1) Match charge pump currents through careful analog design and layout symmetry; (2) Narrow the PLL bandwidth so the loop filter attenuates the f_ref perturbation (but this increases lock time); (3) Use a 3rd-order loop filter for additional high-frequency filtering; (4) Reduce charge pump current (smaller perturbation per cycle, but also narrows bandwidth); (5) Calibrate charge pump mismatch digitally at startup. Typical specs: -40 to -60 dBc for consumer, -60 to -80 dBc for communications.

### Q14: Explain the DLL vs PLL trade-off. When would you choose each?

**A:** DLL (Delay-Locked Loop): first-order system (inherently stable), does not accumulate jitter (each output edge derived from a reference edge), cannot multiply frequency, simpler loop filter (just a capacitor), limited delay range. PLL: second-order system (requires stability analysis), VCO accumulates phase noise between corrections, CAN multiply frequency (f_out = N * f_ref), more complex loop filter (R + C for zero), wide frequency range. Choose DLL for: phase alignment without frequency change (DDR SDRAM interfaces, multiphase clock generation for interleaved ADCs). Choose PLL for: clock synthesis (generating 2.4 GHz from 25 MHz crystal), frequency multiplication, DVFS (changing frequency dynamically), CDR in SerDes. In practice, most SoC clock generators use PLLs because frequency synthesis is almost always needed. DLLs appear mainly in DDR PHY blocks.

### Q15: What is clock mesh and when is it used?

**A:** A clock mesh is a grid of metal wires in the upper metal layers that distributes the clock signal. Multiple buffers drive the mesh from different points, and the mesh shorts them together, equalizing the clock arrival time across the chip. The mesh achieves very low skew (< 5 ps in some designs) because any load variation is shared across all drivers. The cost is high: the mesh wire capacitance is enormous (often the largest single contributor to dynamic power), consuming 10-30% of total chip power. Clock mesh is used in ultra-high-performance processors where skew must be minimized (IBM POWER, Intel server CPUs). It is combined with an H-tree: the H-tree drives the mesh grid points, and the mesh provides local distribution. In ASIC designs below ~1 GHz, a well-balanced CTS (Clock Tree Synthesis) tree is sufficient and much more power-efficient.

### Q16: Explain CPPR (Clock Path Pessimism Removal) and why it matters.

**A:** In OCV-aware STA, the launch and capture clock paths are derated differently (slow vs fast) to account for local variation. But the common portion of both paths — from the clock source to the divergence point — is physically the same gates and wires. Derating the common path slow on launch and fast on capture creates artificial pessimism that doesn't reflect physical reality. CPPR identifies the common clock path and removes the differential derate on it, applying OCV (On-Chip Variation) only from the divergence point onward. The impact is significant: CPPR typically recovers 50-200 ps of timing margin, which can mean the difference between meeting and failing timing closure. Without CPPR, 5-15% of paths may show false violations, requiring unnecessary optimization that increases area and power. All modern STA tools (PrimeTime, Tempus) support CPPR and it is enabled by default in signoff flows.

### Q17: How does a clock gating check differ from a normal setup/hold check?

**A:** A normal setup check ensures data is stable before the active clock edge (e.g., posedge for a rising-edge FF). A clock gating check ensures the enable signal is stable before the INACTIVE edge of the clock (e.g., negedge for an AND-gate gating on a positive-edge clock). The reason: the enable must be settled before the clock becomes active (goes high) to prevent glitches on the gated clock. The check is associated with the clock gate (AND/OR gate) rather than a flip-flop. Clock gating checks can be setup (EN arrives too late) or hold (EN changes too soon after the inactive edge). In STA tools, clock gating checks are reported as a separate category. A common pitfall: forgetting to define clock gating checks on custom gating logic (non-standard ICG cells), which causes the tool to miss the constraint entirely.

### Q18: What happens to the clock during a power domain shutdown?

**A:** When a power domain is shut down (supply gated off), all logic in that domain loses state, including any clock generation or distribution logic. The clocks to that domain must be stopped before shutdown to prevent: (1) glitches on the clock as the supply ramps down (gates in the clock path may generate runt pulses as they pass through the metastable region); (2) shoot-through current in clock buffers as the supply drops below the switching threshold; (3) propagation of undefined clock edges to always-on logic through clock crossings. The proper shutdown sequence is: assert clock gate enable to LOW (stop the clock) → wait for clock to settle → assert isolation cells → assert power switch control → domain powers down. On wake-up: reverse the sequence, and wait for the clock to be stable (PLL locked, if applicable) before de-asserting isolation and releasing clock gates.

### Q19: Describe fractional-N division and its jitter implications.

**A:** A fractional-N divider alternates its division ratio between N and N+1 such that the average ratio is N + K/F (where K and F are integers). A phase accumulator adds K each cycle; when it overflows, the divider uses N+1 instead of N. Over F cycles, it uses N+1 exactly K times, achieving the correct average. The instantaneous jitter is ±T (one input clock period) because the period changes by ±T on each N↔N+1 transition. For a 100 MHz input (T=10ns), this is enormous — 10 ns p-p jitter. When used inside a PLL, the loop filter attenuates this jitter. The PLL bandwidth is set below f_ref/10, and the sigma-delta modulator shapes the divider's quantization noise to high frequencies, where the loop filter provides maximum attenuation. The result: output jitter after PLL filtering can be < 1 ps RMS, despite the divider having 10 ns of instantaneous jitter.

### Q20: What are the common causes of clock-related silicon failures?

**A:** (1) Missing or incorrect SDC constraints: generated clocks not defined, false paths not set, multicycle paths not constrained → timing analysis misses real violations. (2) Clock domain crossing (CDC) bugs: missing synchronizers, incorrect FIFO sizing, or reconvergence issues where a CDC signal is used in multiple places without consistent synchronization. (3) Clock gating violations missed by STA: custom clock gates without proper clock gating check definitions. (4) OCV/CPPR misconfiguration: over-pessimistic or under-pessimistic analysis leading to either over-design or silicon failures. (5) PLL lock time not accounted for in boot sequence: software accesses clocked logic before PLL is locked → random behavior. (6) Glitchy clock MUX: using a combinational MUX for clock switching without the break-before-make synchronizer design. (7) Clock tree imbalance after ECO: metal fills, via changes, or buffer insertions during ECO perturb the balanced clock tree.

### Q21: How do you constrain a divided clock in SDC?

**A:** Use `create_generated_clock`:
```tcl
# For a divide-by-4 from a flip-flop based divider:
create_generated_clock -name clk_div4 \
    -source [get_pins div_reg/C] \
    -divide_by 4 \
    [get_pins div_reg/Q]

# For a MUX-based clock selector:
create_generated_clock -name clk_mux_a \
    -source [get_pins mux/A] \
    -master_clock clk_a \
    [get_pins mux/Y]
create_generated_clock -name clk_mux_b \
    -source [get_pins mux/B] \
    -master_clock clk_b \
    [get_pins mux/Y] -add

# Set the clocks as physically exclusive (only one active at a time):
set_clock_groups -physically_exclusive \
    -group clk_mux_a -group clk_mux_b
```
Common mistakes: forgetting `-add` for multiple clocks on the same pin (the second overwrites the first), not setting clock groups (the tool pessimistically checks timing between both clocks), and incorrect `-source` pin (must be the clock pin of the generating element, not the data pin).

### Q22: What is the relationship between PLL bandwidth and jitter?

**A:** PLL bandwidth creates a crossover between two noise sources. Within the bandwidth, the PLL tracks the reference: reference noise (multiplied by N) passes through, and VCO noise is suppressed. Outside the bandwidth, the VCO free-runs: VCO noise dominates, and reference noise is filtered. Total output jitter = sqrt(reference_contribution^2 + VCO_contribution^2). The optimal bandwidth is where N^2 * S_ref(f) = S_vco(f), i.e., where the scaled reference noise equals the VCO noise. Too wide a bandwidth: VCO noise is well-suppressed, but reference noise (amplified by N^2) dominates — jitter increases. Too narrow a bandwidth: reference noise is filtered, but VCO noise is poorly suppressed — jitter also increases. In practice, the optimal bandwidth is typically f_ref/20 to f_ref/10 for integer-N PLLs. For fractional-N PLLs, wider bandwidth is possible (and desirable to filter sigma-delta noise) because the reference frequency is higher.

### Q23: How do you design a clock divider that allows hitless ratio changes?

**A:** A hitless ratio change means the output clock transitions smoothly from old_ratio to new_ratio without missing edges, producing runt pulses, or having a long dead period. Implementation: (1) Latch the new ratio value only at counter rollover (cnt == 0), as shown in the programmable divider RTL — this ensures the current period completes before the new ratio takes effect. (2) For the transition period itself, the output period may be slightly different from both the old and new ratios (it completes the current half-period at the old ratio, then starts the next at the new ratio). (3) If true hitless operation is required (no period disturbance at all), use a fractional-N divider approach: gradually adjust the fractional part from old_ratio to new_ratio over multiple cycles, so the frequency ramps smoothly. This is used in spread-spectrum clocking where the frequency is continuously modulated. (4) An alternative: have two parallel dividers (old and new ratio) running simultaneously, and use a glitch-free MUX to switch between them once the new divider has phase-aligned with the old one.

### Q24: Explain spread-spectrum clocking (SSC) and its implementation.

**A:** Spread-spectrum clocking intentionally modulates the clock frequency (typically ±0.5% to ±1%) to spread the energy of clock harmonics across a wider bandwidth, reducing the peak EMI (electromagnetic interference) at any single frequency. This helps products pass FCC/CE emission regulations. Implementation: use a fractional-N PLL where the fractional part K is modulated by a triangular or Hershey-kiss profile. The modulation frequency is typically 30-33 kHz (below the resolution bandwidth of EMI measurement receivers). The modulation profile is stored in a small ROM or generated by a counter-based state machine that adjusts K each reference cycle. The PLL bandwidth must be wide enough to track the modulation (~100-500 kHz). Down-spread (only decrease frequency) is common because it maintains backward compatibility with the maximum frequency specification. Used in: PCIe (Peripheral Component Interconnect Express), SATA (Serial ATA), USB, DDR (all have SSC specifications).

### Q25: What is clock jitter's impact on ADC sampling?

**A:** Clock jitter directly limits ADC performance. When the sampling clock has jitter, the actual sampling instant deviates from the ideal, causing an amplitude error proportional to the signal's slew rate at that instant. For a full-scale sinusoidal signal at frequency f_in, the SNR (signal-to-noise ratio) limited by jitter is: SNR_jitter = -20*log10(2*pi*f_in*J_rms) dB. Example: for a 100 MHz input signal and 1 ps RMS jitter: SNR = -20*log10(2*pi*100e6*1e-12) = -20*log10(6.28e-4) = 64 dB ≈ 10.3 ENOB (Effective Number of Bits). This means even a "perfect" ADC with zero quantization noise would be limited to 10.3 effective bits by 1 ps of clock jitter at 100 MHz input. For a 14-bit ADC (86 dB SNR), the jitter requirement is J_rms < 0.1 ps — achievable only with an LC-based PLL or a DLL-cleaned clock. This is why high-speed ADC designs use DLLs (no jitter accumulation) or extremely low-noise LC PLLs for the sampling clock.

### Q26: How is clock skew managed in modern ASIC design?

**A:** Clock skew is managed through the Clock Tree Synthesis (CTS) flow in P&R (Place-and-Route) tools: (1) **Target:** useful skew = 0 for all flip-flop pairs in the same clock domain (or intentional useful skew to help timing). Typical achievable skew: 20-50 ps in 7nm, 50-100 ps in 28nm. (2) **CTS algorithm:** the tool builds a balanced buffer tree from the clock source to all sinks (flip-flops), inserting buffers and inverters to equalize path delays. H-tree topology for critical clocks, or spine-based for general use. (3) **Post-CTS optimization:** the tool adjusts buffer sizes and adds delay cells to fine-tune skew. Useful skew intentionally unbalances launch vs capture to help tight paths. (4) **Clock tree DRVs:** design rule violations specific to clock trees — maximum transition time, maximum capacitance, maximum fanout — are tighter than data path rules. (5) **Multi-corner multi-mode (MCMM):** CTS must balance skew across all PVT (Process, Voltage, Temperature) corners simultaneously, not just typical. A tree balanced at SS (slow-slow) corner may be unbalanced at FF (fast-fast) corner. CTS tools optimize across multiple corners jointly.

---

## SystemVerilog Data Types and Basics -- Senior Engineer Deep Dive

*From [Data_Types_and_Basics.md](../03_Frontend_RTL_and_Verification/02_Data_Types_and_Basics.md)*

### Q1: What is the difference between `logic` and `reg`?

**A:** `logic` is a SystemVerilog 4-state type that can be used anywhere `reg` or `wire` was
used in Verilog, with the restriction that it cannot have multiple drivers. `reg` was
Verilog-only and could only be used in procedural blocks. The name `reg` is misleading because
it does not imply a register -- `logic` eliminates this confusion. In modern SV code, `logic`
is used universally for 4-state signals.

### Q2: When would you use `bit` instead of `logic` in a testbench?

**A:** Use `bit` (2-state) for testbench data structures like scoreboards, transaction fields,
and reference models where X/Z values are meaningless and simulation performance matters. Use
`logic` (4-state) for signals connected to DUT ports where X-propagation detection is critical
for catching uninitialized or multiply-driven signals.

### Q3: What happens when a 4-state value with X is assigned to a 2-state variable?

**A:** X and Z bits are converted to 0. This is silent -- no warning or error. This is why
using 2-state types for DUT-connected signals is dangerous: X-bugs become invisible.

```verilog
logic [3:0] four_state = 4'b10xz;
bit [3:0] two_state = four_state;  // two_state = 4'b1000
```

### Q4: Explain packed vs unpacked arrays. When does the distinction matter?

**A:** Packed arrays are contiguous bit vectors -- dimensions before the variable name.
Unpacked arrays are arrays of elements -- dimensions after the name. The distinction matters
for: (1) Assignments -- packed arrays can be assigned to/from bit vectors of equal width;
(2) Module ports -- must be packed or match exactly; (3) DPI (Direct Programming Interface) calls -- packed maps to simple
C types, unpacked requires complex svOpenArrayHandle; (4) Part-select -- packed supports
arbitrary bit-slicing like `arr[15:8]`, unpacked does not.

### Q5: Write code to find the second-largest value in a dynamic array.

```verilog
function automatic int find_second_largest(int arr[]);
    int sorted_q[$];
    sorted_q = arr.unique;   // Remove duplicates
    sorted_q.sort();         // Ascending
    if (sorted_q.size() < 2) return sorted_q[0];
    return sorted_q[sorted_q.size()-2];
endfunction
```

### Q6: What is the difference between a queue and a dynamic array?

**A:** Both are variable-size. Dynamic arrays require explicit `new[N]` to resize (O(n) copy
operation). Queues support O(1) `push_back`/`push_front`/`pop_back`/`pop_front` without
explicit sizing. Queues also support the `$` operator for last-element access and bounded size
limits. For sequential processing (producer-consumer), queues are the clear choice.

### Q7: How do associative arrays handle memory, and when are they preferred?

**A:** Associative arrays allocate memory only for indices that are actually stored (hash-table
or tree implementation). They are preferred for: sparse data (e.g., modeling a 4GB memory space
with only a few entries), non-integer indexing (strings, class handles), and data where the key
space is much larger than the number of entries.

### Q8: Explain the signedness bug with concatenation.

**A:** In SystemVerilog, the concatenation operator `{}` ALWAYS produces an unsigned result,
regardless of operand signedness. If you concatenate signed values, the result loses its
signedness. This causes bugs when you try to create a sign-extended value:
`{1'b0, signed_val}` creates an unsigned result. Use `$signed()` to restore signedness or
let automatic sign extension handle it via direct assignment to a wider signed variable.

### Q9: Write a parameterized FIFO using a queue.

```verilog
class fifo #(type T = int, int DEPTH = 16);
    T q[$:DEPTH-1];

    function bit push(T data);
        if (q.size() >= DEPTH) return 0;  // Full
        q.push_back(data);
        return 1;
    endfunction

    function bit pop(output T data);
        if (q.size() == 0) return 0;  // Empty
        data = q.pop_front();
        return 1;
    endfunction

    function bit is_full();
        return (q.size() >= DEPTH);
    endfunction

    function bit is_empty();
        return (q.size() == 0);
    endfunction

    function int occupancy();
        return q.size();
    endfunction
endclass
```

### Q10: What is the output of this code?

```verilog
logic signed [7:0] a = -1;    // 8'hFF
logic [15:0] b;
b = a;
$display("b = %0d", b);       // ?
```

**A:** `b = 255`. Because `b` is unsigned, the assignment converts `a` to unsigned BEFORE
extending. `a` as unsigned is 255 (8'hFF), then zero-extended to 16'h00FF = 255.
If `b` were `logic signed [15:0]`, the result would be `16'hFFFF = -1` (sign-extended).

### Q11: How do you model a register with reserved fields using structs?

**A:** Use a packed struct with explicit reserved fields. This maps directly to the hardware
register layout and enables bitfield access by name while maintaining the ability to read/write
the entire register as a single vector. See the `usb_ep_ctrl_t` example above. In UVM (Universal Verification Methodology) RAL (Register Abstraction Layer),
this maps to `uvm_reg_field` objects with `"RO"` or `"RsvdZ"` access policies.

### Q12: Explain the `with` clause in array methods.

**A:** The `with` clause provides an inline predicate or expression evaluated for each element.
The variable `item` refers to the current element. It enables powerful queries without writing
explicit loops:
- `arr.find with (item > 10)` -- returns all elements greater than 10
- `arr.sort with (item.field)` -- sorts by a specific field
- `arr.sum with (item.size)` -- sums a specific property
- `arr.find_index with (item == target)` -- returns indices of matching elements

### Q13: What happens when you read an uninitialized entry from an associative array?

**A:** If you read a key that doesn't exist, SystemVerilog creates a new entry with the
default value (0 for integral types, "" for strings, null for class handles) and returns it.
This is a common bug source -- always use `.exists(key)` before reading.

```verilog
int aa [string];
$display("%0d", aa["missing"]);  // Prints 0, AND creates the entry!
$display("Num: %0d", aa.num());  // 1 -- the entry was created by the read
```

### Q14: Explain the difference between `==` and `===` for 4-state comparisons.

**A:** `==` is logical equality -- returns X if either operand contains X or Z.
`===` is case equality -- compares all 4 states exactly, always returns 0 or 1.

```verilog
logic [3:0] a = 4'b10x1;
logic [3:0] b = 4'b10x1;
if (a == b)  // Result is X (unknown), treated as false in if
if (a === b) // Result is 1 (true) -- X matches X exactly
```

Use `===` in testbench assertions when checking for X. Use `==` in RTL (synthesizable).

### Q15: How do you properly pass a dynamic array to a function?

```verilog
// By value -- copies the ENTIRE array (expensive for large arrays)
function void process_copy(int arr[]);
    arr[0] = 999;  // Modifies the COPY, not the original
endfunction

// By reference -- no copy, modifies original
function void process_ref(ref int arr[]);
    arr[0] = 999;  // Modifies the original
endfunction

// By const reference -- no copy, read-only
function void process_const(const ref int arr[]);
    // arr[0] = 999;  // COMPILE ERROR
    $display("%0d", arr[0]);  // Read OK
endfunction
```

### Q16: Write code to implement a simple hash function for strings.

```verilog
function automatic int unsigned string_hash(string s);
    int unsigned hash = 5381;
    for (int i = 0; i < s.len(); i++) begin
        hash = ((hash << 5) + hash) + s.getc(i);  // djb2 algorithm
    end
    return hash;
endfunction
```

---

## Formal Verification — Senior Engineer Deep Dive

*From [Formal_Verification.md](../03_Frontend_RTL_and_Verification/12_Formal_Verification.md)*

**Q1: What is the difference between simulation and formal verification?**

Simulation applies specific input vectors and checks outputs — it's inherently incomplete
because it only tests the cases you thought of. Formal verification mathematically proves
that a property holds for ALL possible inputs and ALL reachable states. Simulation is good
for datapath validation and system-level testing. Formal excels at control logic, protocol
compliance, and proving absence of bugs (deadlock, starvation, CDC violations).

**Q2: Explain SAT-based bounded model checking.**

The design's transition relation is "unrolled" for K clock cycles. The initial state,
K transitions, and the negation of the target property at step K are encoded as a single
Boolean formula in CNF (Conjunctive Normal Form). A SAT (Boolean satisfiability) solver determines satisfiability: if SAT, the satisfying
assignment is a counterexample trace; if UNSAT (unsatisfiable), the property holds for K cycles. The bound
K is incrementally increased to search for deeper bugs or achieve convergence.

**Q3: What is the difference between BDD-based and SAT-based model checking?**

BDD-based model checking computes the full set of reachable states as a BDD (Binary Decision Diagram; fixed-point
computation). It gives unbounded proofs but can fail due to BDD size explosion.
SAT-based BMC (Bounded Model Checking) unrolls the design for K steps — faster for finding bugs but only provides
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
reachable states using clause learning, similar to how CDCL (Conflict-Driven Clause Learning) SAT solvers learn conflict
clauses. It maintains a sequence of "frames" that progressively tighten the reachable
state set. It often converges much faster than k-induction because it learns exactly
which states need to be excluded, rather than requiring uniform inductive depth.

**Q6: What happens during LEC (Logic Equivalence Checking)? Walk through the flow.**

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
then verify the retiming step separately using SEC (Sequential Equivalence Checking).

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

**Q13: What is the difference between CTL (Computation Tree Logic) and LTL (Linear Temporal Logic)?**

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
1-to-1 register mapping. SEC handles: retiming, FSM (Finite State Machine) re-encoding, pipeline balancing,
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

## Gate-Level Simulation and Emulation

*From [Gate_Level_Sim_and_Emulation.md](../03_Frontend_RTL_and_Verification/13_Gate_Level_Sim_and_Emulation.md)*

**Q: Why run GLS (Gate-Level Simulation) if you already ran RTL sim and STA?** RTL sim doesn't run the actual netlist (misses synthesis/scan bugs and X-pessimism); STA is static (checks timing margins, not functional waveforms, and can't see glitches or async races). SDF (Standard Delay Format) GLS runs the real netlist with real delays and catches reset/X-propagation, scan-chain, and timing-dependent functional bugs neither can.

**Q: Emulation vs FPGA (Field-Programmable Gate Array) prototype — when each?** Emulation: verification wants full signal visibility, fast compile, and HW/SW co-verification of long scenarios — accept ~MHz speed. FPGA prototype: software/system teams want near-real-time to develop apps and demo — accept lower visibility and slow compile. Same design, different consumer.

**Q: How do you cut GLS pain?** Prove logical equivalence with formal LEC (fast), reserve GLS for what LEC can't do (X/reset, SDF timing, DFT), and run a focused suite (power-up + sanity + ATPG (Automatic Test Pattern Generation) patterns) rather than the full regression.

---

## SystemVerilog IPC and Verification Methodology -- Senior Engineer Deep Dive

*From [Procedural_Processes_and_IPC.md](../03_Frontend_RTL_and_Verification/03_Procedural_Processes_and_IPC.md) and [UVM_Methodology.md](../03_Frontend_RTL_and_Verification/10_UVM_Methodology.md)*

### Q1: Explain the difference between mailbox put/get and try_put/try_get.

**A:** `put()` and `get()` are blocking: `put()` blocks if a bounded mailbox is full, `get()`
blocks if the mailbox is empty. `try_put()` and `try_get()` are non-blocking: they return
immediately with 1 (success) or 0 (failure). Use blocking versions for normal flow control
(back-pressure). Use non-blocking for polling patterns or when you can't afford to block.

### Q2: How can events cause a race condition and how do you fix it?

**A:** If `->` (trigger) and `@` (wait) execute in the same time step, the `@` may miss the
trigger because it was registered before the `->` executed. Fix: use `wait(event.triggered)`
which is persistent within the time step, or use `->>` (nonblocking trigger) which schedules
the trigger to the NBA region.

### Q3: Explain UVM build_phase vs connect_phase.

**A:** `build_phase` creates and configures components (top-down: parent creates children).
`connect_phase` connects TLM (Transaction-Level Modeling) ports between components (bottom-up convention). Separation is
needed because you can't connect ports until ALL components exist. Build runs top-down so
parents can configure children before children build. Connect runs bottom-up by convention
but order doesn't strictly matter since ports are just handles.

### Q4: What is the difference between uvm_analysis_port and uvm_blocking_put_port?

**A:** `uvm_analysis_port` is one-to-many broadcast: calling `write()` sends to ALL connected
subscribers simultaneously. It's non-blocking (function). `uvm_blocking_put_port` is
one-to-one: connects to exactly one imp/export. `put()` is a blocking task. Use analysis ports
for monitor-to-scoreboard/coverage broadcast. Use put ports for point-to-point communication.

### Q5: Explain the driver-sequencer handshake (get_next_item/item_done).

**A:** (1) Sequence calls `start_item(txn)` -- waits for sequencer arbitration grant.
(2) Sequence calls `finish_item(txn)` -- sends txn to sequencer, blocks until driver finishes.
(3) Driver calls `get_next_item(txn)` -- retrieves txn from sequencer.
(4) Driver drives txn on the bus interface.
(5) Driver calls `item_done()` -- signals completion, which unblocks `finish_item()`.
This handshake ensures sequencer, sequence, and driver stay synchronized.

### Q6: What is the UVM factory and why is it essential?

**A:** The factory is a lookup table that maps type names to constructors. Instead of
`new()`, components use `type_id::create()`. This allows type overrides (globally replace one
type with another) and instance overrides (replace only at specific hierarchy paths) WITHOUT
modifying source code. Essential for VIP (Verification IP) reuse: end users can substitute custom transactions,
drivers, etc., into reusable VIP without touching VIP code.

### Q7: Explain frontdoor vs backdoor register access in UVM RAL.

**A:** Frontdoor access generates actual bus transactions through the driver, exercising the
real bus protocol. Backdoor access uses the simulator's hierarchical path access to read/write
register values directly in RTL, bypassing the bus. Use frontdoor for protocol verification,
backdoor for fast initialization and when bus access is not the focus.

### Q8: What are UVM objections and why are they needed?

**A:** Objections are a reference-counting mechanism that keeps simulation phases alive.
When a component raises an objection, the phase cannot end. When all objections are dropped,
the phase completes. Without objections, the simulator wouldn't know when sequences are
finished and might end simulation prematurely. Common pattern: test raises objection, starts
sequences, drops objection when sequences complete.

### Q9: Explain TLM FIFO and when you'd use it.

**A:** A TLM FIFO decouples producer and consumer: the producer writes items at its own rate,
and the consumer reads at its own rate, with the FIFO buffering between them. Without it,
producer and consumer must synchronize directly (blocking until the other side is ready).
Use when producer and consumer operate at different rates or when you want to avoid direct
synchronization coupling.

### Q10: How does UVM config_db scope matching work?

**A:** `set(context, path, field, value)` creates an effective scope of
`context.get_full_name() + "." + path`. `get(context, path, field, var)` creates a lookup path
of `context.get_full_name() + "." + path`. The get succeeds if the set's scope pattern matches
the get's path. Wildcards (`*`) are supported in set's path. More specific paths take precedence
over wildcards. Later sets override earlier ones at the same specificity.

### Q11: What is a virtual sequence and why is it needed?

**A:** A virtual sequence coordinates multiple sequences across multiple agents/sequencers.
It doesn't run on any single sequencer (`start(null)`). Instead, it holds handles to multiple
sequencers and starts sub-sequences on each. This enables test scenarios that require
coordinated traffic from multiple bus masters (e.g., CPU and DMA (Direct Memory Access) accessing shared memory).

### Q12: Draw the UVM phase execution order and explain why it matters.

**A:** build (top-down) -> connect (bottom-up) -> end_of_elaboration -> start_of_simulation ->
run (parallel, with sub-phases) -> extract -> check -> report. It matters because: (1) build
must be top-down so parents configure children before children create sub-components; (2) run
phase has objection-controlled lifetime; (3) check phase is where scoreboards report pass/fail
after all simulation activity is done; (4) report phase prints final statistics. Getting the
order wrong means components aren't configured or connected when needed.

### Q13: How do you handle reset in a UVM driver?

**A:** Use a fork-join_any pattern: one thread runs the normal get_next_item/drive/item_done
loop, and another thread watches for reset assertion. When reset fires, join_any returns,
disable fork kills the drive thread, the driver resets interface signals to idle, waits for
reset deassert, then re-enters the fork. This ensures clean recovery from reset during
any point in a transaction. (See the driver example with reset_thread above.)

### Q14: Explain semaphore deadlock and how to prevent it.

**A:** Deadlock occurs when two or more processes each hold a semaphore that the other needs,
and neither can proceed. Classic example: Process 1 holds sem_a and waits for sem_b; Process 2
holds sem_b and waits for sem_a. Prevention: (1) always acquire semaphores in the same global
order; (2) use try_get() with timeout and release if acquisition fails; (3) acquire all
resources atomically.

### Q15: What is the difference between UVM_ACTIVE and UVM_PASSIVE agents?

**A:** An active agent has a driver and sequencer (generates stimulus). A passive agent has
only a monitor (observes bus traffic). Passive agents are used for: (1) monitoring interfaces
that are driven by the DUT (output ports); (2) protocol checking without driving; (3)
read-only bus slaves. The agent's `get_is_active()` method returns the mode, and `build_phase`
conditionally creates driver/sequencer only for active agents.

### Q16: Explain the UVM register model's mirror, desired, and actual values.

**A:** Mirror = what the model predicts the HW register contains (updated on reads and writes).
Desired = what the testbench wants the register to be (set via `set()` without bus access).
Actual = what the HW register really holds in RTL. `write()` updates desired, sends bus
write, and updates mirror on completion. `read()` reads from HW and updates mirror.
`mirror()` reads from HW and checks against the predicted mirror value -- a mismatch indicates
a bug. `update()` writes desired to HW only if desired != mirror (conditional write).

---

## Lint, CDC, and RDC Signoff — Static Frontend Checks

*From [Lint_CDC_RDC_Signoff.md](../03_Frontend_RTL_and_Verification/07_Lint_CDC_RDC_Signoff.md)*

**Q: Why can't simulation find a missing CDC synchronizer?** RTL simulation is cycle-deterministic — it samples the crossing signal as a clean 0/1 each cycle and never models the metastable settling window. The bug only appears as real-time metastability in silicon. Only static CDC analysis (structure) or gate-sim with timing + metastability injection can flag it.

**Q: You 2-FF-synchronized each bit of an 8-bit bus across clocks. What's wrong?** The bits can settle on *different* destination cycles, so the receiver can observe a transient value that never existed (e.g., 0111→1000 seen as 1111). Use gray code (if it's a counter, only 1 bit changes), a handshake/MCP (Multi-Cycle Path) formulation, or an async FIFO.

**Q: What is RDC and how does it differ from CDC?** RDC is reset-domain crossing: a flop in reset-domain A feeding one in reset-domain B. When A asserts asynchronously, B (if running) can capture the mid-cycle transition and go metastable — same failure as CDC but caused by independent *reset assertion* rather than a clock edge. Fix with isolation or reset sequencing.

---

## SystemVerilog OOP and Randomization -- Senior Engineer Deep Dive

*From [OOP_and_Randomization.md](../03_Frontend_RTL_and_Verification/08_OOP_and_Randomization.md)*

### Q1: Explain handle vs object in SystemVerilog. What happens with `Packet p2 = p1;`?

**A:** A handle is a reference/pointer; an object is the actual data in memory. `Packet p2 = p1`
copies the HANDLE, not the object. Both p1 and p2 now point to the same object. Modifying
fields through p2 affects what p1 sees. To get an independent copy, you need an explicit copy
method (deep copy). This is called shallow copy semantics.

### Q2: What is the difference between virtual and non-virtual methods?

**A:** Non-virtual methods use static dispatch (compile-time binding) based on the handle type.
Virtual methods use dynamic dispatch (run-time binding) based on the object type. When you
store a derived class object in a base class handle, non-virtual calls invoke the BASE class
method; virtual calls invoke the DERIVED class method. Virtual is essential for polymorphism
in UVM.

### Q3: Why does UVM need a factory instead of just using `new`?

**A:** The `new` operator hardcodes the type at compile time. In reusable VIP code, you want
users to substitute their own transaction/component types without modifying the VIP source.
The factory pattern replaces `new()` with `type_id::create()`, which looks up the registered
type (or its override) at run-time. This enables type and instance overrides: "whenever the VIP
creates a `base_txn`, give it my `error_txn` instead."

### Q4: Explain the difference between `dist` with `:=` and `:/`.

**A:** `:=` assigns the weight to EACH value. `:/` distributes the weight ACROSS the range.
Example: `value dist {[1:4] := 10, [5:8] :/ 10}` means values 1,2,3,4 each get weight 10
(total 40), while values 5,6,7,8 share weight 10 (2.5 each, total 10). So 1-4 are each 4x
more likely than 5-8.

### Q5: What is `solve...before` and when do you need it?

**A:** `solve X before Y` tells the constraint solver to first determine X's value, then solve
Y given X's fixed value. Without it, the solver considers all (X,Y) pairs in the solution
space equally. This affects probability distribution. Example: if `flag` determines two
different ranges for `value`, without `solve flag before value`, the flag probability depends
on how many solutions each flag value has. With `solve flag before value`, flag is decided
first (50/50), then value is chosen within the resulting range.

### Q6: What happens if randomize() returns 0?

**A:** It means the constraints are unsolvable -- contradictory or over-constrained. The random
variables retain their previous values (they are NOT modified). You MUST check the return value:

```verilog
if (!txn.randomize())
    `uvm_fatal("RAND", "Randomization failed -- check constraints")
```

Common causes: conflicting hard constraints, constraint conflict with inline `with` clause,
`rand_mode(0)` fixing a value that conflicts with constraints.

### Q7: Explain `randc` vs `rand`.

**A:** `rand` produces uniformly distributed values each call (may repeat). `randc` produces
cyclic random: it generates a random permutation of all possible values and returns them one
by one. After all values are used, it starts a new permutation. `randc` guarantees every value
appears before any repeats. Useful for testing all possible values (e.g., all opcodes) with
fair distribution.

### Q8: How do you implement a deep copy in UVM?

**A:** Override `do_copy()` in your sequence item. Use `$cast` to convert the `uvm_object rhs`
parameter to your specific type. Copy each field individually, and for any handle fields, create
new objects and copy their contents recursively rather than just copying the handle. Call
`super.do_copy(rhs)` first to handle base class fields.

### Q9: What is the difference between type override and instance override in UVM factory?

**A:** Type override replaces ALL instances of a type globally: "every base_txn becomes
error_txn." Instance override replaces only specific instances identified by hierarchical path:
"only env.agent.driver's base_txn becomes error_txn." Instance overrides take priority over
type overrides. Both are set during build_phase before objects are created.

### Q10: Explain soft constraints and when to use them.

**A:** Soft constraints provide default values/ranges that can be overridden by hard constraints
without conflict. They are useful in reusable components: the base class provides reasonable
defaults (e.g., `soft addr < 64`), and specific tests override them with inline constraints
(e.g., `with { addr > 200; }`). The hard constraint silently overrides the soft one. Without
`soft`, both constraints would conflict and `randomize()` would fail.

### Q11: Write a constraint for an array where each element is greater than the previous.

```verilog
class sorted_array;
    rand int arr[10];

    constraint c_sorted {
        foreach (arr[i])
            if (i > 0)
                arr[i] > arr[i-1];
    }

    constraint c_range {
        foreach (arr[i])
            arr[i] inside {[0:100]};
    }
endclass
```

### Q12: What is random stability and why does it matter for debug?

**A:** Random stability means that a fixed seed produces the same random sequence regardless
of unrelated code changes. SystemVerilog provides thread stability (each process has its own
RNG (Random Number Generator)) and type stability (each class type has independent state). This matters because: (1) you
can reproduce failures with `+seed=N`; (2) adding a printf in one component doesn't change
the random sequence in another. However, changing object creation ORDER or adding new
randomize() calls within the SAME thread will change that thread's sequence.

### Q13: Explain $cast -- when does it succeed and when does it fail?

**A:** `$cast(target, source)` succeeds when the actual object type of `source` is the same as
or derived from `target`'s type. It fails (returns 0) when the object doesn't match -- e.g.,
casting an Animal object to a Dog handle when the object is actually a Cat. As a function, it
returns 0 on failure. As a task (`$cast(target, source)` without checking return), it throws a
runtime error on failure. Always use the function form and check the return value.

### Q14: How does constraint_mode differ from rand_mode?

**A:** `rand_mode(0)` disables randomization of a specific variable -- it keeps its current
value during randomize(). `constraint_mode(0)` disables a specific named constraint block --
the variable is still randomized but that particular constraint no longer applies. They are
complementary: rand_mode controls WHAT gets randomized, constraint_mode controls HOW it's
constrained.

### Q15: Write a class that generates unique random addresses without repetition until all are used.

```verilog
class unique_addr_gen;
    randc bit [7:0] addr;  // randc cycles through all 256 values

    constraint c_valid {
        addr inside {[8'h10:8'hEF]};  // Exclude reserved ranges
    }

    // randc ensures: 0x10, 0x11, ..., 0xEF all appear once before any repeat
endclass
```

### Q16: Explain the difference between `extends` and `implements` in SystemVerilog.

**A:** `extends` creates a single-inheritance class hierarchy -- a derived class inherits all
fields, methods, and constraints from one parent class. `implements` (SV-2012) allows a class
to declare conformance to one or more interface classes, which define pure virtual method
signatures. A class can extend ONE class but implement MULTIPLE interface classes. This provides
a limited form of multiple inheritance (Java-style interfaces).

---

## SystemVerilog Procedural Blocks and Processes -- Senior Engineer Deep Dive

*From [Procedural_Processes_and_IPC.md](../03_Frontend_RTL_and_Verification/03_Procedural_Processes_and_IPC.md)*

### Q1: Explain the SystemVerilog scheduling regions and why they matter.

**A:** The time slot has regions: Preponed (sample), Active (RTL logic, blocking assigns),
Inactive (#0 delays), NBA (non-blocking assigns), Observed (assertions), Reactive (program
block/testbench code), Postponed ($strobe/$monitor). They matter because: (1) NBA ensures
sequential logic doesn't have read-after-write races; (2) Observed region lets assertions see
settled RTL values; (3) Reactive region lets testbench drive signals after RTL settles. Not
understanding regions causes simulation races and non-deterministic behavior.

### Q2: What are the differences between always_comb and always @(*)?

**A:** Two differences: (1) `always_comb` triggers at time 0 even if no input changes --
`always @(*)` only triggers when a sensitivity-list signal changes. (2) `always_comb` includes
signals read inside function calls in its sensitivity; `always @(*)` only infers sensitivity
from directly-read signals, not function internals. Also, `always_comb` enforces no timing
controls and warns about unintended latch inference.

### Q3: Show the fork-join for-loop bug and explain two ways to fix it.

**A:** The bug is that fork inside a for-loop captures the loop variable by reference, not by
value. By the time spawned processes execute, the loop variable has already reached its final
value. Fix 1: `automatic int j = i;` before the fork captures a copy. Fix 2: Declare
`automatic` variable inside the forked begin-end block. (See code examples in the fork-join
section above.)

### Q4: What does disable fork actually disable?

**A:** `disable fork` disables ALL child processes spawned by the current process. This is
broader than most people expect -- it doesn't just disable the most recent fork, it kills ALL
descendants. To limit scope, use the isolation wrapper pattern: wrap the target fork inside
`fork begin ... end join` so that `disable fork` inside only kills children of that wrapper
process.

### Q5: Why should module tasks default to automatic in a verification context?

**A:** Static tasks share local variables across concurrent calls. In verification, tasks are
frequently called concurrently (e.g., multiple sequence items driving through the same driver
task). Static variables cause silent data corruption between concurrent invocations. Class
methods are always automatic, which is why UVM code doesn't hit this bug. But if you write
module-level helper tasks, you MUST declare them `automatic`.

### Q6: Explain clocking block input #1step and output #0.

**A:** `input #1step` samples the signal in the Preponed region, which is the very start of
the time slot before ANY logic executes. This gives the testbench a clean, race-free view of
DUT outputs as they were at the END of the previous cycle. `output #0` drives in the current
time slot, allowing the driven value to propagate through RTL in the Active region.

### Q7: Why was the program block deprecated for UVM testbenches?

**A:** Program blocks were designed for simple directed tests: enter Reactive region, drive
stimulus, and call `$finish` when all initial blocks complete. UVM needs: (1) persistent
component objects that outlive a single initial block; (2) phase-based simulation control
(not implicit `$finish`); (3) arbitrary fork-join_none spawning; (4) dynamic creation of
sequence items. The program block's restrictions conflict with all of these.

### Q8: What is the difference between blocking (=) and non-blocking (<=) assignments?

**A:** Blocking assignments execute sequentially in the Active region -- the LHS (left-hand side) updates
immediately before the next statement. Non-blocking assignments evaluate the RHS (right-hand side) in Active
but schedule the LHS update to the NBA region (after all Active evaluations). This means all
non-blocking RHS values are sampled before any LHS updates, preventing read-after-write races
in sequential logic. Rule: use `=` in combinational logic (`always_comb`), `<=` in sequential
logic (`always_ff`).

### Q9: How do you implement a timeout with fork-join_any?

```verilog
task run_with_timeout(int timeout_cycles);
    fork
        begin  // Main work
            do_complex_operation();
        end
        begin  // Timeout
            repeat (timeout_cycles) @(posedge clk);
            `uvm_error("TIMEOUT", "Operation timed out")
        end
    join_any
    disable fork;  // Kill the loser (but use isolation wrapper in real code!)
endtask
```

### Q10: What happens if you call a task from a function?

**A:** Compile error. Functions cannot consume simulation time, and tasks can contain delays.
If a function could call a task, the function might block, violating its zero-time contract.
A function CAN call another function. A task CAN call both tasks and functions.

### Q11: Explain the difference between wait(signal) and @(posedge signal).

**A:** `wait(signal)` is level-sensitive -- if signal is already high, it passes immediately
without blocking. `@(posedge signal)` is edge-sensitive -- it always blocks until the NEXT
rising edge, even if signal is already high. Common bug: using `wait(clk)` instead of
`@(posedge clk)` -- if clk is already high, the wait returns immediately.

### Q12: What is process::self() and when would you use it?

**A:** `process::self()` returns a handle to the currently executing process. Use it to:
(1) save a handle for later `kill()` -- e.g., watchdog timers; (2) check process status;
(3) implement process management patterns where one process controls another's lifecycle.

### Q13: Show how always_ff enforces coding style for synthesis.

**A:** `always_ff` tells both the simulator and synthesis tool that this block should infer
flip-flops. It enforces: (1) exactly one event control (clock edge, optional reset); (2) only
non-blocking assignments (some tools warn/error on `=`); (3) signals assigned here cannot be
driven elsewhere. Synthesis tools use this intent to map directly to register primitives without
guessing.

### Q14: What is the difference between `wait fork` and `join`?

**A:** `join` at the end of a `fork` block waits for all processes IN that fork to complete.
`wait fork` is a standalone statement that waits for ALL outstanding child processes of the
current process to complete, including those from previous `fork-join_none` or still-running
processes from `fork-join_any`. `wait fork` has broader scope.

```verilog
initial begin
    fork task_a(); join_none  // Spawns task_a
    fork task_b(); join_none  // Spawns task_b
    wait fork;                // Waits for BOTH task_a and task_b
end
```

### Q15: How does $urandom_range differ from $urandom?

**A:** `$urandom` returns a 32-bit unsigned random number. `$urandom_range(max, min)` returns
a value uniformly distributed between min and max (inclusive). Both are **thread-stable** --
each process has its own random state, so reordering unrelated code doesn't change the random
sequence of other processes. The older `$random` is NOT thread-stable and should be avoided.

---

## RTL Design Methodology — Synchronous Discipline, Reset, Clocking, and Structure

*From [RTL_Design_Methodology.md](../03_Frontend_RTL_and_Verification/01_RTL_Design_Methodology.md)*

**Q: Why async-assert, sync-deassert reset?** Async assert resets the chip even with no clock (power-up). Synchronous de-assert guarantees no flop releases reset inside its recovery/removal window, which would otherwise drive it metastable. The reset synchronizer (2 FF, async-clear, D tied high) implements exactly this.

**Q: You see an inferred latch in the synthesis report. Cause and fix?** A combinational (`always_comb`) block that doesn't assign an output on every path — typically an `if` with no `else` or an incomplete `case`. Fix: default-assign all outputs at the top, or add `else`/`default`.

**Q: When clock-enable vs clock-gating?** Clock-enable (`if (en) q<=d`) keeps one clock and is trivial to verify and time — use it for functional control. Clock-gating (ICG) actually stops the clock to save dynamic power — use it for power, inserted as an ICG cell, never as combinational logic on the clock.

---

## UVM Methodology — Components, Phasing, Sequences, Factory, RAL

*From [UVM_Methodology.md](../03_Frontend_RTL_and_Verification/10_UVM_Methodology.md)*

**Q: Why does UVM need a factory at all — isn't `new()` polymorphic enough?**
**A:** Polymorphism needs the *caller* to choose the subclass at construction. The env code is frozen/reusable; only the test knows it wants `bad_parity_item`. The factory inverts control: env asks for a type by name, test injects the override. Construction-site polymorphism without editing the construction site.

**Q: Monitor vs driver — can the monitor reuse the driver's pin code?**
**A:** No — monitor must be passive (observe-only) and must work when the agent is passive (driver doesn't exist) and at system level where stimulus comes from real RTL neighbors. Independent observation also catches driver bugs; sharing code would verify the driver against itself.

**Q: A test hangs at the end of stimulus. Triage tree?**
**A:** (1) objection never dropped — `+UVM_OBJECTION_TRACE` shows the holder; (2) `finish_item` blocked — driver missing `item_done` (or driver never got the vif and is stuck at time 0); (3) `get_response` without responses; (4) forever-loop sequence started with `start()` (blocking) instead of fork. In that order of frequency.

**Q: Why are sequences objects, not components?**
**A:** They're transient *programs* over the static testbench: created per-run, possibly many concurrent, layered/nested, and they must travel (start on any matching sequencer). Components are fixed topology built at elaboration; stimulus must not be.

**Q: When do you reach for `grab()` over priority arbitration?**
**A:** Atomicity, not preference: a multi-item protocol unit (e.g., locked RMW (Read-Modify-Write), interrupt service burst) where interleaving any other sequence's item corrupts the protocol. Priority biases the long-run mix; grab guarantees an uninterrupted window.

**Q: How does the scoreboard learn about register side-effects (e.g., write to CTRL flushes a FIFO)?**
**A:** Subscribe the explicit RAL predictor to the config-bus monitor; scoreboard queries the register model's mirror (or the predictor publishes typed "config change" analysis transactions). DUT behavioral model keys off mirrored config — never off the stimulus side, or passive/firmware accesses break it.

---

## Verification Planning and Coverage Closure

*From [Verification_Planning_and_Coverage_Closure.md](../03_Frontend_RTL_and_Verification/11_Verification_Planning_and_Coverage_Closure.md)*

**Q: 100% code coverage — are you done?** No. Code coverage only proves the lines *executed*, not that the checks would have caught a bug, and it says nothing about scenarios the spec cares about (functional coverage) or combinations (cross coverage). 100% code + weak checkers can ship a broken chip. Closure needs functional coverage + passing checks too.

**Q: A cover point won't hit after a long random regression. What do you do?** Triage: is it reachable with more seeds/looser constraints (add them)? Reachable only by a specific sequence (write a directed test)? Genuinely unreachable (dead/illegal — waive with justification or fix the coverage model)? Or reachable-but-the-test-fails-there (a real DUT bug)?

**Q: How do constrained-random and functional coverage relate?** They're the two halves of CDV: random stimulus reaches breadth and corners cheaply; functional coverage measures what was actually exercised; the un-hit bins feed back as new constraints/seeds (or coverage-driven generation). One produces volume, the other measures meaning.

