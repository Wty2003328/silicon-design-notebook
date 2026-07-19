# CPU Memory, Translation, and Coherence Implementation Blueprint

> **Abbreviation key:** central processing unit (CPU); load-store unit (LSU); translation lookaside buffer (TLB), including data TLB (D-TLB); miss-status handling register (MSHR); level-one/level-two cache (L1/L2); last-level cache (LLC); least-recently-used replacement (LRU); error-correcting code (ECC); input/output (I/O); instruction set architecture (ISA).

## 0. Purpose and design ideology

The CPU memory subsystem must make a slow, shared, protected memory appear fast and locally accessible without changing which values software may observe. Its design ideology is **separate speculation from permission and separate lookup from ownership**. A load may predict a dependency, translate an address, and start a cache lookup early, but it cannot retire if translation, protection, coherence, or the memory model would reject the access. A cache hit says that bits were found; coherence says whether this agent has permission to use or modify them.

This blueprint connects the load-store unit (LSU), translation lookaside buffer (TLB), caches, miss machinery, page-table walker, coherence controller, and fabric. The mechanism chapters—[Load-Store Unit](../03_Out_of_Order_Backend/02_Load_Store_Unit_and_Memory_Ordering.md), [Cache Microarchitecture](../04_Cache_Hierarchy/01_Cache_Microarchitecture.md), [TLB and Virtual Memory](../05_Virtual_Memory/01_TLB_and_Virtual_Memory.md), and [Cache Coherence](../06_Coherence_and_Consistency/01_Cache_Coherence.md)—provide theory. Here every boundary becomes an implementation obligation.

## 1. Write the memory contract

Freeze these choices before designing arrays:

- virtual and physical address widths, page sizes, address-space and virtualization identifiers;
- supported access sizes, alignment behavior, endianness, atomics, and memory types;
- consistency model, fence meanings, and permitted speculative reordering;
- cache-line size and inclusion/exclusion policy;
- coherence scope and whether instruction, data, device, and direct-memory-access agents are coherent;
- translation/protection scheme, page-table format, physical-memory attributes, and I/O memory-management boundary;
- error behavior for parity, error-correcting code (ECC), poison, timeout, access fault, and machine check;
- fabric payload, ordering, transaction identifiers, retry, credits, and quality of service.

State which layer owns each failure. A TLB hit with denied permission is a translation/protection fault, not a cache miss. An uncacheable access bypasses ordinary allocation but still obeys ordering. An ECC correction may return data and report telemetry; an uncorrectable error usually returns poison or a precise machine exception according to the product contract.

## 2. End-to-end load and store path

~~~mermaid
flowchart LR
    AGU["address generation"] --> LSQ["load/store queue"]
    LSQ --> TLB["D-TLB + permissions"]
    TLB -->|hit| L1["L1 tag/data + coherence state"]
    TLB -->|miss| PTW["page-table walker"]
    PTW --> TLB
    L1 -->|hit| RET["value / store completion"]
    L1 -->|miss| MSHR["miss-status entry"]
    MSHR --> FAB["coherent fabric"]
    FAB --> LLC["shared cache / memory"]
    LLC --> FAB --> MSHR
    MSHR --> L1
~~~

For a load, address generation creates a virtual address plus size and byte mask. The load queue checks older stores: exact-address matches may forward bytes; unresolved older stores may block or be predicted independent. Translation supplies physical page number, access permissions, and memory attributes. The L1 cache checks tag, valid/coherence state, and ECC. A hit returns selected bytes and marks the load complete. A miss allocates or merges into a miss-status handling register (MSHR), sends the necessary coherent request, fills the line, wakes merged dependents, and replays the load.

A store normally records address and data separately because they may become ready at different times. It may acquire line ownership speculatively, but it must not make a value globally visible before retirement permits it. A committed store buffer drains in program order or under explicitly allowed relaxation. Store-to-load forwarding must merge byte lanes from the youngest matching older stores and cache data; partial overlaps are not an all-or-nothing case.

## 3. State schemas and allocation rules

### 3.1 Load queue entry

Store program-order identity, virtual and physical address validity, size, byte mask, destination tag, translation and protection outcome, issued/completed/replayed state, observed store-set or dependency-predictor metadata, forwarded-byte mask, cache request identity, exception, and a record of which older unresolved stores were bypassed. Reclaim at squash or retirement, not merely when data returns, because later detection of a memory-order violation may need the entry.

### 3.2 Store queue and committed store buffer

Store program-order identity, address validity, data validity, byte enables, physical address, memory type, atomic/fence semantics, exception, coherence permission state, and committed/drained bits. Separate speculative store queue from globally visible committed buffering, or make the permission boundary explicit within one structure. A younger load comparing against older stores must understand missing address, missing data, full match, partial match, and no match.

### 3.3 Cache line and MSHR

A cache line needs tag, validity, coherence stable state, dirty state if not implied, replacement metadata, ECC/parity, prefetch/usefulness metadata if used, and data sectors. An MSHR needs block address, requested permission, outstanding transaction identifier, merged requester list, returned-sector mask, victim/writeback dependency, transient coherence state, error/poison, and replay status. Never merge requests that require incompatible ordering or permissions without a defined upgrade protocol.

