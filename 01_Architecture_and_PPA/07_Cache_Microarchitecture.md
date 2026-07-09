# Cache Microarchitecture — Controller Design and Performance

> **Prerequisites:** [CPU_Architecture.md](03_CPU_Architecture.md) (pipeline basics, memory hierarchy),
> [Memory.md](09_Memory.md) (SRAM cell design, DRAM organization)
>
> **Hands off to:** [../Systems/Coherence_Protocols.md](12_ACE_and_CHI.md),
> [../Systems/Memory_Controller.md](10_DDR_Controller.md)

---

## Section 0 — Why This Page Exists

Cache design is the single topic that bridges processor microarchitecture and memory
system performance. Every high-performance core depends on its cache hierarchy to hide
the 100--300 cycle DRAM latency, and the cache controller is the logic that makes this
possible. This page covers the internals of that controller: the tag-compare datapath,
miss-handling state machines, write policies, refill optimizations, replacement
policies, prefetch engines, coherence protocols, and the power-reduction techniques
that modern designs employ.

This material appears in virtually every CPU design interview at companies that build
cores -- Apple, Arm, Qualcomm, AMD, Intel, Google, and the GPU/accelerator vendors.
The goal is to move beyond "caches store frequently used data" and reach the level
where you can whiteboard a four-way set-associative L1 data cache controller with
MSHR miss handling and MESI coherence.

---

## 1. Cache Pipeline

### 1.1 The One-Cycle Hit Path

A cache that can return data in a single cycle must perform tag lookup and data lookup
in parallel. The address from the CPU is decomposed into three fields:

$$
\text{Address} = [\,\underbrace{\text{Tag}}_{\text{upper bits}}\,|\,\underbrace{\text{Index}}_{\text{middle bits}}\,|\,\underbrace{\text{Block Offset}}_{\text{lower bits}}\,]
$$

In cycle 0, the index drives the word lines of all $N$ tag SRAMs and all $N$ data
SRAMs simultaneously. The tag bits from each way are compared against the CPU-provided
tag using $N$ parallel equality comparators. The comparator outputs are OR-reduced into
a single `hit` signal. A multiplexer steered by the winning way selects the correct
data word.

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk", "nodeSpacing": 60, "rankSpacing": 60, "htmlLabels": false}}}%%
flowchart TD
    ADDR["CPU address"] --> IDX["Index"]
    IDX --> T0["Tag SRAM 0"] --> C0["Comparator 0"]
    IDX --> T1["Tag SRAM 1"] --> C1["Comparator 1"]
    C0 --> ORr["OR → hit / miss"]
    C1 --> ORr
    IDX --> D0["Data SRAM 0"]
    IDX --> D1["Data SRAM 1"]
    D0 --> MUX["Way-select MUX"]
    D1 --> MUX
    C0 -.->|winning way| MUX
    C1 -.->|winning way| MUX
    MUX --> OUT["Data out"]
    classDef s fill:#dbeafe,stroke:#1d4ed8,color:#000
    classDef l fill:#fde68a,stroke:#b45309,color:#000
    class T0,T1,D0,D1 s
    class C0,C1,ORr,MUX l
```

This is the one-cycle hit path of a 2-way set-associative cache: the index reads both tag SRAMs and both data SRAMs in parallel; comparators check each way's tag; the OR of the hits gives hit/miss, and the winning way steers the data MUX.

**Timing constraint:** The tag SRAM read, comparator logic, and data MUX must all
complete within one clock period. At 4 GHz this is 250 ps, which is tight. The tag
SRAM is much smaller than the data SRAM, so its access time is shorter; designers
exploit this asymmetry to close timing.

### 1.2 The Two-Cycle Hit (Power-Optimized)

Reading all $N$ data SRAMs in parallel wastes power when only one way contains the
requested data. A two-cycle pipeline splits the operation:

| Cycle | Operation |
|-------|-----------|
| 1     | Read tag SRAMs for all ways; compare tags; determine winning way |
| 2     | Read only the winning way's data SRAM; return data to CPU |

This saves approximately $(N-1)/N$ of the data SRAM read energy at the cost of one
cycle of additional load-to-use latency. Most L2 and L3 caches use this scheme.
L1 caches that can afford the power may also use it when clock frequency is very high
(e.g., Apple Firestorm P-core L1D at 3.2 GHz uses a two-cycle path).

### 1.3 Tag/Data SRAM Partitioning

An $N$-way set-associative cache physically contains:

- $N$ tag SRAM arrays (each stores tag bits + valid bit + dirty bit per line)
- $N$ data SRAM arrays (each stores the data payload per line)

For a 4-way 32 KB cache with 64 B lines:

$$
\text{Sets} = \frac{32\,\text{KB}}{4 \times 64\,\text{B}} = 128\,\text{sets}
$$

Each tag SRAM has 128 entries, each holding:
- Tag bits = $32 - \lceil\log_2 128\rceil - \lceil\log_2 64\rceil = 32 - 7 - 6 = 19$ bits
- 1 valid bit
- 1 dirty bit (write-back caches)
- Total per entry: 21 bits per way, 84 bits per set across 4 ways

Each data SRAM has 128 entries of 64 B = 512 bits.

### 1.4 Hit/Miss Resolution

The hit/miss logic is a simple comparator tree:

$$
\text{hit} = \bigvee_{i=0}^{N-1} (\text{tag}_{\text{CPU}} == \text{tag}_i) \wedge \text{valid}_i
$$

On a hit, the winning way index drives the data MUX select. On a miss, the cache
controller asserts `miss` to the MSHR subsystem. The entire resolution takes one
multiplexer delay plus one OR gate delay after the tag SRAM outputs stabilize.

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk", "nodeSpacing": 60, "rankSpacing": 60, "htmlLabels": false}}}%%
flowchart TD
    A[CPU issues load address] --> B[Decode: Tag, Index, Offset]
    B --> C[Read all N tag SRAMs at Index]
    B --> D[Read all N data SRAMs at Index]
    C --> E{Tag compare per way}
    E -->|Match + Valid=1| F[hit = 1]
    E -->|No match| G[hit = 0]
    F --> H[MUX selects data from winning way]
    H --> I[Return data to CPU]
    G --> J[Miss: allocate MSHR entry]
    J --> K[Issue fill request to next level]
```

---

### 1.4 Worked Geometry Example: 32KB, 4-Way, 64B Lines

**Given:**
- Total cache size: 32 KB = 32,768 bytes
- Associativity: 4-way set-associative
- Line size: 64 bytes

**Derived parameters:**
```verilog
Number of lines = 32768 / 64 = 512 lines total
Number of sets = 512 / 4 = 128 sets
```

**Address breakdown for 32-bit address:**
```verilog
Offset bits = log2(64) = 6 bits  (byte within a line)
Index bits  = log2(128) = 7 bits  (selects the set)
Tag bits    = 32 - 6 - 7 = 19 bits

Address: [31:13 tag | 12:6 index | 5:0 offset]
```

**Hardware structure per set:**
```verilog
Set k:
  Way 0: [Valid][Dirty][Tag (19b)][Data (64B = 512b)]
  Way 1: [Valid][Dirty][Tag (19b)][Data (64B = 512b)]
  Way 2: [Valid][Dirty][Tag (19b)][Data (64B = 512b)]
  Way 3: [Valid][Dirty][Tag (19b)][Data (64B = 512b)]
  LRU state: 3 bits (for pseudo-LRU) or 5 bits (for true LRU)
```

**Total storage:**
- **Data** — 512 lines × 64 bytes = 32 KB (the actual cache data)
- **Tags** — 512 lines × 19 bits = 9728 bits = 1.19 KB
- **Valid** — 512 bits
- **Dirty** — 512 bits
- **LRU** — 128 sets × 3 bits = 384 bits (pseudo-LRU)

- **Total overhead** — ~1.2 KB for 32 KB cache = ~3.7% overhead

**Read operation (hit path):**
```verilog
1. Extract index [12:6], tag [31:13], offset [5:0] from address
2. Use index to read all 4 ways simultaneously:
   - 4 tag comparisons (19-bit each): tag == stored_tag[way_i] AND valid[way_i]
   - 4 data reads (512 bits each, but typically only read the requested word)
3. Hit detection: one-hot vector of 4 compare results
4. MUX: select the data from the matching way
5. Use offset to extract the requested byte/word from the 64B line

Latency: Tag read + comparator + MUX ≈ 2-4 cycles for L1 cache
```

---

## 2. Non-Blocking Cache and MSHR

### 2.1 Motivation

A blocking cache stalls the entire pipeline on every miss. If the L1 miss rate is 5%
and each miss costs 100 cycles, the CPI penalty is $0.05 \times 100 = 5.0$ -- half of
all cycles are wasted. A non-blocking cache allows the processor to continue executing
independent instructions while the miss is serviced.

### 2.2 MSHR Structure -- Detailed Implementation

The **Miss Status Holding Register (MSHR)** tracks every outstanding cache miss. Each
entry contains:

| Field | Width | Purpose |
|-------|-------|---------|
| `valid` | 1 bit | Entry is active |
| `line_addr` | Tag + Index bits (e.g., 26 bits for 32KB/4-way/64B) | Cache line address of the missed line |
| `req_vector` | 1 bit per word in line (e.g., 16 bits for 64B/4B words) | Bit mask: which words within the line have been requested |
| `dest_reg[0..15]` | PReg ID per word (e.g., 7 bits x 16 entries) | Physical register destinations for each requesting instruction |
| `word_valid[0..15]` | 1 bit per word | Which dest_reg entries are valid |
| `state` | 2-3 bits | Current state: IDLE, ISSUED, FILL_PENDING, WRITEBACK_PENDING |
| `way` | $\lceil\log_2(\text{associativity})\rceil$ bits | Allocated way in the cache set for the new line |
| `dirty_victim` | 1 bit | The evicted line in the allocated way is dirty (needs writeback) |
| `byte_mask` | 1 bit per byte (for write merging) | Which bytes have been written by pending stores |

**MSHR entry size calculation (4-way 32KB cache, 64B lines, 7-bit phys reg IDs):**

