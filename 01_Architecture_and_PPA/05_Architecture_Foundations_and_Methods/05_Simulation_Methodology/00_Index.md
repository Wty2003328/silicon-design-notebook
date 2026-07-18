# Architecture Foundations and Methods › Simulation Methodology

> **Abbreviation key — skim now and return as needed:** central processing unit (CPU); graphics processing unit (GPU); neural processing unit (NPU); system on chip (SoC); region of interest (ROI); executable and linkable format (ELF); instruction set architecture (ISA).

**Plain-language purpose:** Choose the least expensive model that can still distinguish the architecture decisions being compared, then validate it honestly.

## Terms introduced here

| Term | Meaning |
|---|---|
| abstraction | intentionally omits detail irrelevant to the question |
| discrete-event simulation | advances between timestamped events rather than every physical instant |
| region of interest (ROI) | execution interval selected for measurement |
| warm-up | establishes representative model state before recording results |
| analytical model | closed-form or algorithmic bound rather than event-by-event execution |
| error budget | allowed contribution of model and measurement uncertainties |
| input artifact | exact executable, trace, graph, or layer description consumed by a simulator |
| dynamic instruction | one runtime execution of a static instruction; a loop creates many instances |
| provenance | chain from result back through counters, configuration, artifact, compiler, input, and source |

## Reading order

1. [Simulation Methodology](01_Simulation_Methodology.md).
2. [Analytical Models](02_Analytical_Models.md).
3. [Benchmark to Results](03_Benchmark_to_Results_End_to_End.md) — source/graph, compiler, assembly or operators, loader, dynamic work, timing events, counters, formulas, and final reporting.

Then use the simulator page inside the CPU, GPU, NPU, or SoC book.

---

[Foundations and Methods](../00_Index.md) · [Architecture book](../../00_Index.md)
