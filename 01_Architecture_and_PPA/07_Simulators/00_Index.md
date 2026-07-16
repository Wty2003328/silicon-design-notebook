# 07 · Architecture › Simulators — Subfolder Index

*How the tools that produce every non-silicon number in this notebook actually work — conceptually, then tool by tool.*

This folder is the conceptual companion to the tool/fidelity ladder in [Full_Chip_Modeling](../01_Modeling/02_Full_Chip_Modeling.md) and the analytical kernels in [Performance_Modeling_and_DSE](../01_Modeling/01_Performance_Modeling_and_DSE.md). Those pages tell you *which tool* to use for a job; this folder tells you *how each tool models compute, memory, and time*, how a workload becomes a statistic, and where the error comes from.

| # | Page | Coverage |
|---|------|----------|
| 01 | [Simulation Methodology](01_Simulation_Methodology.md) | functional vs timing vs physical; the fidelity–speed ladder; the discrete-event engine; trace- vs execution-driven; ROI/warm-up/sampling (SimPoint, SMARTS); how compute and memory are modeled; validation and error budgets |
| 02 | [gem5](02_gem5.md) | SE vs FS mode; Atomic/Timing/Minor/O3 CPU models; Classic vs Ruby memory (SLICC); the event engine, SimObjects, ports and stats; fast-forward/checkpoints/KVM; validation & error vs silicon |
| 03 | [DRAM Simulators](03_DRAM_Simulators.md) | Ramulator 1.0/2.0, DRAMSim3, DRAMPower, USIMM — bank/rank/channel state machines, JEDEC timing, FR-FCFS scheduling, row-buffer policy; achieved BW/latency as contention outputs; energy from IDD × time-in-state |
| 04 | [GPU Simulators](04_GPU_Simulators.md) | GPGPU-Sim / Accel-Sim — the SIMT timing model, warp scheduling, memory coalescing and the L2/HBM contention layer; trace vs execution mode; GPUWattch/AccelWattch power; validation vs silicon |
| 05 | [Accelerator & DNN/NPU Simulators](05_Accelerator_and_NPU_Simulators.md) | analytical mapping/dataflow (Timeloop + Accelergy, MAESTRO), cycle-accurate systolic (SCALE-Sim v3), operator-level industrial (NeuSim, ONNXim); the dataflow taxonomy and how a layer + mapping becomes latency and energy |
| 06 | [Other Architecture Simulators](06_Other_Architecture_Simulators.md) | surveyed catalog: manycore/CPU (Sniper, ZSim, MARSSx86), NoC (BookSim, Garnet, DSENT), CIM/neuromorphic (NeuroSim), datacenter/HPC (SST, ns-3, OMNeT++), chiplet/2.5D-3D (CHIPSIM, HotSpot), RISC-V (Spike, Dromajo, Verilator-RTL, gem5) |
| 07 | [Analytical Models](07_Analytical_Models.md) | roofline (ridge point, ceilings, cache-aware), the interval/mechanistic (Karkhanis–Smith) model, Amdahl with a contention term (USL), Little's-law/M-M-1 queueing, and LogCA/LogGP — the closed-form duals of the simulators above |

---

⬅ [Architecture & PPA index](../00_Index.md) · [Root Index](../../Index.md) · [Flow Overview](../../Chip_Design_Flow_Overview.md) · next ➡ [Simulation Methodology](01_Simulation_Methodology.md)
