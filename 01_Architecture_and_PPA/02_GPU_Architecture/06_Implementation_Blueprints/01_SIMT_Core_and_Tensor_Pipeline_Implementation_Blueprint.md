# GPU SIMT-Core and Tensor-Pipeline Implementation Blueprint

> **Abbreviation key:** graphics processing unit (GPU); single instruction, multiple threads (SIMT); streaming multiprocessor (SM); instruction set architecture (ISA); multiply-accumulate (MAC); program counter (PC).

## 0. Purpose and design ideology

A GPU obtains throughput by holding many groups of threads in hardware and switching among ready groups when another waits. Single instruction, multiple threads (SIMT) means one instruction is issued for a warp—a hardware scheduling group—while an active mask selects which lanes execute it. The core design ideology is therefore **spend state to hide latency, and share control to amortize instruction delivery**. This differs from a CPU: a GPU normally tolerates a long operation by selecting another warp rather than building a large out-of-order window for one thread.

The implementation boundary is the streaming multiprocessor (SM), also called a compute unit in some architectures. It owns resident warp state, register and shared-memory allocation, warp scheduling, execution pipelines, and local memory requests. Grid-level block assignment and global memory live at higher layers.

## 1. Freeze the execution contract

Before choosing lane count or register-file size, specify:

- instruction-set and scalar/vector/tensor operation semantics;
- warp size $W$, maximum threads and warps per block, and synchronization scope;
- divergence and reconvergence behavior, including independent per-thread progress if supported;
- register addressing and allocation granularity;
- shared-memory size, banking, consistency, barriers, and atomics;
- memory address spaces and ordering/fence scopes;
- exception model: per-thread, per-warp, per-block, or context-fatal;
- preemption/context-switch boundary and state to save;
- tensor/matrix operand shapes, precision, accumulation, rounding, saturation, and exceptional values;
- launch metadata and resource-admission rules.

Define non-goals. A first design can omit fine-grained preemption, nested parallelism, or independent-thread scheduling, but it cannot leave their behavior ambiguous.

## 2. SM ownership decomposition

~~~mermaid
flowchart LR
    DISP["block distributor"] --> ADM["resource admission"]
    ADM --> WS["warp slots: PC + masks + state"]
    WS --> SCH["warp schedulers + scoreboards"]
    SCH --> OC["operand collectors"]
    RF["banked register file"] --> OC
    OC --> EX["scalar / vector / tensor / load-store pipelines"]
    EX --> RF
    EX --> SB["completion + scoreboard clear"]
    SB --> SCH
    EX --> MEM["coalescer / shared memory / cache"]
    BAR["barrier state"] --> SCH
~~~

The admission unit calculates whether a new thread block fits all resources simultaneously. It reserves warp slots, register allocation units, shared-memory bytes, barrier identifiers, and block metadata atomically. Partial admission risks resource leaks.

## 3. Required state schemas

### 3.1 Resident block and warp

A block entry stores grid/block identity, base warp range, thread count, allocated register region, shared-memory base/size, barrier state, pending asynchronous-copy groups, completion count, and fault/preemption state.

A warp entry stores validity, block identity, warp-within-block identity, program counter, active-thread mask, completed-thread mask, reconvergence or per-thread program-counter state, stall reason, scoreboard dependency state, outstanding memory/tensor operation identities, barrier membership, and instruction-buffer metadata. With independent thread scheduling, lanes can have different next program counters; hardware must form issuable sub-warps and still implement warp-scope synchronization correctly.

### 3.2 Scoreboard

The scoreboard records whether each logical destination register or special resource has an outstanding producer and which pipeline will complete it. An instruction is eligible only if source registers are available, destination hazards meet the ISA rule, the target execution pipeline can accept it, and ordering/barrier constraints permit it. Track long-latency operations with unique identities so a late completion cannot clear a newer dependency.

A simple scoreboard blocks on write-after-write as well as read-after-write hazards. Renaming could remove false dependencies but adds per-warp maps and recovery complexity; most throughput GPUs instead rely on many warps and compiler scheduling.

