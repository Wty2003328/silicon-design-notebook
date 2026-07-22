# CPU Frontend and Execution-Core Implementation Blueprint

> **Abbreviation key:** central processing unit (CPU); instruction set architecture (ISA); load-store unit (LSU); simultaneous multithreading (SMT); reorder buffer (ROB); program counter (PC).

## 0. Purpose and design ideology

This blueprint explains how to turn an instruction set architecture (ISA)—the software-visible instruction contract—into a speculative out-of-order CPU core. The design ideology is **separate permission from execution**: the frontend may predict a path, rename may remove false dependencies, and the scheduler may execute an instruction early, but only in-order retirement may make the result architectural. Performance comes from doing safe-to-discard work early; correctness comes from one precise commit boundary.

An implementation is therefore not merely a pipeline. It is a collection of ownership rules:

- the fetch target queue owns predicted control-flow history;
- the rename map owns speculative register names;
- the reorder buffer owns program-order completion and exceptions;
- issue structures own readiness, not architectural visibility;
- execution units produce tentative results;
- retirement alone updates committed state and releases old resources.

Read [Branch Prediction](../02_Frontend_and_Prediction/01_Branch_Prediction_Deep_Dive.md), [Speculative Execution](../02_Frontend_and_Prediction/03_Speculative_Execution.md), [Out-of-Order Execution](../03_Out_of_Order_Backend/01_OoO_Execution.md), and [Precise Retirement](../03_Out_of_Order_Backend/03_Retirement_Recovery_and_Precise_State.md) for individual mechanisms. Here the goal is reconstruction.

## 1. Freeze the external contract before choosing widths

Write a one-page core contract containing:

| Contract item | Decision required | Why it changes the core |
|---|---|---|
| ISA and extensions | instruction encodings, integer/vector/floating-point operations | fixes decoder coverage, register files, execution units, and illegal-instruction behavior |
| privilege model | modes, control/status registers, protection rules | determines trap entry, translation context, and which speculative effects must be suppressed |
| memory model | permitted load/store reorderings, fences, atomics | constrains load-store queue and retirement rules |
| precise state | interrupt and exception boundary | requires older instructions to complete and younger effects to be discardable |
| debug | halt, single-step, triggers, architectural access | creates non-speculative stop/drain requirements |
| reset and boot | reset vector, initial privilege, cache/translation state | defines deterministic entry conditions |
| implementation targets | clock, area, power, workload, process | decides whether wide associative structures are feasible |

Also state non-goals. A research core may omit simultaneous multithreading (SMT), vectors, or aggressive memory disambiguation in its first version. An omitted feature is safe; an implicit feature becomes a late redesign.

## 2. Decompose the machine by state ownership

~~~mermaid
flowchart LR
    PC["next-PC selection"] --> FTQ["fetch-target queue"]
    FTQ --> IF["I-TLB + instruction cache"]
    IF --> FQ["fetch byte/instruction queue"]
    FQ --> DEC["decode + micro-op formation"]
    DEC --> REN["rename + checkpoints"]
    REN --> ROB["reorder buffer"]
    REN --> IQ["issue queues"]
    IQ --> EX["execution + bypass"]
    EX --> WB["physical register writeback"]
    WB --> ROB
    ROB --> COMMIT["in-order retirement"]
    EX -->|"branch / memory replay"| REDIR["redirect / recovery"]
    COMMIT -->|"trap / interrupt"| REDIR
    REDIR --> PC
~~~

For every arrow, specify payload, identity, acceptance, and cancellation. A useful minimum set is:

- **Fetch request:** virtual program counter, prediction identifier, privilege/context identifier, requested byte span.
- **Fetch response:** bytes, byte-valid mask, fault metadata, physical-page attributes, prediction identifier.
- **Decoded micro-operation:** operation class, source/destination architectural registers, immediate, control-flow metadata, memory width/sign, exception bits.
- **Renamed operation:** physical source/destination tags, old destination tag, reorder-buffer identifier, branch mask/checkpoint identifier.
- **Issue request:** operation, ready operands or tags, age/priority, execution-port eligibility.
- **Completion:** reorder-buffer identifier, destination tag, value or exception, replay/redirect information.
- **Retirement action:** architectural update, resource reclamation, store permission, predictor-training event, trap or interrupt transition.

Use monotonically distinguishable tags or an index plus generation bit wherever an entry can be reused while an old response remains in flight. Otherwise a late response can corrupt a newly allocated instruction.

## 3. Required state schemas

The following fields are a starting point, not an optional glossary.

### 3.1 Fetch-target queue entry

