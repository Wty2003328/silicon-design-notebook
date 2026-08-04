# System-on-Chip (SoC) and Chiplet Architecture › Input/Output (I/O) and Chiplets

> **Abbreviation key — skim now and return as needed:** quality of service (QoS); Compute Express Link (CXL); Universal Chiplet Interconnect Express (UCIe).

**Plain-language purpose:** Extend shared-memory and service guarantees across device and die boundaries.

## Terms introduced here

| Term | Meaning |
|---|---|
| input/output (I/O) | communication with devices outside a processor core |
| quality of service (QoS) | policy for latency, bandwidth, priority, or fairness |
| chiplet | one die designed to compose with other dies in a package |
| accelerator scale-up | low-latency peer-memory/atomic/collective fabric among GPUs or NPUs |
| link replay | hop-local retransmission after CRC/integrity failure |
| Compute Express Link (CXL) | cache-coherent and memory-semantic link protocol |
| Universal Chiplet Interconnect Express (UCIe) | die-to-die physical/protocol standard |
| serializer/deserializer (SerDes) | circuit pair that converts a parallel word to a serial line and back |
| clock and data recovery (CDR) | recovering the sampling clock from the data transitions themselves |
| inter-symbol interference (ISI) | one symbol's energy smearing into the next through a lossy channel |
| transaction layer packet (TLP) | the PCIe unit that carries a read, a write, or a completion |
| link training and status state machine (LTSSM) | the state machine that brings a serial link from electrical idle to L0 |
| base address register (BAR) | an endpoint register that requests and receives an address window |
| pooling | several hosts or devices share provisioned memory capacity |

## Reading order

1. [QoS, Ordering, and I/O Coherence](01_QoS_Ordering_and_IO_Coherence.md).
2. [Chiplets, CXL, and Die-to-Die](02_Chiplets_CXL_and_Die_to_Die.md) — same-package UCIe/D2D, CXL host/device fabrics, and accelerator scale-up endpoints/switches with remote memory, atomics, software coherence, collectives, and recovery.
3. [High-Speed I/O and Peripheral Protocols](03_High_Speed_IO_and_Peripheral_Protocols.md) — the interfaces that actually leave the package: why serial beat parallel, the SerDes and its CDR, line coding, channel loss and TX FFE / CTLE / DFE equalization, NRZ vs PAM4, the one layered stack shared by PCIe, USB, and Ethernet, the DDR PHY and why it must train, and the low-speed family every SoC has — I2C, SPI, UART, GPIO, timers, and the interrupt controller — plus the pad/package boundary and how an external interface is verified and brought up.
4. [PCI Express — the Protocol Deep Dive](04_PCIe_Protocol_Deep_Dive.md) — the dedicated PCIe page, a peer of the AXI and CHI deep dives: topology and the programming model, configuration space and enumeration with worked BAR sizing, the three layers, TLP headers field by field, sequence numbers/LCRC/ACK-NAK and the replay buffer, the six flow-control credit types derived from a round-trip, the full ordering table, the LTSSM and equalization phases, per-generation bandwidth arithmetic with Max Payload Size, MSI/MSI-X, ATS/PRI/PASID and SR-IOV, the error taxonomy and AER triage, ASPM, how an accelerator actually uses the link, and endpoint bring-up.

**Hands off to:** packaging, signal/power integrity, thermal design, firmware, and system software.

---

[SoC and Chiplet Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
