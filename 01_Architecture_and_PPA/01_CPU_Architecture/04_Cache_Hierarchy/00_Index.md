# Central Processing Unit (CPU) Architecture › Cache Hierarchy

> **Abbreviation key — skim now and return as needed:** average memory access time (AMAT); miss status holding register (MSHR); quality of service (QoS).

**Plain-language purpose:** Explain how a CPU keeps frequently used data close enough to avoid paying main-memory latency on every access.

## Terms introduced here

| Term | Meaning |
|---|---|
| cache line | fixed-size block copied between memory levels |
| hit / miss | requested line is present / must be fetched |
| average memory access time (AMAT) | weighted latency across hits and misses |
| miss-status holding register (MSHR) | tracks one outstanding cache miss and dependent requests |
| prefetch | fetches data before software explicitly requests it |
| quality of service (QoS) | policy that protects latency or bandwidth among requesters |

## Reading order

1. [Cache Microarchitecture](01_Cache_Microarchitecture.md) — addressing, lookup, nonblocking misses, writes, hierarchy.
2. [Prefetching, Replacement, and QoS](02_Prefetching_Replacement_and_QoS.md) — prediction, victim choice, pollution, feedback, fairness.

**Hands off to:** [Virtual Memory](../05_Virtual_Memory/00_Index.md) and [Coherence and Consistency](../06_Coherence_and_Consistency/00_Index.md).

---

[CPU Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
