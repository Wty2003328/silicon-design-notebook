# Part 4 · Architecture › Interconnect — Chapter Index

*Shared part: how blocks, cores, and chips talk — the load/store fabric, the coherence protocols, and the on-chip network that scales past a bus.*

Each chapter is concept-first — why the structure must exist, the mechanism, the trade-offs, then the derivations and worked numbers — closing with **Numbers to memorize** and **Cross-references**.

### [On-Chip Interconnect — AXI, AHB, APB as a Composition Contract](01_AHB_AXI_APB.md)

- §1  The problem a standard interconnect solves: composing IP
- §2  The valid/ready handshake: flow control as *whether*, not *when*
- §3  Why AXI is five independent channels
- §4  Outstanding, ID-tagged, out-of-order transactions: hiding latency
- §5  Bursts: amortizing the address phase
- §6  The three-tier family: AXI vs AHB vs APB as a PPA trade
- §7  Topology: the fabric as a separate, scalable concern
- §8  Bridges: everything reduces to buffering behind a handshake
- §9  Policy sidebands: QoS, security, and atomics
- §10  Numbers to memorize
- §11  Worked problems

### [ACE and CHI — Realizing Cache Coherence at the Interconnect](02_ACE_and_CHI.md)

- §1  What the fabric must provide that a lone cache cannot
- §2  Snoop coherence (ACE): broadcast the question
- §3  The $O(N^2)$ wall: why broadcast cannot scale
- §4  Directory coherence (CHI): the home node
- §5  The price of the directory: storage and indirection
- §6  Why CHI is layered, and why it rides a mesh
- §7  Off-chip coherence: why the story left the die (CXL)
- §8  The ordering point's other duties
- §9  ACE vs CHI: where real silicon lands
- §10  Numbers to memorize
- §11  Worked problems

### [Network-on-Chip — Why On-Chip Communication Becomes a Network](03_Network_on_Chip.md)

- §1  Why a bus and a crossbar both stop scaling
- §2  Topology: the diameter–bisection–cost trade
- §3  Flow control: sharing buffers without dropping flits
- §4  Routing and deadlock — the part with theorems
- §5  The router pipeline as a concept
- §6  Latency under load
- §7  The coherent mesh in practice (Arm CMN-class)
- §8  Physical design of the fabric
- §9  Numbers to memorize
- §10  Worked problems

---
⬅ [Architecture Book Contents](../00_Index.md) · [Root Index](../../Index.md) · [← Part 3 · Memory](../03_Memory/00_Index.md) · [Part 5 · GPU →](../05_GPU/00_Index.md)
