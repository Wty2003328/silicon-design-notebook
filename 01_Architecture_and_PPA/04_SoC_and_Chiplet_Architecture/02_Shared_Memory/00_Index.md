# System-on-Chip (SoC) and Chiplet Architecture › Shared Memory

> **Abbreviation key — skim now and return as needed:** dynamic random-access memory (DRAM); double data rate (DDR); atomic memory operation (AMO); Peripheral Component Interconnect Express (PCIe); Compute Express Link (CXL).

**Plain-language purpose:** Explain how many agents share dense off-chip main memory whose electrical rules require command timing and scheduling, and how they serialize against each other when they touch the same address.

## Terms introduced here

| Term | Meaning |
|---|---|
| double data rate (DDR) | transfers data on both clock edges |
| dynamic random-access memory (DRAM) | dense memory cell that leaks charge and needs refresh |
| bank | independently activated portion of DRAM |
| row buffer | sense-amplifier state holding one open row |
| refresh | periodic restoration of stored charge |
| memory controller | schedules legal commands and returns request data |
| mode register (MR) | on-device configuration written by an MRS command |
| training | the calibration that finds a sampling point the signal actually has |
| DDR PHY interface (DFI) | the standard boundary between a memory controller and its PHY |
| bank group | a set of banks sharing prefetch resources, so back-to-back access to one is slower |
| row hit / miss / conflict | the three outcomes of an access against the currently open row |
| bus turnaround | the dead time paid when the data bus reverses direction between a read and a write |
| address mapping | the assignment of physical-address bits to channel, rank, bank, row, and column |
| exclusive access | a load/store pair a monitor lets complete only if nothing else touched the address in between |
| exclusive monitor | the local or global state machine that watches that address and fails the store when the reservation is lost |
| far atomic | an atomic sent to the home node or memory to be executed there, instead of pulling the line to the requester |

## Reading order

1. [DDR Memory Controller](01_DDR_Controller.md) — the controller's job: the bank state machine, JEDEC timing derived as physics, row-buffer policy, FR-FCFS scheduling, refresh, and the achieved-bandwidth model.
2. [The DRAM Device Protocol and Training](02_DRAM_Device_Protocol_and_Training.md) — the wire-level layer the controller drives: the pin list, the command truth table, bank groups and $t_{CCD\_S}$ vs $t_{CCD\_L}$, prefetch and burst arithmetic, mode registers, the initialization sequence, **training in depth** (why it is mandatory, and every step in the order it must run), ZQ and ODT, DBI and CRC, command-level refresh, what DDR5 and LPDDR5X actually changed, the DFI controller-PHY boundary, and bring-up triage.

3. [Memory Scheduling and Address Mapping](03_Memory_Scheduling_and_Address_Mapping.md) — the controller's *policy*, where most of the achievable bandwidth is actually won or lost: the objective space and why throughput, latency, fairness, and energy conflict; FCFS and FR-FCFS with their failure modes; **read/write turnaround and write draining**, whose cost is one of the largest and least-discussed losses in a real controller; bank and rank parallelism against $t_{RC}$ and $t_{FAW}$; the multicore fairness family derived as a chain of repairs (STFM, PAR-BS, ATLAS, TCM, BLISS, staged memory scheduling) with the fairness metrics themselves; SoC QoS and deadline scheduling; **address mapping**, including the stride pathology and the XOR hash that fixes it; refresh as an optimization rather than a tax; command scheduling and energy; RowHammer as a technology problem with the PARA-TRR-RFM-PRAC mitigation ladder; the latency-reduction and in-DRAM-compute research families surveyed honestly; and how a scheduler is evaluated without fooling yourself.

4. [System Atomics and Exclusive Access](04_System_Atomics_and_Exclusive_Access.md) — the other thing shared memory has to arbitrate, once the agents sharing it are not all cores: the system problem statement and why an atomic is only as atomic as the weakest agent; locking the fabric as the baseline and why it fails; exclusive monitors, local and global, and where each physically sits; why the architecture *permits* spurious failure; the limits on a legal exclusive pair derived from the comparator; why contended exclusives collapse; **far atomics**, sending the operation to the data, and the arithmetic that says where the near/far crossover sits and where far atomics lose; what the home node must implement and must not skip; the ordering obligations an atomic creates in a fabric; PCIe AtomicOps; CXL, pooled memory, and what a die boundary does to the serialization point; DMA, IOMMU, and non-coherent masters; and how to choose and verify.

**Which first?** Read 01 if you want to know how memory requests are *scheduled*; read 02 first if you want to know what the pins actually do, or if you are bringing up a memory interface. 02 assumes the cell physics in [Memory Circuits and Technologies §10](../../../00_Fundamentals/06_Memory_Circuits_and_Technologies.md). 04 is independent of 01–03 and can be read on its own the moment two masters contend for one address; it assumes the transaction layer of [AHB, AXI, and APB](../03_Transaction_Protocols/01_AHB_AXI_APB.md), not DRAM timing.

**Hands off to:** [DRAM Simulation](../06_Simulation/00_Index.md) — which now carries a hands-on §11–§12 taking you from cloning DRAMsim3 or Ramulator to a defensible measured number — and [High-Speed I/O §8](../05_IO_and_Chiplets/03_High_Speed_IO_and_Peripheral_Protocols.md) for the PHY at circuit level. For the core side of 04 — what the instruction has to guarantee and what it costs in a pipeline — see [Atomic Operations](../../01_CPU_Architecture/06_Coherence_and_Consistency/04_Atomic_Operations.md); for the signal encodings it drives, [AMBA Family §3](../03_Transaction_Protocols/02_AMBA_Family_Signals_and_Low_Power_Interfaces.md).

---

[SoC and Chiplet Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