### 3.4 TLB and walker entry

A TLB entry stores virtual page number, physical page number, page size, address-space identifier, virtualization identifier where relevant, read/write/execute/user permissions, accessed/dirty handling, memory attributes, global bit, and replacement state. A page-walk entry stores original request identity, current level, intermediate physical address, accumulated permissions, outstanding memory transaction, and fault state. Translation invalidation must match all affected contexts and page sizes.

### 3.5 Coherence transaction entry

Record line address, initiating request, stable starting state, transient state, requested final permission, outstanding acknowledgement count, data source expectation, queued conflicting probes, requester list, and error/retry state. Transient state is essential: a line waiting for invalidation acknowledgements is neither simply Shared nor Modified.

## 4. Cache organization from address and timing constraints

For capacity $C$, line size $B$, and associativity $A$, the number of sets is

$$S=\frac{C}{AB}, \qquad \text{index bits}=\log_2 S, \qquad \text{offset bits}=\log_2 B.$$

For a virtually indexed, physically tagged L1 cache, all set-index bits used before translation must lie within the page offset unless synonym handling is added. With 4 KiB pages, six line-offset bits for a 64-byte line leave six page-offset bits for indexing: at most 64 sets without additional mechanisms. A 32 KiB, eight-way cache has $32\text{ KiB}/(64\text{ B}\times8)=64$ sets and fits this condition.

Choose bank mapping so common simultaneous accesses distribute, but verify that a bank conflict produces backpressure or replay rather than silent loss. Define whether tag and data arrays access in parallel, serially, or with way prediction. Parallel access lowers hit latency but reads more data ways; serial access saves energy but adds a stage. Way prediction trades occasional replay for reduced energy.

Replacement is a policy with stored state. True least-recently-used replacement scales poorly in highly associative caches; pseudo-LRU, re-reference interval prediction, or insertion policies reduce metadata/logic but behave differently under streaming and mixed reuse. Prefetch fills must state insertion priority and whether they may consume the last MSHR needed by demand traffic.

## 5. Nonblocking miss and writeback protocol

On a miss:

1. compare the block address against live MSHRs;
2. merge a compatible request, or allocate a new entry;
3. select a victim and reserve any required writeback capacity before destroying its state;
4. issue a coherent read, read-for-ownership, or upgrade with a unique identity;
5. accept data beats in any fabric-legal order and track beat validity;
6. verify/correct ECC, install tag/data and the granted coherence state;
7. wake or replay dependents only when their requested bytes and permission are valid;
8. retain transaction state until all protocol acknowledgements are complete.

Reserve forward-progress resources. If every MSHR holds a dirty victim but the writeback queue is full, and writeback responses require an input buffer consumed by fills, circular waiting can deadlock. Use separate virtual channels or reserved buffer classes for requests, responses, probes, and writebacks according to the protocol dependency graph.

By Little’s law, the minimum concurrent miss capacity is approximately

$$N_{MSHR} \ge \lambda_{miss}L_{miss},$$

where $\lambda_{miss}$ is new misses per cycle and $L_{miss}$ is average occupied time in cycles. At 0.25 new misses/cycle and 80 cycles, the average is 20 live misses. A 24-entry design has little burst headroom; 32 may be safer if the fabric can sustain the offered traffic. More entries do not help once memory bandwidth or downstream queue capacity saturates.

## 6. Translation and protection implementation

The TLB is a cache of page-table decisions, not merely address bits. Lookup must match context and page size, then recheck access type and privilege. Superpages complicate matching because different page sizes ignore different low virtual-page bits. A split small/large-page structure can simplify timing.

On a miss, allocate a walker entry and fetch page-table entries through a defined memory path. If walks use the data cache, specify priority and avoid recursive translation; walker memory addresses are already physical after the root is located. Cache intermediate page-table entries only with invalidation rules. Multiple misses to the same translation may merge, but faults and access types can differ.

Translation invalidation is a distributed synchronization operation. Define the initiator, target contexts, acknowledgement point, and the rule for in-flight accesses. The safe simple design drains affected accesses before acknowledgement. A higher-performance design tags requests with translation epochs and rejects stale completions.

TLB reach is

$$R_{TLB}=\sum_i N_iP_i,$$

where $N_i$ and $P_i$ are entries and page size in class $i$. Compare reach with active working set, not total process memory. Adding entries, adding superpage support, or a second-level TLB trades lookup energy/latency against walk traffic.

## 7. Coherence controller specification

Start with stable states such as Invalid, Shared, Exclusive, and Modified, then enumerate a state-event-action-next-state table. Events include local load/store, replacement, incoming probe, data response, acknowledgement, retry, and error. Add transient states for each outstanding sequence: for example, Invalid-to-Shared waiting for data differs from Shared-to-Modified waiting for invalidation acknowledgements.

