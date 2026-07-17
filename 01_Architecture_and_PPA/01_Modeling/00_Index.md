# Part 1 · Architecture › Modeling

Modeling begins with the workload contract, climbs through analytical/cycle fidelity, estimates early PPA with explicit uncertainty, then composes blocks into a coupled chip model.

```mermaid
flowchart LR
    Product["product workload + SLO"] --> Char["characterize + sample"]
    Char --> Perf["performance model + DSE"]
    Perf --> PPA["early PPA + uncertainty"]
    PPA --> Chip["full-chip composition"]
    Chip --> Decision["robust Pareto decision"]
```

## Subdomains

| Subdomain | Chapters | Use it to answer |
|---|---:|---|
| [Performance Analysis](01_Performance_Analysis/00_Index.md) | 2 | Which workload behaviors matter, which regions represent them, and which architecture wins? |
| [System and PPA](02_System_and_PPA/00_Index.md) | 2 | How do block estimates compose, and how confident should the team be? |

## Chapter map

| Chapter | Primary ownership |
|---|---|
| [Performance Modeling and DSE](01_Performance_Analysis/01_Performance_Modeling_and_DSE.md) | fidelity ladder, CPI/roofline bounds, architecture search and PPA objective |
| [Workload Characterization and Sampling](01_Performance_Analysis/02_Workload_Characterization_and_Sampling.md) | workload contracts, phase clustering, warm-up, confidence and aggregation |
| [Full-Chip Modeling](02_System_and_PPA/01_Full_Chip_Modeling.md) | hierarchical composition, contention, overlap, power/thermal fixed points |
| [Early PPA Estimation and Uncertainty](02_System_and_PPA/02_Early_PPA_Estimation_and_Uncertainty.md) | structure proxies, calibration, sensitivity, intervals and risk-aware decisions |

## Reading paths

- **Start a new architecture study:** workload characterization → performance/DSE → early PPA → full-chip composition.
- **Audit a suspicious result:** workload contract → warm-up/aggregation → calibration domain → uncertainty contributors.
- **Choose the next model investment:** run sensitivity × uncertainty, then prototype the largest decision risk.

---

⬅ [Architecture Contents](../00_Index.md) · [Root Index](../../Index.md) · next ➡ [CPU](../02_CPU/00_Index.md)