```verilog
1 (valid) + 26 (line_addr) + 16 (req_vector) + 16*7 (dest_reg) + 16 (word_valid)
+ 2 (state) + 2 (way) + 1 (dirty_victim)
= 1 + 26 + 16 + 112 + 16 + 2 + 2 + 1 = 176 bits per MSHR entry

For 16 MSHRs: 16 x 176 = 2,816 bits = 352 bytes of MSHR storage
```

#### Secondary Miss Handling (Miss Merging)

When a new miss arrives, the controller performs a **CAM lookup** across all valid
MSHR entries, comparing the incoming `line_addr` against stored `line_addr` values:

- **Primary miss** (no matching MSHR): Allocate a free MSHR. Set `valid=1`, store
  the line address, set the requested word's bit in `req_vector`, store the destination
  register, allocate a victim way. If the victim is dirty, set `dirty_victim=1` and
  initiate writeback. Then issue the fill request to the next level.

- **Secondary miss** (matching MSHR found): The same cache line is already being fetched.
  Merge the new request: set the additional word's bit in `req_vector`, store the new
  destination register in `dest_reg[word_index]`, set `word_valid[word_index]=1`.
  **No additional fill request is issued.** When the fill data returns, all merged
  requests are satisfied simultaneously.

**Secondary miss timing example:**

```ascii-graph
Cycle 0:  Load R1, [0x1000]  -- miss, MSHR[0] allocated, fill issued
Cycle 3:  Load R2, [0x1004]  -- miss to same line (0x1000-0x103F)
           MSHR[0] matches (line_addr = 0x1000)
           Merge: req_vector bit[1] set, dest_reg[1] = R2
           No new fill request!
Cycle 20: Fill data returns from memory
           Word 0 (0x1000) → wake R1
           Word 1 (0x1004) → wake R2
           Both instructions resume in the same cycle
```

Without merging, the second miss would occupy a separate MSHR and generate redundant
memory traffic. With merging, the cost of the second miss is zero additional memory
bandwidth and only the MSHR entry update latency.

#### Writeback Before Fill (Dirty Eviction)

When a miss requires evicting a dirty line, the MSHR controller must serialize:

1. **Select victim way** (using replacement policy).
2. **Check dirty bit** of victim.
3. **If dirty**: Write the victim line's data to the writeback buffer. The MSHR enters
   `WRITEBACK_PENDING` state. The writeback buffer drains to the next level while the
   new fill request is queued.
4. **If clean** (or after writeback completes): Issue the fill request for the new line.
   MSHR enters `ISSUED` state.
5. **Fill returns**: Write new data into the victim way's SRAM. Update tag, set valid,
   clear dirty. Wake all dependent instructions. Free the MSHR.

**Writeback buffer optimization:** A dedicated writeback buffer (typically 4-8 entries)
decouples the writeback from the fill. The dirty victim is copied to the writeback buffer
in 1 cycle, and the fill request is issued immediately. The writeback buffer drains to
the next level in the background. This reduces miss penalty by hiding the writeback latency:

**Without writeback buffer:**
   - Miss → writeback dirty victim (20-100 cycles) → fill new line (20-100 cycles)
   - Total: 40-200 cycles

**With writeback buffer:**
   - Miss → copy victim to WB buffer (1 cycle) → issue fill (20-100 cycles)
   - Total: 21-101 cycles (writeback happens in parallel with fill)

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk", "nodeSpacing": 60, "rankSpacing": 60, "htmlLabels": false}}}%%
flowchart TD
    A[Cache miss detected] --> B{MSHR exists for this line?}
    B -->|Yes: secondary miss| C[Merge request into existing MSHR]
    B -->|No: primary miss| D{Free MSHR available?}
    D -->|Yes| E[Allocate MSHR, issue fill to next<br/>level]
    D -->|No| F[Stall: all MSHRs occupied - struct<br/>hazard]
    E --> G[Fill data returns from next level]
    C --> G
    G --> H[Write line into cache SRAM]
    H --> I[Wake all dependent instructions in<br/>MSHR req_vector]
    I --> J[Free MSHR entry]
```

### 2.3 MSHR Entry Format — Detailed Field Breakdown

Each MSHR entry captures all the information needed to service a cache miss, merge
secondary misses, and deliver data to the correct destination registers when the fill
returns. Below is a field-by-field analysis for a 4-way 32 KB L1D cache with 64 B lines
and 7-bit physical register IDs (as found in a modern OoO core).

**MSHR entry fields:**

| Field | Width | Meaning |
|---|---|---|
| valid | 1 bit | entry in use |
| line_addr | 26 bits | Tag[18:0] · Index[6:0] (whole line; block offset is zero) |
| req_vector | 16 bits | one bit per 4-byte word in the 64 B line |
| dest_reg[0..15] | 16 × 7 = 112 bits | destination register per word |
| word_valid[0..15] | 16 bits | which words have arrived |
| state | 2 bits | MSHR state |
| way | 2 bits | target way |
| dirty_victim | 1 bit | victim needs write-back |
| byte_mask | 64 bits | per-byte write mask |

**CAM matching for secondary miss detection:**

The MSHR array includes a content-addressable memory (CAM) on the `line_addr` field.
On every new miss, the incoming line address is broadcast to all valid MSHR entries:

```verilog
Secondary miss detection (combinational):
  match[i] = MSHR[i].valid && (MSHR[i].line_addr == incoming_line_addr)

If (OR_reduce(match) == 1):
  // Secondary miss -- merge into matching MSHR
  word_idx = (incoming_address[5:2])  // word within line
  matching_MSHR.req_vector[word_idx] = 1
  matching_MSHR.dest_reg[word_idx]   = incoming_preg_id
  matching_MSHR.word_valid[word_idx] = 1
  // No new fill request issued!

Else:
  // Primary miss -- allocate free MSHR
  free_idx = priority_encode(~MSHR[].valid)  // find first IDLE entry
  MSHR[free_idx].valid     = 1
  MSHR[free_idx].line_addr = incoming_line_addr
  MSHR[free_idx].state     = WRITEBACK_PENDING or FILL_ISSUED
  // ... (allocate victim way, check dirty, issue fill)
```

**Total MSHR storage for a 16-entry L1D MSHR:**

$$
16 \times 176 = 2{,}816 \text{ bits} = 352 \text{ bytes}
$$

Plus the CAM comparator logic: 16 x 26-bit comparators = 416 bits of XOR + NOR trees.
This is modest compared to the 32 KB data SRAM (262,144 bits), representing only ~1% overhead.

### 2.4 Writeback-Before-Fill Sequence — Cycle-by-Cycle

When a cache miss evicts a dirty line, the controller must write back the victim before
installing the new line. The detailed sequence:

```verilog
Cycle 0: Miss detected. Select victim way (LRU/PLRU).
Cycle 1: Check victim dirty bit.
          If dirty:  copy victim data to writeback buffer (1-cycle SRAM read).
          If clean:  skip to fill issue (cycle 3).

Cycle 2: (dirty only) Writeback buffer holds victim data.
          Issue writeback request to next level (L2 or memory).
          MSHR state = WRITEBACK_PENDING.
          Meanwhile, the fill request is queued in the request buffer.

Cycle 3: Issue fill request for the new line.
          MSHR state = FILL_ISSUED.
          (Writeback may still be draining to next level in background.)

Cycle 4..N: Fill data returns from next level (N = memory latency).
          Write new line into the allocated way's SRAM.
          Update tag, set valid=1, clear dirty=0.
          Wake all dependent instructions (dest_reg[] entries with word_valid=1).
          Free MSHR entry.
```

**Writeback buffer optimization detail:**

Without a writeback buffer, the dirty victim must complete its write to the next level
before the fill can begin. This serializes:

```verilog
Without WB buffer:
  Miss -> writeback dirty victim (L2 write latency = 20 cycles)
        -> fill new line (L2 read latency = 20 cycles)
  Total miss penalty: 40 cycles

With WB buffer:
  Miss -> copy victim to WB buffer (1 cycle)
        -> issue fill immediately (20 cycles)
        -> WB buffer drains in background (overlapped with fill)
  Total miss penalty: 21 cycles (nearly 2x improvement!)
```

The writeback buffer is typically 4-8 entries deep, allowing multiple dirty victims
to queue up while fills proceed. This is critical for sustained throughput: without it,
a burst of misses to different sets that all evict dirty lines would serialize every
writeback before every fill.

### 2.5 MSHR Count

| Cache Level | Typical MSHR Entries | Rationale |
|-------------|---------------------|-----------|
| L1 I-cache  | 4--8                | Few simultaneous I-fetch streams |
| L1 D-cache  | 8--16               | Multiple outstanding loads from OoO execution |
| L2          | 32--64              | Must track misses from all L1 MSHRs |
| L3          | 64--128+            | Aggregation from multiple cores |

The total number of outstanding misses that can be tracked simultaneously equals the
MSHR count. Intel Skylake L1D has 10 load-buffer entries (functionally similar to
MSHRs) plus 6 fill-buffer entries; Apple M1 L1D has 14.

### 2.4 Hit-Under-Miss

The key property of a non-blocking cache is **hit-under-miss**: while an MSHR is
tracking an outstanding miss, the cache can still service hits to other lines. The
pipeline only stalls when:
1. A new miss occurs and no MSHR is free (structural hazard).
2. An instruction depends on the data from a pending miss (data hazard).

### 2.5 Split-Transaction Bus

In a **split-transaction** (or split-phase) bus, the request and response are
decoupled. The cache issues a fill request with a transaction ID and continues
processing other accesses. When the response arrives (potentially many cycles later),
the transaction ID is used to match it to the correct MSHR. This is essential for
bandwidth utilization -- a single miss does not lock the bus for its entire latency.

---

## 3. Write Policy

### 3.1 Write-Allocate vs. No-Write-Allocate

| Policy | On Write Miss | Used By |
|--------|---------------|---------|
| Write-allocate | Fetch the line into cache, then write the word | Most D-caches (L1, L2, L3) |
| No-write-allocate | Write directly to next level; line not fetched | Some I-caches, write-through L1s |

Write-allocate is dominant because most programs exhibit spatial locality on writes
(e.g., zeroing a buffer, copying a struct). Fetching the line amortizes the miss cost
over multiple subsequent writes to adjacent words.

### 3.2 Write-Through vs. Write-Back

**Write-through:** Every store writes to both the cache (if the line is present) and
the next level immediately.

- Advantage: coherence is simple -- the next level always has up-to-date data.
- Disadvantage: high write bandwidth consumption.

**Write-back:** A store modifies only the cache line. The dirty bit is set. The line
is written to the next level only upon eviction.

- Advantage: multiple writes to the same line are coalesced; write bandwidth is
  reduced by the ratio of writes-per-line to 1.
- Disadvantage: coherence complexity -- other caches may hold stale copies.

Modern high-performance designs overwhelmingly use write-back at every level. ARM
Neoverse N2, Apple M-series, AMD Zen 4, and Intel Golden Cove all use write-back L1D.
Write-through is found in simpler embedded cores and in some L1 designs where
coherence simplicity is valued over bandwidth.

### 3.3 Write Buffer

A **write buffer** (or writeback buffer) sits between the cache and the next level,
decoupling the CPU from the next-level write latency:

- On a write-through: the CPU writes to the cache and the write buffer simultaneously.
  The CPU continues as soon as the write buffer accepts the entry.
- On a write-back eviction: the dirty line is placed in the writeback buffer, and the
  new line is fetched immediately. The writeback buffer drains to the next level in
  the background.

**Write buffer merging:** If the buffer contains a pending write to line $L$ and a
new write to $L$ arrives, the two are merged into a single entry. This can reduce
traffic by 2--8x for sequential write patterns.

Typical write buffer depth: 4--16 entries (L1), 16--32 entries (L2).

---

**Write-back controller FSM (RTL view):**

```ascii-graph
States: IDLE → TAG_CHECK → {HIT_UPDATE | MISS_ALLOCATE} → WRITEBACK → REFILL → IDLE

