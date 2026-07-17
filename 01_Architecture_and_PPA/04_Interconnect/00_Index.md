# Part 4 · Architecture › Interconnect

Interconnect is organized as three layers: endpoint transaction protocols, on-chip transport, and system-level policy/off-die fabrics.

```mermaid
flowchart LR
    Endpoint["masters / caches / devices"] --> Proto["AXI / ACE / CHI semantics"]
    Proto --> NoC["routing / VCs / routers / links"]
    NoC --> Fabric["QoS / ordering / I/O / chiplets"]
    Fabric --> Target["homes / memory / peer dies"]
```

## Subdomains

| Subdomain | Chapters | Boundary it owns |
|---|---:|---|
| [Protocols](01_Protocols/00_Index.md) | 2 | transaction and coherence messages visible at endpoints |
| [Network on Chip](02_Network_on_Chip/00_Index.md) | 2 | topology, routing, flow control, router timing and liveness |
| [System Fabrics](03_System_Fabrics/00_Index.md) | 2 | QoS/order/I/O coherence and off-die chiplet/CXL semantics |

## Chapter map

| Chapter | Primary ownership |
|---|---|
| [AHB, AXI, and APB](01_Protocols/01_AHB_AXI_APB.md) | channels, valid/ready, bursts, IDs, outstanding transactions and bridges |
| [ACE and CHI](01_Protocols/02_ACE_and_CHI.md) | snoop versus home/directory coherent message fabrics |
| [Network-on-Chip Architecture](02_Network_on_Chip/01_Network_on_Chip.md) | topology, router pipeline, latency/saturation and physical design |
| [Routing, Flow Control, and Deadlock](02_Network_on_Chip/02_Routing_Flow_Control_and_Deadlock.md) | channel dependencies, VCs/credits, adaptive escape and protocol liveness |
| [QoS, Ordering, and I/O Coherence](03_System_Fabrics/01_QoS_Ordering_and_IO_Coherence.md) | service contracts, arbitration, identities, device visibility and isolation |
| [Chiplets, CXL, and Die-to-Die](03_System_Fabrics/02_Chiplets_CXL_and_Die_to_Die.md) | partition economics, UCIe/CXL, credits, reset/RAS/security and package co-design |

## Reading paths

- **Build an SoC fabric:** AXI → NoC Architecture → Routing/Deadlock → QoS/I/O.
- **Build a coherent mesh:** Cache Coherence → CHI → NoC pair → QoS.
- **Partition into chiplets:** Full-Chip Modeling → Chiplets/CXL → IC Packaging.

---

⬅ [Memory](../03_Memory/00_Index.md) · [Architecture Contents](../00_Index.md) · next ➡ [GPU](../05_GPU/00_Index.md)
