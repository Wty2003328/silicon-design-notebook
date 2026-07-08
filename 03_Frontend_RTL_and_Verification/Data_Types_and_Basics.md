# SystemVerilog Data Types and Basics -- Senior Engineer Deep Dive

---

## 2-State vs 4-State Types

### The Fundamentals

4-state types hold values 0, 1, X (unknown), Z (high-impedance).
2-state types hold only 0 and 1.

```verilog
// 4-state types: logic, reg, wire, integer, time
logic       [7:0] data;       // Default value: 8'hxx
reg         [7:0] old_style;  // Default value: 8'hxx
wire        [7:0] w;          // Default value: 8'hzz
integer            i;          // Default value: 32'hxxxxxxxx

// 2-state types: bit, byte, shortint, int, longint, real, shortreal
bit         [7:0] fast_data;  // Default value: 8'h00
byte               b;          // Default value: 0 (signed 8-bit)
shortint           si;         // Default value: 0 (signed 16-bit)
int                n;          // Default value: 0 (signed 32-bit)
longint            li;         // Default value: 0 (signed 64-bit)
```

### Why 2-State Is ~2x Faster in Simulation

Simulators represent 4-state values using TWO bits per signal bit internally -- one "value" bit
and one "strength/unknown" bit. A 32-bit `logic` variable requires 64 bits of simulator memory
and every operation (AND, OR, addition) must compute on both bit planes. 2-state types use
exactly one bit per signal bit, halving memory and simplifying every operation to native CPU
instructions. On large testbench data structures (scoreboards with thousands of transactions),
2-state types can deliver 1.5x-2x simulation speedup.

### X-Propagation: The Critical Verification Trade-Off

**X-optimism** (the dangerous default in most simulators): when an X feeds into control logic
(if/case), the simulator picks a branch -- typically the else/default branch. This HIDES bugs.

**X-pessimism**: when X feeds into data logic, it spreads aggressively. `0 & X = 0` is correct,
but `X + 0 = X` means one uninitialized field corrupts an entire computation.

```verilog
// BUG EXAMPLE: 2-state misses uninitialized register
module buggy_mux (input logic sel, input logic [7:0] a, b, output logic [7:0] y);
    logic [7:0] internal_reg; // OOPS: never assigned in one path

    always_comb begin
        if (sel)
            internal_reg = a;
        // BUG: no else branch -- internal_reg retains old value (latch)
        y = internal_reg;
    end
endmodule

// In 4-state simulation: internal_reg starts as 8'hxx, y shows X on waveform
//   => Engineer sees the X, finds the bug immediately
// In 2-state simulation: internal_reg starts as 8'h00, y shows 00
//   => Bug is HIDDEN -- appears to work with specific test vectors
```

**Rule of thumb for industry practice:**
- RTL signals: ALWAYS use 4-state (`logic`). You want X to propagate and catch bugs.
- Testbench variables: Use 2-state (`bit`, `int`) for performance. Testbench data is
  initialized by your code, so X detection is irrelevant.
- Scoreboards/coverage: 2-state for speed.

### The `logic` Type -- Why It Replaced `reg` and `wire`

In Verilog, you needed `reg` for procedural blocks and `wire` for continuous assignments.
The names were confusing -- `reg` does NOT mean register. SystemVerilog's `logic` works in both
contexts with ONE restriction: it cannot have multiple drivers (use `wire` for that).

```verilog
logic [7:0] data;

// Works in procedural context
always_comb data = a & b;

// Also works in continuous assignment
assign data = a & b;

// But NOT this -- multiple drivers require wire/tri
// assign data = a;  // ERROR if data is also driven procedurally
```

---

## Packed vs Unpacked Arrays

### Memory Layout

**Packed arrays** are stored as contiguous bit vectors. You can treat the entire array as a
single vector, bit-select into it, and pass it across module ports or DPI boundaries cleanly.

**Unpacked arrays** are stored as arrays of elements -- each element may have its own memory
alignment. The simulator is free to add padding between elements.

