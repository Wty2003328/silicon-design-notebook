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

1. [On-Chip Interconnect — AXI, AHB, APB](01_AHB_AXI_APB.md).

**Hands off to:** CPU-owned [ACE and CHI](../../01_CPU_Architecture/06_Coherence_and_Consistency/03_ACE_and_CHI.md) for coherence, and [On-Chip Networks](../04_On_Chip_Networks/00_Index.md) for scalable transport.

---

[SoC and Chiplet Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
