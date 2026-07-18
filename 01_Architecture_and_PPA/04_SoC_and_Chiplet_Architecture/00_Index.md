# System-on-Chip (SoC) and Chiplet Architecture

> **Abbreviation key — skim now and return as needed:** central processing unit (CPU); graphics processing unit (GPU); neural processing unit (NPU); dynamic random-access memory (DRAM); double data rate (DDR);
> network on chip (NoC); quality of service (QoS); Advanced eXtensible Interface (AXI); Advanced High-performance Bus (AHB); Advanced Peripheral Bus (APB);
> Compute Express Link (CXL); Universal Chiplet Interconnect Express (UCIe); input/output (I/O).

A system on chip composes CPU, GPU, NPU, memory controllers, peripherals, and shared communication into one product. A chiplet design extends that composition across multiple silicon dies in one package. This book owns the structures whose behavior is defined by multiple agents rather than one processor family.

> **First-time reader:** Read the [Architecture Primer](../05_Architecture_Foundations_and_Methods/01_Reader_Foundations/01_Architecture_Primer_and_Glossary.md), then at least one compute-architecture overview. A bus, network, or memory controller only makes sense once you know what requests its endpoints generate.

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
~~~

## Subdomains

| Order | Subdomain | Chapters | What it owns |
|---:|---|---:|---|
| 1 | [System Modeling](01_System_Modeling/00_Index.md) | 1 | composing block behavior, contention, power, thermal limits |
| 2 | [Shared Memory](02_Shared_Memory/00_Index.md) | 1 | DDR commands, scheduling, refresh, errors, delivered bandwidth |
| 3 | [Transaction Protocols](03_Transaction_Protocols/00_Index.md) | 1 | APB/AHB/AXI channels, handshakes, IDs, bursts, bridges |
| 4 | [On-Chip Networks](04_On_Chip_Networks/00_Index.md) | 2 | topology, routers, routing, flow control, deadlock/liveness |
| 5 | [I/O and Chiplets](05_IO_and_Chiplets/00_Index.md) | 2 | end-to-end QoS/order, device visibility, CXL/UCIe partitioning |
| 6 | [SoC Memory Simulation](06_Simulation/00_Index.md) | 1 | executable DRAM timing, scheduling, and power models |

## Ownership boundary

CPU cache coherence remains in [CPU Architecture](../01_CPU_Architecture/06_Coherence_and_Consistency/00_Index.md), because its states and ordering rules arise from CPU caches and the CPU memory model. This book begins where several architecture-owned agents share transport, memory, I/O, packaging, or product-level budgets.

---

← [NPU Architecture](../03_NPU_Architecture/00_Index.md) · [Architecture book](../00_Index.md) · [Foundations and Methods](../05_Architecture_Foundations_and_Methods/00_Index.md) →
