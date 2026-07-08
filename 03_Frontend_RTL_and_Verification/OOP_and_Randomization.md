# SystemVerilog OOP and Randomization -- Senior Engineer Deep Dive

---

## UVM Class Hierarchy -- Complete Reference

### The Two Branches: uvm_object and uvm_component

```ascii-graph
uvm_void (abstract root)
├── uvm_object (data containers -- no hierarchy, no phasing)
│   ├── uvm_transaction (deprecated, use uvm_sequence_item)
│   │   └── uvm_sequence_item (THE base class for transactions)
│   │       └── your_packet (user-defined transaction)
│   ├── uvm_sequence #(REQ, RSP) (generates sequence_items)
│   │   └── your_sequence
│   ├── uvm_reg (register model objects)
│   ├── uvm_reg_field
│   ├── uvm_reg_map
│   └── uvm_pool, uvm_queue, uvm_event (utility classes)
│
└── uvm_component (hierarchical, phased, permanent during simulation)
    ├── uvm_test (top-level test class)
    │   └── your_test
    ├── uvm_env (encapsulates the verification environment)
    │   └── your_env
    ├── uvm_agent (bundles driver, monitor, sequencer, coverage)
    │   └── your_agent
    ├── uvm_driver #(REQ, RSP) (drives signals to DUT)
    │   └── your_driver
    ├── uvm_monitor (observes DUT signals, creates transactions)
    │   └── your_monitor
    ├── uvm_sequencer #(REQ, RSP) (arbitrates between sequences)
    │   └── your_sequencer
    ├── uvm_scoreboard (compares expected vs actual)
    │   └── your_scoreboard
    ├── uvm_subscriber #(T) (analysis port consumer)
    │   ├── your_coverage_collector
    │   └── your_logger
    ├── uvm_reg_block (register block)
    └── uvm_phase (phase management)
```

### How the Hierarchy Relates -- Data Flow

```ascii-graph
uvm_test
  └── uvm_env
        ├── uvm_agent (ACTIVE mode: drives stimulus)
        │     ├── uvm_sequencer
        │     │     └── uvm_sequence
        │     │           └── creates uvm_sequence_item objects
        │     │                 (these are uvm_object, NOT uvm_component)
        │     ├── uvm_driver
        │     │     └── receives uvm_sequence_item from sequencer
        │     │           drives DUT pins via virtual interface
        │     └── uvm_monitor
        │           └── observes DUT pins, creates uvm_sequence_item
        │                 publishes to analysis port
        │
        ├── uvm_agent (PASSIVE mode: observation only)
        │     └── uvm_monitor only (no driver, no sequencer)
        │
        ├── uvm_scoreboard
        │     └── subscribes to monitors via analysis ports
        │           compares expected vs actual transactions
        │
        └── uvm_subscriber (coverage collector)
              └── subscribes to monitors
                    collects functional coverage on transactions

Key relationships:
  uvm_sequence IS-A uvm_object (created dynamically, garbage collected)
  uvm_sequence_item IS-A uvm_object (created/destroyed per transaction)
  uvm_driver IS-A uvm_component (exists for entire simulation)
  uvm_sequencer IS-A uvm_component (exists for entire simulation)
  
  The sequencer is the bridge between the two branches:
    - Sequencer is a component (permanent, phased)
    - It holds a queue of uvm_sequence_item objects (temporary)
    - Driver requests items from sequencer via get_next_item()
    - Sequencer runs sequences which produce items
```

---

## Class Fundamentals

### Handle vs Object -- The #1 Source of Confusion

A **handle** is a pointer/reference. An **object** is the actual memory allocation. Multiple
handles can point to the same object. A handle can be `null`.

```verilog
class Packet;
    int id;
    function new(int id);
        this.id = id;
    endfunction
endclass

initial begin
    Packet p1;          // Handle declared -- currently NULL
    Packet p2;          // Another NULL handle

    // p1.id = 5;       // RUNTIME ERROR: null handle dereference

    p1 = new(1);        // Object created, p1 points to it
    p2 = p1;            // p2 points to SAME object (not a copy!)
    p2.id = 99;
    $display(p1.id);    // Prints 99 -- p1 and p2 share the same object!

    p1 = new(2);        // p1 now points to NEW object (id=2)
    $display(p2.id);    // Still 99 -- p2 still points to OLD object

    p2 = null;          // p2 no longer references the old object
    // If no other handles reference it, garbage collector reclaims it
end
```

### Complete Class Lifecycle

