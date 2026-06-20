# SystemVerilog Assertions and Coverage -- Senior Engineer Deep Dive

---

## Immediate Assertions

Executed as procedural statements at the point they appear in code.

```systemverilog
// Simple immediate assertion
always_comb begin
    assert (state != ILLEGAL_STATE)
        else $error("Illegal state: %s", state.name());
end

// Three forms: assert, assume, cover
always @(posedge clk) begin
    a_valid: assert (data !== 'x)
        $display("Data valid");      // Pass action
    else
        $error("Data is X at T=%0t", $time);  // Fail action

    a_input: assume (req inside {0, 1});  // Input constraint (formal)

    c_full: cover (fifo_full)             // Reachability check
        $display("FIFO full observed");
end
```

### Deferred Immediate Assertions

Standard immediate assertions fire in the Active region, so they can trigger on combinational
glitches before values settle. Deferred assertions fix this.

```systemverilog
always_comb begin
    // #0: deferred to Observed region (after all Active/NBA settle)
    assert #0 (onehot(grant))
        else $error("Grant not one-hot: %b", grant);

    // "final": deferred to Reactive region (after Observed)
    assert final (checksum_valid)
        else $error("Checksum failed");
end
```

**When to use:** Always prefer `assert #0` or `assert final` in combinational logic to avoid
false failures from glitches during delta cycles.

---

## SVA (Concurrent Assertions) Deep Dive

### Sampled Values and the Preponed Region

Concurrent assertions use **sampled values**: the value of a signal at the END of the previous
time step (technically, the Preponed region of the current time step). This avoids race
conditions between assertion evaluation and RTL updates.

```systemverilog
// At posedge clk, the assertion sees the values that were STABLE
// before the clock edge, not the values being computed in this cycle
property p_stable;
    @(posedge clk) req |-> data == $past(data);
endproperty

// GOTCHA: if a signal changes in the same cycle as the clock edge,
// the assertion sees the OLD value (from before the edge)
always @(posedge clk) data <= new_data;  // NBA update
// Assertion sees data's value from BEFORE this NBA update
```

### Where Assertions Execute in the Scheduler

```
Preponed:   Sample signal values for assertions
Active:     RTL logic executes
NBA:        Non-blocking assignments
Observed:   CONCURRENT ASSERTIONS EVALUATED HERE (using Preponed samples)
Reactive:   Program block code, assertion pass/fail actions
```

---

## Sequence Operators with Exact Semantics

### ##N: Fixed Cycle Delay

```
##N means "N clock cycles later"

Signal:   req ----+          ack ----+
                  |                  |
Clock:    __|--|__|--|__|--|__|--|__|--|
Cycle:      0     1     2     3     4

req ##2 ack: req at cycle 0, ack at cycle 2
```

```systemverilog
sequence s_req_ack;
    req ##2 ack;  // req true now, ack true exactly 2 cycles later
endsequence
```

### ##[M:N]: Window (Range Delay)

```systemverilog
sequence s_window;
    req ##[1:3] ack;  // req now, ack true 1, 2, OR 3 cycles later
endsequence

// CRITICAL: this creates MULTIPLE threads!
// At time of req, the checker spawns evaluation threads for:
//   Thread 1: check ack at cycle+1
//   Thread 2: check ack at cycle+2
//   Thread 3: check ack at cycle+3
// If ANY thread matches, the sequence matches
// If used with |-> and NONE match, assertion fails

// ##[0:$] means "zero or more cycles" ($ = unbounded)
// ##[1:$] means "eventually" -- ack must happen at some future cycle
```

### [*N]: Consecutive Repetition

```systemverilog
// Signal must be true for N consecutive cycles
sequence s_stable;
    valid[*4];  // valid must be 1 for 4 consecutive cycles
endsequence

// [*M:N]: range of consecutive repetitions
sequence s_hold;
    valid[*2:5];  // valid true for 2 to 5 consecutive cycles
endsequence

// [*0]: zero repetitions (empty sequence -- matches immediately)
// [*0:N]: optional presence (0 to N consecutive)
```

### [->N]: Goto Repetition (THE COMMON INTERVIEW QUESTION)

```
[->N] means: eventually true for the Nth time, AND the match point
is at that Nth occurrence.

Waveform for ack[->2]:
Cycle:   1  2  3  4  5  6  7
ack:     0  1  0  0  1  0  0
              ^           ^
              1st         2nd (MATCH POINT IS HERE -- cycle 5)
```

```systemverilog
sequence s_goto;
    req ##1 ack[->2];
    // req at cycle 0
    // then ack goes high for the 2nd time (non-consecutive OK)
    // match ends at the cycle where ack is high the 2nd time
endsequence
```

### [=N]: Non-Consecutive Repetition

```
[=N] means: true for N times total (not necessarily consecutive).
Unlike [->N], the match does NOT have to end on the Nth occurrence.

Waveform for ack[=2]:
Cycle:   1  2  3  4  5  6  7
ack:     0  1  0  0  1  0  0
              ^        ^
              1st      2nd occurrence met
              But match point can be cycles 5, 6, OR 7
```

### THE CRITICAL DIFFERENCE: [->2] vs [=2]

```
Consider: req ##1 ack[->2] ##1 done
vs:       req ##1 ack[=2]  ##1 done

Timeline:
Cycle:    0    1    2    3    4    5    6
req:      1
ack:           0    1    0    1    0    0
done:                                   ?

[->2]: Match point is cycle 4 (2nd ack). Then ##1 checks done at cycle 5.
[=2]:  Match point is cycle 4 OR LATER. Then ##1 checks done at cycle 5, 6, 7...
       The [=2] is satisfied once 2 acks occur, but it doesn't end there.
       It allows more non-ack cycles before ##1 done.

Key: [->N] ENDS at Nth occurrence.
     [=N] requires N occurrences but allows trailing cycles.
```