### 3.3 Operand collector and register file

An operand-collector entry stores issued instruction identity, warp/lane mask, source register numbers, collected-source bits, destination, and target pipeline. It arbitrates register-file banks over several cycles and releases the instruction when all operands are present. This decouples issue from bank conflicts.

The register-file address derives from resident warp allocation, logical register, and lane. Specify bank mapping, read/write ports, operand reuse buffers, bypass, and collision policy. If two operands map to one single-ported bank, the collector serializes them; scheduling must not assume one-cycle operand collection.

### 3.4 Reconvergence and barrier state

A stack-based divergence design records reconvergence program counter, pending path program counter, and masks. An immediate-post-dominator convention works for structured control flow but needs well-defined behavior for unstructured branches. Independent-thread designs track per-lane PCs and convergence tokens. In both cases, the invariant is that each non-completed thread executes exactly the instructions required by its control flow—never twice and never while masked off.

A barrier entry stores participating warps/threads, arrival mask/count, phase or generation, memory-fence requirements, and release state. Generation prevents a late arrival from a previous barrier instance releasing a later one.

## 4. Cycle-level instruction path

1. A warp slot requests fetch at its current program counter.
2. Instruction cache/decode returns an operation and control metadata; a small buffer may decouple fetch from scheduling.
3. The scheduler considers ready warps. Eligibility combines scoreboard, barrier, memory-throttle, operand-collector availability, and pipeline acceptance.
4. Arbitration chooses warps by a documented policy—round-robin, greedy-then-oldest, age, locality, or two-level grouping.
5. Issue reserves an operand-collector entry and marks destination dependencies before another instruction can slip through.
6. The collector reads register banks and applies immediate/reuse/bypass sources.
7. The execution pipeline accepts operation, mask, identity, and operands. Inactive lanes do not update state and should be power gated where possible.
8. Completion writes register banks, clears the matching scoreboard entry, reports faults, and makes the warp eligible.
9. Branch resolution updates masks/PC/reconvergence state. Memory operations remain pending until all required transactions return.
10. When all threads finish and all block-scoped effects are complete, admission releases resources.

Backpressure must propagate without losing warp state: full operand collectors prevent issue; a full memory queue keeps the instruction pending; full completion/writeback ports delay results and therefore scoreboard clearing. Avoid a cycle in which issue marks a destination busy but the downstream reservation fails.

## 5. Tensor and asynchronous pipeline contract

A matrix/tensor instruction often spans many lanes and cycles. Specify the logical matrix fragments each lane supplies, register encoding, internal tile decomposition, accumulator ownership, pipeline latency/initiation interval, supported mixed precision, and writeback grouping. “One tensor operation” at the ISA may expand into repeated sub-array operations.

For an array with $M\times N$ multiply-accumulate units at clock $f$, peak multiply-accumulate rate is

$$T_{MAC}=MNf\eta_{issue},$$

where $\eta_{issue}\le1$ captures initiation gaps and structural limits. If counting one multiply plus one add as two operations, reported operations/s doubles. State the convention.

Asynchronous global-to-shared copies require producer/consumer state: copy-group identifier, destination shared-memory range, expected sectors, arrived sectors, error, commit, and wait threshold. A wait instruction must not release consumers until the required data is visible in shared memory. Double buffering overlaps load and compute but needs disjoint buffer ownership and a phase protocol.

## 6. Resource sizing and occupancy

For registers per SM $R_{SM}$, registers per thread $r$, threads per block $T_b$, and allocation granularity $G_R$, register-limited blocks are

$$B_R=\left\lfloor\frac{R_{SM}}{\lceil rT_b/G_R\rceil G_R}\right\rfloor.$$

For shared memory $S_{SM}$ and per-block allocation $s_b$ rounded to $G_S$, $B_S=\lfloor S_{SM}/\lceil s_b/G_S\rceil G_S\rfloor$. Resident blocks are the minimum of register, shared-memory, warp-slot, thread, barrier, and architectural block limits. Occupancy is resident warps divided by maximum warp slots, but high occupancy is only a means: extra warps help until latency is hidden or bandwidth/issue becomes saturated.