```verilog
// Packed: stored as one contiguous 32-bit vector
logic [3:0][7:0] packed_data;   // 4 bytes packed into 32 bits
// Memory: [byte3][byte2][byte1][byte0] -- one 32-bit word

// Unpacked: stored as 4 separate 8-bit elements
logic [7:0] unpacked_data [4];  // 4 bytes, potentially with gaps
// Memory: [byte0][pad?][byte1][pad?][byte2][pad?][byte3][pad?]

// Key difference: packed can be used as a whole vector
logic [31:0] flat = packed_data;          // OK -- direct assignment
// logic [31:0] flat2 = unpacked_data;    // ERROR -- cannot flatten unpacked
```

### Why Packed Matters for Ports and DPI

Module ports that connect to RTL signals must be packed (or scalar). When calling C functions
via DPI-C, packed arrays map cleanly to C integer types or `svBitVecVal` arrays, while unpacked
arrays require `svOpenArrayHandle` and complex accessor functions.

```verilog
// DPI-C import -- packed maps to simple C types
import "DPI-C" function int c_checksum(input bit [31:0] data);
// In C: int c_checksum(const svBitVecVal* data);  // simple pointer

// Unpacked requires complex open array handling
import "DPI-C" function void c_process(input bit [7:0] data[]);
// In C: void c_process(const svOpenArrayHandle data);
//        Access: *((int8_t*)svGetArrayPtr(data) + svLow(data,1))
```

### Packed Array Bit-Select Gotcha

```verilog
logic [3:0][7:0] pdata;  // packed: 32 bits total
pdata[2] = 8'hAB;        // Selects the THIRD byte (bits [23:16])
pdata[2][3] = 1'b1;      // Selects bit 19 (byte 2, bit 3)

// Unpacked -- different indexing semantics
logic [7:0] udata [4];
udata[2] = 8'hAB;        // Selects element at index 2
udata[2][3] = 1'b1;      // Selects bit 3 of element 2
```

### Mixed Packed/Unpacked

```verilog
// Packed dimensions are to the LEFT of the name, unpacked to the RIGHT
logic [1:0][7:0] memory [256];
// memory is: 256 entries (unpacked), each 16-bit (2 packed bytes)
// memory[5]      => 16-bit packed value
// memory[5][1]   => upper byte of entry 5
// memory[5][1][3]=> bit 3 of upper byte of entry 5
```

---

## Dynamic Arrays

### Complete Lifecycle

```verilog
int dyn_arr[];              // Declare -- handle is null, size is 0

initial begin
    dyn_arr = new[10];      // Allocate 10 elements, initialized to 0
    $display("Size: %0d", dyn_arr.size());  // 10

    foreach (dyn_arr[i]) dyn_arr[i] = i * 10;

    dyn_arr = new[20](dyn_arr);  // Resize to 20, COPY old data
    // dyn_arr[0..9] = old values, dyn_arr[10..19] = 0

    dyn_arr = new[5];       // Resize to 5, OLD DATA IS LOST (no copy arg)

    dyn_arr.delete();       // Free memory, size becomes 0
    // dyn_arr = new[0];    // Equivalent to delete
end
```

### Performance: Dynamic Array vs Queue

| Operation | Dynamic Array | Queue |
|---|---|---|
| Random access `arr[i]` | O(1) | O(1) -- but higher constant factor |
| Push to back | O(n) -- must resize+copy | O(1) amortized |
| Push to front | O(n) | O(1) amortized |
| Pop from back | O(n) -- must resize+copy | O(1) |
| Pop from front | O(n) | O(1) |
| Resize | O(n) -- allocate+copy | N/A (automatic) |
| Memory overhead | Minimal | Higher (linked list / deque internals) |

**Rule:** Use dynamic arrays for fixed-size-after-creation data. Use queues when you need
push/pop. In UVM, queues are overwhelmingly preferred because transactions arrive one at a time.

---

## Associative Arrays

### Hash-Based Implementation

Associative arrays are implemented as hash tables (or balanced trees depending on the
simulator). They allocate memory only for indices that are actually used -- perfect for
sparse data.

