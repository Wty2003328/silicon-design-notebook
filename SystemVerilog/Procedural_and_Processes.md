# SystemVerilog Procedural Blocks and Processes -- Senior Engineer Deep Dive

---

## SystemVerilog Scheduling Semantics

Understanding the simulation scheduler is non-negotiable for senior verification engineers.
Race conditions, assertion failures, and testbench/RTL interaction bugs all trace back to
which region executes when.

### The Simulation Time Slot Regions

Each simulation time step is divided into these ordered regions:

```
    +-----------------------+
    | Preponed              |  Sample values for assertions (read-only)
    +-----------------------+
    | Active                |  always, always_comb, assign, blocking assigns
    | Inactive              |  #0 delayed assignments
    | NBA (Non-Blocking     |  <= assignments execute here
    |      Assignment)      |
    +-----------------------+
    | Observed              |  Concurrent assertions evaluated here
    +-----------------------+
    | Reactive              |  program block code, pass/fail of assertions
    | Re-Inactive           |  #0 in program blocks
    | Re-NBA                |  <= in program blocks
    +-----------------------+
    | Postponed             |  $monitor, $strobe execute here (read-only)
    +-----------------------+
```

Within Active/Reactive, events iterate until no more are pending (delta cycle convergence).

### How This Matters in Practice

```systemverilog
module scheduler_demo;
    logic clk = 0;
    logic [7:0] a, b, c;

    always #5 clk = ~clk;

    // Active region: combinational logic evaluates
    always_comb c = a + b;

    // Active region: blocking assignment in always block
    always @(posedge clk) begin
        a = 8'h10;   // Executes in Active region
    end

    // NBA region: non-blocking assignment
    always @(posedge clk) begin
        b <= 8'h20;  // Scheduled to NBA region
    end

    // Observed region: assertion checks sampled values from Preponed
    property p_sum;
        @(posedge clk) (c == a + b);
    endproperty
    assert property (p_sum);

    // Reactive region: program block testbench code
    // (If using program block -- deprecated in modern UVM)
endmodule
```

### Why Non-Blocking Assignments Use NBA Region

The NBA region exists to prevent race conditions in sequential logic. All RHS values are
captured in Active, then all LHS updates happen together in NBA. This ensures:

```systemverilog
always @(posedge clk) begin
    a <= b;   // Captures b's current value
    b <= a;   // Captures a's current value (not the one just assigned!)
end
// Result: a and b SWAP correctly. With blocking (=), order matters and one value is lost.
```

---

## always_comb vs always @(*)

### The TWO Critical Differences

**Difference 1: always_comb triggers at time 0**

```systemverilog
module time_zero_demo;
    logic [7:0] a = 8'hFF;
    logic [7:0] y1, y2;

    // always_comb: evaluates at time 0, y1 = 8'hFF immediately
    always_comb y1 = a;

    // always @(*): does NOT trigger at time 0, y2 is X until a changes
    always @(*) y2 = a;

    initial begin
        #0;
        $display("y1=%h y2=%h", y1, y2);  // y1=FF, y2=xx
        // y2 only updates when 'a' changes -- but a never changes!
        // This is a REAL BUG that always_comb catches
    end
endmodule
```

**Difference 2: always_comb is sensitive to function contents**

```systemverilog
logic [7:0] lut [256];    // Lookup table
logic [7:0] addr, result1, result2;

function logic [7:0] lookup(logic [7:0] idx);
    return lut[idx];       // Reads from lut[] internally
endfunction

// always_comb: sensitive to addr AND lut (reads inside lookup)
always_comb result1 = lookup(addr);
// If lut[x] changes, result1 re-evaluates

// always @(*): sensitive to addr ONLY (doesn't analyze function body)
always @(*) result2 = lookup(addr);
// If lut[x] changes but addr doesn't, result2 is STALE -- BUG
```

### Other always_comb Restrictions

```systemverilog
always_comb begin
    // These are all ILLEGAL in always_comb:
    // #10 y = a;            // No delays
    // @(posedge clk) y = a; // No event controls
    // wait (enable) y = a;  // No waits
    // fork ... join         // No fork
end

// always_comb also enforces: variable assigned in always_comb
// cannot be assigned by any other process
logic y;
always_comb y = a & b;
// assign y = c;  // ERROR: y already driven by always_comb
```

---

## always_ff

### Single-Clock Enforcement

```systemverilog
// CORRECT: single clock edge, optional async reset
always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        q <= '0;
    else
        q <= d;
end

// GOTCHA: multiple clock edges -- some tools allow, some flag as error
always_ff @(posedge clk or posedge clk2) begin  // Questionable
    // Which clock drives this register? Ambiguous for synthesis
end

// always_ff enforces that only <= (non-blocking) is used
always_ff @(posedge clk) begin
    q = d;    // WARNING or ERROR: blocking assignment in always_ff
end
```

