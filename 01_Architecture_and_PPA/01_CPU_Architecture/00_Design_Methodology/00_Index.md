# CPU Architecture › Design Methodology

> **Purpose:** Define the CPU workload, quantify the bottleneck, price the physical structures, and choose evidence strong enough to support the decision before studying individual pipeline blocks.

## Terms introduced here

| Term | Meaning in CPU design |
|---|---|
| workload contract | exact programs, inputs, software stack, operating conditions, and success metrics the CPU must satisfy |
| dynamic instruction | one runtime execution of a machine instruction; a loop creates many dynamic instances of the same static instruction |
| cycles per instruction (CPI) | average target-CPU cycles divided by retired architectural instructions |
| design-space exploration (DSE) | controlled comparison of widths, depths, predictors, caches, clocks, and policies |
| physical resource ledger | explicit inventory of arrays, queues, ports, wires, clock load, power, and area created by a feature |
| provenance | chain from a reported result back to counters, simulator configuration, binary, compiler, input, and source |

## Reading order

1. [CPU Workloads, Performance Modeling, and DSE](01_CPU_Workloads_Performance_and_DSE.md) — turn product behavior into measurable CPU requirements and a bounded experiment.
2. [CPU PPA and Physical Implementation](02_CPU_PPA_and_Physical_Implementation.md) — translate predictors, windows, register files, caches, and clock targets into power, timing, and area consequences.
3. [CPU Simulation Methodology and Evidence](03_CPU_Simulation_Methodology_and_Evidence.md) — follow source and input through an executable, dynamic instructions, timing events, raw counters, validation, and final results.

Then continue to [Core Foundations](../01_Core_Foundations/00_Index.md). Later chapters return to these methods when making a specific frontend, backend, cache, translation, coherence, or simulation decision.

---

← [CPU book index](../00_Index.md) · next → [CPU Workloads, Performance Modeling, and DSE](01_CPU_Workloads_Performance_and_DSE.md)