```verilog
// Syntax: type name [index_type];
int scores [string];           // String-indexed
bit [31:0] mem [bit [63:0]];   // 64-bit address indexed (sparse memory)
Transaction txn_db [int];      // Object-indexed scoreboard

initial begin
    scores["Alice"] = 95;
    scores["Bob"] = 87;

    // Check existence BEFORE access (avoids creating default entry)
    if (scores.exists("Charlie"))
        $display("Charlie: %0d", scores["Charlie"]);
    else
        $display("Charlie not found");

    // Iterate in sorted order of keys (for integral keys)
    // For string keys: lexicographic order
    foreach (scores[name])
        $display("%s => %0d", name, scores[name]);

    // Navigation methods
    string first_key;
    if (scores.first(first_key))
        $display("First: %s", first_key);

    scores.delete("Bob");      // Remove one entry
    scores.delete();           // Remove all entries
    $display("Num entries: %0d", scores.num());
end
```

### Practical Use Case: Scoreboard Transaction Tracking

```verilog
class scoreboard extends uvm_scoreboard;
    // Track outstanding transactions by ID
    Transaction expected_txns [int];  // Associative array indexed by txn ID

    function void add_expected(Transaction t);
        if (expected_txns.exists(t.id))
            `uvm_error("SCB", $sformatf("Duplicate ID %0d", t.id))
        expected_txns[t.id] = t;
    endfunction

    function void check_actual(Transaction actual);
        if (!expected_txns.exists(actual.id)) begin
            `uvm_error("SCB", $sformatf("Unexpected txn ID %0d", actual.id))
            return;
        end
        if (!expected_txns[actual.id].compare(actual))
            `uvm_error("SCB", "Data mismatch")
        expected_txns.delete(actual.id);  // Remove matched entry
    endfunction

    function void check_phase(uvm_phase phase);
        if (expected_txns.num() > 0)
            `uvm_error("SCB", $sformatf("%0d unmatched transactions", expected_txns.num()))
    endfunction
endclass
```

### When to Use Associative vs Dynamic

| Scenario | Use |
|---|---|
| Sparse memory model (4GB address space, few entries) | Associative `[bit[31:0]]` |
| String-keyed lookup table | Associative `[string]` |
| Dense sequential data, known size | Dynamic array |
| Protocol ID tracking | Associative `[int]` |
| FIFO-like processing | Queue |

---

## Queues

### Complete API with Edge Cases

```verilog
int q[$];                     // Unbounded queue
int bq[$:15];                 // Bounded queue: max 16 elements (0..15)

initial begin
    // Push operations
    q.push_back(10);          // q = {10}
    q.push_back(20);          // q = {10, 20}
    q.push_front(5);          // q = {5, 10, 20}

    // Insert at position
    q.insert(1, 7);           // q = {5, 7, 10, 20}

    // Pop operations
    int val = q.pop_front();  // val = 5, q = {7, 10, 20}
    val = q.pop_back();       // val = 20, q = {7, 10}

    // Size and indexing
    $display("Size: %0d", q.size());  // 2
    $display("First: %0d", q[0]);     // 7
    $display("Last: %0d", q[$]);      // 10 -- $ means last index

    // Slice operations (queue = concatenation)
    int q2[$] = {1, 2, 3, 4, 5};
    int slice[$] = q2[1:3];   // {2, 3, 4} -- elements 1 through 3

    // Delete
    q2.delete(2);             // Remove element at index 2, q2 = {1, 2, 4, 5}
    q2.delete();              // Remove all elements
end
```

### Edge Cases That Cause Bugs

```verilog
// GOTCHA 1: Pop from empty queue
int q[$];
int val = q.pop_front();  // RUNTIME ERROR or returns default (0)
// ALWAYS check size() first:
if (q.size() > 0) val = q.pop_front();

// GOTCHA 2: Bounded queue overflow
int bq[$:3];               // Max 4 elements
bq = {1, 2, 3, 4};         // Full
bq.push_back(5);           // SILENTLY DROPPED or error (simulator-dependent)
// ALWAYS check: if (bq.size() < 4) bq.push_back(5);

// GOTCHA 3: Queue vs dynamic array assignment
int q3[$] = {1, 2, 3};
int da[];
da = q3;                    // OK -- copies queue to dynamic array
q3 = da;                    // OK -- copies dynamic array to queue
// But they are DIFFERENT types in function signatures
```