```verilog
class Transaction;
    static int count = 0;
    int id;
    bit [31:0] data;

    function new(bit [31:0] data = 0);
        this.data = data;
        this.id = count++;
    endfunction

    // No explicit destructor in SV -- garbage collection handles cleanup
    // Objects are reclaimed when no handles reference them
endclass

initial begin
    Transaction txn_array[100];

    // Create 100 objects
    foreach (txn_array[i])
        txn_array[i] = new($urandom);

    // Use them...
    foreach (txn_array[i])
        $display("Txn %0d: data=%h", txn_array[i].id, txn_array[i].data);

    // Deallocation: set handles to null or let them go out of scope
    txn_array.delete();  // For dynamic arrays
    // Garbage collector will reclaim all 100 Transaction objects
end
```

### this Keyword

```verilog
class Node;
    int value;
    Node next;  // Handle to another Node

    function new(int v);
        this.value = v;    // this disambiguates parameter from field
        this.next = null;
    endfunction

    function void append(Node n);
        Node current = this;  // Start from this node
        while (current.next != null)
            current = current.next;
        current.next = n;
    endfunction
endclass
```

---

## Inheritance

```verilog
class base_transaction;
    rand bit [7:0] addr;
    rand bit [31:0] data;
    bit [3:0] kind;

    function new(bit [3:0] kind = 0);
        this.kind = kind;
    endfunction

    virtual function void display();
        $display("BASE: addr=%h data=%h kind=%0d", addr, data, kind);
    endfunction

    virtual function base_transaction copy();
        base_transaction t = new(this.kind);
        t.addr = this.addr;
        t.data = this.data;
        return t;
    endfunction
endclass

class error_transaction extends base_transaction;
    rand bit       inject_error;
    rand bit [2:0] error_type;

    constraint c_error_rate { inject_error dist {0 := 90, 1 := 10}; }

    function new();
        super.new(4'hE);  // Call parent constructor with error kind
    endfunction

    // Override virtual functions
    function void display();
        super.display();  // Call parent version first
        $display("  ERROR: inject=%b type=%0d", inject_error, error_type);
    endfunction

    function base_transaction copy();
        error_transaction t = new();
        t.addr = this.addr;
        t.data = this.data;
        t.inject_error = this.inject_error;
        t.error_type = this.error_type;
        return t;
    endfunction
endclass
```

---

## Polymorphism -- The #1 OOP Interview Question

### Virtual vs Non-Virtual: Proof by Example

```verilog
class Animal;
    function string speak_static();       // NOT virtual
        return "...";
    endfunction

    virtual function string speak_virtual();  // Virtual
        return "...";
    endfunction
endclass

class Dog extends Animal;
    function string speak_static();
        return "Woof";
    endfunction

    function string speak_virtual();
        return "Woof";
    endfunction
endclass

class Cat extends Animal;
    function string speak_static();
        return "Meow";
    endfunction

    function string speak_virtual();
        return "Meow";
    endfunction
endclass

initial begin
    Animal animals[3];
    animals[0] = new();
    animals[1] = Dog::new();   // Upcast Dog to Animal handle
    animals[2] = Cat::new();   // Upcast Cat to Animal handle

    foreach (animals[i]) begin
        // NON-VIRTUAL: compile-time binding based on HANDLE type (Animal)
        $display("Static:  %s", animals[i].speak_static());
        // Prints: "...", "...", "..."  -- WRONG for Dog and Cat

        // VIRTUAL: run-time binding based on OBJECT type
        $display("Virtual: %s", animals[i].speak_virtual());
        // Prints: "...", "Woof", "Meow"  -- CORRECT
    end
end
```

**Key insight:** Without `virtual`, SystemVerilog uses the HANDLE type to determine which
function to call (compile-time/static dispatch). With `virtual`, it uses the OBJECT type
(run-time/dynamic dispatch). This is fundamental to UVM where base class handles store
derived class objects.

### Practical Polymorphism in Verification

```verilog
// A single scoreboard function handles ALL transaction types
class scoreboard;
    function void check(base_transaction expected, base_transaction actual);
        // compare() is virtual -- calls the right version for error_transaction,
        // burst_transaction, etc. without knowing the actual type
        if (!expected.compare(actual)) begin
            expected.display();  // Virtual -- shows error_transaction fields
            actual.display();
            `uvm_error("SCB", "Mismatch")
        end
    endfunction
endclass
```

---

## Deep Copy vs Shallow Copy

### The Bug That Bites Everyone

```verilog
class Payload;
    bit [7:0] data[];

    function new(int size = 4);
        data = new[size];
    endfunction
endclass

class Packet;
    int id;
    Payload pl;  // Handle to another object

    function new(int id, int pl_size = 4);
        this.id = id;
        this.pl = new(pl_size);
    endfunction
endclass

