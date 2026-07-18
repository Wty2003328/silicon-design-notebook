# Neural Processing Unit (NPU) Architecture › Simulation

> **Abbreviation key — skim now and return as needed:** processing element (PE); deep neural network (DNN).

**Plain-language purpose:** Turn an NPU design and tensor mapping into estimated cycles, traffic, utilization, area, and energy.

## Terms introduced here

| Term | Meaning |
|---|---|
| deep neural network (DNN) | layered numerical model being accelerated |
| ONNX graph | portable operator/tensor representation exported from a framework |
| lowering | converts graph nodes into supported operators, loop bounds, or GEMM dimensions |
| mapping | assignment of tensor loops/data to time, PEs, and memory levels |
| cycle model | estimates when modeled operations occur |
| energy model | assigns energy cost to compute, storage, and movement events |
| design-space exploration | evaluates many legal hardware/mapping combinations |

## Reading order

1. Read the NPU-specific [Graph-to-Results Workflow](../00_Design_Methodology/03_NPU_Simulation_Methodology_and_Evidence.md).
2. [Accelerator and NPU Simulators](01_Accelerator_and_NPU_Simulators.md) — framework/ONNX graph → optimized operator list → loop/GEMM dimensions → mapping → cycles/access counts/energy in SCALE-Sim, Timeloop, Accelergy, MAESTRO, NeuSim, and ONNXim.

**Prerequisite:** [NPU Simulation Methodology and Evidence](../00_Design_Methodology/03_NPU_Simulation_Methodology_and_Evidence.md).

---

[NPU Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
