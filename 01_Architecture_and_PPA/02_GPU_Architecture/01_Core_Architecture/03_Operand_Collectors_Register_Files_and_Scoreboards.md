# GPU Operand Delivery — Register Files, Operand Collectors, Scoreboards, and Replay

> **First-time reader orientation:** A graphics processing unit (GPU) can keep thousands of threads ready, but its arithmetic lanes still need actual operand bits every cycle. The structures that choose a ready warp, read a heavily banked register file, collect its operands, and route them to the correct pipeline often limit performance before the arithmetic units do.

> **Abbreviation key — skim now and return as needed:** graphics processing unit (GPU); single instruction, multiple threads (SIMT); streaming multiprocessor (SM); register file (RF); vector register file (VRF); scalar register file (SRF); arithmetic logic unit (ALU); floating point (FP); load/store unit (LSU); special function unit (SFU); matrix multiply-accumulate (MMA); level-one cache (L1); instructions per cycle (IPC); first in, first out (FIFO); read-after-write (RAW); write-after-read (WAR); write-after-write (WAW).

> **Prerequisites:** [GPU Architecture](01_GPU_Architecture.md) for the streaming-multiprocessor map and [SIMT Scheduling and Occupancy](02_SIMT_Scheduling_and_Occupancy.md) for warps, residency, and eligibility.
> **Hands off to:** [Independent Threads and Asynchronous Pipelines](04_Independent_Thread_Scheduling_and_Asynchronous_Pipelines.md) for modern execution control and [GPU Memory System](../02_Memory_System/00_Index.md) for coalescing, caches, and outstanding misses.

---

## 0. Why “many ready warps” is only half the design

A warp scheduler may identify an eligible instruction, yet issue can still fail because:

- its source register banks conflict;
- no operand collector is free;
- a destination writeback collides with another result;
- the required execution pipeline is busy;
- a long-latency load has not cleared the scoreboard;
- a barrier or memory-order dependency blocks the warp.

The execution path is therefore a sequence of separately capacity-limited stages:

```mermaid
flowchart LR
    W["resident warps"] --> S["scoreboard<br/>dependency eligibility"]
    S --> I["warp scheduler<br/>select + issue"]
    I --> C["operand collectors"]
    R["banked register files"] --> C
    C --> D["dispatch / crossbar"]
    D --> X["ALU / LSU / SFU / matrix"]
    X --> B["result / writeback network"]
    B --> R
    B --> S
```

Peak lane count describes only `X`. Sustained throughput is the minimum capacity along the whole chain.

## 1. Warp state and the scoreboard

A **scoreboard** tracks whether an instruction's dependencies are safe. In a simple design, decoding a destination register marks it pending; writeback clears it. A later instruction reading that register is ineligible while the bit is set.

GPU scoreboards differ from a CPU issue queue in emphasis:

- a GPU keeps state per warp and chooses among many warps rather than searching one large pool of individual operations;
- register names are usually not dynamically renamed, so false dependencies may remain within a warp;
- long memory latency is tolerated by switching to another warp instead of building a very deep OoO window;
- barriers, fences, and asynchronous-copy tokens add non-register dependencies.

An advanced scoreboard may track readiness per register, instruction class, memory transaction, barrier phase, or asynchronous event. “Warp ready” is therefore shorthand for a conjunction:

$$
ready = operands\_ready \land barrier\_clear \land memory\_legal \land pipe\_available.
$$

Keeping pipeline availability out of the persistent scoreboard can simplify state: the scheduler first finds dependency-ready warps, then arbitrates among them for current resources.

## 2. Why the GPU register file is difficult

Each resident thread owns architectural registers. If an SM holds $T$ threads and each thread receives $R_t$ 32-bit registers, storage is

$$
C_{RF}=4T R_t\ \text{bytes}.
$$

For 2,048 threads at 64 registers/thread, this is 512 KiB before error protection and peripheral logic. More important, one warp instruction may request two or three operands for 32 lanes at once. Building 64–96 independent reads in a conventional multiported memory is impractical.

GPU RFs therefore use many banks and time/space multiplexing. Register mapping spreads lane/register combinations across banks. If several requests target the same bank in the same service slot, some must wait unless the hardware can broadcast one shared value.

The register file creates three coupled limits:

1. **capacity** limits resident warps and therefore latency hiding;
2. **bank bandwidth** limits issued operand groups;
3. **wire energy** grows because large lane-wide values travel between banks, collectors, and execution units.

This is why reducing registers per thread can improve performance even when no spill occurs: it may admit another block or warp and increase the number of alternatives the scheduler can choose.

## 3. Operand collectors

An **operand collector** is a temporary holding structure between issue and execution. It accepts an instruction, requests its source operands from RF banks over one or more cycles, stores arriving values, and dispatches only when the full operand set is ready.

Collectors decouple two mismatched widths:

- issue sees a whole warp instruction;
- the RF may deliver only a subset of its lane operands per cycle.

They also let bank arbitration interleave operands from several instructions. While instruction A waits for a conflicting bank, instruction B may collect from idle banks.

A collector entry typically holds:

- warp and instruction identity;
- destination and execution-unit class;
- one valid bit per source operand segment;
- pending bank requests;
- active-lane mask and control metadata;
- replay or exception state.

Too few collectors backpressure issue. Too many increase storage, crossbar inputs, and arbitration cost. Allocate them by observed residency time:

$$
N_{collector}\gtrsim \lambda_{issue}T_{collect},
$$

using Little's law, then add headroom for bank-conflict bursts.

## 4. Bank conflicts and broadcast

Assume $B$ banks and $K$ independent operand requests whose bank mapping is approximately uniform. The expected number of distinct banks used is

$$
E[U]=B\left(1-\left(1-\frac{1}{B}\right)^K\right).
$$

The difference $K-E[U]$ is a first estimate of requests that collide. It is optimistic when register allocation or lane mapping creates patterns.

Hardware can mitigate conflicts with:

- more banks or dual-pumped banks;
- compiler-aware register allocation;
- bank-aware warp selection;
- operand reuse caches;
- broadcast when all lanes read one scalar value;
- separate scalar and vector register files;
- collector scheduling that steals otherwise idle banks.

AMD-style scalar datapaths make a useful principle explicit: values uniform across a wavefront need not be stored and read 32 or 64 times. A scalar register file and scalar execution unit remove redundant vector RF traffic for addresses, loop bounds, and common control.

## 5. Issue, dispatch, and pipeline specialization

“Issued” can mean the scheduler selected an instruction, while “dispatched” means collected operands entered an execution pipeline. Keeping those events separate explains why headline issue width may not equal completed instruction width.

An SM usually has several pipeline families:

- integer and FP ALUs;
- branch/control;
- load/store address and data paths;
- special functions such as reciprocal or transcendental approximation;
- tensor/matrix pipelines;
- asynchronous data-movement engines.

One instruction cannot use any arbitrary lane. Scheduler partitions, collector sets, and dispatch crossbars are often specialized to a subset of pipelines. Dual issue helps only when a warp or scheduler partition finds two independent instructions targeting compatible resources and when RF/collector bandwidth can feed both.

## 6. Writeback, bypass, and result reuse

Results return through another bandwidth-limited network. A design may:

- write the RF and wake the scoreboard;
- bypass a result directly to a following instruction;
- keep recently used operands in a small reuse cache;
- accumulate matrix results internally for several cycles before exposing them;
- route load data through a separate path.

Unlike a CPU's deeply connected bypass network, a GPU can often tolerate an extra latency cycle by scheduling a different warp. This favors energy-efficient RF writeback over universal forwarding. But tensor kernels create long dependency chains inside a small number of warps; modern designs add targeted forwarding or accumulator storage where switching warps does not hide the delay.

Writeback arbitration must preserve identity across replay. A late load response needs a warp, destination register, active mask, and generation tag. If the warp has been canceled or its slot reused, the response must be dropped.

## 7. Memory scoreboard and replay

A global-memory instruction can generate several coalesced transactions. Some sectors hit while others miss. The warp cannot blindly clear its destination when the first sector returns.

Implementations track:

- the destination register range;
- outstanding sector or request count;
- per-lane participation;
- fault and retry state;
- ordering against fences, atomics, and barriers.

Replay may occur for a cache miss, translation miss, structural conflict, or failed coalescer allocation. The instruction can remain in a replay buffer while the warp scheduler runs other work. Completion clears the scoreboard only when every required fragment has arrived and any exception decision is resolved.

This is a distributed transaction: scheduler, coalescer, cache, translation unit, and writeback path all hold pieces of its state. Transaction identity is as important in a GPU as in a CPU MSHR.

## 8. Matrix and tensor operand delivery

An MMA instruction consumes fragments rather than ordinary scalar operands. The visible instruction may represent many fused operations, but the microarchitecture still has to stage matrices into the format expected by the matrix pipeline.

Possible paths include:

- RF fragments collected and routed to tensor cores;
- shared-memory tiles loaded into registers first;
- asynchronous matrix instructions reading selected operands from shared memory;
- dedicated accumulator registers retained across multiple MMA steps.

The design goal is to avoid using the general RF as a high-energy conveyor belt. Hopper-era asynchronous warp-group MMA and Tensor Memory Accelerator flows make this explicit: producer warps move tiles into shared memory, consumer warp groups launch matrix work, and barriers carry readiness instead of every thread executing address and copy instructions.

## 9. Register pressure versus occupancy

If an SM contains $R_{SM}$ registers and a block has $T_b$ threads using $R_t$ registers each, the RF capacity bound on resident blocks is