IDLE:
  Wait for CPU request
  → TAG_CHECK on any read/write

TAG_CHECK:
  Read tags for the indexed set
  Compare with request tag
  → HIT_UPDATE if any way matches (hit)
  → MISS_ALLOCATE if no match (miss)

HIT_UPDATE:
  Read hit: return data, update LRU
  Write hit: update cache line, set dirty bit, update LRU
  → IDLE

MISS_ALLOCATE:
  Select victim way using LRU
  Check if victim is dirty:
  → WRITEBACK if dirty (must write victim to memory first)
  → REFILL if clean (can immediately fetch new line)

WRITEBACK:
  Send victim line to next level (L2 or main memory)
  Wait for acknowledgment
  → REFILL

REFILL:
  Send request to next level for the missing line
  Wait for data
  Install new line: update tag, set valid, clear dirty
  Supply requested word to CPU (can do "critical word first")
  → IDLE
```

---

## 4. Refill Optimization

### 4.1 Critical-Word-First (CWF)

When a cache line fill begins, the memory system returns the **requested word first**,
before the rest of the line. The CPU can resume execution as soon as the critical word
arrives, without waiting for the full 64 B transfer.

Example: CPU accesses address 0x1004 (word at offset +4 in a 64 B line starting at
0x1000). The memory controller returns bytes [0x1004..0x1007] first, then fills
[0x1000..0x1003] and [0x1008..0x103F].

Savings: up to $(B/4 - 1) \times T_{\text{bus-width}}$ cycles, where $B$ is the line
size. For a 64 B line on a 16-byte-wide bus, a full line transfer takes 4 beats; CWF
saves up to 3 cycles of stall time.

### 4.2 Early Restart

A simpler variant: the fill proceeds in order from the base of the line, but as soon
as the requested word arrives (wherever it falls in the beat order), the CPU is
un-stalled. This avoids the need to reorder the memory bus but provides less savings
if the requested word is near the end of the line.

### 4.3 Line Fill Buffer

A **line fill buffer** holds the incoming cache line as it arrives beat by beat. The
cache SRAM is written only after the entire line is received. This serves two purposes:

1. The SRAM write port is not tied up during the multi-cycle fill.
2. If the CPU re-accesses the filling line (secondary miss to same MSHR), the fill
   buffer can supply the data directly if the requested word has already arrived.

---

## 5. Prefetch Engines

### 5.1 Stream Prefetcher

Detects sequential (stride = +1 cache line) access patterns and prefetches the next
$D$ lines, where $D$ is the **prefetch degree** (typically 1--4).

- Detection: maintain a small table of recent access addresses per page or per
  stride-tracking entry. If consecutive accesses hit adjacent line addresses,
  a stream is detected.
- Action: prefetch lines $+1, +2, \ldots, +D$ ahead of the current access.
- Limitation: only handles unit-stride patterns. Pointer-chasing, strided, or
  irregular patterns are missed.

#### Stream Prefetcher Hardware Implementation

**Stream Detection Table (SDT)** — 16–32 entries. Per-entry format:

| Page Tag | Direction | Head Ptr | Count | Valid | Train State |
|---|---|---|---|---|---|
| bits | +1 or −1 | line # | — | — | — |

Detection FSM per entry:

- **IDLE** — entry free for allocation.
- **TRAIN** — one access recorded (Head Ptr = line); waiting for confirmation.
- **ACTIVE** — two or more consecutive sequential accesses confirmed; begin prefetching ahead.

On a cache access to line L at page P: (1) look up P in the SDT; (2) if hit and ACTIVE with matching direction, `Count++` and prefetch `Head + direction × D`; (3) if hit and TRAIN, promote to ACTIVE when `L == Head + direction`, otherwise update Head/direction and stay in TRAIN; (4) on miss, allocate an entry with `Head = L`, state TRAIN.

Prefetch when ACTIVE: `Prefetch_addr = current_access + direction × Degree`, issued only if the line is not already cached, not already in an MSHR, and the prefetch buffer has room.

### 5.2 Stride Prefetcher

Generalizes the stream prefetcher by tracking arbitrary constant strides. Each entry
records a base address and the last observed stride $\Delta$. On a new access:

1. Compute the new stride: $\Delta_{\text{new}} = A_{\text{new}} - A_{\text{prev}}$.
2. If $\Delta_{\text{new}} == \Delta_{\text{prev}}$ for two consecutive observations,
   the pattern is confirmed.
3. Prefetch address $= A_{\text{new}} + \Delta$.

Handles both forward and backward strides. Common in L1 D-cache prefetchers.
Intel calls this the "stride prefetcher"; ARM Neoverse uses it in the L1.

#### Stride Prefetcher Hardware (PC-Indexed Stride Table)

```wavedrom
{"reg":[
  {"bits":1,"name":"Valid"},
  {"bits":2,"name":"State"},
  {"bits":16,"name":"Stride"},
  {"bits":30,"name":"Last Addr"},
  {"bits":10,"name":"PC Tag"}
]}
```

RPT (Reference Prediction Table): 64–256 entries, indexed by `PC[11:2]`. `State`: `00` initial (first access), `01` training (one stride observed), `10` steady (two+ matching strides — actively prefetching), `11` no-prediction (irregular).

### 5.3 Delta Correlation Prefetcher

An advanced stride prefetcher that tracks sequences of deltas (stride differences)
rather than single strides. This captures complex repeating access patterns like
A, A+8, A+16, A+32, A+40, A+48 (deltas: +8, +8, +16, +8, +8).

```verilog
Delta History Buffer (DHB): circular buffer, 64-256 entries
  Each entry: {delta, PC, address}  // delta = current - previous address

Delta Prediction Table (DPT): maps delta patterns to predicted next deltas

Training:
  For each miss, record delta in DHB
  Extract last K deltas (K = 2 typical): [d(n-1), d(n)]
  Look up pattern in DPT
  If pattern exists: record that d(n+1) followed this pattern

Prediction:
  On miss, extract current K deltas
  Look up in DPT
  If hit: predict next delta from DPT
  Prefetch: current_address + predicted_delta

Advantage over simple stride: captures non-constant but repeating patterns
  e.g., array-of-structs traversal: +8, +8, +16, +8, +8, +16, ...
  Simple stride sees +8, +16 alternating and gets confused
  Delta correlation learns the [+8, +16] -> +8, [+16, +8] -> +8 pattern
```

### 5.3.1 PC-Indexed Stride Prefetcher — Worked Example

The PC-indexed stride prefetcher uses the load/store instruction address (PC) rather
than the data address to index its prediction table. This is critical because the same
data address accessed by different instructions may have different strides (e.g., one
instruction walks an array forward, another walks it backward).

```verilog
Worked example:

Load instruction at PC = 0x4008A0:
  Access 1: addr = 0x1000 -> RPT entry created: PC_tag=0x8A0, Last_Addr=0x1000, Stride=--, State=Initial
  Access 2: addr = 0x1080 -> delta = 0x80. RPT updated: Stride=0x80, State=Training
  Access 3: addr = 0x1100 -> delta = 0x80. Matches Stride! State -> Steady.
             Prefetch: 0x1100 + 0x80 = 0x1180
             Prefetch: 0x1100 + 2*0x80 = 0x1200  (degree = 2)
  Access 4: addr = 0x1180 -> HIT in cache (prefetched!). Stride confirmed.
             Prefetch: 0x1180 + 0x80 = 0x1200 (may already be prefetched)
             Prefetch: 0x1180 + 2*0x80 = 0x1280
  ...

Meanwhile, a different load at PC = 0x4008F0 walks a linked list:
  Access 1: addr = 0x5000 -> RPT entry: PC_tag=0x8F0, Last_Addr=0x5000, State=Initial
  Access 2: addr = 0x7820 -> delta = 0x2820 (not a useful stride). State -> Training, Stride=0x2820
  Access 3: addr = 0x3F10 -> delta != 0x2820. State stays Training, Stride updated.
  ...never reaches Steady... no prefetches issued (correctly, since linked list is unpredictable)

The PC-indexed approach correctly handles:
  - Multiple instructions accessing the same array with different strides
  - Pointer-chasing code (never trains, no wasted prefetches)
  - The same instruction accessing different arrays (one stride per PC, may need
    set-associative RPT to avoid aliasing)
```

### 5.3.2 Stride Prefetcher vs Stream Prefetcher — When to Use Each

```verilog
Stride prefetcher:
  Detects: constant stride (any value: +64, -128, +256, etc.)
  Best for: array traversals, struct field accesses, strided BLAS
  Limitation: only one stride per PC; confused by irregular patterns
  Typical location: L1 D-cache (needs PC, which is only available at L1)