initial begin
    Packet p1 = new(1, 8);
    p1.pl.data = '{8'h11, 8'h22, 8'h33, 8'h44, 8'h55, 8'h66, 8'h77, 8'h88};

    // SHALLOW COPY: copies handle values, NOT the objects they point to
    Packet p2 = new p1;       // Built-in shallow copy
    p2.id = 2;                // p2 gets its own id -- OK
    p2.pl.data[0] = 8'hFF;   // OOPS: modifies p1.pl.data[0] too!
    // p1 and p2 share the SAME Payload object!

    $display("p1.pl.data[0] = %h", p1.pl.data[0]);  // FF -- corrupted!
end
```

### Implementing Proper Deep Copy

```verilog
class Payload;
    bit [7:0] data[];

    function Payload copy();
        Payload c = new(data.size());
        foreach (data[i]) c.data[i] = this.data[i];
        return c;
    endfunction
endclass

class Packet;
    int id;
    Payload pl;

    function Packet copy();
        Packet c = new(this.id, 0);
        c.pl = this.pl.copy();  // Deep copy -- create new Payload
        return c;
    endfunction
endclass

// UVM approach: do_copy()
class my_txn extends uvm_sequence_item;
    `uvm_object_utils(my_txn)
    rand bit [7:0] addr;
    rand bit [31:0] data;
    bit [7:0] payload[];

    function void do_copy(uvm_object rhs);
        my_txn rhs_txn;
        super.do_copy(rhs);  // Copy base class fields
        if (!$cast(rhs_txn, rhs))
            `uvm_fatal("COPY", "Cast failed in do_copy")
        this.addr = rhs_txn.addr;
        this.data = rhs_txn.data;
        this.payload = new[rhs_txn.payload.size()];
        foreach (rhs_txn.payload[i])
            this.payload[i] = rhs_txn.payload[i];
    endfunction
endclass

// Usage:
my_txn t1 = my_txn::type_id::create("t1");
my_txn t2 = my_txn::type_id::create("t2");
t2.copy(t1);  // Calls do_copy -- full deep copy
```

---

## Factory Pattern (UVM Context)

### Why Polymorphism Alone Is Not Enough

```verilog
// Problem: you can't override "new" with polymorphism
class driver;
    task run();
        base_transaction txn;
        txn = new();  // ALWAYS creates base_transaction
        // Even if you want error_transaction, you'd have to modify this code
        // In a reusable VIP, you CAN'T modify driver source code
    endtask
endclass
```

### UVM Factory Solves This

```verilog
class base_txn extends uvm_sequence_item;
    `uvm_object_utils(base_txn)  // Register with factory
    rand bit [7:0] addr;
    rand bit [31:0] data;

    function new(string name = "base_txn");
        super.new(name);
    endfunction
endclass

class error_txn extends base_txn;
    `uvm_object_utils(error_txn)  // Register with factory
    rand bit inject_error;

    function new(string name = "error_txn");
        super.new(name);
    endfunction
endclass

// In the driver (VIP code -- never modified):
class my_driver extends uvm_driver #(base_txn);
    task run_phase(uvm_phase phase);
        base_txn txn;
        forever begin
            txn = base_txn::type_id::create("txn");
            // Factory returns whatever type is registered
            // Default: base_txn. After override: error_txn.
        end
    endtask
endclass

// In the test (user code -- freely modified):
class error_test extends uvm_test;
    function void build_phase(uvm_phase phase);
        super.build_phase(phase);
        // TYPE override: ALL base_txn creations return error_txn
        base_txn::type_id::set_type_override(error_txn::get_type());

        // INSTANCE override: only specific instances affected
        // factory.set_inst_override_by_type(
        //     base_txn::get_type(), error_txn::get_type(),
        //     "env.agent.driver.*");
    endfunction
endclass
```

---

## Randomization Engine Deep Dive

### How the Solver Works (Conceptually)

SystemVerilog constraint solvers use either **BDD** (Binary Decision Diagram) or **SAT/SMT**
(Boolean Satisfiability) based approaches:

- **BDD-based**: builds a compact representation of all valid solutions, then uniformly samples.
  Good for small constraint spaces. Exponential memory for complex constraints.
- **SAT-based**: searches for satisfying assignments. Faster for large problems but may not
  provide uniform distribution without extra work.

**Solver complexity**: constraint solving is NP-complete in general. Certain patterns cause
exponential blowup:
- Multiplication/division constraints: `a * b == c` is very hard
- Large `unique` constraints: N! possible orderings
- Complex cross-variable dependencies

### Basic Randomization

```verilog
class packet extends uvm_sequence_item;
    rand bit [7:0]  addr;
    rand bit [31:0] data;
    rand bit [3:0]  burst_len;
    randc bit [1:0] channel;  // Cyclic: all values before repeating

    constraint c_addr_range {
        addr inside {[8'h00:8'h3F], [8'hC0:8'hFF]};
    }

    constraint c_burst {
        burst_len >= 1;
        burst_len <= 8;
    }

    function new(string name = "packet");
        super.new(name);
    endfunction