### Synthesis Mapping

`always_ff` tells synthesis tools this is a register. The sensitivity list determines:
- `@(posedge clk)` -> flip-flop with synchronous behavior only
- `@(posedge clk or negedge rst_n)` -> flip-flop with async active-low reset
- `@(posedge clk or posedge rst)` -> flip-flop with async active-high reset

---

## always_latch

```systemverilog
always_latch begin
    if (enable)
        q <= d;
    // No else: intentional latch behavior
end

// This is equivalent to always_comb for latches, but makes intent explicit
// Tools won't warn about latch inference (they would with always_comb)
```

---

## Fork-Join Deep Dive

### Three Variants

```systemverilog
// fork-join: waits for ALL child processes to complete
fork
    begin task_A(); end   // Process 1
    begin task_B(); end   // Process 2
    begin task_C(); end   // Process 3
join                      // Blocks until ALL three finish

// fork-join_any: waits for ANY ONE child to complete
fork
    begin task_A(); end
    begin task_B(); end
    begin task_C(); end
join_any                  // Blocks until FIRST one finishes
// Other two still running in background!

// fork-join_none: doesn't wait at all, spawns and continues
fork
    begin task_A(); end
    begin task_B(); end
join_none                 // Returns immediately, tasks run in background
// Parent continues executing here while tasks run concurrently
```

### Timing Diagram Example

```
Time:    0   10   20   30   40   50
Task A:  |=========|                   (finishes at t=20)
Task B:  |==================|          (finishes at t=30)
Task C:  |===========================| (finishes at t=50)

fork-join:      parent resumes at t=50
fork-join_any:  parent resumes at t=20 (A finishes first)
fork-join_none: parent resumes at t=0  (immediate)
```

### THE CLASSIC BUG: For-Loop + Fork

This is one of the most common interview questions and real-world bugs.

```systemverilog
// BUG: All spawned processes see the SAME variable i
initial begin
    for (int i = 0; i < 4; i++) begin
        fork
            begin
                #10;
                $display("i = %0d", i);  // ALL print "i = 4"!
            end
        join_none
    end
    wait fork;
end
// Output: i = 4, i = 4, i = 4, i = 4
// WHY: fork captures the VARIABLE i, not its VALUE.
// By the time #10 passes, the for loop is done and i == 4.
```

```systemverilog
// FIX 1: Automatic variable captures a copy
initial begin
    for (int i = 0; i < 4; i++) begin
        automatic int j = i;  // j gets a COPY of i at each iteration
        fork
            begin
                #10;
                $display("j = %0d", j);  // Prints 0, 1, 2, 3
            end
        join_none
    end
    wait fork;
end

// FIX 2: Extra begin-end with automatic (more idiomatic)
initial begin
    for (int i = 0; i < 4; i++) begin
        fork
            begin
                automatic int k = i;
                #10;
                $display("k = %0d", k);  // Prints 0, 1, 2, 3
            end
        join_none
    end
    wait fork;
end
```

---

## Disable Fork: Scope and the Isolation Wrapper

### The Problem: disable fork Kills TOO Much

```systemverilog
initial begin
    fork
        begin  // Background monitor -- should run forever
            forever @(posedge clk) check_protocol();
        end
    join_none

    // Later, run a transaction with timeout
    fork
        begin drive_transaction(); end
        begin #1000 $display("Timeout!"); end
    join_any
    disable fork;  // KILLS EVERYTHING -- including the background monitor!
end
```

### The Fix: Isolation Wrapper Pattern

```systemverilog
initial begin
    fork
        begin  // Background monitor
            forever @(posedge clk) check_protocol();
        end
    join_none

    // Isolate the disable fork scope
    fork begin  // <-- This begin creates a NEW child process scope
        fork
            begin drive_transaction(); end
            begin #1000 $display("Timeout!"); end
        join_any
        disable fork;  // Only kills processes within THIS begin-end
    end join  // <-- join waits for the isolating process

    // Background monitor is still alive!
end
```

The key insight: `disable fork` disables all child processes of the **current** process.
By wrapping in `fork begin...end join`, you create a new process whose children are only
the timeout and the transaction -- not the background monitor.

---

## Process Class (IEEE 1800)

### Complete Watchdog Timer Using process::self()