Stream prefetcher:
  Detects: sequential (stride = +1 cache line) access patterns
  Best for: memcpy, sequential file I/O, video/image scanlines
  Limitation: cannot handle non-unit strides
  Typical location: L2 cache (can operate on cache line addresses without PC)

Combined approach (most modern designs):
  L1 has a stride prefetcher (PC-indexed, catches all constant strides)
  L2 has a stream prefetcher (catches sequential patterns missed by L1)
  Together they cover the majority of regular access patterns

  Neither catches: pointer chasing, random hash table lookups, graph traversals.
  These require correlation prefetchers (delta correlation, Markov) or software prefetch.
```

### 5.4 Markov Prefetcher

Records sequences of cache miss addresses (or page + offset pairs) in a history table.
When a miss occurs, the table is consulted to predict future misses based on past
sequences.

**Markov prefetch table** — maps a miss address to likely next miss addresses with probabilities. Structure (direct-mapped or set-associative, 256–1024 entries):

| Miss Address Tag (partial) | Next Addr[0] + confidence | Next Addr[1] + confidence |
|---|---|---|

Training: record the miss stream M0, M1, M2, …; for each pair (Mᵢ, Mᵢ₊₁), increment the confidence of Mᵢ₊₁ as a successor of Mᵢ. Prediction: on a miss at address A, look it up and prefetch the highest-confidence successor(s).

Captures pointer-chasing patterns (linked lists, tree traversal) where the effective "stride" is the pointer value rather than a constant. Downsides: large storage, cold-start (needs repetition to train), and it predicts only one step ahead (multi-step Markov is expensive).

### 5.5 Bandwidth-Aware Prefetch Throttling

Prefetches consume memory bandwidth. If prefetching is too aggressive, it can
interfere with demand (non-prefetch) requests, increasing demand latency.

```verilog
Prefetch Throttling Logic:

Track per-cycle:
  demand_pending    = number of outstanding demand requests
  prefetch_pending  = number of outstanding prefetch requests
  total_bandwidth   = demand_pending + prefetch_pending

Throttle condition (example):
  if prefetch_pending > THRESHOLD_HIGH:
    disable_prefetch_issue  // too many prefetches in flight
  elif prefetch_pending < THRESHOLD_LOW:
    enable_prefetch_issue   // safe to issue prefetches

Dynamic threshold adjustment:
  Monitor demand latency (moving average over last N requests)
  If demand latency increases > 20% above baseline:
    Reduce prefetch degree and/or increase throttle threshold
  If demand latency is near baseline:
    Increase prefetch degree (up to configured maximum)

Accuracy-based throttling:
  Track: useful_prefetches / total_prefetches (rolling window)
  If accuracy < 50%: reduce degree
  If accuracy > 90%: increase degree

Timing:
  Prefetch issue also checks MSHR availability
  Prefetch cannot use the last MSHR (reserved for demand)
  Prefetch is lower priority than demand in the MSHR allocator
```

### 5.4 Prefetch Metrics

$$
\text{Accuracy} = \frac{\text{Useful Prefetches}}{\text{Total Prefetches Issued}}
$$

$$
\text{Coverage} = \frac{\text{Useful Prefetches}}{\text{Total Cache Misses}}
$$

$$
\text{Prefetch Overhead} = \frac{\text{Useless Prefetches} \times \text{BW per Prefetch}}{\text{Total Available Bandwidth}}
$$

Design target for L1 prefetchers: accuracy > 90% (to avoid polluting the small
cache). L2 prefetchers can tolerate lower accuracy (70--80%) because the L2 is larger
and pollution is less catastrophic.

### 5.5 Prefetch Degree and Timeliness

The **prefetch degree** $D$ controls how many lines ahead to prefetch. Higher $D$
increases coverage for sequential patterns but risks:
- **Pollution:** evicting useful lines from the cache.
- **Bandwidth saturation:** consuming memory bandwidth needed by demand accesses.

**Timeliness** means the prefetched line should arrive before it is demanded but not
so early that it gets evicted again. Late prefetches waste bandwidth; early prefetches
waste cache capacity. The ideal prefetch distance (in cycles) is:

$$
\text{Prefetch Distance} = \frac{\text{Miss Latency}}{\text{Cycles Between Accesses}}
$$

### 5.6 L1 vs. L2 Prefetching

| Attribute | L1 Prefetcher | L2 Prefetcher |
|-----------|---------------|---------------|
| Accuracy target | > 90% | 70--85% |
| Types | Stride, next-line | Stream, stride, correlation |
| Degree | 1--2 | 2--8 |
| Pollution cost | High (small cache) | Moderate (large cache) |
| Coverage target | 20--40% of L1 misses | 50--80% of L2 misses |

---

## 6. Cache Hierarchy Design

### 6.1 Typical Parameters

| Parameter | L1 I\$ | L1 D\$ | L2 | L3 |
|-----------|--------|--------|-----|-----|
| Size | 32--64 KB | 32--64 KB | 256 KB--1 MB | 2--64 MB |
| Associativity | 4--8 way | 4--8 way | 8--16 way | 12--16 way |
| Line size | 64 B | 64 B | 64--128 B | 64--128 B |
| Hit latency | 3--4 cyc | 3--4 cyc | 8--14 cyc | 30--50 cyc |
| MSHR count | 4--8 | 8--16 | 16--64 | 64--128+ |
| Ports | 1--2R | 2R+1W or 2RW | 1R+1W | 1R+1W |
| Private/Shared | Private | Private | Private or shared | Shared |

### 6.2 Cache Sizing Math

The total number of sets in an $N$-way set-associative cache with line size $L$ and
total capacity $C$ is:

$$
\text{Sets} = \frac{C}{N \times L}
$$

The number of index bits is $\lceil\log_2(\text{Sets})\rceil$. The number of block
offset bits is $\lceil\log_2 L\rceil$. The tag size is:

$$
\text{Tag bits} = \text{Address bits} - \text{Index bits} - \text{Offset bits}
$$

Total SRAM bits (data + tag overhead) for one way:

$$
\text{Data SRAM per way} = \text{Sets} \times L \times 8 \text{ bits}
$$

$$
\text{Tag SRAM per way} = \text{Sets} \times (\text{Tag bits} + 1_{\text{valid}} + 1_{\text{dirty}})
$$

### 6.3 Inclusive vs. Exclusive Hierarchies

**Inclusive:** All data in L1/L2 is also present in L3.

- Advantage: snooping for coherence only needs to check L3. If L3 does not contain a
  line, no private cache does. This simplifies directory and snoop logic.
- Disadvantage: wastes capacity. L3 must be at least as large as the sum of all
  private caches (otherwise back-invalidation is needed).
- Used by: Intel (most generations), IBM Power.

**Exclusive:** L3 contains only lines evicted from L2 (no duplication).

- Advantage: effective capacity = L2 capacity + L3 capacity (no duplication waste).
- Disadvantage: on a coherence check, all private caches must be probed in parallel.
  Victim allocation: evicted L2 lines are placed into L3 rather than discarded.
- Used by: AMD Zen, some ARM designs.

**Non-inclusive / non-exclusive (NINE):** No strict relationship. Lines may be
allocated independently at any level. Flexible but complex coherence.

### 6.4 Victim Cache

A small (4--16 entry) fully-associative cache that captures lines recently evicted
from L1. On an L1 miss, the victim cache is probed in parallel with L2. If the line
is found, it is swapped back into L1 (and the evicted line goes into the victim cache).
Effective at reducing conflict misses in low-associativity L1 caches. Originally
proposed by Jouppi (1990).

---

## 7. Replacement Policies

### 7.1 LRU (Least Recently Used)

Exact LRU maintains a total ordering of all $N$ ways within a set. On an access, the
accessed way moves to the "most recently used" position. On eviction, the "least
recently used" way is chosen.

Storage cost: $\lceil\log_2(N!)\rceil$ bits per set. For $N=4$, this is
$\lceil\log_2(24)\rceil = 5$ bits. For $N=16$, it is $\lceil\log_2(16!)\rceil = 45$
bits. This grows quickly and makes exact LRU impractical for high associativity.

### 7.2 PLRU (Pseudo-LRU)

Tree-based approximation using $N-1$ bits per set. Organize the $N$ ways as leaves of
a binary tree. Each internal node stores 1 bit. The convention is:

- **On access:** set all bits on the path from root to the accessed leaf to point
  *away* from that leaf (marking it as most recently used).
- **On eviction:** follow the bits from root to leaf to find the
  least-recently-used candidate.

For $N=4$: 3 bits per set.

```verilog
        b0
       /  \
     b1    b2
    / \   / \
   W0  W1 W2  W3

Bit update on access:
  Access W0: b0=1 (point right), b1=1 (point right within left subtree)
  Access W1: b0=1 (point right), b1=0 (point left within left subtree)
  Access W2: b0=0 (point left),  b2=1 (point right within right subtree)
  Access W3: b0=0 (point left),  b2=0 (point left within right subtree)

Eviction: follow bits from root to leaf.
  b0=0: go left,  b0=1: go right
  b1=0: go left (W0),  b1=1: go right (W1)
  b2=0: go left (W2),  b2=1: go right (W3)