endclass

initial begin
    packet pkt = new("pkt");
    if (!pkt.randomize())
        `uvm_fatal("RAND", "Randomization failed!")
    // ALWAYS check return value -- constraint conflicts cause silent failure
end
```

---

## Constraint Patterns (10+ Real Patterns)

### Pattern 1: Weighted Distribution for Error Injection

```verilog
class error_injection_txn;
    rand enum {NORMAL, SHORT_PKT, LONG_PKT, CRC_ERROR, TIMEOUT} txn_type;

    constraint c_error_dist {
        txn_type dist {
            NORMAL    := 90,   // 90% weight
            SHORT_PKT := 3,    // 3% weight
            LONG_PKT  := 3,
            CRC_ERROR := 2,
            TIMEOUT   := 2
        };
    }
endclass
// dist with := assigns weight to each value
// dist with :/ distributes weight across range:
//   value [1:4] :/ 40  means each value gets 40/4 = 10% weight
```

### Pattern 2: Unique Array Elements

```verilog
class unique_demo;
    rand bit [3:0] arr[8];

    constraint c_unique {
        unique {arr};  // All elements must be different
        // Only works if array size <= number of possible values (16 here)
    }

    // Alternative for older tools without 'unique' support:
    constraint c_unique_manual {
        foreach (arr[i])
            foreach (arr[j])
                if (i < j)
                    arr[i] != arr[j];
    }
endclass
```

### Pattern 3: Conditional Constraints with Implication

```verilog
class protocol_txn;
    rand bit        is_write;
    rand bit [31:0] addr;
    rand bit [31:0] data;
    rand bit [3:0]  burst_len;

    // Implication: if write, then specific address range
    constraint c_write_addr {
        is_write -> addr inside {[32'h1000:32'h1FFF]};
    }

    // Bidirectional: if-else constraint
    constraint c_burst {
        if (is_write) {
            burst_len inside {1, 2, 4, 8};
        } else {
            burst_len == 1;  // Reads are always single-beat
        }
    }

    // Chain: A -> B -> C
    constraint c_chain {
        (addr[31:28] == 4'hF) -> (is_write == 0);  // High addr = read only
        (is_write == 0) -> (data == 0);              // Read data field unused
    }
endclass
```

### Pattern 4: Circular Dependency Detection

```verilog
class circular_demo;
    rand int a, b;

    // UNSOLVABLE circular dependency:
    // constraint c_circular {
    //     a > b;
    //     b > a;  // Impossible! randomize() returns 0
    // }

    // Subtler circular issue with implication:
    // (a > 5) -> (b < 3)
    // (b < 3) -> (a < 2)
    // Combined: (a > 5) -> (a < 2) -- contradiction when a > 5

    // Fix: ensure constraint graph is acyclic or solvable
    constraint c_valid {
        a inside {[1:10]};
        b inside {[1:10]};
        a > b;  // Solvable: just needs a > b
    }
endclass
```

### Pattern 5: Array Size Constraint with Element Constraints

```verilog
class variable_payload;
    rand bit [7:0] payload[];
    rand int unsigned payload_size;

    constraint c_size {
        payload_size inside {[4:64]};
        payload.size() == payload_size;
    }

    constraint c_elements {
        foreach (payload[i]) {
            payload[i] != 8'h00;  // No zero bytes
            payload[i] != 8'hFF;  // No all-ones bytes
        }
    }

    // GOTCHA: constraining size AND elements simultaneously can be slow
    // for large arrays. Solver must reason about every element.
endclass
```

### Pattern 6: solve...before for Controlling Solver Order

```verilog
class solve_before_demo;
    rand bit       flag;
    rand bit [7:0] value;

    constraint c_flag_value {
        flag -> (value inside {[200:255]});
        !flag -> (value inside {[0:55]});
    }

    // WITHOUT solve-before:
    // Solver picks (flag, value) pairs uniformly from solution space
    // flag=1 has 56 solutions (200-255), flag=0 has 56 solutions (0-55)
    // So flag is ~50/50

    // WITH solve-before:
    constraint c_order {
        solve flag before value;
    }
    // Now solver picks flag first (50% each), then value given flag
    // Distribution of value CHANGES based on flag probability

    // Real example: solve burst_type before burst_len
    // Ensures each burst_type is equally likely, then length varies
endclass
```

### Pattern 7: Soft Constraints

