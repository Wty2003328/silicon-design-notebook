# Architecture Foundations and Methods

> **Abbreviation key — skim now and return as needed:** central processing unit (CPU); graphics processing unit (GPU); neural processing unit (NPU); system on chip (SoC); register-transfer level (RTL);
> power, performance, and area (PPA).

This is the shared toolbox, not a fifth chip type. It contains only concepts that retain the same meaning across CPU, GPU, NPU, and SoC work: beginner vocabulary, workload/performance analysis, early power-performance-area estimation, reusable memory structures, simulation methodology, and tool selection.

> **First-time reader:** Start with [Architecture Primer — A First Map of the Machine](01_Reader_Foundations/01_Architecture_Primer_and_Glossary.md). It expands the core abbreviations and explains latency, throughput, queues, caches, translation, coherence, and PPA in plain language.

~~~mermaid
flowchart LR
    Primer["reader foundations"] --> Work["workload + performance"]
    Work --> PPA["early PPA + uncertainty"]
    Primer --> Struct["memory structures"]
    Work --> Method["simulation methodology"]
    Method --> Tools["tool landscape"]
    PPA --> Decision["architecture decision"]
    Tools --> Decision
~~~

## Subdomains

| Order | Subdomain | Chapters | Use it to answer |
|---:|---|---:|---|
| 1 | [Reader Foundations](01_Reader_Foundations/00_Index.md) | 1 | What do the basic concepts, units, and abbreviations mean? |
| 2 | [Performance Analysis](02_Performance_Analysis/00_Index.md) | 2 | Which workloads matter, and which design performs better? |
| 3 | [PPA Estimation](03_PPA_Estimation/00_Index.md) | 1 | What power, performance, and area range is credible before RTL? |
| 4 | [Hardware Structures](04_Hardware_Structures/00_Index.md) | 1 | What circuit structures store architecture state? |
| 5 | [Simulation Methodology](05_Simulation_Methodology/00_Index.md) | 3 | How does source become a simulator artifact, dynamic events, raw counters, and a defensible result? |
| 6 | [Tool Landscape](06_Tool_Landscape/00_Index.md) | 1 | Which specialized simulator family should we investigate? |

## How to use this book

Do not read it as a detached tools catalog. Start with a product question, choose the relevant chip architecture, then use these methods to characterize the workload, estimate tradeoffs, select fidelity, and report uncertainty.

---

← [SoC and Chiplet Architecture](../04_SoC_and_Chiplet_Architecture/00_Index.md) · [Architecture book](../00_Index.md) · [Power and Low-Power](../../02_Power_and_Low_Power/00_Index.md) →
