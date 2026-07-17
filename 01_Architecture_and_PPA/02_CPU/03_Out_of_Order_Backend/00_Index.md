# Out-of-Order CPU Backend

This subdomain owns the speculative dataflow engine behind the in-order architectural façade.

| Chapter | Owns |
|---|---|
| [Out-of-Order Execution](01_OoO_Execution.md) | renaming, ROB, issue, scheduling, window sizing |
| [Load-Store Unit and Memory Ordering](02_Load_Store_Unit_and_Memory_Ordering.md) | address generation, disambiguation, forwarding, replay, ordering machinery |
| [Retirement, Recovery, and Precise State](03_Retirement_Recovery_and_Precise_State.md) | commit, exceptions, branch recovery, machine clears, checkpoint design |

**Up:** [CPU](../00_Index.md) · [Architecture + PPA](../../00_Index.md)