```systemverilog
class watchdog;
    process proc_handle;
    int timeout_ns;

    function new(int timeout = 10000);
        this.timeout_ns = timeout;
    endfunction

    task start();
        fork begin
            proc_handle = process::self();
            #(timeout_ns * 1ns);
            `uvm_fatal("WATCHDOG", $sformatf("Timeout after %0d ns", timeout_ns))
        end join_none
    endtask

    task stop();
        if (proc_handle != null) begin
            if (proc_handle.status() != process::FINISHED &&
                proc_handle.status() != process::KILLED) begin
                proc_handle.kill();
                $display("Watchdog stopped");
            end
        end
    endtask

    function bit is_running();
        if (proc_handle == null) return 0;
        return (proc_handle.status() == process::RUNNING ||
                proc_handle.status() == process::WAITING);
    endfunction
endclass

// Usage in a test:
task run_phase(uvm_phase phase);
    watchdog wd = new(50000);  // 50us timeout
    phase.raise_objection(this);

    wd.start();
    run_all_sequences();
    wd.stop();

    phase.drop_objection(this);
endtask
```

### process::status() Values

| Status | Meaning |
|---|---|
| `process::FINISHED` | Process completed normally |
| `process::RUNNING` | Currently executing |
| `process::WAITING` | Blocked on event/delay/wait |
| `process::SUSPENDED` | Explicitly suspended |
| `process::KILLED` | Killed by kill() or disable |

---

## Task vs Function

### Fundamental Rule

- **Functions** cannot consume simulation time. No `#delay`, `@event`, `wait`, `fork-join`.
  They execute in zero time and return a value.
- **Tasks** can consume time. They can have delays, waits, event controls, fork-join.
  They do not return values (use output/inout arguments instead).

```systemverilog
// Function: zero-time computation
function logic [7:0] compute_crc(logic [31:0] data);
    // Cannot have: #10, @(posedge clk), wait(signal), fork-join
    logic [7:0] crc = 0;
    for (int i = 0; i < 32; i++)
        crc = crc ^ data[i];
    return crc;
endfunction

// Task: can consume time
task drive_data(input logic [31:0] data);
    @(posedge clk);        // Wait for clock edge
    valid <= 1'b1;
    bus_data <= data;
    @(posedge clk);
    wait (ready);           // Wait for handshake
    valid <= 1'b0;
endtask

// Calling context restrictions:
always_comb begin
    crc = compute_crc(data);   // OK -- function is zero-time
    // drive_data(data);        // ERROR -- cannot call task from always_comb
end
```

### Void Functions: Not Syntactic Sugar for Tasks

A `void function` returns nothing but is still a FUNCTION -- zero time, no delays. It is NOT
equivalent to a task. You can call void functions from always_comb; you cannot call tasks.

```systemverilog
function void log_event(string msg);  // No return value, but zero-time
    $display("T=%0t: %s", $time, msg);
endfunction

always_comb begin
    log_event("Combinational block evaluated");  // OK
end
```

### Functions Can Call Functions, Tasks Can Call Both

```
Function -> Function:    OK
Function -> Task:        ERROR (task may consume time)
Task -> Function:        OK
Task -> Task:            OK
```

---

## Automatic vs Static

### The Default Is DANGEROUS

- **Module-level** tasks/functions default to **static** (shared variables)
- **Class methods** are always **automatic** (each call gets its own copy)
- **Program block** tasks/functions default to **automatic**

### The Critical Bug With Static Tasks

```systemverilog
module static_bug;
    task count(input int id);  // DEFAULT: static!
        int local_var;          // Shared across all concurrent calls!
        local_var = 0;
        repeat (3) begin
            #10;
            local_var++;
            $display("T=%0t ID=%0d count=%0d", $time, id, local_var);
        end
    endtask

    initial begin
        fork
            count(1);  // Both calls share the SAME local_var!
            count(2);
        join
    end
endmodule

// Output (WRONG -- variables collide):
// T=10 ID=1 count=1
// T=10 ID=2 count=2   <- Expected 1, got 2!
// T=20 ID=1 count=3   <- Expected 2, got 3!
// T=20 ID=2 count=4   <- Expected 2, got 4!
```

```systemverilog
// FIX: declare as automatic
module fixed;
    task automatic count(input int id);
        int local_var;  // Each call gets its OWN copy
        local_var = 0;
        repeat (3) begin
            #10;
            local_var++;
            $display("T=%0t ID=%0d count=%0d", $time, id, local_var);
        end
    endtask
endmodule

// Output (CORRECT):
// T=10 ID=1 count=1
// T=10 ID=2 count=1
// T=20 ID=1 count=2
// T=20 ID=2 count=2
```

### Making an Entire Module Automatic

```systemverilog
module automatic my_module;  // ALL tasks/functions in this module are automatic
    // ...
endmodule
```

---

## Clocking Blocks

### Purpose: Model Setup/Hold Timing Relationships