$$
B_{RF}=\left\lfloor\frac{R_{SM}}{T_bR_t}\right\rfloor,
$$

after applying architecture-specific allocation granularity. The actual block count is the minimum of RF, shared-memory, warp, thread, and block limits.

Reducing $R_t$ may raise occupancy but can introduce spills to local memory. More occupancy is useful only if it produces additional *eligible* warps. The correct optimization is not “maximize occupancy”; it is “provide enough alternatives to hide the measured stall without creating spill traffic.”

## 10. Observability and verification

Counters should distinguish:

- no dependency-ready warp;
- ready warp but target pipeline busy;
- collector full;
- RF bank conflict;
- dispatch crossbar conflict;
- writeback conflict;
- memory scoreboard wait;
- barrier wait;
- register-capacity occupancy limit;
- local-memory spill traffic.

Key correctness properties include:

1. A scoreboard bit clears only after all writers for that logical destination complete.
2. A collector never combines operands from different warp generations.
3. Bank retries neither lose nor duplicate an operand.
4. Inactive lanes cannot write architectural destinations.
5. A replayed instruction completes exactly once.
6. Barriers release only the threads or warps belonging to the same barrier phase.
7. A canceled warp drops late cache, translation, and execution responses.

## 11. Worked examples

**1 — RF occupancy bound.** An SM has 65,536 32-bit registers. A 256-thread block uses 80 registers/thread: 20,480 registers/block, so at most $\lfloor65{,}536/20{,}480\rfloor=3$ blocks fit by RF capacity. Reducing to 64 registers/thread uses 16,384 registers/block and permits four blocks—a 33% increase in block residency if no other resource binds.

**2 — Collector sizing.** Two warp instructions issue per cycle and operand collection takes 3 cycles on average. Little's law gives six occupied collectors. Eight collectors leave two entries of burst headroom; four guarantee issue backpressure even without unusual bank conflicts.

**3 — Expected bank use.** Eight requests map independently to eight banks. $E[U]=8(1-(7/8)^8)\approx5.25$ banks, so about $8-5.25=2.75$ requests collide in the first service opportunity. Mapping and broadcast can improve this; a single-cycle eight-request design cannot assume eight useful banks merely because eight banks exist.

## Numbers to remember

| Quantity | Typical scale | Why it matters |
|---|---:|---|
| warp/wave width | commonly 32 or 64 threads | multiplies logical operand demand |
| RF capacity per SM | tens to hundreds of KiB | often the largest local storage and an occupancy bound |
| source operands | commonly 2–3 | sets bank and collector pressure |
| collector residence | several cycles under conflict | sizes the decoupling buffer |
| resident warps | often tens per SM | scheduler alternatives for latency hiding |
| matrix instruction | many fused lane-level operations | makes operand staging more important than instruction count |

## Cross-references

- [SIMT Scheduling and Occupancy](02_SIMT_Scheduling_and_Occupancy.md) explains how eligible warps are chosen.
- [Independent Threads and Asynchronous Pipelines](04_Independent_Thread_Scheduling_and_Asynchronous_Pipelines.md) adds per-thread control, barriers, and producer–consumer specialization.
- [Coalescing, Caches, and Shared Memory](../02_Memory_System/01_Coalescing_Caches_and_Shared_Memory.md) continues the memory transaction after issue.
- [NPU Systolic and Spatial Dataflows](../../03_NPU_Architecture/01_Compute_Dataflows/02_Systolic_Spatial_and_Vector_Dataflows.md) contrasts GPU RF-fed lanes with explicit accelerator dataflow.

## References

1. NVIDIA, “Fermi Compute Architecture Whitepaper” — [PDF](https://www.nvidia.com/content/pdf/fermi_white_papers/nvidiafermicomputearchitecturewhitepaper.pdf).
2. NVIDIA, “Volta Architecture Whitepaper” — [PDF](https://images.nvidia.com/content/volta-architecture/pdf/volta-architecture-whitepaper.pdf).
3. AMD, “CDNA Architecture Whitepaper” — [PDF](https://www.amd.com/content/dam/amd/en/documents/instinct-business-docs/white-papers/amd-cdna-white-paper.pdf).
4. A. Zulfiqar et al., “Software-Directed Techniques for Improved GPU Register File Utilization,” TACO 2018 — [NVIDIA Research](https://research.nvidia.com/publication/2018-09_software-directed-techniques-improved-gpu-register-file-utilization).
5. S. Chen et al., “Bank Stealing for Conflict Mitigation in GPGPU Register File,” 2015 — [project page](https://sc2682cornell.github.io/publication/bank/).

---

← [SIMT Scheduling and Occupancy](02_SIMT_Scheduling_and_Occupancy.md) · [Core Architecture index](00_Index.md) · next → [Independent Threads and Asynchronous Pipelines](04_Independent_Thread_Scheduling_and_Asynchronous_Pipelines.md)