```systemverilog
// Interview example: check that exactly 2 acks occur, then done on next cycle
// Use [->2] (not [=2]) because you want the sequence to end at the 2nd ack:
property p_two_acks_then_done;
    @(posedge clk) req |-> ##1 ack[->2] ##1 done;
endproperty
```

---

## Implication Operators

### |-> Overlapping (Same Cycle)

```
The consequent starts in the SAME cycle the antecedent completes.

Timeline for: req |-> ack
Cycle:  0    1    2
req:    1
ack:    1              <-- checked in same cycle as req

Timeline for: (req ##2 grant) |-> ack
Cycle:  0    1    2
req:    1
grant:            1
ack:              1    <-- checked at cycle 2 when grant completes
```

### |=> Non-Overlapping (Next Cycle)

```
The consequent starts ONE cycle AFTER the antecedent completes.
Equivalent to: |-> ##1

Timeline for: req |=> ack
Cycle:  0    1    2
req:    1
ack:         1         <-- checked one cycle AFTER req

Timeline for: (req ##2 grant) |=> ack
Cycle:  0    1    2    3
req:    1
grant:            1
ack:                   1  <-- checked at cycle 3 (one after grant)
```

### Vacuous Truth

If the antecedent never matches, the implication is **vacuously true** (passes by default).
This is correct mathematically but can hide bugs.

```systemverilog
// If req never goes high, this assertion ALWAYS passes -- no checking done!
property p_req_ack;
    @(posedge clk) req |-> ##[1:5] ack;
endproperty

// To detect vacuous passes, use cover to verify the antecedent fires:
cover property (@(posedge clk) req);
// If this never hits, your assertion is vacuously passing and not testing anything
```

---

## first_match

```systemverilog
// Without first_match, ##[1:5] creates 5 threads:
property p_multi_thread;
    @(posedge clk) req |-> ##[1:5] ack;
endproperty
// 5 separate evaluation threads -- expensive, confusing failure messages

// With first_match, only the earliest matching thread survives:
property p_first;
    @(posedge clk) req |-> first_match(##[1:5] ack);
endproperty
// Only one thread succeeds -- cleaner, lower simulator overhead

// When it matters: in complex sequences with multiple windows,
// thread count can explode exponentially. first_match prevents this.
```

---

## disable iff

```systemverilog
property p_handshake;
    @(posedge clk) disable iff (!rst_n)  // Disable during reset
    req |-> ##[1:10] ack;
endproperty

// GOTCHA: disable iff is ASYNCHRONOUS -- checked every evaluation,
// not just at clock edges. A glitch on rst_n during the cycle can
// cause a vacuous pass.

// If rst_n has a short low pulse (glitch) that happens between clock edges,
// the assertion may be disabled even though the RTL didn't see a reset.

// Recommendation: use a synchronous version when possible:
property p_handshake_sync;
    @(posedge clk)
    (!rst_n) || (req |-> ##[1:10] ack);
endproperty
```

---

## System Functions for Assertions

### $rose, $fell, $stable

```systemverilog
// $rose(sig): true if sig changed from 0 to 1 (based on SAMPLED values)
// $fell(sig): true if sig changed from 1 to 0
// $stable(sig): true if sig didn't change

property p_valid_rose;
    @(posedge clk) $rose(valid) |-> !$isunknown(data);
endproperty
// When valid rises, data must not be X

// GOTCHA: $rose/$fell compare current and previous SAMPLED values
// At the very first clock edge, previous value is X (4-state) or 0 (2-state)
// $rose(sig) at first edge: true if sig is 1 (previous was X->treated as 0)
```

### $past

```systemverilog
// $past(signal, N) -- value of signal N cycles ago (default N=1)
property p_data_held;
    @(posedge clk) valid && !ready |-> ##1 (data == $past(data));
endproperty
// If valid but not ready, data must be held stable next cycle

// $past with clock gating:
// $past(signal, N, gating_signal, @(posedge clk))
// Returns value of signal N clock edges ago WHEN gating_signal was true
```

### $countones, $onehot, $isunknown

```systemverilog
// $onehot(expr): exactly one bit is 1
property p_onehot_grant;
    @(posedge clk) |grant |-> $onehot(grant);
endproperty
// If any grant is active, exactly one must be active

// $onehot0(expr): zero or one bit is 1
// $countones(expr): number of 1-bits
// $isunknown(expr): true if any bit is X or Z

property p_no_x;
    @(posedge clk) disable iff (!rst_n)
    !$isunknown(data_bus);
endproperty
```

---

## Bind Construct

### Why Bind Is Essential

You CANNOT modify RTL source code to add verification assertions (IP may be encrypted, or you
need clean RTL for synthesis). `bind` lets you attach assertion modules to RTL modules from
outside.

```systemverilog
// assertion module -- separate file
module fifo_assertions (
    input logic        clk,
    input logic        rst_n,
    input logic        wr_en,
    input logic        rd_en,
    input logic        full,
    input logic        empty,
    input logic [7:0]  wr_data,
    input logic [7:0]  rd_data,
    input logic [3:0]  count
);
    // Never write when full
    property p_no_write_when_full;
        @(posedge clk) disable iff (!rst_n)
        full |-> !wr_en;
    endproperty
    assert property (p_no_write_when_full)
        else $error("Write attempted on full FIFO");

    // Never read when empty
    property p_no_read_when_empty;
        @(posedge clk) disable iff (!rst_n)
        empty |-> !rd_en;
    endproperty
    assert property (p_no_read_when_empty)
        else $error("Read attempted on empty FIFO");

    // Count consistency
    property p_count;
        @(posedge clk) disable iff (!rst_n)
        (wr_en && !rd_en && !full) |=> (count == $past(count) + 1);
    endproperty
    assert property (p_count)
        else $error("Count mismatch after write");
endmodule

// Bind statement -- in testbench top or separate file
bind fifo_rtl fifo_assertions u_fifo_asserts (
    .clk     (clk),
    .rst_n   (rst_n),
    .wr_en   (wr_en),
    .rd_en   (rd_en),
    .full    (full),
    .empty   (empty),
    .wr_data (wr_data),
    .rd_data (rd_data),
    .count   (count)
);
// This instantiates fifo_assertions INSIDE every instance of fifo_rtl
// without modifying fifo_rtl source code
```