A latency-hiding estimate is

$$N_{ready}\gtrsim \frac{L}{I},$$

where $L$ is cycles until a warp can issue its next dependent operation and $I$ is useful independent issue opportunities contributed per resident warp. If a dependency returns in 24 cycles and each warp supplies one eligible instruction before waiting, roughly 24 independently ready warp opportunities are needed for one issue slot; multiple schedulers and instruction-level parallelism change the estimate.

Register-file bandwidth follows issued operands: $B_{RF}=N_{issue}\times N_{src}\times W\times b$ bits/cycle before reuse and banking, where $b$ is operand width. This enormous value explains banked files, collector serialization, reuse caches, and physically distributed scheduler partitions.

## 7. Policy and physical trade-offs

| Choice | Benefit | Cost or losing case |
|---|---|---|
| more resident warps | latency tolerance | register-file area/leakage and scheduler state; no help when bandwidth-bound |
| greedy scheduling | instruction locality, cache reuse | one warp can dominate; poor fairness/latency |
| round-robin | fairness and simplicity | may sacrifice locality and oldest-first progress |
| larger operand collectors | tolerate bank conflicts | area, arbitration, and longer residency |
| wider register banks | more throughput | wire/clock energy and writeback timing |
| independent-thread scheduling | handles divergent/blocking lanes | larger control state and harder barrier semantics |
| larger tensor arrays | higher peak density | utilization collapses on small/irregular shapes; accumulator/data bandwidth grows |

Replicated SM structures amplify small costs by the SM count. Physically cluster scheduler, collector, register banks, and their execution lanes. Global crossbar access to one monolithic register file is usually untenable. Partition writeback and make cross-partition operands explicit. Schedule pipeline stages around wire distance, then reflect added latency in scoreboard release and performance models.

## 8. Invariants, verification, and counters

Prove or assert:

- an admitted block never exceeds any reserved resource;
- a logical register has at most the permitted live producer per warp;
- a masked lane cannot write architectural, shared-memory, or global-memory state;
- each issued instruction completes, faults, or is canceled exactly once;
- a barrier releases only the correct generation after all required participants and memory effects arrive;
- register-bank conflicts delay but do not drop operands;
- tensor fragment mapping and accumulation match the numerical contract;
- under fair downstream service, a ready warp is not starved indefinitely.

Differentially compare scalar/thread semantics against an instruction reference model, then compare warp/block kernels. Directed kernels must cover nested divergence, loops with partial exit, barrier under divergence, register bank collisions, simultaneous writebacks, tensor edge tiles, special floating-point values, asynchronous-copy races, and maximum resource fragmentation.

Counters should distinguish no eligible warp, eligible but not selected, scoreboard dependency, instruction fetch, operand-collector full, register-bank conflict, execution-pipe full, memory throttle, barrier, and writeback conflict. Also measure active lanes per issued instruction, tensor array utilization, issued instructions per scheduler, resident/eligible warps, and resource fragmentation. These explain why occupancy and peak operations do or do not translate to throughput.

## 9. Minimum viable implementation

1. One SM, one resident warp, scalar/vector arithmetic, no divergence beyond a simple mask.
2. Add multiple warps and a scoreboard with one scheduler.
3. Add banked registers and operand collectors with arbitrary conflict stalls.
4. Add structured divergence/reconvergence and block barriers.
5. Add shared and global memory, then atomics and scoped fences.
6. Add multiple schedulers/execution partitions and validate steering fairness.
7. Add tensor pipelines and asynchronous copies with a numerical reference.
8. Add independent-thread scheduling, preemption, and wider scale only when earlier invariants remain observable.

## 10. Reconstruction checklist

You can specify the SM when every resident resource has allocation/release rules, every stall reason is represented in state, register and tensor bandwidth are numerically budgeted, divergence/barrier invariants are testable, and the minimum correct design can grow without changing its execution contract.

---

Next → [GPU Memory and Scale-Up Implementation Blueprint](02_GPU_Memory_and_Scale_Up_Implementation_Blueprint.md)