```

### 7.3 RRIP (Re-Reference Interval Prediction)

Each cache line has an $M$-bit **RRPV** (Re-Reference Interval Prediction) counter.
RRPV values:
- $2^M - 1$: distant re-reference (evict first)
- $2^M - 2$: long re-reference
- $0$: near-immediate re-reference

On access: set the line's RRPV to 0 (will be reused soon).
On eviction: choose the line with RRPV = $2^M - 1$. If no such line exists, increment
all RRPVs until one reaches $2^M - 1$.

With $M=2$ (2-bit RRIP), this is called **SRRIP** (Static RRIP). Lines are inserted
with RRPV = $2^M - 2$ (long re-reference), giving new lines one chance to prove
useful before being evicted. This avoids the "scan thrash" problem of LRU where a
sequential scan through a working set larger than the cache evicts all useful data.

### 7.4 SHiP (Shared-access-aware HiSTory-based Prefetching-aware Replacement)

Extends RRIP with a signature-based predictor. Shared lines (accessed by multiple
cores) and lines brought in by prefetch are given different insertion RRPV values
based on a small history table indexed by a signature (e.g., PC of the accessing
instruction). Lines whose signatures predict "low reuse" are inserted at RRPV =
$2^M - 1$ (likely to be evicted soon). Lines with "high reuse" signatures are
inserted at RRPV = 0.

### 7.5 Worked Example: 4-Way Set, Access Sequence A B C D A E

Initial state: all ways empty (Invalid).

**LRU trace:**

| Access | Way Assignment | MRU order (newest...oldest) | Eviction |
|--------|---------------|------------------------------|----------|
| A | Way 0 | A | -- |
| B | Way 1 | B, A | -- |
| C | Way 2 | C, B, A | -- |
| D | Way 3 | D, C, B, A | -- |
| A | Way 0 (hit) | A, D, C, B | -- |
| E | Way 1 (evicts B, LRU) | E, A, D, C | B evicted |

**PLRU trace (3 bits: b0, b1, b2):** Initial state b0=b1=b2=0.
Convention: on access, set bits to point away from the accessed way. On eviction,
follow bits (0=left, 1=right).

| Access | Eviction path / action | Bits after | Way assigned |
|--------|------------------------|------------|--------------|
| A | b0=0->L, b1=0->L = W0 (empty) | b0=1, b1=1, b2=0 | Way 0 |
| B | b0=1->R, b2=0->L = W2 (empty) | b0=0, b1=1, b2=1 | Way 2 |
| C | b0=0->L, b1=1->R = W1 (empty) | b0=1, b1=0, b2=1 | Way 1 |
| D | b0=1->R, b2=1->R = W3 (empty) | b0=0, b1=0, b2=0 | Way 3 |
| A | Hit Way 0, update bits | b0=1, b1=1, b2=0 | Way 0 |
| E | b0=1->R, b2=0->L = W2 (has B) | b0=0, b1=1, b2=1 | Evict **B** |

LRU evicts **B**; PLRU also evicts **B** for this sequence. They may differ for
other sequences -- PLRU is an approximation.

**RRIP trace (2-bit SRRIP, insert at RRPV=2):**

| Access | Way 0 | Way 1 | Way 2 | Way 3 | Action |
|--------|-------|-------|-------|-------|--------|
| A | 2 | -- | -- | -- | Insert A at RRPV=2 |
| B | 2 | 2 | -- | -- | Insert B |
| C | 2 | 2 | 2 | -- | Insert C |
| D | 2 | 2 | 2 | 2 | Insert D |
| A | 0 | 2 | 2 | 2 | Hit A, set RRPV=0 |
| E | 2 | 3 | 2 | 2 | Evict B (highest RRPV was tied, age-order breaks tie), insert E at 2 |

RRIP evicts **B** (same as LRU in this case, but the mechanism differs).

### 7.6 Replacement Policy Deep Comparison

This section compares all major replacement policies on the same access sequence,
highlighting where each diverges and why.

**Access sequence: A B C D E A B C D F** (4-way set, working set = 5 unique lines,
cache holds only 4).

#### LRU (Exact)

Storage: $\lceil\log_2(N!)\rceil$ bits per set. For 4-way: 5 bits. For 8-way: 16 bits.
For 16-way: 45 bits. Impractical beyond 8-way due to storage and update logic complexity.

```verilog
Matrix method: M[i][j] = 1 if way i was accessed more recently than way j.
On access to way k: set M[k][*] = 1, set M[*][k] = 0.
Victim: way with all zeros in its row.
```

| Access | MRU→LRU order | Eviction |
|--------|---------------|----------|
| A | A _ _ _ | -- |
| B | B A _ _ | -- |
| C | C B A _ | -- |
| D | D C B A | -- |
| E | E D C B | **A** evicted |
| A | A E D C | **B** evicted |
| B | B A E D | **C** evicted |
| C | C B A E | **D** evicted |
| D | D C B A | **E** evicted |

**LRU thrashes**: A is evicted and immediately re-requested. With working set (5) > associativity (4),
LRU cycles through all entries, evicting the one that will be needed next. **Hits after warmup: 0/5.**

#### PLRU (Tree-Based)

Storage: $N - 1$ bits per set. For 4-way: 3 bits. For 8-way: 7 bits. For 16-way: 15 bits.
Very practical at all associativities.

Same result as LRU for this sequence (PLRU approximates LRU well for 4-way). Divergence
occurs with pathological sequences where two ways in the same subtree are accessed
repeatedly while a way in the other subtree ages.

#### Random

No storage per set (uses a PRNG). Selects a random way on eviction.

Expected hit rate after warmup: $3/5 \times (3/4) \approx 45\%$ (probabilistic). Random
avoids thrashing because it sometimes evicts a non-immediately-needed line. For working
sets slightly larger than associativity, random outperforms LRU.

#### FIFO (First-In, First-Out)

Evicts the oldest-inserted line, regardless of recency of access. Uses a circular pointer.
Storage: $\lceil\log_2(N)\rceil$ bits per set.

| Access | Insert order (oldest→newest) | Eviction |
|--------|------------------------------|----------|
| A | A | -- |
| B | A B | -- |
| C | A B C | -- |
| D | A B C D | -- |
| E | B C D E | **A** evicted (oldest) |
| A | C D E A | **B** evicted |
| B | D E A B | **C** evicted |
| C | E A B C | **D** evicted |
| D | A B C D | **E** evicted |

Same thrashing as LRU for this sequence. FIFO ignores recency of access, so a frequently-
hit line can be evicted simply because it was inserted long ago. **Belady's anomaly:**
FIFO can have a *higher* miss rate when given *more* associativity (counterintuitive).

#### SRRIP (Static RRIP, 2-bit)

Each line has a 2-bit RRPV counter. Insert at RRPV=2 (long re-reference). On access,
set RRPV=0. On eviction, pick highest RRPV; if tie, age-order or random.

| Access | W0 | W1 | W2 | W3 | Eviction |
|--------|----|----|----|----|----------|
| A | 2 | -- | -- | -- | -- |
| B | 2 | 2 | -- | -- | -- |
| C | 2 | 2 | 2 | -- | -- |
| D | 2 | 2 | 2 | 2 | -- |
| E | 3 | 2 | 2 | 2 | Insert E→2 at W0. A aged to 3, evicted. |
| A | 0 | 2 | 2 | 3 | Hit: A→0. W3(E) ages to 3. |
| B | 0 | 0 | 2 | 3 | Hit: B→0. W3(E) ages to 3, evicted on next miss. |
| C | 0 | 0 | 0 | 2 | Hit: C→0. |
| D | 0 | 0 | 0 | 0 | Hit: D→0. All at 0. |
| F | 3 | 0 | 0 | 0 | Age all by 1 until one hits 3. W0→1, W1→1, W2→1, W3→1. Age again: W0→2, W1→2, W2→2, W3→2. Age again: W0→3 (A evicted). |

Wait -- that's the same result as LRU. Let me redo with correct SRRIP insertion logic:

| Access | W0 | W1 | W2 | W3 | Action |
|--------|----|----|----|----|--------|
| A | 2 | -- | -- | -- | Insert A |
| B | 2 | 2 | -- | -- | Insert B at W1 |
| C | 2 | 2 | 2 | -- | Insert C at W2 |
| D | 2 | 2 | 2 | 2 | Insert D at W3 |
| E | **2** | 2 | 2 | 2 | All at 2. Increment all: 3,3,3,3. Evict W0 (A). Insert E at W0, RRPV=2. |
| A | **2** | 2 | 2 | 2 | All at 2 again. Increment all: 3,3,3,3. Evict W0 (E, not A!). Insert A at W0. |

**Key SRRIP difference:** SRRIP evicts E (the scan line) preferentially because E was
inserted at RRPV=2 and was never re-accessed before the eviction decision. When A is
re-accessed (step "A" after E), A becomes the "new" line at RRPV=2, but it has already
proven itself useful. The anti-thrashing property emerges over longer sequences: scan
lines (accessed once) are evicted before working-set lines (accessed multiple times).

**SRRIP anti-thrashing:** New lines are inserted at RRPV=2, not 3. This gives them one
"chance" to be accessed before being evicted. In the scan-heavy sequence above, SRRIP
detects that the scan line (E) is not reused and evicts it preferentially, preserving
the working set {A, B, C, D} with higher probability than LRU.

**Key result:** For scan-resistant workloads (streaming + working set), SRRIP achieves
5-15% lower miss rate than LRU/PLRU. For purely random access, all policies converge.

#### BRRIP (Bimodal RRIP)

Inserts most new lines at RRPV=3 (distant, likely evicted) but a small fraction
(e.g., 1/32 probability determined by a policy bit) at RRPV=2. This provides even
stronger scan resistance: 97% of scan lines are inserted at the "evict me first" level
and quickly discarded, while the 3% that happen to be useful get a second chance.

**Use case:** Large L3 caches serving many cores with mixed streaming and random-access
workloads. BRRIP can reduce LLC miss rate by 10-20% over LRU for memory-intensive
server workloads.

#### Summary Table

| Policy | Bits per set (4-way) | Anti-thrash | Scan-resistant | Hardware complexity |
|--------|---------------------|-------------|----------------|---------------------|
| LRU | 5 | No | No | High (matrix or shift register) |
| PLRU | 3 | No | No | Low (tree of MUXes) |
| FIFO | 2 | No | No | Lowest (circular pointer) |
| Random | 0 | Yes (probabilistic) | Yes (probabilistic) | Minimal (PRNG) |
| SRRIP | 8 (2 bits x 4 ways) | Yes | Yes | Low (counters + comparator) |
| BRRIP | 8 + 1 (policy bit) | Yes | Strong | Low (same as SRRIP + PRNG) |

### 7.7 Replacement Policy Side-by-Side Comparison on Identical Sequence

All four policies applied to the same access sequence on a 4-way set.
**Sequence:** A B C D A B C D E F G H (working set grows from 4 to 8).

#### LRU (Exact) — 5 bits/set, matrix method

```ascii-graph
Access:  A  B  C  D  A  B  C  D  E  F  G  H
MRU→LRU after each access:
  A:     A  _  _  _                        (miss, fill W0)
  B:     B  A  _  _                        (miss, fill W1)
  C:     C  B  A  _                        (miss, fill W2)
  D:     D  C  B  A                        (miss, fill W3)
  A:     A  D  C  B                        (HIT, A moves to MRU)
  B:     B  A  D  C                        (HIT)
  C:     C  B  A  D                        (HIT)
  D:     D  C  B  A                        (HIT)
  E:     E  D  C  B   evict A (LRU)        (miss)
  F:     F  E  D  C   evict B              (miss)
  G:     G  F  E  D   evict C              (miss)
  H:     H  G  F  E   evict D              (miss)