### Bind to Specific Instance

```systemverilog
// Bind to ALL instances of a module type:
bind fifo_rtl fifo_assertions u_assert (...);

// Bind to a SPECIFIC instance by hierarchical path:
bind tb.dut.u_tx_fifo fifo_assertions u_assert (...);
```

---

## Real Protocol Assertions

### AXI Handshake Protocol

```systemverilog
module axi_protocol_checker (
    input logic        ACLK,
    input logic        ARESETn,
    input logic        AWVALID,
    input logic        AWREADY,
    input logic [31:0] AWADDR,
    input logic        WVALID,
    input logic        WREADY,
    input logic [31:0] WDATA,
    input logic        BVALID,
    input logic        BREADY
);
    // AXI RULE: Once VALID is asserted, it must not deassert until READY
    property p_valid_stable(valid, ready);
        @(posedge ACLK) disable iff (!ARESETn)
        valid && !ready |=> valid;
    endproperty

    assert property (p_valid_stable(AWVALID, AWREADY))
        else $error("AWVALID deasserted before AWREADY handshake");
    assert property (p_valid_stable(WVALID, WREADY))
        else $error("WVALID deasserted before WREADY handshake");
    assert property (p_valid_stable(BVALID, BREADY))
        else $error("BVALID deasserted before BREADY handshake");

    // AXI RULE: Data must be stable while VALID is high and READY is low
    property p_data_stable(valid, ready, data);
        @(posedge ACLK) disable iff (!ARESETn)
        valid && !ready |=> $stable(data);
    endproperty

    assert property (p_data_stable(AWVALID, AWREADY, AWADDR))
        else $error("AWADDR changed before handshake complete");
    assert property (p_data_stable(WVALID, WREADY, WDATA))
        else $error("WDATA changed before handshake complete");

    // AXI RECOMMENDATION: VALID should not depend on READY
    // (VALID can assert independently of READY's state)
    // This is a design guideline, not always assertable directly.
    // One way: check VALID can rise without READY being high
    property p_valid_no_wait_ready(valid, ready);
        @(posedge ACLK) disable iff (!ARESETn)
        $rose(valid) |-> 1;  // VALID can always rise (vacuously true check)
    endproperty

    // Handshake must complete within N cycles (liveness)
    property p_handshake_timeout(valid, ready);
        @(posedge ACLK) disable iff (!ARESETn)
        valid |-> ##[0:100] (valid && ready);  // Complete within 100 cycles
    endproperty

    assert property (p_handshake_timeout(AWVALID, AWREADY))
        else $error("AW handshake timeout");
endmodule
```

### Arbiter Assertions

```systemverilog
module arbiter_checker #(parameter N = 4) (
    input logic       clk,
    input logic       rst_n,
    input logic [N-1:0] req,
    input logic [N-1:0] grant
);
    // Mutual exclusion: at most one grant at a time
    property p_mutex;
        @(posedge clk) disable iff (!rst_n)
        |grant |-> $onehot(grant);
    endproperty
    assert property (p_mutex)
        else $error("Multiple grants: %b", grant);

    // Grant only if requested
    property p_grant_needs_req;
        @(posedge clk) disable iff (!rst_n)
        |grant |-> |(grant & req);
    endproperty
    assert property (p_grant_needs_req)
        else $error("Grant without request");

    // No starvation: every persistent request eventually granted
    generate
        for (genvar i = 0; i < N; i++) begin : gen_fair
            property p_no_starvation;
                @(posedge clk) disable iff (!rst_n)
                req[i] |-> ##[0:100] grant[i];
            endproperty
            assert property (p_no_starvation)
                else $error("Starvation: req[%0d] not granted in 100 cycles", i);
        end
    endgenerate

    // No grant during reset
    property p_no_grant_in_reset;
        @(posedge clk) !rst_n |-> grant == '0;
    endproperty
    assert property (p_no_grant_in_reset);
endmodule
```

### FIFO Data Integrity

```systemverilog
module fifo_data_checker #(parameter DEPTH = 16, WIDTH = 8) (
    input logic             clk,
    input logic             rst_n,
    input logic             wr_en,
    input logic             rd_en,
    input logic [WIDTH-1:0] wr_data,
    input logic [WIDTH-1:0] rd_data,
    input logic             full,
    input logic             empty
);
    // Reference model for data integrity
    logic [WIDTH-1:0] ref_queue[$];

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            ref_queue.delete();
        end else begin
            if (wr_en && !full)
                ref_queue.push_back(wr_data);
            if (rd_en && !empty) begin
                automatic logic [WIDTH-1:0] expected = ref_queue.pop_front();
                assert (rd_data == expected)
                    else $error("Data mismatch: expected %h, got %h", expected, rd_data);
            end
        end
    end

    // Overflow protection
    property p_no_overflow;
        @(posedge clk) disable iff (!rst_n)
        full |-> !wr_en;
    endproperty

    // Underflow protection
    property p_no_underflow;
        @(posedge clk) disable iff (!rst_n)
        empty |-> !rd_en;
    endproperty

    assert property (p_no_overflow)
        else $error("FIFO overflow: write while full");
    assert property (p_no_underflow)
        else $error("FIFO underflow: read while empty");
endmodule
```

---

## Functional Coverage Deep Dive

### Covergroups and Coverpoints

