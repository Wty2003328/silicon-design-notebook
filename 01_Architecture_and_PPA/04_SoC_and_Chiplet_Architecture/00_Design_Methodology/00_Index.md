# SoC and Chiplet Architecture › Design Methodology

> **Purpose:** Define complete-system workloads, compose heterogeneous engines and shared resources, price wires/memory/PHY/package costs, and validate transaction-to-application behavior before studying individual protocols and fabrics.

## Terms introduced here

| Term | Meaning in SoC/chiplet design |
|---|---|
| use-case contract | concurrent software, compute engines, traffic, QoS/SLO, power modes, and external interfaces active in a product scenario |
| critical path | dependency/resource path that determines transaction, frame, request, or job completion |
| offered load | requests/bytes presented to a shared resource before its service limitation |
| delivered bandwidth | useful completed bytes per time after protocol, contention, refresh, and turnaround losses |
| composition | coupling validated component models through explicit transaction, timing, clock, power, and state boundaries |
| trace point | exact hierarchy/interface boundary at which requests are recorded |

## Reading order

1. [SoC/Chiplet Workloads, Performance Modeling, and DSE](01_SoC_Chiplet_Workloads_Performance_and_DSE.md).
2. [SoC/Chiplet PPA and Physical Implementation](02_SoC_Chiplet_PPA_and_Physical_Implementation.md).
3. [SoC/Chiplet Simulation Methodology and Evidence](03_SoC_Chiplet_Simulation_Methodology_and_Evidence.md).

Then continue to [System Modeling](../01_System_Modeling/00_Index.md).

---

← [SoC/chiplet book index](../00_Index.md) · next → [SoC/Chiplet Workloads, Performance Modeling, and DSE](01_SoC_Chiplet_Workloads_Performance_and_DSE.md)