### Why Queues Are Preferred in UVM

1. Transactions arrive one at a time -- `push_back` is O(1)
2. Scoreboard drains from front -- `pop_front` is O(1)
3. No need to know size in advance
4. Natural FIFO behavior matches protocol ordering
5. `$` indexing for peeking at newest/oldest without removal

---

## Structs and Unions

### Packed Struct for Register Modeling

```verilog
// Real-world register definition -- USB endpoint control register
typedef struct packed {
    logic        ep_enable;    // Bit [31]
    logic        ep_stall;     // Bit [30]
    logic [1:0]  ep_type;      // Bits [29:28] -- 00=ctrl, 01=iso, 10=bulk, 11=intr
    logic [3:0]  ep_number;    // Bits [27:24]
    logic        data_toggle;  // Bit [23]
    logic [2:0]  rsvd;         // Bits [22:20] -- reserved
    logic [9:0]  max_pkt_size; // Bits [19:10]
    logic [9:0]  rsvd2;        // Bits [9:0]
} usb_ep_ctrl_t;

// Because it's PACKED, this struct is exactly 32 bits, stored contiguously
usb_ep_ctrl_t ep0_ctrl;
logic [31:0] reg_data;

// Direct assignment between packed struct and vector
reg_data = ep0_ctrl;                       // Flatten to 32-bit vector
ep0_ctrl = 32'h8400_0200;                  // Assign from raw value
$display("EP type: %b", ep0_ctrl.ep_type); // Access named field

// Bit-select works on the entire packed struct
logic msb = ep0_ctrl[31];                  // Same as ep0_ctrl.ep_enable
```

### Tagged Union for Type Safety

```verilog
// Regular union -- NO type safety, you must track which member is valid
typedef union packed {
    logic [31:0] raw;
    struct packed {
        logic [15:0] upper;
        logic [15:0] lower;
    } halves;
} word_u;

// Tagged union -- compiler tracks which member is active
typedef union tagged {
    void       Invalid;
    int        IntVal;
    real       RealVal;
    string     StrVal;
} tagged_data_t;

tagged_data_t td;
td = tagged IntVal 42;
case (td) matches
    tagged IntVal .v:  $display("Int: %0d", v);
    tagged RealVal .v: $display("Real: %f", v);
    tagged StrVal .v:  $display("Str: %s", v);
    tagged Invalid:    $display("Invalid");
endcase
```

### Struct Padding and Alignment

```verilog
// UNPACKED struct -- simulator may add padding
typedef struct {
    byte  a;      // 8 bits + possible padding to 32-bit boundary
    int   b;      // 32 bits
    bit   c;      // 1 bit + padding
} padded_s;
// sizeof(padded_s) is implementation-defined -- could be 5, 8, or 12 bytes

// PACKED struct -- NO padding, exact bit count
typedef struct packed {
    logic [7:0]  a;   // 8 bits
    logic [31:0] b;   // 32 bits
    logic        c;   // 1 bit
} compact_s;          // Exactly 41 bits, no padding
```

---

## Casting

### $cast for Class Hierarchy

```verilog
class Animal;
    virtual function string speak();
        return "...";
    endfunction
endclass

class Dog extends Animal;
    function string speak();
        return "Woof";
    endfunction
    function void fetch();
        $display("Fetching!");
    endfunction
endclass

initial begin
    Animal a;
    Dog d = new();
    a = d;           // Upcast: ALWAYS safe, implicit

    // Downcast: MUST use $cast
    Dog d2;
    // d2 = a;       // COMPILE ERROR -- cannot assign base to derived
    if (!$cast(d2, a)) begin
        $fatal("Cast failed -- a doesn't point to a Dog");
    end
    d2.fetch();      // Now safe to call Dog-specific method
end
```

### The FAILURE Case in UVM (Factory Overrides)

```verilog
class base_txn extends uvm_sequence_item;
    `uvm_object_utils(base_txn)
    rand bit [7:0] addr;
endclass