```verilog
class configurable_txn;
    rand bit [7:0] addr;
    rand bit [31:0] data;

    // Soft: provides defaults that can be overridden
    constraint c_default_addr {
        soft addr inside {[0:63]};  // Default: low address range
    }

    constraint c_default_data {
        soft data < 32'h0000_FFFF;  // Default: small data values
    }
endclass

// In a specific test:
initial begin
    configurable_txn txn = new();

    // Override soft constraint with inline constraint
    if (!txn.randomize() with {
        addr inside {[200:255]};  // Hard constraint OVERRIDES soft
    })
        `uvm_fatal("RAND", "Failed")
    // addr will be in [200:255] -- soft constraint yields
    // data still < 0xFFFF (soft not overridden)
end
```

### Pattern 8: constraint_mode() On/Off

```verilog
class multi_constraint_txn;
    rand bit [7:0] addr;

    constraint c_low_addr  { addr inside {[0:63]}; }
    constraint c_high_addr { addr inside {[192:255]}; }

    function new();
        // Start with only low_addr enabled
        c_high_addr.constraint_mode(0);  // Disable
    endfunction
endclass

initial begin
    multi_constraint_txn txn = new();
    txn.randomize();  // addr in [0:63]

    txn.c_low_addr.constraint_mode(0);   // Disable low
    txn.c_high_addr.constraint_mode(1);  // Enable high
    txn.randomize();  // addr in [192:255]

    // Disable ALL constraints
    txn.constraint_mode(0);  // Called on object, disables all
    txn.randomize();         // addr = any 8-bit value
end
```

### Pattern 9: randc for Permutation

```verilog
class fair_arbiter_test;
    randc bit [1:0] channel;  // Cycles through 0,1,2,3 before repeating

    // randc guarantees: each value appears exactly once per cycle
    // After all values used, the cycle restarts (new random permutation)

    // Example sequence: 2,0,3,1 | 1,3,0,2 | 3,2,1,0 | ...
    //                    cycle 1   cycle 2   cycle 3
endclass

// GOTCHA: randc + constraints can make cycling impossible
// If constraint eliminates some values, randc cycles only through remaining
```

### Pattern 10: Complex Cross-Variable Constraint

```verilog
class axi_txn;
    rand bit [1:0]  burst_type;  // 0=FIXED, 1=INCR, 2=WRAP
    rand bit [7:0]  burst_len;   // 0-255 (actual length = burst_len + 1)
    rand bit [2:0]  burst_size;  // 2^burst_size bytes per beat
    rand bit [31:0] addr;

    // AXI WRAP burst: length must be 2, 4, 8, or 16
    constraint c_wrap_len {
        (burst_type == 2) -> burst_len inside {1, 3, 7, 15};
    }

    // AXI: address must be aligned to burst_size
    constraint c_addr_align {
        (addr % (1 << burst_size)) == 0;
    }

    // AXI4: burst_len max depends on burst_type
    constraint c_max_len {
        (burst_type == 0) -> (burst_len <= 15);   // FIXED: max 16 beats
        (burst_type == 2) -> (burst_len inside {1, 3, 7, 15}); // WRAP
        // INCR: up to 256 beats (burst_len up to 255)
    }

    // 4KB boundary check (AXI cannot cross 4KB boundary)
    constraint c_4kb {
        (burst_type == 1) ->
            ((addr % 4096) + ((burst_len + 1) * (1 << burst_size))) <= 4096;
    }