Store start program counter, predicted next program counter, prediction type, branch positions within the block, predictor metadata needed for later training, global/local history snapshots, privilege/context, instruction-cache fault status, and a validity/generation marker. Reclaim only after the corresponding instructions retire or are squashed. Predictor training normally uses resolved outcome plus the original prediction context; reconstructing history after the fact is unreliable.

### 3.2 Rename state

Maintain a speculative map from each architectural register to its latest physical register, a committed map or recovery image, and a free list. A destination allocation records both the new physical register and the displaced old mapping. The old register cannot return to the free list when the new register is allocated; it remains needed by older readers and is released only when the overwriting instruction retires.

A branch checkpoint must be sufficient to restore the speculative map and free-list allocation frontier. Full snapshots recover quickly but cost bits and write energy. Incremental logs reduce storage but lengthen recovery. A hybrid stores periodic full snapshots plus deltas.

### 3.3 Reorder-buffer entry

Store validity, program-order identity, operation class, completion, exception cause and fault address, destination architectural and physical register, old physical register, branch outcome, memory-operation identity, serialization flags, and debug/interrupt metadata. The head may retire only when complete and non-faulting, and only under any ordering constraints. Multiple retirement slots require logic that stops at the first incomplete, excepting, or serializing entry.

### 3.4 Issue and execution state

An issue entry needs physical source tags, ready bits, optional captured values, destination tag, age, execution eligibility, and speculative dependencies such as unresolved loads. The physical register file needs data, validity or poison state, and enough read/write banking to serve scheduled operations. Execution pipelines need stage-valid bits and a cancellation identity so flushes cannot write stale results.

## 4. Causal instruction flow

1. **Predict:** choose a next fetch address and allocate prediction context. If no queue space exists, stop creating predictions.
2. **Fetch and align:** translate the address, access instruction bytes, identify instruction boundaries, and carry faults alongside bytes rather than taking a trap immediately.
3. **Decode:** convert encodings into one or more internal operations. Complex instructions may expand; fused instructions must retain enough metadata for architectural accounting and precise faults.
4. **Rename:** read current mappings, allocate destinations, update the speculative map, and create recovery state. Process same-cycle dependencies in program order so a later instruction sees an earlier same-group destination.
5. **Dispatch:** atomically reserve all required resources—reorder entry, issue entry, load/store entry, and destination register. If any resource is unavailable, do not partially allocate the instruction group unless a rollback protocol is specified.
6. **Wake and select:** mark operands ready from register-file state and completion broadcasts; arbitrate ready operations by age, port compatibility, fairness, and timing limits.
7. **Read and execute:** obtain operands, execute, and either produce a result or a reason to replay. Variable-latency units require explicit completion arbitration.
8. **Resolve:** a branch compares actual and predicted direction/target. A mismatch redirects fetch and invalidates all younger state. A load may detect an ordering violation and request selective or full replay.
9. **Write back:** update only the matching live physical destination and completion entry. Generation checks reject killed or stale returns.
10. **Retire:** in program order, expose results, authorize committed stores, release old physical registers, train retirement-based structures, and take the oldest precise exception.

Backpressure is causal too: retirement blockage fills the reorder buffer, which blocks rename, which fills decode/fetch queues, which stops prediction. Specify this chain and ensure there is no circular dependence requiring a blocked younger operation to free an older one.

## 5. Recovery is a protocol, not a wire

Define redirect priority, the recovery point, and acknowledgement. A typical priority is reset/debug, exception, memory-order replay, branch misprediction, then ordinary prediction. Each redirect carries the oldest responsible instruction identity, corrected program counter, and recovery checkpoint. Every structure compares age and invalidates younger entries. The frontend discards responses with killed prediction generations.

Two invariants are central:

1. after recovery, the speculative rename state equals the committed state plus exactly the surviving older instructions; and
2. no killed instruction may update architectural state, memory, predictor state that is specified as retirement-only, or a reallocated physical destination.

Selective replay preserves unrelated work but requires dependency tracking and careful poison propagation. Full pipeline squash is simpler and is the right minimum viable implementation.

## 6. Size resources from latency hiding

Let target committed throughput be $I$ instructions per cycle and let $L$ cycles be the average interval from rename to retirement for the useful instruction window. A first-order reorder-buffer bound is

$$N_{ROB} \gtrsim I L.$$

If the target is four instructions per cycle and the useful interval is 45 cycles, the baseline is 180 entries. Add headroom for bursty misses and retirement barriers; then check whether a 192- or 224-entry structure meets physical timing. This calculation is a demand estimate, not proof that a larger window helps: instruction-level parallelism and branch accuracy may saturate first.

For physical registers,

$$N_{phys} \ge N_{arch} + N_{inflight\ destinations} + N_{recovery\ reserve}.$$