class error_txn extends base_txn;
    `uvm_object_utils(error_txn)
    rand bit inject_error;
endclass

// In a test using factory override:
class error_test extends uvm_test;
    function void build_phase(uvm_phase phase);
        // Tell factory: whenever someone creates base_txn, give them error_txn
        base_txn::type_id::set_type_override(error_txn::get_type());
    endfunction
endclass

// In the sequence:
task body();
    base_txn txn;
    txn = base_txn::type_id::create("txn");  // Factory returns error_txn

    // If you need to access error_txn-specific fields:
    error_txn etxn;
    if (!$cast(etxn, txn)) begin
        `uvm_fatal("SEQ", "Factory didn't return error_txn -- check override")
    end
    etxn.inject_error = 1;  // Now accessible
endtask
```

### Static Cast for Basic Types

```verilog
// int'(), bit'(), signed'(), unsigned'() -- compile-time cast
real r = 3.14;
int i = int'(r);           // i = 3 (truncated)

logic signed [7:0] s = -5;
logic [7:0] u = unsigned'(s);  // u = 251 (8'hFB)

// Type casting for enums
typedef enum {RED, GREEN, BLUE} color_e;
color_e c = color_e'(1);   // c = GREEN
int v = int'(c);            // v = 1
```

---

## Array Manipulation Methods

### Non-Trivial Examples

```verilog
// find_index with complex predicate
int data[] = '{10, 25, 30, 45, 50, 65, 70};
int idx[$] = data.find_index with (item > 40 && item < 60);
// idx = {3, 4} (indices of 45 and 50)

// find_first / find_last
int first_big[$] = data.find_first with (item > 50);
// first_big = {65}

// sort with custom comparator using 'with' clause
typedef struct {
    string name;
    int    score;
} student_t;
student_t students[];
// ... populate students ...
students.sort with (item.score);           // Sort by score ascending
students.rsort with (item.score);          // Sort by score descending

// unique -- remove duplicates (returns unique VALUES, not in-place)
int dupes[] = '{1, 3, 2, 3, 1, 4};
int uniq[$] = dupes.unique;  // {1, 3, 2, 4} -- first occurrence of each