Hits: 4 (A, B, C, D re-accessed). Misses: 8.
LRU thrashes once working set exceeds 4: E evicts A, F evicts B, etc.
```

#### PLRU (Tree-Based) — 3 bits/set

```verilog
PLRU evictions match LRU for this sequence (confirmed in Section 7.5).
The tree approximation produces identical results for sequential cyclic patterns
where the working set equals the associativity.

Hits: 4. Misses: 8. Same as LRU.
```

#### Random — 0 bits/set

```verilog
Expected hit rate after warmup is probabilistic.
For each re-access of {A,B,C,D} after all 4 are loaded:
  P(hit) = 3/4 (3 out of 4 ways have useful data, assuming no prior eviction)

For {E,F,G,H} (new lines): always miss (compulsory).

Expected hits from re-accessing {A,B,C,D}: 4 * (3/4) = 3.0
Expected misses: 9.0

Random does NOT thrash deterministically. It sometimes evicts the "right" line
(preserving a soon-to-be-accessed entry). For working sets slightly larger than
associativity, random often outperforms LRU because it avoids the systematic
eviction of the next-needed line.
```

#### SRRIP (2-bit) — 8 bits/set (2 bits x 4 ways)

```ascii-graph
Insert at RRPV=2. On access, set RRPV=0. On eviction, pick max RRPV; if tie,
increment all until one reaches 3.

Access: A   B   C   D   A   B   C   D   E   F   G   H
W0:     2   2   2   2   0   0   0   0   2   2   2   2
W1:     --  2   2   2   2   0   0   0   0   2   2   2
W2:     --  --  2   2   2   2   0   0   0   0   2   2
W3:     --  --  --  2   2   2   2   0   2   2   2   2

After D loaded: all at RRPV=2.
Re-access A: W0=0, others stay 2.
Re-access B: W1=0.
Re-access C: W2=0.
Re-access D: W3=0. All at 0 now.

E arrives: all at 0. Increment all: 1,1,1,1. Again: 2,2,2,2. Again: 3,3,3,3.
Pick W0 (tie-break: lowest index). Evict A. Insert E at RRPV=2.
  W0(E)=2, W1(B)=3(aged), W2(C)=3, W3(D)=3

F arrives: W1(B) at 3 → evict B. Insert F at 2.
  W0(E)=2, W1(F)=2, W2(C)=3, W3(D)=3

G arrives: W2(C) at 3 → evict C. Insert G at 2.
  W0(E)=2, W1(F)=2, W2(G)=2, W3(D)=3

H arrives: W3(D) at 3 → evict D. Insert H at 2.

Hits: 4 (same as LRU for this sequence). Misses: 8.

KEY DIFFERENCE: SRRIP evicts the aged "3" entries first. After the re-accesses
warm all entries to RRPV=0, the subsequent new inserts (E,F,G,H) all start at 2.
SRRIP ages the entries uniformly, so eviction order depends on tie-breaking.
For a SCAN-HEAVY workload (e.g., A B C D A B C D E E E E), SRRIP would recognize
that E is not reused (stays at RRPV=2, then ages to 3) and evict it preferentially,
preserving the working set {A,B,C,D}. LRU would thrash.
```

**Bottom line for interviews:**

| Sequence | LRU | PLRU | Random | SRRIP |
|----------|-----|------|--------|-------|
| A B C D A B C D E F G H | 4 hits / 8 misses | 4 / 8 | ~3 / 9 expected | 4 / 8 |
| A B C D E A B C D E (scan+reuse) | 0 after warmup (thrash) | 0 | ~1-2 expected | 2-4 (anti-thrash) |
| A A A A B C D E (hot + scan) | 0 after warmup | 0 | ~1-2 | 3-4 (preserves hot A) |

SRRIP/BRRIP excel at scan-resistant workloads; LRU/PLRU excel at LRU-friendly locality.

---

## 8. Cache Power Optimization

### 8.1 Way Prediction

Predict which way will hit before reading any data SRAM. A small **way predictor**
(typically a tag-hash table indexed by PC or address) produces a predicted way. Only
the predicted way's data SRAM is read in the first cycle.

- If prediction is correct: single-cycle access with $1/N$ of the data SRAM energy.
- If prediction is wrong: the remaining ways are read in the second cycle (two-cycle
  total). Misprediction rate is typically 5--15%.

Used in: ARM Cortex-A series L1D, Intel L1I (way-prediction for instruction fetch).

### 8.2 Sequential (Tag-First) Access

As described in Section 1.2, reading tags first and only reading the winning way's
data SRAM saves $(N-1)/N$ of data SRAM read energy. This is the most common power
optimization for L2 and L3 caches. The trade-off is one additional cycle of latency.

### 8.3 Drowsy Caches

Reduce the supply voltage of cache lines that have not been accessed recently. The
line's data is retained (not lost) but cannot be read at full speed. When accessed,
the line is "woken up" to full voltage (typically 1--2 cycles). Can reduce L2 cache
leakage power by 40--60% with < 1% performance loss.

### 8.4 Cache Way Gating

Disable entire ways when the working set is small. A control register or hardware
monitor disables ways by preventing their word lines from firing. Effective for L3
caches in low-load situations. Modern Intel processors dynamically enable/disable L3
ways based on demand (e.g., RAPL power management).

---

## 9. Cache Coherence

*Scope: the implementation view — how coherence lands in a real cache pipeline. Protocol-level MESI/MOESI state tables, snooping examples, directory scaling: [CPU_Architecture](03_CPU_Architecture.md) §8.*

### 9.1 The Coherence Problem

In a multicore system, each core has private L1 (and possibly L2) caches. When Core 0
writes to address X, Core 1's copy of X (if cached) becomes stale. A coherence
protocol ensures that all cores observe a consistent order of writes and never read
stale data.

**Coherence invariant (single-writer, multi-reader):** For any memory address, at any
time, either (a) exactly one cache has the line in Modified state (can write), or
(b) zero or more caches have the line in Shared state (can read), but never both (a)
and (b) simultaneously.

### 9.2 MESI Protocol

Four states per cache line:

| State | Meaning | Can Read? | Can Write? | Dirty? | In Other Caches? |
|-------|---------|-----------|------------|--------|-------------------|
| **M**odified | Line is dirty; this cache has the only valid copy | Yes | Yes | Yes | No |
| **E**xclusive | Line is clean; this cache has the only copy | Yes | No | No | No |
| **S**hared | Line is clean; other caches may also have it | Yes | No | No | Possibly |
| **I**nvalid | Line is not valid | No | No | -- | -- |

Bus transactions:

| Transaction | Meaning |
|-------------|---------|
| BusRd | Request a shared copy of a line |
| BusRdX | Request an exclusive (writable) copy |
| BusUpgr | Announce upgrade from S to M (already have the line) |
| Flush | Write back modified data to memory |

### 9.3 MESI Complete State Transition Table

The table below shows every transition for every event. Each cell shows the
**new state / action taken**. "--" means the event cannot occur from that state
(no valid transition). BusUpd is included for completeness (used by MOESI
variants and some directory protocols).

| Current State | PrRd (local read) | PrWr (local write) | BusRd (other core reads) | BusRdX (other core writes) | BusUpgr (other upgrades S→M) | BusUpd (partial write broadcast) |
|---------------|-------------------|--------------------|--------------------------|-----------------------------|-------------------------------|-----------------------------------|
| **I** | **S/E**: issue BusRd. If Shared line asserted → S, else → E. Memory supplies data. | **M**: issue BusRdX. Memory supplies data. No other cache has it. | -- (no action) | -- (no action) | -- | -- |
| **S** | **S**: cache hit, no bus transaction. | **M**: issue BusUpgr. All other S copies → I. | **S**: stay S. Assert Shared signal so requester knows multiple copies exist. | **I**: invalidate silently. | **I**: invalidate silently. | **S**: apply update if matching address (rare in MESI; more relevant in MOESI/directory). |
| **E** | **E**: cache hit, no bus transaction. | **M**: silent upgrade. No bus transaction. This is the key E-state benefit. | **S**: Flush data to requester and memory. Assert Shared. Both caches now S. | **I**: Flush data to requester. Invalidate. | -- (E cannot see BusUpgr; already only copy) | -- |
| **M** | **M**: cache hit, no bus transaction. | **M**: cache hit, no bus transaction. | **S**: Flush dirty data to requester (and memory). Requester gets S. Both are S. | **I**: Flush dirty data to requester. Invalidate. Requester gets M. | -- (M implies exclusive, no S copies to upgrade) | -- |

**Event definitions:**

| Event | Who initiates | Meaning |
|-------|---------------|---------|
| PrRd | Local processor | Read request from this core |
| PrWr | Local processor | Write request from this core |
| BusRd | Other core (snooped) | Another core requests a shared copy |
| BusRdX | Other core (snooped) | Another core requests exclusive ownership for writing |
| BusUpgr | Other core (snooped) | Another core upgrades from S to M (already has a copy) |
| BusUpd | Other core (snooped) | Another core broadcasts a partial write (used in update-based protocols, rare in MESI) |

**Key transitions explained in detail:**

- **I → S or E on PrRd:** The cache issues a BusRd. If any other cache currently holds the
  line (in S, E, or M), it asserts the Shared signal on the bus. If Shared is asserted,
  the line arrives in S (multiple copies may exist). If no cache responds (Shared not
  asserted), the line arrives in E -- this cache has the only copy, and can silently
  upgrade to M on a subsequent write without any bus transaction. This is the critical
  optimization that the E state provides over a pure MSI protocol.

- **I → M on PrWr:** The cache issues a BusRdX (read-for-ownership). All other caches
  must invalidate any copies. Memory supplies the data. The line arrives in M (dirty)
  because a write will immediately follow.

- **E → M on PrWr (silent upgrade):** No bus transaction needed. The core already has
  the only copy (guaranteed by the E state). This eliminates a BusUpgr transaction that
  would be required in a pure MSI protocol. For workloads with predominantly private
  data, this optimization eliminates 20-50% of coherence bus traffic.

- **M → S on BusRd:** The modified (dirty) cache has the only up-to-date copy. It must
  intervene: supply the data to the requesting cache (and optionally write it back to
  memory). Both the original holder and the requester end in S. The memory is updated
  as a side effect.

- **M → I on BusRdX:** Same as M→S, but the requesting cache wants exclusive ownership.
  The dirty cache flushes data to the requester and invalidates. The requester enters M.

- **S → I on BusUpgr:** Another core is upgrading its copy from S to M. All other S copies
  must be invalidated. No data transfer needed (the upgrader already has a clean copy).

**Bus transaction count analysis:**

| Operation | MESI Bus Transactions | Notes |
|-----------|----------------------|-------|
| Read miss, no other copy | 1 (BusRd) | Line arrives in E |
| Read miss, other copies exist | 1 (BusRd) + 1 (Flush) | Line arrives in S |
| Write miss | 1 (BusRdX) | All others invalidated |
| Write hit in S | 1 (BusUpgr) | Others invalidated |
| Write hit in E | 0 (silent) | Key E-state benefit |
| Write hit in M | 0 (silent) | Already exclusive |
| Read hit in any state | 0 | No bus traffic |

### 9.3.1 MESI State Transition Summary — Compact Reference

The full MESI state transition table above (Section 9.3) is the definitive reference.
Below is a compact matrix for quick interview recall. Each cell is "resulting state / bus action":

| From \ Event | PrRd | PrWr | BusRd | BusRdX |
|---|---|---|---|---|
| **M** | M / -- | M / -- | S / Flush | I / Flush |
| **E** | E / -- | M / -- (silent) | S / Flush | I / Flush |
| **S** | S / -- | M / BusUpgr | S / Assert Shared | I / -- |
| **I** | S or E / BusRd | M / BusRdX | -- / -- | -- / -- |

Quick-interview shorthand:
- PrRd always keeps or improves the current state (M stays M, E stays E, S stays S, I upgrades to S or E).
- PrWr always drives toward M (M stays M, E silently upgrades, S requires BusUpgr, I requires BusRdX).
- BusRd (another core reads) forces sharing: M/E flush data and downgrade toward S; S stays S and asserts Shared.
- BusRdX (another core writes) forces invalidation: any holder must flush (if M/E) and go to I.

### 9.4 MOESI Protocol

Extends MESI with an **O** (Owned) state:

| State | Meaning |
|-------|---------|
| **O**wned | Dirty but shared. This cache is responsible for supplying the line to other requesters (avoids memory read). |

When a modified line is read by another core via BusRd, instead of writing back to
memory and transitioning both to S, the original holder transitions to O and the
requester enters S. The owner supplies data on subsequent BusRd events without
touching memory. Only when the owner evicts the line does it write back to memory.

**MOESI Complete State Transition Table:**

| Current State | PrRd | PrWr | BusRd | BusRdX | BusUpgr | BusUpd |
|---------------|------|------|-------|--------|---------|--------|
| **I** | BusRd → **S/E** | BusRdX → **M** | -- | -- | -- | -- |
| **S** | Hit → **S** | BusUpgr → **M** | Stay **S** (assert Shared) | → **I** | → **I** | Stay **S** (apply update) |
| **O** | Hit → **O** | BusUpgr → **M** | Stay **O** (supply data, no memory write) | → **I** (flush) | -- (O implies no other M) | Stay **O** (apply update) |
| **E** | Hit → **E** | Silent → **M** | Flush → **O** (become owner, supply to requester) | → **I** (flush) | -- | -- |
| **M** | Hit → **M** | Hit → **M** | Flush → **O** (supply data, no memory writeback yet) | Flush → **I** | -- | -- |

**Key MOESI difference from MESI:** The O state avoids memory write-back on sharing.
When a dirty line (M) is read by another core via BusRd, in MESI both caches end in S
and memory is updated. In MOESI, the original holder transitions to O (not S) and the
requester enters S. The O-state cache is responsible for supplying data on future BusRd
events without going to memory. Only when the O-state cache evicts the line does it
write back to memory. This saves memory bandwidth when multiple cores read data that was
recently written by one core (common for producer-consumer patterns).

Advantage: reduces memory bandwidth for shared-read data that was recently written.
Used by: AMD (all modern designs), ARM (optionally).

### 9.5 Directory-Based Coherence

For systems with many cores (16+), snooping every cache on every bus transaction
becomes impractical. A **directory** maintains coherence metadata per cache line:

| Directory Entry Field | Meaning |
|-----------------------|---------|
| State | Uncached, Shared, Modified |
| Sharer vector | Bit vector: bit $i$ = 1 if core $i$ has a copy |
| Owner | Which core has the line in Modified state |

Protocol (simplified):

1. **Core $i$ reads line $L$ (miss):** Directory checks state.
   - Uncached: supply from memory, set state=Shared, set bit $i$.
   - Shared: supply from memory (or owner), set bit $i$.
   - Modified: send fetch request to owner; owner downgrades to Shared and sends
     data to core $i$; directory sets state=Shared, sets bit $i$.

2. **Core $i$ writes line $L$:** Directory checks state.
   - Uncached: supply from memory, set state=Modified, owner=$i$.
   - Shared: send invalidate to all sharers (bits set in sharer vector); wait for
     acknowledgments; set state=Modified, owner=$i$, clear all other bits.
   - Modified by other core $j$: send fetch-invalidate to $j$; $j$ sends data to $i$
     and transitions to I; directory sets owner=$i$.

**Coherence traffic calculation:** If a line is shared by $K$ cores and one core writes
to it, the directory must send $K-1$ invalidate messages and receive $K-1$
acknowledgments. Total messages: $2(K-1)$. For a 64-core system with a hot shared
variable, a single write can generate 126 messages.

### 9.6 Coherence Scope Example

Consider a 4-core system with MESI. Line X is initially in memory only (all caches I).

```ascii-graph
Step 1: Core 0 reads X (miss)
  BusRd issued. No other cache responds.
  Core 0: I --> E
  Bus transactions: 1 BusRd, memory supplies data

