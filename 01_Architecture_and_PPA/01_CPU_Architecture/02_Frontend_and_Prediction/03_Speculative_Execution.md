# Speculative Execution — Predict, Validate, Recover, and Contain Side Effects

> **First-time reader orientation:** A fast central processing unit (CPU) often starts an operation before it knows that the operation is on the correct program path or that all of its inputs are safe to use. That early work is *speculative*. The result may become real only after the CPU validates the guess; otherwise the CPU must discard the work and restore an older correct state.

> **Abbreviation key — skim now and return as needed:** central processing unit (CPU); instruction set architecture (ISA); out-of-order (OoO); program counter (PC); branch prediction unit (BPU); fetch target queue (FTQ); reorder buffer (ROB); register alias table (RAT); physical register file (PRF); load-store queue (LSQ); load queue (LQ); store queue (SQ); memory dependence predictor (MDP); store-set identifier table (SSIT); last fetched store table (LFST); translation lookaside buffer (TLB); miss status holding register (MSHR); instructions per cycle (IPC); misses per thousand instructions (MPKI).

> **Prerequisites:** [Branch Prediction](01_Branch_Prediction_Deep_Dive.md) for next-PC prediction, [Out-of-Order Execution](../03_Out_of_Order_Backend/01_OoO_Execution.md) for rename and the ROB, and [Retirement and Recovery](../03_Out_of_Order_Backend/03_Retirement_Recovery_and_Precise_State.md) for precise architectural state.
> **Hands off to:** [Advanced Scheduling, Wakeup, and Replay](../03_Out_of_Order_Backend/04_Advanced_Scheduling_Wakeup_and_Replay.md) for the timing-critical implementation, [Load-Store Unit](../03_Out_of_Order_Backend/02_Load_Store_Unit_and_Memory_Ordering.md) for memory ordering, and [XiangShan](../07_Core_Case_Studies/01_Xiangshan_CPU_Design.md) for a current open implementation.

---

## 0. Why speculation deserves its own chapter

Branch prediction is one form of speculation, but it is not the whole subject. A modern CPU guesses in several places at once:

- **Control speculation:** which instruction address comes next?
- **Memory-dependence speculation:** may a load pass older stores whose addresses are not known yet?
- **Latency speculation:** will a producer finish at its usual time, so dependents can wake before writeback?
- **Cache and translation speculation:** did a way, TLB entry, or permission check hit as expected?
- **Value speculation and runahead:** can the machine predict a value, or temporarily execute only to discover future misses?

These guesses share one engineering pattern. Each needs a prediction, a place to keep recoverable state, a validation event, and a rule for which effects may escape. Studying them as one pattern prevents a common misunderstanding: **speculation is not “executing randomly and hoping.” It is a controlled transaction with an undo boundary.**

```mermaid
flowchart LR
    P["Predict<br/>choose an assumption"] --> A["Allocate recovery state<br/>ROB / checkpoint / queue entry"]
    A --> E["Execute early<br/>tag all dependent work"]
    E --> V{"Validate"}
    V -->|correct| C["Commit<br/>make architectural"]
    V -->|wrong| R["Recover<br/>cancel / replay / redirect"]
    R --> P
```

## 1. Architectural, speculative, and transient state

The **instruction set architecture (ISA)** defines the state software is allowed to observe: registers, memory, exceptions, privilege state, and the order in which these appear. Microarchitecture adds much more state—predictor tables, cache tags, replacement bits, queue entries, physical registers, and in-flight requests.

It is useful to divide that implementation state into three categories:

1. **Architectural state** has passed retirement and is part of the software-visible machine.
2. **Speculative state** belongs to instructions that might retire but have not yet done so. ROB entries, newly allocated physical registers, and an uncommitted store are examples.
3. **Transient side effects** are changes caused by speculative work that may remain after the work is squashed: a filled cache line, trained predictor entry, occupied port, or altered replacement state.

Correctness requires that a wrong guess never corrupt architectural state. Security is stricter: secret-dependent transient effects must not become an observable side channel. A design can therefore be architecturally correct and still insecure.

## 2. The speculation contract

Every speculative mechanism should answer six questions before register-transfer level (RTL) design begins:

| Question | Why it is required |
|---|---|
| What is predicted? | Defines the hypothesis: target, dependency, latency, hit, or value. |
| What identifies the speculation? | A ROB index, FTQ index, generation number, or queue tag distinguishes live work from stale responses. |
| What state is recoverable? | RAT snapshots, free-list pointers, history state, and queue tails must return to a known point. |
| What validates the guess? | Branch execution, address comparison, writeback, tag check, or retirement closes the speculation. |
| What is the recovery scope? | One instruction may replay, a dependency slice may replay, or the whole younger machine may flush. |
| Which effects may escape? | Stores, exceptions, predictor training, cache fills, and external requests need explicit policy. |

The key invariant is:

> An instruction may produce tentative data early, but it may update architectural state only when every older instruction is known to permit it.

The ROB normally enforces the age part of that invariant. Stores add another boundary: their address and data may be computed early, but a store must not become globally visible until it is non-speculative under the memory model.

## 3. Control speculation: following a predicted path

The frontend predicts a next PC because waiting for a branch to execute would starve the pipeline. A practical high-performance predictor is itself pipelined. A small, fast structure gives an early answer; larger structures later override it with a more accurate answer. The FTQ records each predicted fetch block and the metadata needed to train or repair the predictor.

A control-speculation lifetime is:

1. The branch prediction unit predicts direction and target.
2. The FTQ records the predicted block and speculative history.
3. Instructions on that path enter rename and receive ROB entries.
4. The branch executes and computes its actual outcome.
5. If correct, execution continues and training eventually uses a trusted outcome.
6. If wrong, the redirect selects the correct PC and all younger work is invalidated.

The lost work per misprediction is roughly

$$
W_{lost} \approx P_{redirect}\times I_{front},
$$

where $P_{redirect}$ is the redirect-to-refill penalty in cycles and $I_{front}$ is the average number of operations delivered per cycle. This is why a wider machine raises the value of accurate prediction even when the number of lost *cycles* is unchanged.

### 3.1 Speculative predictor history

History-based predictors need the outcome of recent branches to predict the next branch. Waiting until retirement would make the history stale, so the predictor shifts predicted outcomes into a speculative history register. A misprediction must restore the old history and then insert the actual outcome. The same issue appears in a return-address stack: calls and returns on a wrong path must not permanently corrupt the stack.

Common recovery choices are:

- save a complete history checkpoint per in-flight branch;
- save a pointer into a persistent history structure;
- reconstruct history from the FTQ or ROB;
- keep separate speculative and committed structures.

Full checkpoints recover quickly but cost storage and ports. Reconstruction stores less but adds redirect latency.

## 4. Memory-dependence speculation: loads before unknown stores

Register dependencies are visible after decode and rename. Memory dependencies are not: the CPU may not know whether `load [r1]` overlaps an older `store [r2]` until both addresses are calculated.

The conservative policy waits for every older store address. It is safe but often stalls independent loads. The aggressive policy issues the load, compares its address against older stores later, and recovers on a violation. Its expected cost is

$$
C_{spec}=p_{vio}C_{recover},
$$

where $p_{vio}$ is the probability of a true ordering violation. Speculation wins while this is lower than the delay avoided by not waiting.

An **MDP (memory dependence predictor)** learns which static loads have violated before. A store-set design usually contains:

- an **SSIT (store-set identifier table)** mapping an instruction PC to a learned dependency group;
- an **LFST (last fetched store table)** or related structure naming an older in-flight store in that group;
- an update path that merges or refreshes groups after observed violations.

The prediction does not need to name an exact byte address. It answers the scheduling question: “which older store must be far enough along before this load may issue?”

### 4.1 Validation and replay

The LSQ validates speculation using age and address comparisons. It must distinguish at least:

- a true store-to-load ordering violation;
- store data not ready for forwarding;
- a TLB miss or permission delay;
- a cache-bank conflict or miss;
- a coherence probe or invalidation race.

Not all causes justify flushing the whole pipeline. A blocked cache port can replay one load. A load that supplied a wrong value to many dependents may require selective replay of its dependency chain or a redirect that removes all younger instructions. Recovery scope is a performance-versus-complexity choice, not merely a correctness choice.

## 5. Latency speculation and speculative wakeup

Suppose an add normally produces its result one cycle after issue. If the scheduler waits for physical-register writeback before waking a dependent add, it inserts an avoidable cycle. Instead, issue logic can send a **speculative wakeup** based on the producer's expected latency, allowing the dependent to select the bypassed result as it becomes available.

This creates a new obligation: if the producer is canceled, delayed, or misses in the cache, every consumer awakened by that prediction must be canceled or replayed. Implementations carry dependency tags or short load-dependency vectors so a late cancel reaches the affected consumers.