Clocking blocks define the timing relationship between testbench and DUT signals, abstracting
away clock edges and skew. `input #1step` samples one step before the clock edge (Preponed
region). `output #0` drives at the exact clock edge (effectively the same time slot).

```systemverilog
interface bus_if (input logic clk);
    logic [7:0] data;
    logic       valid;
    logic       ready;

    clocking driver_cb @(posedge clk);
        default input #1step output #0;
        output data;
        output valid;
        input  ready;   // Sampled 1step before posedge clk
    endclocking

    clocking monitor_cb @(posedge clk);
        default input #1step output #0;
        input data;
        input valid;
        input ready;
    endclocking

    modport driver_mp (clocking driver_cb);
    modport monitor_mp (clocking monitor_cb);
endinterface
```

### Using Clocking Blocks in a Testbench

```systemverilog
class driver;
    virtual bus_if.driver_mp vif;

    task drive_transaction(Transaction txn);
        // Wait for clock edge using clocking block
        @(vif.driver_cb);

        // Drive outputs through clocking block (automatically clocked)
        vif.driver_cb.data  <= txn.data;
        vif.driver_cb.valid <= 1'b1;

        // Wait for ready (sampled at previous 1step)
        while (!vif.driver_cb.ready)
            @(vif.driver_cb);

        @(vif.driver_cb);
        vif.driver_cb.valid <= 1'b0;
    endtask
endclass
```

### Why #1step and #0 Specifically?

`#1step` is the smallest representable time unit in the simulator -- it samples in the
Preponed region, BEFORE any Active-region logic executes. This models setup time: the
testbench sees values that were stable before the clock edge, exactly as real hardware would.

`#0` output drives at the clock edge itself, in the same time slot. Since RTL evaluates in
Active region and testbench drives in Reactive region (via program blocks) or the same Active
region with clocking block scheduling, this ensures proper ordering.

---

## Program Block

### Why It Exists: Race-Free Testbench Execution

```systemverilog
program automatic test (bus_if.driver_mp drv_if);
    initial begin
        // This executes in the REACTIVE region
        // All RTL (Active region) and assertions (Observed) are done
        // So testbench sees stable, settled values -- no races
        @(drv_if.driver_cb);
        drv_if.driver_cb.data <= 8'hFF;
    end
endprogram
```

### Why UVM Deprecated Program Blocks

1. **OOP incompatibility**: program blocks don't support class-based component hierarchies well
2. **Implicit $finish**: program blocks call `$finish` when all initial blocks complete, which
   can terminate simulation prematurely when using phase-based UVM control
3. **Limited fork-join**: some simulators restrict concurrent processes in program blocks
4. **One-shot execution**: program blocks run once; UVM needs phases that can re-execute
5. **Modern alternative**: UVM uses `uvm_test` with phase mechanism for simulation control,
   and clocking blocks + careful coding practices to avoid races

---

## Wait Statements and Event Control

```systemverilog
// Edge-triggered wait
@(posedge clk);           // Wait for rising edge of clk
@(negedge rst_n);         // Wait for falling edge of rst_n
@(posedge clk or negedge rst_n);  // Wait for either

// Level-sensitive wait
wait (ready == 1);         // Block until ready is 1
// GOTCHA: if ready is already 1, wait returns immediately (no blocking)

// Named event
event done;
-> done;                   // Trigger the event (Active region)
@(done);                   // Wait for trigger

// RACE CONDITION: if -> and @ happen in same time step
// The @ might miss the trigger (it was registered before the ->)
// Fix: use .triggered
wait (done.triggered);     // Catches same-time-step triggers

// Nonblocking event trigger
->> done;                  // Trigger in NBA region (avoids some races)
```

---

## Disable Statement

```systemverilog
// Disable a named block
begin : my_block
    forever begin
        @(posedge clk);
        if (condition) disable my_block;  // Exits the named block
    end
end

// Disable a task (all instances!)
task automatic long_task();
    // ...
endtask

initial begin
    fork
        long_task();   // Instance 1
        long_task();   // Instance 2
    join_none

    #100;
    disable long_task;  // Disables ALL active instances of long_task!
end
```

---

## Interview Questions and Answers

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

**A:** Blocking assignments execute sequentially in the Active region -- the LHS updates
immediately before the next statement. Non-blocking assignments evaluate the RHS in Active
but schedule the LHS update to the NBA region (after all Active evaluations). This means all
non-blocking RHS values are sampled before any LHS updates, preventing read-after-write races
in sequential logic. Rule: use `=` in combinational logic (`always_comb`), `<=` in sequential
logic (`always_ff`).

### Q9: How do you implement a timeout with fork-join_any?

```systemverilog
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

```systemverilog
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