endclass
```

---

## Constraint Solving Worked Example

### A Class With 5 Constraint Types -- Step by Step

```verilog
class axi_burst_txn;
    rand bit [1:0]  burst;     // 00=FIXED, 01=INCR, 10=WRAP
    rand bit [7:0]  len;       // 0-255 (beats = len+1)
    rand bit [2:0]  size;      // bytes per beat = 2^size
    rand bit [31:0] addr;
    rand bit        is_cacheable;

    // Constraint 1: RANGE constraint
    constraint c_size_range {
        size inside {[0:4]};  // 1, 2, 4, 8, or 16 bytes per beat
    }

    // Constraint 2: IMPLICATION constraint
    constraint c_wrap_len {
        // WRAP burst: len must be power-of-2 minus 1
        (burst == 2'b10) -> (len inside {1, 3, 7, 15});
    }

    // Constraint 3: SET MEMBERSHIP constraint
    constraint c_burst_type {
        burst inside {2'b00, 2'b01, 2'b10};  // exclude reserved 2'b11
    }

    // Constraint 4: SOFT constraint (can be overridden)
    constraint c_default_addr {
        soft addr[3:0] == 4'b0000;  // Default: aligned to 16 bytes
    }

    // Constraint 5: SOLVE-BEFORE constraint
    constraint c_solve_order {
        solve burst before len;  // Pick burst type first, then solve len
    }
endclass
```

**Step-by-step solver walkthrough:**

``` text
1. SOLVE-BEFORE:
   - Solver picks `burst` first.
   - Available: {00, 01, 10} (Constraint 3 excludes 11).
   - Probability: 1/3 each.
   - Example: Solver picks `burst = 10` (WRAP).

2. RANGE Constraint (`c_size_range`): 
   - `size` must be in {0, 1, 2, 3, 4}.
   - Example: Solver picks `size = 2` (4 bytes per beat), probability 1/5.

3. IMPLICATION Constraint (`c_wrap_len`):
   - If `burst == 10`, `len` must be in {1, 3, 7, 15}.
   - Available: {1, 3, 7, 15} (4 values).
   - Example: Solver picks `len = 7` (8-beat burst), probability 1/4.

4. SOFT Constraint (`c_default_addr`):
   - `addr[3:0] == 4'b0000` (default: aligned).
   - Since no hard constraint overrides it, `addr[3:0] = 0`.
   - `addr[31:4] = random` (any value).
   - Result: `addr` is aligned to a 16-byte boundary.

5. Remaining Unconstrained Variables:
   - `is_cacheable`: random (50% each).

Sample Output:
- burst = 10 (WRAP)
- len = 7 (8 beats)
- size = 2 (4 bytes/beat)
- addr = 32'h0001_0010 (16-byte aligned)
- is_cacheable = 1
```

**Overriding the soft constraint:**
```verilog
initial begin
    axi_burst_txn txn = new();
    
    // Override soft addr alignment with hard constraint
    txn.randomize() with {
        addr inside {[32'h1000_0000:32'h1000_FFFF]};
    };
    // addr[3:0] is now random (soft constraint yields to hard inline constraint)
    // addr is in range 0x10000000-0x1000FFFF
    // burst, len, size still follow their hard constraints
    
    // If we try to violate a hard constraint:
    // txn.randomize() with { burst == 2'b11; };  // FAILS! Returns 0.
end
```

---

## Coverage-Driven Verification -- Complete Reference

### Covergroup Definition and Sampling

```verilog
class axi_coverage extends uvm_subscriber #(axi_burst_txn);
    `uvm_component_utils(axi_coverage)

    axi_burst_txn txn;

    covergroup cg_axi @(posedge vif.clk);
        // Coverpoint 1: Auto bins (each value gets its own bin)
        cp_burst: coverpoint txn.burst {
            bins fixed = {2'b00};
            bins incr  = {2'b01};
            bins wrap  = {2'b10};
            bins reserved = {2'b11};  // Should never hit!
        }

        // Coverpoint 2: Explicit range bins
        cp_len: coverpoint txn.len {
            bins single    = {0};           // 1 beat
            bins short_bst = {[1:3]};       // 2-4 beats
            bins med_bst   = {[4:15]};      // 5-16 beats
            bins long_bst  = {[16:255]};     // 17-256 beats
        }

        // Coverpoint 3: Transition bins (state machine coverage)
        cp_burst_trans: coverpoint txn.burst {
            bins fixed_to_incr = (2'b00 => 2'b01);
            bins incr_to_wrap  = (2'b01 => 2'b10);
            bins wrap_to_fixed = (2'b10 => 2'b00);
            bins same_burst    = (2'b00 => 2'b00), (2'b01 => 2'b01), (2'b10 => 2'b10);
        }

        // Coverpoint 4: Cross coverage (combinatorial explosion management)
        cp_size: coverpoint txn.size {
            bins byte_1  = {0};
            bins byte_2  = {1};
            bins byte_4  = {2};
            bins byte_8  = {3};
            bins byte_16 = {4};
        }

        // Cross: burst type x length category
        cx_burst_len: cross cp_burst, cp_len {
            // WRAP burst only has specific lengths (from AXI spec)
            ignore_bins wrap_short = binsof(cp_burst.wrap) && 
                                     binsof(cp_len) intersect {0};
        }

        // Cross: burst type x size x cacheable
        cp_cacheable: coverpoint txn.is_cacheable;
        cx_burst_size_cache: cross cp_burst, cp_size, cp_cacheable {
            // Non-cacheable FIXED burst is common; cacheable FIXED is unusual
            // Still legal, so no ignore_bins, but coverage target is lower
        }

        // Options
        option.per_instance = 1;
        option.at_least = 3;  // Each bin needs 3 hits for coverage credit
        option.goal = 95;     // 95% coverage target
    endgroup

    function new(string name, uvm_component parent);
        super.new(name, parent);
        cg_axi = new();
    endfunction

    function void write(axi_burst_txn t);
        txn = t;
        cg_axi.sample();
    endfunction

    function void report_phase(uvm_phase phase);
        $display("AXI Coverage: %.1f%%", cg_axi.get_coverage());
    endfunction
endclass
```

### Functional Coverage vs Code Coverage

```text
FUNCTIONAL COVERAGE vs. CODE COVERAGE

1. Functional Coverage
   - What it measures: "Did we test all SPECIFIED behaviors?"
   - How it's collected: Covergroups and coverpoints explicitly written by the verification engineer.

2. Code Coverage
   - What it measures: "Did we execute all lines, branches, FSM states, and conditions?"
   - How it's collected: Automatically extracted by the simulator from the RTL source.

Types of Code Coverage:
- Line coverage: Did each line of RTL execute?
- Toggle coverage: Did each net toggle 0->1 and 1->0?
- Branch coverage: Did each if-else branch execute?
- FSM coverage: Did each state transition occur?
- Condition coverage: Did each Boolean sub-expression evaluate to both true and false?
- Expression coverage: Did each logical expression hit all input combinations?

Relationship & Pitfalls:
- 100% Code Coverage ≠ Correct Design: You can execute all code but still miss important scenarios.
  Example: A 32-bit counter counts from 0 to 2^32-1. Code coverage is 100%, but functional coverage fails if overflow or wrap-around wasn't tested.
- 100% Functional Coverage ≠ All Bugs Found: The specification itself may be incomplete.
  Example: A FIFO is tested with all read/write/empty/full combinations (100% functional coverage). However, if the RTL has a typo where data is inverted on even clock cycles, neither coverage catches this unless explicitly specified.

Best Practice: 
Use BOTH. Treat code coverage as a baseline floor and functional coverage for spec-specific scenarios. True coverage closure requires both metrics to be above their respective thresholds.
```

### Coverage Closure Methodology

Phase 1: Define coverage model from specification
- Extract all functional requirements from the design spec
   - For each requirement, define coverpoints and cross coverage
   - Set coverage goals (typically 95% for functional, 100% for code)
   - Estimate total bin count (watch for cross coverage explosion)

Phase 2: Run initial regression with random stimulus
- Use constrained-random sequences (UVM factory + sequences)
   - Collect coverage database (.ucdb for Questa, .covr for VCS)
   - Check initial coverage: typically 40-70% after first run

Phase 3: Analyze coverage holes
- Identify uncovered bins (which scenarios weren't hit?)
   - For each hole, determine: is it a test gap or an impossible scenario?
   - Impossible scenarios: add ignore_bins
   - Test gaps: add directed sequences or adjust constraints

Phase 4: Targeted stimulus for coverage closure
- Write directed sequences for specific uncovered bins
   - Adjust random constraints to steer stimulus toward holes
   - Use coverage feedback to guide regression (coverage-driven closure)
   - Run multiple seeds to increase random coverage

Phase 5: Coverage merge and final regression
- Merge coverage from all tests and seeds:
   - vcover merge total.ucdb seed1.ucdb seed2.ucdb ... seedN.ucdb
   - Verify total coverage meets signoff threshold
   - Document any remaining holes with justification

**Signoff criteria:**
   - Functional coverage: >= 95% (with documented exceptions)
   - Code coverage: >= 100% line, >= 95% branch, >= 90% condition
   - All illegal_bins: zero hits
   - Coverage model reviewed by design and verification leads

---

## rand_mode: Enable/Disable Randomization Per Variable

```verilog
class txn;
    rand bit [7:0] addr;
    rand bit [31:0] data;

    function void set_directed_addr(bit [7:0] a);
        this.addr = a;
        this.addr.rand_mode(0);  // Don't randomize addr anymore
    endfunction
endclass

initial begin
    txn t = new();
    t.set_directed_addr(8'hAB);
    t.randomize();  // Only data is randomized, addr stays 8'hAB
end
```

---

## pre_randomize and post_randomize

```verilog
class sequenced_packet extends uvm_sequence_item;
    static int seq_counter = 0;
    rand bit [7:0] addr;
    rand bit [31:0] data;
    int sequence_num;
    bit [7:0] checksum;

    function void pre_randomize();
        // Called BEFORE randomization -- set up state
        $display("About to randomize packet");
    endfunction

    function void post_randomize();
        // Called AFTER randomization -- compute derived fields
        sequence_num = seq_counter++;
        checksum = addr ^ data[7:0] ^ data[15:8] ^ data[23:16] ^ data[31:24];
        $display("Packet #%0d: addr=%h data=%h csum=%h",
                 sequence_num, addr, data, checksum);
    endfunction
endclass
```

---

## std::randomize -- Randomize Without a Class

```verilog
initial begin
    bit [7:0] addr;
    bit [31:0] data;
    bit [3:0] len;

    // Randomize local variables with inline constraints
    if (!std::randomize(addr, data, len) with {
        addr inside {[0:63]};
        len < 8;
        data[7:0] == addr;  // Low byte of data matches addr
    })
        $fatal("std::randomize failed");

    $display("addr=%h data=%h len=%0d", addr, data, len);
end
```

### The local:: Scope Qualifier

```verilog
class driver;
    bit [7:0] min_addr;
    bit [7:0] max_addr;

    task generate_addr();
        bit [7:0] addr;
        // local:: refers to variables in the calling scope (this class)
        if (!std::randomize(addr) with {
            addr >= local::min_addr;
            addr <= local::max_addr;
        })
            $fatal("randomize failed");
    endtask
endclass
```

---

## Random Stability

### How Seeds Propagate

SystemVerilog guarantees **thread stability** and **type stability**:

- **Thread stability**: each process (initial, always, fork branch) has its own random number
  generator (RNG) state. Adding/removing code in one process doesn't affect random sequences
  in other processes.

- **Type stability**: each class type has independent random state. Creating more objects of
  class A doesn't change the random sequence for class B objects.

### Why Reordering Code Changes Random Sequences

```verilog
initial begin
    Packet p1 = new();
    Packet p2 = new();
    p1.randomize();  // Gets random value sequence R1
    p2.randomize();  // Gets random value sequence R2
end

// If you swap creation order:
initial begin
    Packet p2 = new();  // p2 is created first
    Packet p1 = new();
    p1.randomize();  // Gets DIFFERENT sequence than before!
    // Because p1's RNG was seeded differently (creation order matters)
end
```

### Getting Reproducible Results

- Use `+seed=<value>` on the command line to set the root seed
- From a fixed root seed, the hierarchical seeding is deterministic
- `srandom(seed)` on an object sets its RNG explicitly
- For debug: print the seed with `get_randstate()` and restore with `set_randstate()`

```verilog
Packet p = new();
string saved_state = p.get_randstate();  // Save RNG state
p.randomize();  // Some random values
p.set_randstate(saved_state);  // Restore state
p.randomize();  // SAME random values as before
```

---

## Parameterized Classes

```verilog
class fifo #(type T = int, int DEPTH = 16);
    T storage[$:DEPTH-1];

    function bit push(T item);
        if (storage.size() >= DEPTH) return 0;
        storage.push_back(item);
        return 1;
    endfunction

    function bit pop(output T item);
        if (storage.size() == 0) return 0;
        item = storage.pop_front();
        return 1;
    endfunction
endclass

// Usage:
fifo #(Packet, 32) pkt_fifo = new();
fifo #(int, 256) int_fifo = new();
fifo #(bit [7:0]) byte_fifo = new();  // Default DEPTH = 16
```

---

## Abstract Classes and Pure Virtual Methods

```verilog
virtual class base_driver;  // Cannot instantiate directly
    pure virtual task drive(input Transaction txn);  // Must be overridden
    pure virtual function bit check_ready();

    // Concrete methods allowed in abstract class
    function void log(string msg);
        $display("T=%0t [%s] %s", $time, get_name(), msg);
    endfunction

    pure virtual function string get_name();
endclass

class axi_driver extends base_driver;
    task drive(input Transaction txn);
        // AXI-specific driving logic
    endtask

    function bit check_ready();
        return vif.ARREADY | vif.AWREADY;
    endfunction

    function string get_name();
        return "AXI_DRV";
    endfunction
endclass

// base_driver d = new();  // COMPILE ERROR -- abstract class
axi_driver d = new();      // OK
base_driver handle = d;    // OK -- polymorphic handle
```

---

## Typedef Class (Forward Declaration)

```verilog
// When two classes reference each other:
typedef class B;  // Forward declare B

class A;
    B b_handle;  // OK because B is forward-declared
endclass

class B;
    A a_handle;
endclass
```

---

## Interface Classes (SystemVerilog 2012)

```verilog
interface class printable;
    pure virtual function void print();
endclass

interface class comparable;
    pure virtual function bit compare(uvm_object rhs);
endclass

// A class can implement multiple interface classes
class my_txn extends uvm_sequence_item implements printable, comparable;
    function void print();
        $display("my_txn: addr=%h", addr);
    endfunction

    function bit compare(uvm_object rhs);
        my_txn rhs_txn;
        if (!$cast(rhs_txn, rhs)) return 0;
        return (this.addr == rhs_txn.addr && this.data == rhs_txn.data);
    endfunction
endclass
```

---

