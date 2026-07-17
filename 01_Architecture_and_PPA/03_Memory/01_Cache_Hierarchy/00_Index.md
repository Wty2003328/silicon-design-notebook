# Cache Hierarchy

This subdomain owns cache organization, miss concurrency, replacement, prefetch, and shared-cache resource control.

| Chapter | Owns |
|---|---|
| [Cache Microarchitecture](01_Cache_Microarchitecture.md) | hit path, associativity, MSHRs, writes, hierarchy and AMAT |
| [Prefetching, Replacement, and QoS](02_Prefetching_Replacement_and_QoS.md) | prediction mechanisms, pollution control, insertion policy, partitioning and fairness |

Coherence is intentionally separate because it is a distributed correctness protocol, not a cache-local policy.

**Related:** [Coherence and Consistency](../03_Coherence_and_Consistency/00_Index.md) · **Up:** [Memory](../00_Index.md)