Define a single serialization point for conflicting requests to one line—home directory, shared cache slice, snoop filter, or bus order. For a directory, store owner and/or sharer information plus pending transaction state. Exact bit vectors are simple but scale linearly with agents; sparse vectors, limited pointers, or broadcast-on-overflow trade metadata for traffic.

Core invariants include:

- at most one writer permission exists for a line;
- a reader’s returned data comes from the latest serialized write allowed by the memory model;
- memory and a dirty owner cannot both be treated as authoritative;
- every accepted transaction eventually completes or returns an architected retry/error under fair service;
- probes cannot be permanently blocked behind demand requests that depend on those probes.

Race handling must be explicit. If a local eviction, upgrade response, and remote probe meet, the transient-state table decides whether data is forwarded, installed then invalidated, retried, or written back. “Handle races in the controller” is not a specification.

## 8. Memory ordering and atomics

Coherence orders accesses to one line; consistency constrains the order software observes across addresses. Implement the ISA memory model through rules on issue, completion, retirement, store-buffer drain, and fences. State whether loads may pass older loads, loads may pass unresolved stores, or stores may drain out of order.

A memory-order violation occurs when a younger load executed before an older store’s address resolved and both ultimately overlap. Detect it by comparing the resolved store against younger completed loads, then squash or selectively replay from the offending load. Dependency prediction can reduce unnecessary stalls but needs training and a correctness fallback.

An atomic read-modify-write must appear indivisible at the coherence serialization point. This can use line ownership plus local exclusion, a locked transaction, or a home-agent atomic operation. Define interaction with eviction, interrupts, faults, and reservation granules. Fences complete only when the required older operations reach the memory-model-defined visibility point—not necessarily when they merely leave the LSU.

## 9. Policy trade-offs and physical consequences

| Decision | Advantage | Cost/failure region |
|---|---|---|
| larger line | lower tag cost, spatial locality | false sharing, wasted bandwidth, longer fill |
| higher associativity | fewer conflict misses | tag energy, hit time, replacement complexity |
| inclusive shared cache | simpler snoop filtering | back-invalidations and duplicated capacity |
| exclusive/non-inclusive | more effective capacity | ownership/location tracking becomes harder |
| aggressive load bypass | hides store-address latency | replay and side-channel exposure |
| directory coherence | scalable targeted probes | metadata, home hot spots, transient states |
| snooping | simple small-system ordering | broadcast bandwidth and energy |
| hardware page walker concurrency | hides translation latency | cache/fabric contention and invalidation races |

Place L1 tags, data banks, TLB, load queue, and forwarding comparison close to address-generation units. Fully associative store comparisons and load-replay detection can dominate timing. Filter comparisons by age, partial address, or store sets, but prove that filters only create false positives, never false negatives. Larger lower-level caches are wire-dominated; bank/slice them and include NoC traversal in hit latency.

## 10. Verification, performance evidence, and staged build

Use four complementary oracles:

1. an ISA reference for architectural values and faults;
2. memory-model litmus tests for allowed/forbidden outcomes;
3. protocol assertions or formal checking for line-state invariants and progress;
4. a transaction scoreboard for data integrity from stores through evictions, probes, and refills.

Test every stable/transient state against every legal event, plus invalid responses, duplicate/late identities, response reordering, full queues, simultaneous eviction/probe, TLB invalidation during a walk, page fault after cache lookup, partial forwarding, atomics at line boundaries, ECC correction, poison, and reset with traffic outstanding.

Expose load latency by source—store forward, L1, L2, last-level cache, memory—plus TLB level, walk cycles, MSHR occupancy, merge rate, bank conflicts, replay causes, writeback blockage, probe latency, coherence retry, directory occupancy, and fabric backpressure. Derive average memory-access time only after checking the latency distribution and overlap; a long miss need not stall retirement if independent work covers it.

Build in this order:

1. uncached, blocking physical memory with precise faults;
2. blocking write-through or write-back L1 with one outstanding miss;
3. store buffer and forwarding;
4. nonblocking misses and writebacks;
5. virtual translation and protection, then concurrent walks;
6. a two-agent coherence protocol with exhaustive state testing;
7. directory/many-core scale, speculative loads, prefetch, and advanced replacement.

Each step retains the earlier correctness oracle and adds a new performance hypothesis. Do not introduce coherence, nonblocking replay, and virtual memory simultaneously; their races become indistinguishable.

## 11. Reconstruction checklist

- Can every bit in a load, store, MSHR, TLB, walker, and coherence entry be justified?
- Is the permission point for speculative loads and globally visible stores unambiguous?
- Are all stable and transient coherence transitions tabulated, including collisions?
- Can request/response/probe/writeback buffer dependencies be shown acyclic?
- Do queue depths follow workload rate and service latency, with burst headroom?
- Can translation invalidation, replay, reset, and stale responses be handled without identity ambiguity?
- Do counters identify latency source and blocked resource rather than report one aggregate miss rate?

---

← [Core Blueprint](01_Frontend_and_Execution_Core_Implementation_Blueprint.md) · next → [CPU Integration, Verification, and Bring-up](03_CPU_Integration_Verification_and_Bringup_Blueprint.md)