Step 2: Core 1 reads X (miss)
  BusRd issued. Core 0 sees BusRd, asserts Shared, flushes.
  Core 0: E --> S (flush)
  Core 1: I --> S
  Bus transactions: 1 BusRd, 1 Flush from Core 0

Step 3: Core 0 writes X
  Core 0 in S, needs exclusive. Issues BusUpgr.
  Core 1 sees BusUpgr: S --> I
  Core 0: S --> M
  Bus transactions: 1 BusUpgr, Core 1 invalidates silently

Step 4: Core 1 reads X (miss)
  BusRd issued. Core 0 sees BusRd, has M, must flush.
  Core 0: M --> S (flush to memory + supply to Core 1)
  Core 1: I --> S
  Bus transactions: 1 BusRd, 1 Flush from Core 0
```

---

## 10. Numbers to Memorize

| Parameter | L1 I\$ | L1 D\$ | L2 | L3 | DRAM |
|-----------|--------|--------|-----|-----|------|
| Size | 32--64 KB | 32--64 KB | 256 KB--1 MB | 2--64 MB | 8--128 GB |
| Associativity | 4--8 | 4--8 | 8--16 | 12--16 | -- |
| Line size | 64 B | 64 B | 64--128 B | 64--128 B | 64 B row window |
| Hit latency | 3--4 cyc | 3--4 cyc | 8--14 cyc | 30--50 cyc | 100--300 cyc |
| Miss rate (SPEC) | 1--3% | 5--10% | 0.5--2% | < 0.5% | -- |
| MSHR count | 4--8 | 8--16 | 16--64 | 64--128 | -- |
| Bandwidth | 64--256 GB/s | 64--256 GB/s | 128--512 GB/s | 64--256 GB/s | 25--50 GB/s |
| Write policy | -- | Write-back | Write-back | Write-back | -- |
| Allocation | -- | Write-alloc | Write-alloc | Write-alloc | -- |

Additional key numbers:

| Quantity | Value |
|----------|-------|
| Typical L1D AMAT | 1.5--3.0 cycles |
| Typical full-hierarchy AMAT | 5--15 cycles |
| MESI state bits per line | 2 (M, E, S, I encoded as 2 bits) |
| MOESI state bits per line | 3 |
| LRU bits for 4-way set | 5 |
| PLRU bits for 4-way set | 3 |
| PLRU bits for 8-way set | 7 |
| PLRU bits for N-way set | $N - 1$ |
| Write buffer entries (L1) | 4--16 |
| Prefetch accuracy target (L1) | > 90% |
| Prefetch accuracy target (L2) | 70--85% |
| DDR4-3200 peak bandwidth | 25.6 GB/s (single channel) |
| DDR5-5600 peak bandwidth | 44.8 GB/s (single channel) |

---

## 10A. Cache-Adjacent Compute: AMX (Apple Matrix eXtensions)

### 11.1 Architecture

Apple's M1/M2/M3/M4 chips include a dedicated matrix multiplication unit called
AMX (Apple Matrix eXtensions) that sits conceptually "next to" the load/store
unit and operates on data from the L2 cache. AMX is not a traditional ISA-
visible coprocessor; instead, it is accessed via special ARM instructions that
the Apple CPU decodes into micro-ops targeting the matrix unit.

**Key parameters (M2):**

| Parameter | Value |
|-----------|-------|
| Matrix size (X/Y registers) | 16x16 elements (FP16) or 8x8 (FP32) |
| Register file | 16 X-registers + 16 Y-registers + 16 Z-registers |
| Compute throughput | Up to 128 FP16 FLOPs/cycle per AMX unit |
| Latency (matmul tile) | 8--12 cycles |
| Data path | Connected to L2 cache fill bandwidth |

### 11.2 Cache Interaction

AMX operates on data that flows from the L2 cache (or from registers loaded by
the scalar core). The critical performance path is:

1. Software loads a tile (16x16 FP16 = 512 bytes) from memory into AMX
   registers via AMX load instructions.
2. AMX performs a matrix multiply-accumulate: $Z \mathrel{+}= X\,Y^{T}$.
3. Software stores the result tile back to memory.

Because each tile is 512 bytes (8 cache lines), the AMX unit's sustained
throughput is limited by L2 cache bandwidth, not by the compute unit. This is
a general principle: **for cache-adjacent compute, the cache hierarchy's
bandwidth and the prefetcher's ability to keep data flowing are the bottleneck.**

### 11.3 Comparison with Other Matrix Units

| Unit | ISA | Tile Size | Peak FP16 | Cache Connection |
|------|-----|-----------|-----------|------------------|
| Apple AMX | ARM (Apple-specific) | 16x16 | 128 FLOP/cyc | L2 cache |
| Intel AMX | x86 (TILECFG/TILELOAD) | 16x16 (8-bit) / 4x8 (FP32) | 4096 INT8 OP/cyc | L2 cache |
| ARM SME | AArch64 (Streaming SVE) | VL-dependent | 2x SVE bandwidth | L2 cache |
| NVIDIA Tensor Core | SASS (CUDA) | 16x8x16 (FP16) | 312 TFLOPS (H100) | Shared L2 |

Intel's AMX (Advanced Matrix Extensions), introduced in Sapphire Rapids (2023),
is architecturally similar but ISA-visible. It uses `TILECFG` to configure tile
dimensions, `TILELOAD`/`TILESTORE` to move data, and `TDPBSSD`/`TDPBF16PS` for
matrix operations. The Intel AMX register file holds 8 tiles of up to 16x16
x 4 bytes = 1 KB each.

---

## 11. Cache Partitioning for AI Workloads

### 12.1 The Problem

In a multi-core system running mixed workloads (e.g., an LLM inference server
alongside a web server), an AI workload with a large working set can evict
useful data from the shared L3 cache, degrading the co-running workload's
performance by 20--50%. This is called **cache interference** or **noisy
neighbor** problem.

### 12.2 Intel CAT (Cache Allocation Technology)

Intel CAT (introduced in Xeon Skylake) allows the OS or hypervisor to partition
the L3 cache into **classes of service (CLOS)**. Each CLOS is assigned a
bitmask that determines which cache ways it can allocate into.

**Mechanism:**

- L3 cache has N ways (e.g., 20 ways in a 2.5 MB slice).
- A CLOS is defined by a **capacity bitmask (CBM)** of N bits. A CLOS with
  CBM = `0x0000F` can only allocate into ways 0--3 (4 out of 20 ways = 20% of
  L3 capacity).
- Each core (or thread) is assigned a CLOS ID. The core's fill requests can
  only allocate into the ways permitted by its CLOS.
- **Allocation enforcement:** On a cache fill, the replacement logic only
  considers ways within the CLOS bitmask. This guarantees the CLOS will not
  exceed its allocation.

**Typical configuration:**

| CLOS | CBM | Capacity | Workload |
|------|-----|----------|----------|
| 0 | 0xFFFFF | 100% | System / hypervisor |
| 1 | 0x0FFF0 | 50% | AI inference (LLM) |
| 2 | 0x0000F | 20% | Web server |
| 3 | 0x00003 | 10% | Latency-sensitive service |

**Limitation:** CAT partitions by ways, not by sets. A CLOS with 4 ways out
of 20 gets exactly 4/20 = 20% of the cache, regardless of its actual working
set. This can waste capacity if the CLOS's working set is smaller than its
allocation.

### 12.3 ARM MPAM (Memory System Partitioning and Monitoring)

ARM MPAM (introduced in ARMv8.4, widely deployed in Neoverse V2/N2) provides
a more fine-grained partitioning mechanism:

- **Partition IDs (PARTID):** Each thread or VM is assigned a PARTID (up to
  256 partitions).
- **Cache maximum portion:** A partition can be limited to a percentage of the
  cache (in increments of ~1--4 KB, not whole ways). This is implemented by
  partition-aware replacement logic that tracks occupancy per PARTID.
- **Memory bandwidth partitioning:** MPAM also partitions DRAM bandwidth.
  Each PARTID can be assigned a minimum guaranteed bandwidth and a maximum
  limit, enforced by the memory controller's scheduler.
- **Monitoring (MPAM monitor):** Hardware counters track cache occupancy and
  memory bandwidth usage per PARTID, allowing the OS to dynamically adjust
  partitions based on actual behavior.

**Comparison:**

| Feature | Intel CAT | ARM MPAM |
|---------|-----------|----------|
| Partition granularity | Cache ways (coarse) | Bytes / percentage (fine) |
| Bandwidth partitioning | No (separate RDT feature: MBA) | Yes (unified with cache) |
| Monitoring | CQM (Cache QoS Monitoring) | Built-in MPAM monitors |
| Max partitions | ~8--16 CLOS | 256 PARTID |
| Applicable caches | L3 only | Any cache level (configurable) |

### 12.4 Interview Insight

Cache partitioning is increasingly important for cloud and edge AI. An
interview candidate should be able to explain:
1. Why shared caches suffer from interference (conflict misses from co-runners).
2. How way-partitioning (CAT) works and its granularity limitation.
3. How fine-grained partitioning (MPAM) tracks occupancy per partition.
4. The tradeoff between partition isolation and overall cache utilization.

---

## 12. Non-Inclusive Cache Hierarchies

### 13.1 Background

Section 6.3 covered inclusive and exclusive hierarchies. Modern high-performance
CPUs increasingly use **non-inclusive** (also called non-inclusive/non-exclusive,
or NINE) hierarchies, which offer the best capacity utilization but add
coherence complexity.

### 13.2 Why Non-Inclusive?

**The inclusion problem:** If the L3 is inclusive of L1/L2, the L3 must be at
least as large as the sum of all private caches. For a 16-core chip with 48 KB
L1 + 1 MB L2 per core, that is $16 \times (48 + 1024) = 17$ MB of L3 "wasted"
on duplicates. In practice, the L3 must be even larger because some lines exist
only in L3 (not in any private cache). For a 32-core design, the inclusion
requirement becomes prohibitive.

**Intel's shift:** Intel used inclusive L3 caches from Nehalem (2008) through
Skylake (2017). Starting with Ice Lake Server (2020), Intel switched to a
non-inclusive L3. The reason: server chips grew from 8 to 28+ cores, and the
inclusion overhead became too costly.

### 13.3 Non-Inclusive Coherence

In a non-inclusive hierarchy, the L3 does not know which private caches hold a
given line. This requires a **snoop filter** or **directory at the L3 level**
to track which cores might have a copy.

**Snoop filter:** A tag-only structure in the L3 controller that records which
cores have cached each line. On a coherence request (BusRdX, BusUpgr):
1. Look up the snoop filter.
2. If the line is present in one or more private caches, direct the
   invalidation to those specific cores (directed probe).
3. If the line is not in any private cache, the L3 can supply the data
   (if it has it) or forward the request to memory.

The snoop filter consumes tag storage but not data storage, so it is much
smaller than a fully inclusive L3.

**Non-inclusive back-invalidation:** Unlike an exclusive hierarchy, the L3 in
a NINE design can hold a copy even when a private cache also holds it. On an L3
eviction of a line that is also in a private cache, the L3 must send a
**back-invalidation** to that private cache. This is an additional coherence
message not needed in inclusive or exclusive designs.

### 13.4 Modern Examples

| Processor | L3 Policy | Snoop Mechanism | Notes |
|-----------|-----------|-----------------|-------|
| Intel Ice Lake / Sapphire Rapids | Non-inclusive | Snoop filter + directory | Per-tile L3 with mesh interconnect |
| AMD Zen 4 | Exclusive (L3 as victim cache) | Directory-based | L3 receives evicted L2 lines |
| Apple M2 Max | Non-inclusive | Snoop filter | System-level cache (SLC) with per-core tracking |
| ARM Neoverse V2 | Non-inclusive | Directed probe via CMN mesh | Coherent mesh network with home-node directory |
| IBM POWER10 | Non-inclusive | Distributed directory | Each core complex has a local directory slice |

### 13.5 Interview Insight

When asked about cache hierarchy design, explain the three-way tradeoff:

1. **Inclusive:** Simple coherence (probe only L3), but wastes L3 capacity.
   Good for small-core-count designs.
2. **Exclusive:** Maximizes effective capacity (L2 + L3), but requires probe
   broadcast on coherence checks. Good for mid-range designs.
3. **Non-inclusive:** Best capacity efficiency, but requires snoop filter or
   directory. The standard for high-core-count server chips.

---

## References

1. Hennessy, J. L. and Patterson, D. A., *Computer Architecture: A Quantitative
   Approach*, 6th ed., Morgan Kaufmann, 2019. Chapters 2 and 5.
2. Jouppi, N. P., "Improving Direct-Mapped Cache Performance by the Addition of a
   Small Fully-Associative Cache and Prefetch Buffers," *ISCA*, 1990.
3. Kharbutli, M. and Solihin, Y., "Counter-Based Cache Replacement and Bypassing
   Algorithms," *IEEE Transactions on Computers*, 2008. (RRIP/SHiP origins.)
4. Jaleel, A. et al., "High Performance Cache Replacement Using Re-Reference Interval
   Prediction (RRIP)," *ISCA*, 2010.
5. Culler, D. E. and Singh, J. P., *Parallel Computer Architecture: A
   Hardware/Software Approach*, Morgan Kaufmann, 1999. (Directory coherence.)
6. Sorin, D. J., Hill, M. D., and Wood, D. A., *A Primer on Memory Consistency and
   Cache Coherence*, 2nd ed., Morgan & Claypool, 2020.
7. ARM Ltd., "Arm Cortex-A78 Core Technical Reference Manual," ARM DDI 0414.
8. Intel Corp., "Intel 64 and IA-32 Architectures Optimization Reference Manual,"
   Order 248966.
9. Intel Corp., "Intel Cache Allocation Technology," Intel Developer Manual,
   Chapter 17 (RDT/CAT).
10. ARM Ltd., "ARM Memory System Partitioning and Monitoring (MPAM)," ARM DDI 0598.

---

[Prev: CPU Architecture](03_CPU_Architecture.md) | [Up: Architecture](03_CPU_Architecture.md) | [Next: Memory Design](09_Memory.md)