The benefit is largest in a recurrence:

$$
t_{loop}=t_{select}+t_{operand}+t_{execute}+t_{wakeup}.
$$

Removing even one registered cycle from this loop can raise dependent-operation throughput. The cost is a fast broadcast network and a precise cancellation path, both of which become timing-critical as issue width grows.

## 6. Cache, TLB, and way speculation

An L1 cache may read data before a full physical tag comparison finishes, or predict the likely way to save power and mux delay. A TLB hierarchy may allow a fast translation to drive a parallel cache access before all permission and alias checks settle.

These mechanisms are safe only if:

- returned data remains tagged as speculative until tag and permission checks complete;
- exceptions are recorded but not architecturally raised before the instruction reaches the retirement boundary;
- a failed way or translation prediction cancels dependent wakeups;
- stale refills carry an epoch or transaction identity so they cannot complete a reallocated queue entry.

This is the same speculate–validate–recover pattern again, applied to a memory lookup rather than control flow.

## 7. Value prediction and runahead

Two more aggressive techniques are important even when a particular core does not implement them.

**Value prediction** predicts the output of a load or arithmetic instruction and lets dependents execute with that value. Validation compares the prediction with the real result. A mismatch can invalidate a large dependency slice, so confidence estimation is essential; predicting too often may waste more energy and recovery bandwidth than it saves.

**Runahead execution** addresses a different problem. When an old cache miss blocks retirement and the ROB fills, the core checkpoints architectural state, marks the missing value invalid, and continues executing only to discover independent future misses. Those speculative misses act as prefetches. When the original miss returns, the machine restores the checkpoint and executes normally. Runahead does not try to commit the temporary instructions; its product is memory-level parallelism.

These techniques show the outer boundary of speculation: the machine may use wrong-path or non-committing work if the useful microarchitectural effect—usually an early memory request—outweighs its energy and recovery cost.

## 8. Recovery granularity

| Recovery method | Removes | Advantage | Cost |
|---|---|---|---|
| local retry | one request | cheapest when no consumer observed bad data | queue must retain complete request state |
| selective replay | producer and affected consumers | preserves unrelated work | dependency tracking and cancel network |
| checkpoint restore | speculative mappings/history after a saved point | fast state repair | snapshot storage and checkpoint selection |
| ROB walk | younger instructions in program order | simple and general | recovery takes several cycles |
| full redirect/flush | all younger work | easiest correctness argument | loses maximum useful work |

An advanced core normally uses several. Cache-bank conflicts retry locally; load miss wakeup failures selectively cancel dependents; a branch misprediction redirects the frontend and restores a checkpoint; an exception may drain to a precise ROB boundary.

## 9. Speculation and security

Squashing an instruction removes its architectural result, but does not automatically undo cache fills, predictor changes, port contention, or DRAM activity. A transient instruction that reads a secret and then uses it to choose a cache line can encode the secret in timing even though the instruction never retires.

A secure design therefore distinguishes **permission to execute** from **permission to transmit an effect**. Mitigation families include:

- block speculative access until authorization is known;
- partition or flush predictor state across protection domains;
- delay cache-visible changes or keep them in speculative buffers;
- prevent forwarding of faulting or unauthorized values;
- provide speculation barriers and predictor-control instructions;
- verify non-interference properties in addition to ISA correctness.

There is no universal free defense. Delaying all speculative effects sacrifices much of the performance speculation was added to obtain, so threat model and trust boundary must be explicit architecture inputs.

## 10. XiangShan as a current open example

Current Kunminghu documentation exposes several forms of speculation that are often hidden in commercial CPUs:

- a multi-stage BPU whose later answers may override earlier answers;
- an FTQ that retains prediction metadata through commit and recovers its pointers on redirects;
- speculative branch history and a persistent return-address stack;
- rename snapshots used to shorten recovery compared with walking from committed state;
- a memory-dependence predictor based on a load-wait table and a store-set variant;
- speculative wakeup in distributed issue queues, with cancellation feedback;
- a load replay queue that records causes such as TLB miss, cache miss, and forwarding violation;
- redirect arbitration among branches, memory violations, and ROB flushes.

The important lesson is not that XiangShan “has speculation.” It is that speculation is a **cross-pipeline protocol**. Frontend, rename, scheduler, LSU, cache, and retirement all exchange identities, cancel events, and recovery state. Leaving any one of those paths out produces either silent corruption or a deadlock.

