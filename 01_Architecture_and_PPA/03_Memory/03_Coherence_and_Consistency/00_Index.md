# Coherence and Consistency

This subdomain separates the distributed permission protocol from the ordering contract exposed to software.

| Chapter | Owns |
|---|---|
| [Cache Coherence](01_Cache_Coherence.md) | stable/transient states, directories, races, safety/liveness, verification |
| [Memory Consistency and Atomics](02_Memory_Consistency_and_Atomics.md) | ordering models, litmus tests, fences, atomicity, speculation constraints |

Read both: coherence answers *which copy is authoritative*; consistency answers *which observation orders are legal*.

**Up:** [Memory](../00_Index.md) · [Architecture + PPA](../../00_Index.md)
