# System-on-Chip (SoC) and Chiplet Architecture › Transaction Protocols

> **Abbreviation key — skim now and return as needed:** central processing unit (CPU); Advanced eXtensible Interface (AXI); Advanced High-performance Bus (AHB); Advanced Peripheral Bus (APB); AXI Coherency Extensions (ACE);
> Coherent Hub Interface (CHI).

**Plain-language purpose:** Define how independently designed blocks exchange requests, data, responses, and backpressure without assuming they share one internal implementation.

## Terms introduced here

| Term | Meaning |
|---|---|
| Advanced Peripheral Bus (APB) | simple low-bandwidth peripheral protocol |
| Advanced High-performance Bus (AHB) | pipelined shared-bus protocol |
| Advanced eXtensible Interface (AXI) | multi-channel protocol supporting many outstanding transactions |
| handshake | sender and receiver jointly agree a transfer occurred |
| burst | several data beats associated with one address request |
| transaction identifier (ID) | tag used to match and order outstanding work |

## Reading order

1. [On-Chip Interconnect — AXI, AHB, APB](01_AHB_AXI_APB.md) — a single chapter covering first principles and complete designer deep dives: APB SETUP/ACCESS timing and peripheral RTL; AHB/AHB-Lite pipelining, `HREADY`, bursts, and arbitration; AXI channels, IDs, ordering, endpoint/fabric microarchitecture, bridges, verification, and waveform debug.

2. [The AMBA Family in Full — Signal Groups, Protocol Versions, and the Low-Power Interfaces](02_AMBA_Family_Signals_and_Low_Power_Interfaces.md) — everything around the core three that a working engineer needs: the family map and how to choose a member; **the AXI memory-attribute signals bit by bit** — `AxCACHE`, `AxPROT`, `AxLOCK`, `AxQOS`, `AxREGION`, `AxUSER` — and what a wrong value costs in correctness or performance; exclusive access and AXI5 atomics; narrow, unaligned, and WRAP bursts with the 4 KB rule; ordering, write interleaving, and the ID-reuse deadlock; AXI4-Stream as a genuinely different protocol; what each protocol version added and why; **the AMBA Low Power Interface — Q-Channel and P-Channel derived as state machines**, which is how a component and its power controller actually negotiate a state change; where AXI meets coherence via `AxDOMAIN`/`AxSNOOP`/`AxBAR`; ATB, DTI, and LTI; the protocol-checker assertion set; and an integration checklist.

**Hands off to:** CPU-owned [ACE and CHI](../../01_CPU_Architecture/06_Coherence_and_Consistency/03_ACE_and_CHI.md) for coherence concepts and full designer deep dives; [On-Chip Networks](../04_On_Chip_Networks/00_Index.md) for scalable transport.

---

[SoC and Chiplet Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
