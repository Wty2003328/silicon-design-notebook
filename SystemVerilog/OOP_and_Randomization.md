# SystemVerilog OOP and Randomization -- Senior Engineer Deep Dive

---

## Class Fundamentals

### Handle vs Object -- The #1 Source of Confusion

A **handle** is a pointer/reference. An **object** is the actual memory allocation. Multiple
handles can point to the same object. A handle can be `null`.

```systemverilog
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

```systemverilog
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

```systemverilog
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

```systemverilog
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

```systemverilog
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

```systemverilog
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

```systemverilog
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

```systemverilog
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

```systemverilog
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

```systemverilog
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

```systemverilog
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

```systemverilog
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

```systemverilog
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

```systemverilog
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

```systemverilog
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

```systemverilog
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

```systemverilog
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

```systemverilog
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

```systemverilog
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

```systemverilog
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

```systemverilog
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

## rand_mode: Enable/Disable Randomization Per Variable

```systemverilog
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

```systemverilog
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

```systemverilog
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

```systemverilog
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

```systemverilog
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

```systemverilog
Packet p = new();
string saved_state = p.get_randstate();  // Save RNG state
p.randomize();  // Some random values
p.set_randstate(saved_state);  // Restore state
p.randomize();  // SAME random values as before
```

---

## Parameterized Classes

```systemverilog
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

```systemverilog
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

```systemverilog
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

```systemverilog
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

## Interview Questions and Answers

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

```systemverilog
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

```systemverilog
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
RNG) and type stability (each class type has independent state). This matters because: (1) you
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

```systemverilog
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