Estimate destination density $d$ among in-flight instructions, giving roughly $N_{arch}+dN_{ROB}$, then add headroom so one stalled resource does not deadlock rename. Fetch/decode queue depth should cover instruction-cache and predictor bubbles without hiding redirects so long that wrong-path energy dominates.

Issue width is constrained by useful ready operations, execution ports, register-file bandwidth, and completion buses. Doubling issue width can more than double associative compare, arbitration, bypass, and wiring cost. Banked or distributed schedulers improve timing and locality but introduce steering imbalance.

## 7. Policy choices and their losses

| Choice | Benefit | Cost and losing workload |
|---|---|---|
| centralized issue queue | global age/port choice | associative timing and wiring scale poorly with width/window |
| distributed queues | local timing and energy | wrong steering strands ready work when operation mix is uneven |
| value-capture issue entries | fewer register reads at issue | larger entries and heavy broadcast wiring |
| tag-only entries | smaller queue | register-file ports and read-after-wakeup timing become difficult |
| full branch checkpoints | single-cycle recovery | capacity and checkpoint-write energy limit unresolved branches |
| selective replay | retains independent work | dependency bookkeeping and corner-case verification grow sharply |
| aggressive predictor training | adapts sooner | wrong-path or later-squashed training can create pollution/security leakage |
| SMT sharing | fills idle slots | fairness, side channels, partitioning, and worst-case latency become product requirements |

Design ideology should follow the workload. A high-frequency client core may distribute queues and pipeline wakeup/select; a smaller research core may accept a centralized queue for observability and correctness.

## 8. Physical implementation consequences

The likely critical structures are branch target selection, same-cycle rename dependency resolution, issue wakeup/select, register-file read, bypass selection, and multi-wide retirement. Place producer pipelines near their issue queue and register-file banks; long broadcast networks otherwise dominate delay and energy. Partition completion buses by execution cluster and explicitly handle cross-cluster latency.

Clock-gate invalid queue entries and idle execution lanes, but never gate state needed for a pending wakeup or recovery. Wide flush signals are high-fanout; distribute registered epochs or local kill computation rather than a single unbounded combinational net. Model parity or error-correcting code where arrays are large enough that silent corruption violates product reliability.

## 9. Verification and observability contract

Expose a retirement trace containing program counter, instruction encoding, destination/value, memory effect, privilege, and exception. Compare it against an ISA reference model at the architectural boundary. Add assertions for:

- no physical register is allocated twice;
- every live speculative mapping points to an allocated register;
- reorder age remains total and retirement is in order;
- killed operations cannot complete visibly;
- each dispatched instruction is eventually retired, trapped, or squashed under fair memory response;
- backpressure holds payload stable;
- recovery restores the expected rename/free-list state.

Directed tests must include same-cycle rename dependencies, free-list exhaustion, branch plus exception collisions, stale variable-latency completion, simultaneous redirects, interrupt at every pipeline stage, and debug halt around outstanding memory. Performance counters should separately count frontend starvation, rename stalls by resource, issue-ready-but-not-selected cycles, execution-port pressure, replays, redirect penalty, and retirement blockage. A single “pipeline stall” counter cannot guide design.

## 10. Minimum viable implementation and growth path

1. Build a single-issue in-order core with precise traps and a retirement trace.
2. Add instruction/data memory handshakes and prove arbitrary backpressure.
3. Add a simple predictor, but recover by flushing the whole pipeline.
4. Introduce physical registers and a reorder buffer while still issuing in order.
5. Add one centralized issue queue and out-of-order integer execution.
6. Add load/store speculation only after memory-order tests pass without it.
7. Add wider fetch/decode/retire, distributed scheduling, vectors, or SMT one at a time, retaining counters that demonstrate the benefit.

At every step, preserve the same architectural trace contract. That stable boundary lets the internal implementation evolve without changing the definition of correctness.

## 11. Reconstruction checklist

You are ready to specify this core when you can answer, without hand-waving:

- What exact state is allocated for an instruction at prediction, rename, dispatch, issue, completion, and retirement?
- Which operation wins when exception, replay, branch redirect, interrupt, and debug requests coincide?
- How is every stale response recognized after entry reuse?
- What formula justified every major queue depth and port count?
- Which invariant prevents double allocation, imprecise state, and deadlock?
- Which trace and counters distinguish frontend, dependency, execution, memory, and retirement limits?
- What is the smallest correct version, and which evidence authorizes each later optimization?

---

Next → [CPU Memory, Translation, and Coherence Implementation Blueprint](02_Memory_Translation_and_Coherence_Implementation_Blueprint.md)