```systemverilog
class my_coverage extends uvm_subscriber #(my_txn);
    `uvm_component_utils(my_coverage)

    my_txn txn;

    covergroup cg_txn @(cov_event);
        // Coverpoint with automatic bins
        cp_addr: coverpoint txn.addr {
            bins low_addr    = {[0:63]};
            bins mid_addr    = {[64:191]};
            bins high_addr   = {[192:255]};
            bins zero        = {0};
            bins max_val     = {255};
            illegal_bins reserved = {[240:249]};
            ignore_bins dont_care = {128};
        }

        // Coverpoint with automatic bins (auto-created per value)
        cp_opcode: coverpoint txn.opcode {
            bins opcodes[] = {[0:15]};  // 16 individual bins
        }

        // Transition coverage
        cp_state: coverpoint txn.state {
            bins idle_to_active = (IDLE => ACTIVE);
            bins active_to_done = (ACTIVE => DONE);
            bins full_seq       = (IDLE => ACTIVE => DONE => IDLE);
            bins any_to_error   = (IDLE, ACTIVE, DONE => ERROR);
        }

        // Wildcard bins
        cp_cmd: coverpoint txn.cmd {
            wildcard bins read_cmds  = {4'b00??};  // Any read command
            wildcard bins write_cmds = {4'b01??};  // Any write command
        }
    endgroup

    function new(string name, uvm_component parent);
        super.new(name, parent);
        cg_txn = new();
    endfunction

    function void write(my_txn t);
        txn = t;
        cg_txn.sample();
    endfunction
endclass
```

### Cross Coverage and the Explosion Problem

```systemverilog
covergroup cg_cross;
    cp_op:   coverpoint txn.opcode   { bins ops[] = {[0:7]}; }   // 8 bins
    cp_size: coverpoint txn.size     { bins sizes[] = {1,2,4,8}; } // 4 bins
    cp_addr: coverpoint txn.addr_type { bins types[] = {[0:3]}; }  // 4 bins
    cp_prot: coverpoint txn.prot     { bins prots[] = {[0:3]}; }   // 4 bins

    // FULL CROSS: 8 * 4 * 4 * 4 = 512 bins
    // This is manageable, but add more coverpoints and it explodes

    cx_all: cross cp_op, cp_size, cp_addr, cp_prot;
endgroup

// Managing cross coverage explosion:
covergroup cg_managed;
    cp_op:   coverpoint txn.opcode   { bins ops[] = {[0:7]}; }
    cp_size: coverpoint txn.size     { bins sizes[] = {1,2,4,8}; }
    cp_addr: coverpoint txn.addr_type { bins types[] = {[0:3]}; }

    // Selective cross -- only interesting combinations
    cx_op_size: cross cp_op, cp_size {
        // Ignore irrelevant combinations
        ignore_bins ig = binsof(cp_op) intersect {0,1}
                      && binsof(cp_size) intersect {8};
        // Read ops (0,1) never use size 8

        // Only care about specific combos
        bins write_large = binsof(cp_op) intersect {2,3}
                        && binsof(cp_size) intersect {4,8};
    }
endgroup
```

### Coverage Options

```systemverilog
covergroup cg_options;
    option.per_instance = 1;     // Track coverage per instance (not merged)
    option.at_least = 5;         // Each bin needs 5 hits (default: 1)
    option.auto_bin_max = 64;    // Max auto-generated bins (default: 64)
    option.weight = 2;           // Weight in overall coverage calculation
    option.goal = 90;            // Coverage goal percentage

    cp_addr: coverpoint addr {
        option.auto_bin_max = 8; // Override per-coverpoint
    }
endgroup
```

---

## Coverage-Driven Verification Methodology

### The Feedback Loop

```
+------------------+      +-------------------+      +------------------+
| Write Coverage   | ---> | Run Regressions   | ---> | Analyze Coverage |
| Model from Spec  |      | (random + directed)|      | Reports          |
+------------------+      +-------------------+      +------------------+
        ^                                                      |
        |                                                      v
        |                                             +------------------+
        +------------ Identify Holes <-------------- | Coverage Holes?  |
        |             Tune constraints                | Missing combos?  |
        |             Add directed tests              +------------------+
        |             Adjust weights                          |
        +-----------------------------------------------------+
```

### Methodology Stages

1. **Feature extraction**: Read the spec, identify functional features, corner cases, error
   scenarios, boundary conditions.

2. **Coverage model design**: Create covergroups for:
   - Input stimulus coverage (did we send all transaction types?)
   - Output/status coverage (did we see all response types?)
   - Cross coverage (interesting combinations)
   - Transition coverage (state machine paths)
   - Corner cases (boundary values, back-to-back, empty/full)

3. **Initial regression**: Run random tests, check coverage.

4. **Coverage closure**: Iterate on constraints, add directed sequences for hard-to-hit bins.

5. **When 100% is not needed**: illegal/impossible states (use `illegal_bins`), don't-care
   combinations (`ignore_bins`), or when diminishing returns make the last 2% not worth the
   effort. Document why each uncovered bin is excluded.

### Coverage Merging

```systemverilog
// Per-instance vs type coverage:
// option.per_instance = 1: each component tracks its own coverage
//   Useful: "did agent0 AND agent1 both see all opcodes?"
// option.per_instance = 0 (default): all instances merge into one
//   Useful: "across all agents, did we see all opcodes?"

// Across regression runs, coverage databases are merged:
// Simulator command: vcover merge total.ucdb run1.ucdb run2.ucdb run3.ucdb
// This accumulates hits across all seeds/tests
```

---

## Multi-Clock Assertions

```systemverilog
// Sequence spanning two clock domains
// (Use with caution -- multi-clock assertions are complex)
sequence s_cross_domain;
    @(posedge clk_a) req_a ##1 @(posedge clk_b) ack_b;
endsequence

// The ##1 here means: one tick of clk_b AFTER the posedge clk_a event
// Not one cycle of clk_a!
// This is useful for CDC (Clock Domain Crossing) verification
```

---

## Complete SVA Property Operator Reference

One worked example per operator -- the cheat sheet you need in an interview.

### $rose / $fell / $stable

```systemverilog
// $rose(sig): true when sampled value changed from 0→1 since previous clock edge
// $fell(sig): true when sampled value changed from 1→0
// $stable(sig): true when sampled value is same as previous clock edge

// Example: req must stay high until ack (both edges are meaningful)
property p_req_holds;
    @(posedge clk) $rose(req) |-> req throughout ##[1:20] $fell(req);
endproperty
// When req rises, it must stay 1 until it eventually falls

// Example: data must be stable when valid is high
property p_data_stable_with_valid;
    @(posedge clk) valid |-> $stable(data);
endproperty
```

### $past

```systemverilog
// $past(expr, N, gate, clk): value of expr N clock ticks ago
// Default N=1, gate=1 (ungated), clock = current clock

// Example: counter should increment by 1
property p_cnt_incr;
    @(posedge clk) disable iff (!rst_n)
    cnt == $past(cnt) + 1;
endproperty

// Example with gating: check data matches what was on bus when en was asserted
property p_gated_past;
    @(posedge clk)
    $rose(ready) |-> data == $past(wr_data, 1, wr_en, @(posedge clk));
endproperty
// Returns value of wr_data from the last cycle where wr_en was 1
```

### $onehot / $onehot0 / $countones / $isunknown

```systemverilog
// $onehot(expr): exactly one bit is 1
// $onehot0(expr): at most one bit is 1 (zero or one)
// $countones(expr): number of 1 bits
// $isunknown(expr): any bit is X or Z

// Example: arbiter one-hot grant with X detection
property p_grant_valid;
    @(posedge clk) disable iff (!rst_n)
    |grant |-> $onehot(grant) && !$isunknown(grant);
endproperty

// Example: bus ownership -- at most one master active
property p_bus_owner;
    @(posedge clk) $onehot0(bus_req);
endproperty
```

### ##N (fixed delay) and ##[M:N] (range delay)

```systemverilog
// ##N: exact N cycles later
// ##[M:N]: between M and N cycles later (creates N-M+1 threads)
// ##[0:$]: zero or more cycles (unbounded)

// Example: pipeline with known latency of 3 cycles
property p_pipe_latency;
    @(posedge clk) $rose(issue) |-> ##3 $rose(done);
endproperty

// Example: interrupt must be serviced within 10-100 cycles
property p_irq_service;
    @(posedge clk) $rose(irq) |-> ##[1:100] $rose(irq_ack);
endproperty
```

### [*N] (consecutive repetition) and [*M:N]

```systemverilog
// expr[*N]: expr true for N consecutive cycles
// expr[*M:N]: expr true for M to N consecutive cycles
// expr[*0]: empty match (matches immediately, zero-length)

// Example: reset must be held low for at least 5 cycles
sequence s_reset_duration;
    !rst_n[*5];
endsequence

// Example: backoff period -- bus idle for 3-8 cycles between transactions
property p_backoff;
    @(posedge clk) $fell(bus_valid) |=> !bus_valid[*3:8] ##1 bus_valid;
endproperty
```

### [->N] (goto repetition)

```systemverilog
// expr[->N]: expr becomes true for the Nth time (non-consecutive OK)
// Match point IS at the Nth occurrence.
// Equivalent to: (!expr[*0:$] ##1 expr) repeated N times

// Example: receive exactly 4 data beats, then assert last
property p_rx_4_beats;
    @(posedge clk) $rose(rx_start) |-> data_valid[->4] ##1 rx_last;
endproperty
// Matches: rx_start → (eventually data_valid 4 separate times) → then rx_last
```

### [=N] (non-consecutive repetition)

```systemverilog
// expr[=N]: expr has been true N times (non-consecutive)
// Unlike [->N], match continues AFTER the Nth occurrence
// Allows trailing cycles where expr is 0

// Example: monitor that at least 3 errors occurred before system halt
property p_3_err_then_halt;
    @(posedge clk) $rose(monitor_start) |-> error[=3] ##1 halt;
endproperty
// After 3 errors, halt can come on any subsequent cycle (not necessarily
// the cycle right after the 3rd error)
```

### |-> (overlapping implication) and |=> (non-overlapping)

```systemverilog
// |-> : consequent evaluated SAME cycle as antecedent match
// |=> : consequent evaluated NEXT cycle after antecedent match
// |=> B  is equivalent to  |-> ##1 B

// Example (|->): grant must be one-hot in the same cycle as any grant
property p_grant_onehot;
    @(posedge clk) |grant |-> $onehot(grant);
endproperty

// Example (|=>): after write, data appears in SRAM one cycle later
property p_sram_write;
    @(posedge clk) wr_en |=> (sram[$past(addr)] == $past(wr_data));
endproperty
```

### throughout

```systemverilog
// expr1 throughout sequence2: expr1 must hold continuously for the
// entire duration of sequence2

// Example: burst transfer -- burst_active must stay high for entire burst
property p_burst_active;
    @(posedge clk)
    $rose(burst_start) |-> burst_active throughout (data_valid[->4] ##1 burst_end);
endproperty
// burst_active must be true from burst_start through the 4th data_valid and burst_end
```

### intersect

```systemverilog
// seq1 intersect seq2: both sequences must start AND end at the same time
// Both must match over the SAME number of cycles

// Example: verify that a read completes in exactly 5 cycles AND
// the bus is continuously owned during those 5 cycles
sequence s_read_takes_5;
    ##[5:5] read_done;  // exactly 5 cycles
endsequence

sequence s_bus_owned;
    bus_owned[*5];  // bus_owned for 5 consecutive cycles
endsequence

property p_read_5_with_bus;
    @(posedge clk) $rose(read_req) |-> s_read_takes_5 intersect s_bus_owned;
endproperty
```

### within

```systemverilog
// seq1 within seq2: seq1 must complete entirely during seq2
// seq1 can start AFTER seq2 starts and end BEFORE seq2 ends

// Example: NACK must occur somewhere within a transaction window
property p_nack_in_window;
    @(posedge clk)
    $rose(txn_start) |-> (nack within (txn_start ##[1:50] txn_end));
endproperty
// nack must occur somewhere between txn_start and txn_end
```

### first_match

```systemverilog
// first_match(seq): only the first matching thread of seq is kept
// Critical for performance when ranges create many threads

// Example: only care about first ack after req, ignore redundant matches
property p_first_ack;
    @(posedge clk) req |-> first_match(##[1:100] ack);
endproperty
// Without first_match, 100 threads would be spawned
// With first_match, only the first ack match survives
```

### ended

```systemverilog
// seq.ended: true at the clock tick where sequence seq completes
// Useful for checking that one sequence ended when another event occurs

// Example: verify that a training sequence has completed before data transfer
sequence s_training;
    txd ##1 txd ##1 txd;  // 3 training words
endsequence

property p_data_after_training;
    @(posedge clk) data_valid |-> s_training.ended;
endproperty
// data_valid is only allowed after the training sequence has completed
```

---

## Worked AXI4 Write Channel Protocol Assertion

This is the single most common protocol assertion asked in interviews. It exercises multiple SVA features simultaneously.

### Protocol Requirements

```
AXI4 Write Address (AW) channel:
  AWVALID, AWREADY, AWADDR, AWLEN, AWSIZE, AWBURST

AXI4 Write Data (W) channel:
  WVALID, WREADY, WDATA, WSTRB, WLAST

AXI4 Write Response (B) channel:
  BVALID, BREADY, BRESP

Rules to check:
  1. AW→W→B ordering: write address before or with write data; response last
  2. WLAST must be asserted on the last W beat
  3. BVALID must not assert until after WLAST handshake completes
  4. Number of W transfers must equal AWLEN + 1
  5. Once VALID asserted, it must stay high until READY handshake
  6. Payload must be stable while VALID is high and READY is low
```

### Complete Assertion Module

```systemverilog
module axi4_write_protocol_checker #(
    parameter ADDR_WIDTH = 32,
    parameter DATA_WIDTH = 64,
    parameter ID_WIDTH   = 4
)(
    input logic                  ACLK,
    input logic                  ARESETn,
    // AW channel
    input logic                  AWVALID,
    input logic                  AWREADY,
    input logic [ADDR_WIDTH-1:0] AWADDR,
    input logic [7:0]            AWLEN,    // burst length - 1
    input logic [2:0]            AWSIZE,
    input logic [1:0]            AWBURST,
    input logic [ID_WIDTH-1:0]   AWID,
    // W channel
    input logic                  WVALID,
    input logic                  WREADY,
    input logic [DATA_WIDTH-1:0] WDATA,
    input logic [DATA_WIDTH/8-1:0] WSTRB,
    input logic                  WLAST,
    // B channel
    input logic                  BVALID,
    input logic                  BREADY,
    input logic [1:0]            BRESP,
    input logic [ID_WIDTH-1:0]   BID
);

    // -------------------------------------------------------
    // Rule 1: VALID must stay asserted until READY handshake
    // -------------------------------------------------------
    property p_valid_stable(valid, ready);
        @(posedge ACLK) disable iff (!ARESETn)
        valid && !ready |=> valid;
    endproperty

    assert property (p_valid_stable(AWVALID, AWREADY))
        else $error("AWVALID deasserted before AWREADY at T=%0t", $time);
    assert property (p_valid_stable(WVALID, WREADY))
        else $error("WVALID deasserted before WREADY at T=%0t", $time);
    assert property (p_valid_stable(BVALID, BREADY))
        else $error("BVALID deasserted before BREADY at T=%0t", $time);

    // -------------------------------------------------------
    // Rule 2: Payload stability while VALID && !READY
    // -------------------------------------------------------
    property p_payload_stable(valid, ready, payload);
        @(posedge ACLK) disable iff (!ARESETn)
        valid && !ready |=> $stable(payload);
    endproperty

    assert property (p_payload_stable(AWVALID, AWREADY, {AWADDR, AWLEN, AWSIZE, AWBURST, AWID}))
        else $error("AW payload changed before handshake at T=%0t", $time);
    assert property (p_payload_stable(WVALID, WREADY, {WDATA, WSTRB}))
        else $error("W payload changed before handshake at T=%0t", $time);

    // -------------------------------------------------------
    // Rule 3: WLAST must be 0 for all beats except the last
    // -------------------------------------------------------
    // Count W handshakes per burst using a reference model
    logic [7:0] w_beat_count;
    logic       aw_captured;
    logic [7:0] expected_len;

    always @(posedge ACLK or negedge ARESETn) begin
        if (!ARESETn) begin
            w_beat_count  <= 8'd0;
            aw_captured   <= 1'b0;
            expected_len  <= 8'd0;
        end else begin
            // Capture AWLEN on AW handshake
            if (AWVALID && AWREADY) begin
                aw_captured  <= 1'b1;
                expected_len <= AWLEN;
            end

            // Count W handshakes
            if (WVALID && WREADY) begin
                if (WLAST) begin
                    // Verify beat count matches AWLEN + 1
                    assert (w_beat_count == expected_len)
                        else $error("W beat count mismatch: expected %0d transfers, got %0d at T=%0t",
                                    expected_len + 1, w_beat_count + 1, $time);
                    w_beat_count <= 8'd0;
                    aw_captured  <= 1'b0;
                end else begin
                    // WLAST must be 0 for non-final beats
                    assert (!WLAST)
                        else $error("WLAST=0 expected on non-final beat at T=%0t", $time);
                    w_beat_count <= w_beat_count + 8'd1;
                end
            end
        end
    end

    // -------------------------------------------------------
    // Rule 4: BVALID must not appear before WLAST handshake
    // -------------------------------------------------------
    sequence s_wlast_handshake;
        WVALID && WREADY && WLAST;
    endsequence

    property p_b_after_wlast;
        @(posedge ACLK) disable iff (!ARESETn)
        !BVALID throughout (first_match(s_wlast_handshake)) |=> 1;
        // Before WLAST handshake, BVALID must be 0
    endproperty

    // Alternative (simpler) using $past:
    property p_b_after_wlast_alt;
        @(posedge ACLK) disable iff (!ARESETn)
        BVALID |-> $past(WVALID && WREADY && WLAST, 1, 1, @(posedge ACLK))
               || $past(WVALID && WREADY && WLAST, 2, 1, @(posedge ACLK))
               || $past(WVALID && WREADY && WLAST, 3, 1, @(posedge ACLK));
        // BVALID can appear 1-3 cycles after WLAST handshake (pipelining allowed)
    endproperty

    // -------------------------------------------------------
    // Rule 5: BRESP must be OKAY (0) or SLVERR (2) or DECERR (3)
    // Never undefined
    // -------------------------------------------------------
    property p_bresp_valid;
        @(posedge ACLK) disable iff (!ARESETn)
        BVALID |-> BRESP inside {2'b00, 2'b10, 2'b11};
    endproperty
    assert property (p_bresp_valid)
        else $error("BRESP undefined value: %b at T=%0t", BRESP, $time);

    // -------------------------------------------------------
    // Rule 6: EXOKAY (2'b01) not allowed on BRESP for non-exclusive
    // -------------------------------------------------------
    property p_no_exokay;
        @(posedge ACLK) disable iff (!ARESETn)
        BVALID |-> BRESP != 2'b01;
    endproperty
    assert property (p_no_exokay)
        else $error("EXOKAY on BRESP without exclusive access at T=%0t", $time);

    // -------------------------------------------------------
    // Rule 7: WLAST must be asserted exactly once per burst
    // (WLAST can only be 1 when WVALID is 1, and only on the last beat)
    // -------------------------------------------------------
    property p_wlast_only_with_valid;
        @(posedge ACLK) disable iff (!ARESETn)
        WLAST |-> WVALID;
    endproperty
    assert property (p_wlast_only_with_valid)
        else $error("WLAST asserted without WVALID at T=%0t", $time);

    // -------------------------------------------------------
    // Coverage: ensure all protocol events are exercised
    // -------------------------------------------------------
    cover property (@(posedge ACLK) AWVALID && AWREADY);       // AW handshake
    cover property (@(posedge ACLK) WVALID  && WREADY);        // W handshake
    cover property (@(posedge ACLK) WVALID  && WREADY && WLAST); // Last W beat
    cover property (@(posedge ACLK) BVALID  && BREADY);        // B handshake
    cover property (@(posedge ACLK) BRESP == 2'b00);           // OKAY response
    cover property (@(posedge ACLK) BRESP == 2'b10);           // SLVERR response

    // Coverage: back-to-back W beats (full bandwidth)
    cover property (@(posedge ACLK)
        (WVALID && WREADY) ##1 (WVALID && WREADY));

    // Coverage: AW and W handshake in same cycle
    cover property (@(posedge ACLK)
        (AWVALID && AWREADY) && (WVALID && WREADY));

endmodule
```

### Key Design Decisions Explained

```
1. Why reference model for beat counting instead of pure assertions?
   The number of W transfers depends on AWLEN (runtime variable).
   Pure SVA cannot express "count exactly AWLEN+1 W handshakes" without
   auxiliary state. The always block acts as a reference model.

2. Why first_match on s_wlast_handshake?
   WVALID && WREADY && WLAST could be true for multiple consecutive cycles
   (if WREADY stays high). first_match ensures we only trigger on the
   first occurrence.

3. Why p_b_after_wlast_alt uses $past with gating?
   AXI4 allows pipelining: BVALID can appear several cycles after WLAST
   handshake, not necessarily the very next cycle. The $past check with
   gating allows a window of 1-3 cycles.

4. Why cover property on each handshake?
   Without these covers, the assertions could pass vacuously if the
   stimulus never exercises the protocol. The covers ensure test quality.
```

---

## CDC (Clock Domain Crossing) Assertions

### Why You Cannot Assert at the Crossing Point

```
Signal crossing from clk_a to clk_b:

  clk_a domain          CDC signal (metastable)     clk_b domain
  ┌──────────┐          ═══════════════════         ┌──────────┐
  │ source   ├─────────►│ glitch / metastable │────►│ 2-FF     ├─► synchronized
  │ FF       │          ═════════════════════         │ sync     │   signal
  └──────────┘                                        └──────────┘

At the crossing point (between source and synchronizer input), the signal
is METASTABLE. It may be X, it may oscillate, it may settle to the wrong
value temporarily. You CANNOT write a meaningful assertion here.

The correct place to assert: AFTER the synchronizer, in the destination
clock domain.
```

### CDC Assertion Patterns

```systemverilog
// Pattern 1: Verify synchronizer output is stable (no glitches after sync)
// After a 2-FF synchronizer, the signal should be clean in clk_b domain
module cdc_assertions #(
    parameter SYNC_STAGES = 2
)(
    input  logic clk_a,        // source domain
    input  logic clk_b,        // destination domain
    input  logic rst_n,
    input  logic cdc_data_a,   // signal in source domain (before crossing)
    input  logic cdc_synced_b  // signal after 2-FF synchronizer in clk_b
);

    // After the synchronizer, the signal should not glitch within a clk_b cycle
    property p_no_glitch_after_sync;
        @(posedge clk_b) disable iff (!rst_n)
        $stable(cdc_synced_b);  // No intra-cycle changes after synchronizer
    endproperty
    // Note: this checks sampled stability only; actual glitch detection
    // requires real-time comparison (not possible in SVA)

    // Pattern 2: Gray-code pointer crossing for async FIFO
    // After synchronization, the gray-code pointer must differ by at most 1 bit
    // from its previous value (gray code property: adjacent values differ by 1 bit)
    property p_gray_code_ptr_stability;
        @(posedge clk_b) disable iff (!rst_n)
        $countones(cdc_synced_b ^ $past(cdc_synced_b)) <= 1;
    endproperty
    // If this fails, the synchronizer output was corrupted (metastability
    // caused a multi-bit change, violating gray-code assumption)

    // Pattern 3: Verify data is held stable during crossing (req-ack handshake)
    // Source must hold data stable until destination acknowledges
    property p_data_held_during_cross;
        @(posedge clk_a) disable iff (!rst_n)
        cdc_req_a && !cdc_ack_a |-> $stable(cdc_data_a);
    endproperty
    // In clk_a domain: while req is high and ack has not returned,
    // data must not change

    // Pattern 4: Verify req-ack handshake completion
    // After req is asserted, ack must eventually return
    property p_handshake_completion;
        @(posedge clk_a) disable iff (!rst_n)
        $rose(cdc_req_a) |-> ##[1:500] cdc_ack_a;
    endproperty

    // Pattern 5: Verify ack deasserts after req deasserts
    property p_ack_follows_req;
        @(posedge clk_a) disable iff (!rst_n)
        $fell(cdc_req_a) |-> ##[0:10] !cdc_ack_a;
    endproperty

endmodule
```

### Common CDC Mistake: Asserting Before the Synchronizer

```systemverilog
// WRONG: Asserting on the metastable signal
property p_bad_cdc;
    @(posedge clk_b) disable iff (!rst_n)
    $stable(metastable_signal);  // This WILL fire false failures!
    // metastable_signal can change mid-cycle due to metastability resolution
endproperty

// CORRECT: Assert after the synchronizer
property p_good_cdc;
    @(posedge clk_b) disable iff (!rst_n)
    $stable(synchronized_signal);  // Clean, settled signal
endproperty

// CORRECT: Assert source-side stability (in source domain)
property p_source_holds;
    @(posedge clk_a) disable iff (!rst_n)
    cdc_req && !cdc_ack |-> $stable(cdc_data);
    // Verify source discipline in source clock domain -- no metastability issues
endproperty
```

### Multi-bit CDC Assertion Strategy

```systemverilog
// For a multi-bit bus crossing clock domains:
// Option 1: Verify gray-code encoding (async FIFO pointers)
//   Adjacent gray-code values differ by exactly 1 bit
//   After synchronization, check single-bit transitions

// Option 2: Verify req-ack handshake discipline
//   Source holds bus stable from req assertion until ack return
//   Destination samples bus only when req is high after synchronization

// Option 3: Verify MUX-based synchronization discipline
//   A control signal (crossed via 2-FF sync) gates the data bus
//   Data bus is only sampled when control signal is stable (after sync)

// Example: MUX-based CDC bus
module cdc_bus_checker #(
    parameter WIDTH = 32
)(
    input logic             clk_b,
    input logic             rst_n,
    input logic             enable_synced,   // control signal after synchronizer
    input logic [WIDTH-1:0] data_bus_b,      // data bus in destination domain
    input logic [WIDTH-1:0] captured_data    // latched data when enable was high
);

    // Data should only be captured when enable is stable for 2 cycles
    // (after the synchronizer)
    property p_capture_only_when_stable;
        @(posedge clk_b) disable iff (!rst_n)
        $rose(enable_synced) |=> ##1 $stable(enable_synced);
        // Enable must be stable for at least 2 cycles (sync latency)
    endproperty

    // Captured data must be stable
    property p_captured_data_valid;
        @(posedge clk_b) disable iff (!rst_n)
        !enable_synced |-> $stable(captured_data);
        // When enable is low, captured data must not change
    endproperty

endmodule
```

---

## Assertion Coverage and Action Blocks

```systemverilog
// Track how often an assertion fires (antecedent matches)
a_req_ack: assert property (
    @(posedge clk) req |-> ##[1:5] ack
) begin
    // PASS action block
    pass_count++;
end else begin
    // FAIL action block
    fail_count++;
    $error("Handshake failed at T=%0t", $time);
end

// Cover property: track that the property was exercised
c_req_ack: cover property (
    @(posedge clk) req ##[1:5] ack
);
// Coverage tools report how many times this property matched
```

---

## Interview Questions and Answers

### Q1: What are sampled values in SVA and why do they matter?

**A:** Sampled values are signal values captured in the Preponed region (start of the time
slot, before any Active-region logic). Assertions evaluate in the Observed region using these
pre-captured values. This prevents race conditions: the assertion sees a consistent snapshot
of all signals from before any RTL updates in the current cycle. Without sampled values,
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
source. This is essential because: (1) RTL IP may be encrypted; (2) RTL should stay clean for
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

```systemverilog
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

**A:** `assert`: the DUT must satisfy this property -- failures indicate bugs. `assume`:
the environment/stimulus must satisfy this -- in simulation, acts like assert; in formal
verification, constrains the input space (tells the prover not to try illegal inputs). `cover`:
checks that a property CAN be satisfied -- verifies reachability. Use all three together: assume
legal inputs, assert DUT behavior, cover interesting scenarios.

### Q12: How do you check for X/Z in assertions?

```systemverilog
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

```systemverilog
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
evaluates in the Observed region (same time slot, after Active/NBA settle). `assert final`
evaluates in the Reactive region (after Observed assertions and program block scheduling).
Use `assert #0` for most cases. Use `assert final` when you need to check values after
concurrent assertion processing has completed.

### Q16: How do you measure assertion coverage?

**A:** Assertion coverage tracks: (1) how many times the antecedent matched (attempts); (2)
how many times the full property passed; (3) how many times it failed; (4) how many times
the antecedent didn't match (vacuous passes). Use `cover property` alongside `assert property`
to ensure non-trivial coverage. Most tools report assertion coverage in the same database as
functional coverage, allowing unified analysis.