// reduce for checksum
bit [7:0] payload[] = '{8'hDE, 8'hAD, 8'hBE, 8'hEF};
bit [7:0] xor_sum = payload.xor with (item);        // DE ^ AD ^ BE ^ EF
bit [7:0] add_sum = payload.sum with (int'(item));   // (DE + AD + BE + EF) & FF

// min/max
int values[] = '{-5, 100, 42, -200, 0};
int mn[$] = values.min;  // {-200}
int mx[$] = values.max;  // {100}

// shuffle -- randomize order in place
int deck[] = '{1,2,3,4,5,6,7,8,9,10};
deck.shuffle();
```

---

## Packages and Namespace Management

### Wildcard Import Gotcha

```verilog
package pkg_a;
    typedef enum {READ, WRITE} op_e;
    function void hello(); $display("pkg_a::hello"); endfunction
endpackage

package pkg_b;
    typedef enum {READ, WRITE, RMW} op_e;
endpackage

module top;
    import pkg_a::*;   // Wildcard: makes names CANDIDATES, not visible yet
    import pkg_b::*;   // Also candidates

    initial begin
        op_e op;  // COMPILE ERROR: ambiguous -- op_e exists in both packages
        // The fix: explicit import takes priority
    end
endmodule

module top_fixed;
    import pkg_a::*;
    import pkg_b::op_e;  // Explicit import wins over wildcard

    initial begin
        op_e op;   // OK -- uses pkg_b::op_e
        hello();   // OK -- only in pkg_a, no ambiguity
    end
endmodule
```

### Proper Package Pattern in UVM Environments

```verilog
package my_env_pkg;
    import uvm_pkg::*;
    `include "uvm_macros.svh"

    `include "my_transaction.sv"
    `include "my_driver.sv"
    `include "my_monitor.sv"
    `include "my_agent.sv"
    `include "my_scoreboard.sv"
    `include "my_env.sv"
endpackage

module tb_top;
    import uvm_pkg::*;
    import my_env_pkg::*;
    // ...
endmodule
```

---

## Signed Arithmetic Gotchas

### The Classic Sign-Extension Bug

```verilog
logic signed [7:0] a = -3;    // 8'hFD (signed)
logic signed [7:0] b = 5;     // 8'h05 (signed)

// CORRECT: both operands signed, result is signed
logic signed [8:0] sum = a + b;  // 9'h002 = 2, correct

// BUG: concatenation ALWAYS produces unsigned result
logic [8:0] bad = {1'b0, a};
// bad = 9'h0FD = 253 (unsigned!), NOT -3
// The leading 1'b0 doesn't sign-extend -- it just prepends a zero bit

// FIX: use signed cast or $signed
logic signed [8:0] good = $signed({1'b0, a});
// Or better: let the compiler sign-extend
logic signed [8:0] good2 = a;  // Automatic sign extension: 9'h1FD = -3
```

### Verilog Expression Width and Sign Rules

1. **Width determination** (before evaluation): result width = max(LHS width, RHS width)
   for most operators. Context-dependent -- assignment LHS sets the width context.
2. **Sign determination**: result is signed ONLY if ALL operands are signed.
   One unsigned operand makes the entire expression unsigned.
3. **Concatenation** `{}` and **replication** `{{}}` are ALWAYS unsigned regardless of operands.

```verilog
logic signed [3:0] s4 = -1;   // 4'hF
logic [3:0] u4 = 4'hF;        // 15

// GOTCHA: mixed signed/unsigned comparison
if (s4 > u4)  // s4 is CONVERTED to unsigned: 4'hF = 15
    $display("This prints -- 15 > 15 is false, 15 == 15");
// Actually 15 > 15 is false, so this doesn't print.
// But: if (s4 < 0) is true; if ({s4} < 0) is FALSE -- concat makes unsigned

logic signed [7:0] x = -1;    // 8'hFF
logic [15:0] y = x;           // y = 16'h00FF = 255, NOT -1!
// Why: y is unsigned, so x is zero-extended, not sign-extended
logic signed [15:0] z = x;    // z = 16'hFFFF = -1 (correct sign extension)
```

---

## Enums

```verilog
typedef enum logic [2:0] {
    IDLE   = 3'b000,
    SETUP  = 3'b001,
    ACCESS = 3'b010,
    ERROR  = 3'b100
} state_e;

state_e current_state;

// Methods
$display("Name: %s", current_state.name());  // "IDLE"
$display("Num states: %0d", current_state.num());  // 4

state_e next = current_state.next();   // Next in declaration order (wraps)
state_e prev = current_state.prev();   // Previous (wraps)
state_e first_s = current_state.first(); // IDLE
state_e last_s = current_state.last();   // ERROR

// GOTCHA: enum underlying type matters for X detection
typedef enum logic [1:0] {A, B, C} abc_e;  // 4-state base -> starts as X
typedef enum bit [1:0] {D, E, F} def_e;    // 2-state base -> starts as 0 (D)
```

---

## String Type

```verilog
string s1 = "Hello";
string s2 = "World";
string s3 = {s1, " ", s2};       // Concatenation: "Hello World"

// Methods
$display("Length: %0d", s1.len());         // 5
$display("Char: %c", s1.getc(0));         // 'H' (ASCII 72)
$display("Upper: %s", s1.toupper());      // "HELLO"
$display("Substr: %s", s1.substr(1, 3));  // "ell"
s1.putc(0, "J");                          // s1 = "Jello"

// Comparison
if (s1 == "Jello") $display("Match");
if (s1 < s2) $display("Lexically before");

// Formatting
string formatted;
formatted = $sformatf("Addr: 0x%08h Data: 0x%08h", addr, data);
```

---

## Type Compatibility and Equivalence

```verilog
// Type-equivalent: same structure, possibly different names
typedef logic [7:0] byte_a;
typedef logic [7:0] byte_b;
byte_a a; byte_b b;
a = b;    // OK -- structurally equivalent

// Type-compatible: assignment possible with implicit conversion
int i;
shortint si;
i = si;   // OK -- implicit widening
si = i;   // OK but may truncate -- some tools warn
```

---

## Interview Questions and Answers

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
(2) Module ports -- must be packed or match exactly; (3) DPI calls -- packed maps to simple
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
the entire register as a single vector. See the `usb_ep_ctrl_t` example above. In UVM RAL,
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