## 11. Verification checklist

For each speculation source, verify both the prediction and every way it can be wrong:

1. A correct prediction commits exactly once.
2. A wrong prediction cannot update architectural state.
3. A canceled producer cancels every speculatively awakened consumer.
4. A stale response cannot match a recycled ROB, LSQ, FTQ, or MSHR entry.
5. The oldest redirect wins when several redirects arrive together.
6. Recovery restores RAT, free list, history, queue tails, and privilege context to the same logical boundary.
7. Exceptions remain precise even when detected early.
8. Stores never become visible from a squashed path.
9. Forward progress holds under repeated replay and backpressure.
10. Security tests cover observable transient state, not only retired results.

## 12. Worked examples

**1 — Is memory speculation worthwhile?** A load would wait an average of 5 cycles for unresolved stores. Violations occur for 1% of dynamic loads and cost 18 cycles to recover. The expected speculative cost is $0.01\times18=0.18$ cycles per load, far below the 5-cycle conservative wait. Even at a 10% violation rate the expected cost is 1.8 cycles, so speculation still wins—assuming recovery bandwidth does not saturate.

**2 — Why selective replay matters.** A delayed load has 12 younger dependent operations and the ROB contains 140 younger operations total. Replaying the dependency slice discards 13 operations including the load; a full flush discards 141. If both recover in the same number of cycles, selective replay reduces wasted execution by roughly $141/13\approx10.8\times$. The tracking logic is justified only if such events are frequent enough.

**3 — Checkpoint capacity.** Four rename snapshots are available and branches eligible for snapshots arrive every 8 cycles. A snapshot lives for 28 cycles on average. Little's law predicts $28/8=3.5$ live snapshots, so four covers the mean with little burst headroom. The design must either throttle snapshot creation, fall back to ROB walking, or snapshot additional periodic boundaries when the queue is full.

## Numbers to remember

| Quantity | Typical scale | Design meaning |
|---|---:|---|
| branch frequency | about 15–25% of instructions | control speculation is continuously active |
| high-performance redirect penalty | roughly 8–20 cycles | recovery latency multiplies predictor mistakes |
| L1 load-use latency | roughly 3–5 cycles | makes speculative wakeup valuable and fragile |
| ROB size | roughly 100–600 operations | maximum ordinary speculative window |
| store/load violation rate | workload-dependent, often low | low rate is why aggressive load issue pays |
| stale-response protection | at least identity plus generation/epoch | prevents recycled-entry corruption |

## Cross-references

- [Branch Prediction](01_Branch_Prediction_Deep_Dive.md) explains the predictor structures that create control speculation.
- [Advanced Scheduling, Wakeup, and Replay](../03_Out_of_Order_Backend/04_Advanced_Scheduling_Wakeup_and_Replay.md) follows speculative data through the issue machinery.
- [Memory Consistency and Atomics](../06_Coherence_and_Consistency/02_Memory_Consistency_and_Atomics.md) separates legal architectural reordering from internal speculation.
- [XiangShan](../07_Core_Case_Studies/01_Xiangshan_CPU_Design.md) maps these mechanisms onto current open modules.

## References

1. G. Z. Chrysos and J. S. Emer, “Memory Dependence Prediction using Store Sets,” ISCA 1998 — [paper](https://www.princeton.edu/~rblee/ELE572Papers/MemoryDependencePredictionStoreSets_Emer.pdf).
2. O. Mutlu et al., “Runahead Execution: An Alternative to Very Large Instruction Windows,” HPCA 2003 — [paper](https://www.cs.cmu.edu/~18742/papers/Mutlu2003.pdf).
3. P. Kocher et al., “Spectre Attacks: Exploiting Speculative Execution,” 2018 — [paper](https://arxiv.org/abs/1801.01203).
4. XiangShan Team, “Kunminghu V3 Backend, FTQ, CtrlBlock, IssueQueue, and LSU Design Documents” — [documentation](https://docs.xiangshan.cc/projects/design/en/kunminghu-v3/).
5. XiangShan Team, “Memory Dependence Prediction” — [documentation](https://docs.xiangshan.cc/zh-cn/dev/memory/mdp/mdp/).

---

← [Fetch, Decode, and µop Delivery](02_Fetch_Decode_and_Uop_Delivery.md) · [Frontend index](00_Index.md) · next → [Out-of-Order Backend](../03_Out_of_Order_Backend/00_Index.md)
