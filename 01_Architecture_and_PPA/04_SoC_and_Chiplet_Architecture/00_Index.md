# System-on-Chip (SoC) and Chiplet Architecture

> **Abbreviation key — skim now and return as needed:** central processing unit (CPU); graphics processing unit (GPU); neural processing unit (NPU); dynamic random-access memory (DRAM); double data rate (DDR);
> network on chip (NoC); quality of service (QoS); Advanced eXtensible Interface (AXI); Advanced High-performance Bus (AHB); Advanced Peripheral Bus (APB);
> Compute Express Link (CXL); Universal Chiplet Interconnect Express (UCIe); input/output (I/O).

A system on chip composes CPU, GPU, NPU, memory controllers, peripherals, and shared communication into one product. A chiplet design extends that composition across multiple silicon dies in one package. This book owns the structures whose behavior is defined by multiple agents rather than one processor family.

> **First-time reader:** Read [SoC and Chiplet Workloads, Performance Modeling, and Design-Space Exploration](00_Design_Methodology/01_SoC_Chiplet_Workloads_Performance_and_DSE.md), then at least one compute-architecture overview. A bus, network, or memory controller only makes sense once you know what requests its endpoints generate.
>
> **AI-serving path:** after one compute architecture, read [AI Workloads and Serving](07_AI_Workloads_and_Serving/00_Index.md) to trace model provisioning and one request across CPU, accelerator, HBM/DDR, NoC, chiplet links, NIC, and storage, then derive TTFT, TPOT, goodput, communication, capacity, and tail-latency bounds.

~~~mermaid
flowchart LR
    CPU["CPU subsystem"] --> Proto["transaction protocols"]
    GPU["GPU subsystem"] --> Proto
    NPU["NPU subsystem"] --> Proto
    Proto --> NoC["network on chip"]
    NoC --> DDR["shared DDR memory"]
    NoC --> IO["I/O + chiplet links"]
    Model["full-chip model"] -. budgets .-> CPU
    Model -. budgets .-> GPU
    Model -. budgets .-> NPU
    Build["implementation blueprints"] -. specifies .-> Proto
    Build -. specifies .-> NoC
~~~

## Subdomains

| Order | Subdomain | Chapters | What it owns |
|---:|---|---:|---|
| 0 | [SoC/Chiplet Design Methodology](00_Design_Methodology/00_Index.md) | 3 | concurrent use-case contracts, shared-resource DSE, chip/package PPA, and composed simulation evidence |
| 1 | [System Modeling](01_System_Modeling/00_Index.md) | 1 | composing block behavior, contention, power, thermal limits |
| 2 | [Shared Memory](02_Shared_Memory/00_Index.md) | 1 | DDR commands, scheduling, refresh, errors, delivered bandwidth |
| 3 | [Transaction Protocols](03_Transaction_Protocols/00_Index.md) | 1 | APB/AHB/AXI channels, handshakes, IDs, bursts, bridges |
| 4 | [On-Chip Networks](04_On_Chip_Networks/00_Index.md) | 2 | topology, routers, routing, flow control, deadlock/liveness |
| 5 | [I/O and Chiplets](05_IO_and_Chiplets/00_Index.md) | 2 | end-to-end QoS/order, device visibility, CXL/UCIe partitioning |
| 6 | [SoC Memory Simulation](06_Simulation/00_Index.md) | 1 | executable DRAM timing, scheduling, and power models |
| 7 | [AI Workloads and Serving](07_AI_Workloads_and_Serving/00_Index.md) | 6 | inference/mapping/research plus heterogeneous AI control plane, distributed data/state plane, validation, operations, and deployment |
| 8 | [Implementation Blueprints](08_Implementation_Blueprints/00_Index.md) | 3 | address/protocol/memory, NoC/QoS/I/O/chiplet, and full-chip verification/bring-up specifications |

## Ownership boundary

CPU cache coherence remains in [CPU Architecture](../01_CPU_Architecture/06_Coherence_and_Consistency/00_Index.md), because its states and ordering rules arise from CPU caches and the CPU memory model. This book begins where several architecture-owned agents share transport, memory, I/O, packaging, or product-level budgets.

---

← [NPU Architecture](../03_NPU_Architecture/00_Index.md) · [Architecture book](../00_Index.md) · [SoC/Chiplet Design Methodology](00_Design_Methodology/00_Index.md)
