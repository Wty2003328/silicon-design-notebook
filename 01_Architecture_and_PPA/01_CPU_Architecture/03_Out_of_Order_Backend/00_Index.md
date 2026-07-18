# Central Processing Unit (CPU) Architecture › Out-of-Order Backend

> **Abbreviation key — skim now and return as needed:** out-of-order (OoO); reorder buffer (ROB); load-store queue (LSQ).

**Plain-language purpose:** Show how a CPU executes ready work early while preserving the illusion that instructions completed one at a time in program order.

## Terms introduced here

| Term | Meaning |
|---|---|
| out-of-order (OoO) execution | ready younger instructions may execute before stalled older ones |
| register renaming | maps software register names to a larger pool to remove false dependencies |
| reorder buffer (ROB) | holds speculative instructions until they can retire in order |
| load-store queue (LSQ) | tracks unfinished memory operations and their ordering |
| precise state | architectural state corresponds to one exact instruction boundary |

## Reading order

1. [Out-of-Order Execution](01_OoO_Execution.md) — rename, scheduling, execution, and window sizing.
2. [Load-Store Unit and Memory Ordering](02_Load_Store_Unit_and_Memory_Ordering.md) — address speculation, forwarding, violations, and replay.
3. [Retirement, Recovery, and Precise State](03_Retirement_Recovery_and_Precise_State.md) — commit, exceptions, branch recovery, and stale-response safety.

**Hands off to:** [Cache Hierarchy](../04_Cache_Hierarchy/00_Index.md) and [Coherence and Consistency](../06_Coherence_and_Consistency/00_Index.md).

---

[CPU Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
