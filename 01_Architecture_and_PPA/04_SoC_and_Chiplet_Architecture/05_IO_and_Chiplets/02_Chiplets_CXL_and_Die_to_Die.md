# Chiplets, Compute Express Link (CXL), and Die-to-Die Architecture

> **First-time reader orientation:** A chiplet is one silicon die intended to compose with others inside a package. Compute Express Link provides cache-coherent and memory-semantic protocols beyond a single die; Universal Chiplet Interconnect Express (UCIe) defines die-to-die connectivity. Partitioning trades yield, reuse, and modularity against link latency, energy, bandwidth, packaging, test, security, and failure recovery.

> **Abbreviation key — skim now and return as needed:** central processing unit (CPU); graphics processing unit (GPU); static random-access memory (SRAM); dynamic random-access memory (DRAM); high-bandwidth memory (HBM);
> virtual channel (VC); quality of service (QoS); direct memory access (DMA); AXI Coherency Extensions (ACE); Coherent Hub Interface (CHI);
> Universal Chiplet Interconnect Express (UCIe); Peripheral Component Interconnect Express (PCIe); die-to-die (D2D); reliability, availability, and serviceability (RAS); non-uniform memory access (NUMA);
> physical-layer interface (PHY); input/output (I/O); gigabyte (GB); kibibyte (KiB).

> **Prerequisites:** [Full-Chip Modeling](../01_System_Modeling/01_Full_Chip_Modeling.md), [ACE and CHI](../../01_CPU_Architecture/06_Coherence_and_Consistency/03_ACE_and_CHI.md), [Network on Chip](../04_On_Chip_Networks/01_Network_on_Chip.md), and [IC Packaging](../../../07_Manufacturing_and_Bringup/02_IC_Packaging.md).
> **Hands off to:** package implementation, signal/power integrity, thermal design, and system software. This page owns partitioning and link/protocol architecture.

---

## 0. Why this page exists

A chiplet system moves boundaries that were once internal wires onto standardized or proprietary die-to-die links. The move can improve yield, reuse, reticle scaling, technology specialization, and product composition. It also adds serialization, PHY power, protocol conversion, package dependencies, and new reset/RAS/security domains.

```mermaid
flowchart LR
    CPU["compute chiplet"] --> D2D0["D2D adapter + PHY"]
    GPU["accelerator chiplet"] --> D2D1["D2D adapter + PHY"]
    IO["I/O chiplet"] --> D2D2["D2D adapter + PHY"]
    D2D0 --> Pkg["package fabric / interposer / bridge"]
    D2D1 --> Pkg
    D2D2 --> Pkg
    Pkg --> Mem["HBM / memory chiplets"]
    CXL["off-package CXL fabric"] --> IO
    Mgmt["discovery, security, RAS, power"] --> D2D0
    Mgmt --> D2D1
    Mgmt --> D2D2
```

The architecture must decide **where semantics terminate**, not only how many gigabits cross the package.

## Before the details: a boundary converts wires into a protocol

Inside one die, nearby blocks may exchange wide signals with tight clocking and shared reset assumptions. Across a chiplet boundary, the design needs physical lanes, training, clock tolerance, flow control, error detection, retry, discovery, and reset sequencing. The boundary therefore adds latency and energy and creates new failure modes even when logical behavior is preserved.

Partitioning can improve yield by replacing one large die with smaller dies, reuse mature process nodes for analog or I/O functions, and enable several products from common components. The benefits depend on known-good-die test, package yield, substrate routing, thermal density, and the traffic placed across the boundary. Fine-grained feedback loops suffer most from added latency; bulk transfers tolerate it better.

**Beginner checkpoint:** quantify bytes per second, round-trip sensitivity, energy per bit, outstanding-credit requirement, and failure behavior for every cut edge. “Use chiplets” is not an architecture until the partition and link contract are explicit.

## 1. Why partition a die?

Partitioning benefits:

- yield improvement from smaller dies;
- reuse of validated I/O, compute, cache, and analog tiles;
- mixing process nodes (dense logic, SRAM, analog/PHY);
- scaling past reticle limits;
- product binning and configurable chiplet counts;
- independent development schedules and suppliers.

Costs:

- duplicated PHY/adapters, clocks, test, management, and keepout;
- higher latency/energy per crossing;
- package/interposer area, yield, and routing constraints;
- limited edge/bump bandwidth;
- cross-die coherence/directory complexity;
- known-good-die test and repair;
- thermal and power-delivery coupling;
- security/trust between dies.

The right partition cuts few latency-critical, high-bandwidth paths and many modular/technology-specific ones.

## 2. A first partition cost model

For boundary $b$ with traffic $T_b$ bytes/s, energy $E_b$ J/byte, and latency sensitivity coefficient $\lambda_b$, crossing cost is

$$
C_b=T_bE_b+\lambda_bL_b+ A_{adapter,b}+P_{idle,b}.
$$

Total value also includes die yield and reuse. With defect density $D_0$ and die area $A$, a simple Poisson yield is

$$
Y\approx e^{-D_0A}.
$$

Splitting reduces individual die area but package yield multiplies component/assembly yields and adds link/test overhead. More chiplets are not automatically cheaper.

Use traffic matrices from representative workloads. An average boundary traffic number hides bursts and coherence fanout that set queue/credit requirements.

## 3. Semantic partition choices

### 3.1 Protocol tunneling

Carry an existing on-die transaction/coherence protocol across the link. Benefits: preserves semantics and can make remote agents look local. Costs: link must carry fine-grained messages, ordering, retries, and virtual channels; latency may expose protocol assumptions designed for short wires.

### 3.2 Protocol termination and translation

Terminate on-die protocol at an adapter, then use a die-to-die transport and reconstruct transactions remotely. This localizes domains and supports heterogeneous chiplets, but bridges need reorder buffers, flow-control translation, error mapping, and precise completion semantics.

### 3.3 Message/software boundary

Partition at coarse queues, DMA, or software messages. It minimizes coherence and fine-grained latency coupling, but shifts data placement and synchronization to software/compiler.

Choose the coarsest boundary compatible with product latency and programmability.

## 4. UCIe stack as an architectural framework

Universal Chiplet Interconnect Express (UCIe) standardizes die-to-die physical/link capabilities and protocol mappings so chiplets can interoperate. Conceptually:

| Layer | Owns |
|---|---|
| protocol | PCIe/CXL or streaming/raw protocol semantics |
| die-to-die adapter | flitization, CRC/retry, ordering/flow mapping |
| physical | lanes, training, repair, clocking, electrical signaling |
| management/DFx | discovery, test, telemetry, reset, power, debug |

The exact feature set depends on specification version and package class. Architecture models should parameterize lane count, data rate, flit overhead, retry buffer, training, and repair rather than hard-coding a headline bandwidth.

Effective bandwidth:

$$
BW_{eff}=N_{lane}R_{lane}\eta_{encoding}\eta_{flit}\eta_{protocol}\eta_{retry}.
$$

Small coherence/control packets can have lower efficiency than large streaming transfers because headers/CRC/flow-control consume a larger fraction.

## 5. Link latency and credit sizing

Crossing latency includes source adapter, serialization, PHY/link, package propagation, destination adapter, and protocol reconstruction:

$$
L_{D2D}=L_{src}+L_{ser}+L_{phy}+L_{pkg}+L_{dst}+L_{protocol}.
$$

Round-trip coherence or load latency crosses twice and may visit a remote home/cache. Link replay after CRC errors adds a tail, so retry-buffer depth must cover the link round-trip bandwidth-delay product.

Credits need enough outstanding bytes to fill the pipe:

$$
C_{bytes}\gtrsim BW_{target}L_{credit}.
$$

Partition credits by virtual channel/protocol class to keep mandatory responses moving. A wide physical link can still underperform if transaction limits or adapter reorder entries are too small.

## 6. Coherence across chiplets

Options:

- one system-wide coherent domain with distributed homes/directories;
- coherent within compute chiplets, noncoherent/DMA across selected boundaries;
- hierarchical coherence: local directories summarized to a global level;
- memory-side coherence through a host/home agent;
- explicit software-managed sharing.

System-wide coherence improves programmability but makes remote latency, directory placement, snoop filtering, and failure recovery architectural. A hierarchical directory can track one bit per chiplet at the upper level and finer sharers locally, trading extra hop/lookup for scalable storage.

Coherence deadlock proof must include adapter queues and link VCs. A chiplet reset must not discard dirty owned data or outstanding acknowledgements silently.

### 6.1 Follow one coherent read across the boundary—and recover without completing it twice

Take a CPU on compute chiplet A that loads cache line `X`. `X` is absent locally, and the address-to-home map assigns its coherence home and attached memory to I/O/memory chiplet B. This one load crosses four different contracts: CPU/cache, coherence protocol, die-to-die transport, and physical link. Keeping their state and completion meanings separate is the key to a correct adapter.

**1. Admit the miss before transmitting it.** The local cache allocates a miss-status entry and requester transaction tag `T37`. That entry records at least `{line X, requesting core/load, requested permission, returned-beat mask, error state}`. The coherence requester creates a `ReadShared(X, requester=A, txn=T37)` message. If the miss table, request queue, or remote-link credits are exhausted, admission backpressures the cache; it must not emit a request it cannot later match.

**2. Convert protocol identity into transport identity.** Chiplet A's protocol adapter maps the coherence request class to a request virtual channel (VC), adds destination/home and ordering attributes, and packetizes it. The link layer then assigns link sequence number `S418`, consumes request-VC credit, calculates a cyclic redundancy check (CRC), and retains an exact copy in a **retry buffer**. `T37` survives end to end so the coherence response finds the cache miss. `S418` exists only between the two adjacent link adapters so a corrupted flit can be replayed. Confusing these tags leads either to duplicate cache transactions or to a retry buffer that can never retire.

**3. Receive or replay at link granularity.** Chiplet B checks framing and CRC before exposing the request to its coherence engine. If the check fails, B does not create a home transaction; it requests replay of `S418`. A resends the retained bytes, not a newly constructed coherence request. If the check passes, B accepts `S418` once, suppresses any duplicate sequence, advances its receive sequence, and acknowledges it. A may then release retry-buffer entry `S418`. Separately, B returns flow-control credit when the receive-buffer slot is actually reusable. An acknowledgment retires replay state; a credit grants future storage. Combining them is possible in an encoding, but their logical invariants remain different.

**4. Execute coherence at the remote home.** The B-side protocol adapter reconstructs `ReadShared(X,T37)` and allocates a home transaction entry. The directory lookup determines whether memory is authoritative or another cache owns modified data. If memory is current, the home reads it. If an owner holds dirty data, the home sends a snoop, waits for owner data and required acknowledgments, updates directory sharer/owner state, and only then forms the permitted data response. Link receipt of the request was therefore *not* architectural completion; coherence can still be waiting hundreds of cycles after `S418` was acknowledged.

**5. Return data under an independent progress path.** The response/data message carries `{requester=A, txn=T37, line X, permission/state, beat number, error/poison}` on response/data VCs. B's transmit adapter repeats the credit, sequence, CRC, and retry procedure in the reverse direction. Independent request and response credit pools are a liveness feature: a flood of new reads must not consume the storage needed by the data that releases their miss entries.

**6. Complete exactly once at the requester.** A verifies and accepts each return flit, reconstructs the coherence data response, and uses `T37` to update the original miss entry. Only after all required data beats and coherence conditions arrive may the cache install the line in the granted state, wake the CPU load, and free `T37`. A duplicate link flit is suppressed by link sequence state; a duplicate protocol completion is rejected because `T37` is no longer live. These are complementary defenses at different layers.

```mermaid
sequenceDiagram
    participant CPU as "CPU/cache A"
    participant AA as "coherence + D2D adapter A"
    participant AB as "D2D + coherence adapter B"
    participant H as "home/directory B"
    participant M as "memory or dirty owner"
    CPU->>AA: "miss X; allocate T37; ReadShared(X,T37)"
    Note over AA: "request VC credit--; assign S418; save retry copy; CRC"
    AA->>AB: "flit S418 containing txn T37"
    alt "CRC/framing error"
        AB-->>AA: "NAK/replay from S418; no coherence request created"
        AA->>AB: "replay identical S418"
    end
    AB-->>AA: "link ACK S418; credit when receive slot frees"
    AB->>H: "deliver ReadShared(X,T37) exactly once"
    H->>M: "directory lookup; memory read or snoop dirty owner"
    M-->>H: "data + required ownership response"
    H-->>AB: "Data(X,T37,state,beats)"
    Note over AB: "response/data VC credit--; new link sequence + retry copy"
    AB->>AA: "response flits with txn T37"
    AA-->>AB: "link ACK/credits"
    AA->>CPU: "match T37; install line; wake load; free T37"
```

The enabling state can be reviewed as a layered ledger:

| Layer/state owner | Minimum live metadata | Freed when |
|---|---|---|
| cache requester/miss entry | line, requester, `T37`, permission, beat mask, error | coherence data/completion conditions satisfied |
| source coherence adapter | opcode, home/destination, ordering class, protocol VN, `T37` | protocol handoff/response contract permits |
| D2D link transmitter | link sequence `S418`, exact flit copy, CRC, retry/ack state, VC | peer acknowledges valid receipt |
| D2D flow control | credits and in-flight accounting per VC | receiver declares buffer reusable |
| remote home transaction | line lock/transient directory state, requester/`T37`, snoop/ack/data masks | directory transition commits and response launches |
| destination reassembly | packet length/flit mask, poison/error, protocol class | complete message is delivered or aborted |

**Transient corruption uses replay; link-state loss uses recovery.** A CRC error leaves both adapters in the same session with retry/receive sequence state intact, so link-local replay is invisible to coherence apart from latency. A link-down event, peer reset, or lost retry/credit state is different: the receiver may no longer know whether `S418` was accepted. Blindly replaying or resetting credits can create a duplicate request or overwrite a full buffer. Recovery must be an explicit distributed state machine:

```mermaid
stateDiagram-v2
    [*] --> Serving
    Serving --> LinkReplay: "CRC error; session state intact"
    LinkReplay --> Serving: "replayed sequence acknowledged"
    Serving --> Quiescing: "link down / timeout / peer reset"
    Quiescing --> Training: "stop injection; snapshot or abort in-flight state"
    Training --> Reconcile: "lanes train/repair; exchange capabilities and epoch"
    Reconcile --> Serving: "same epoch and replay window safely reconciled"
    Reconcile --> ProtocolRecovery: "peer epoch changed or transport state lost"
    ProtocolRecovery --> Serving: "old transactions resolved; credits rebuilt; routes enabled"
```

During **quiescing**, stop new injection, mark the route unavailable, and retain retry/protocol tables. During **training**, repair or down-width lanes and re-establish electrical/link framing. During **reconciliation**, exchange a session epoch, accepted-sequence/replay-window state, and fresh credit baseline. Replay is safe only if both peers prove the same session and agree which sequences were accepted. If the peer epoch changed, raise a transport abort into the coherence layer: the requester/home tables must resolve each old transaction, roll back or finish transient directory state, and reissue only under a new transaction identity after the old one cannot complete. If an unreachable chiplet may own dirty data, recovery cannot invent a clean copy; policy must preserve the owner through reset, reach it over another route, poison/isolate the affected lines, or declare a fatal containment event.

**PPA and losing cases.** Retry depth follows link bandwidth × acknowledgment round trip; credit storage follows bandwidth × credit round trip; home/requester outstanding depth follows coherence bandwidth × full remote latency. These are three related but non-interchangeable windows. Deeper windows sustain throughput and tolerate bursts, but cost SRAM, leakage, sequence comparators, timers, muxing, and reset retention. More protocol VCs isolate request/response/data progress but multiply buffers and arbitration. A narrower repaired link preserves availability at reduced bandwidth and longer serialization; continuing at full injection rate merely moves congestion into adapters. Fine-grained false sharing, ownership ping-pong, or remote atomics can be latency-bound despite a link that reaches peak streaming bandwidth, so “GB/s passed” is not a sufficient partition result.

**Counters, fault injection, and assertions.** Measure protocol requests/completions by class, remote-home/dirty-owner cases, request-to-first-data and full completion tails, credits and retry-buffer high-water marks, credit/ack stalls separately, CRC/sequence errors, replays and replayed bytes, duplicate suppressions, lane repair/down-width time, link epochs, quiesce/retrain/reconcile latency, aborted/reissued transactions, and outstanding dirty ownership at faults. Assert:

- credits never underflow/overflow and `free + occupied + in_flight` matches declared capacity;
- an unacknowledged sequence retains an immutable retry copy, and an accepted sequence is delivered upward at most once;
- link ACK cannot complete `T37`; only the coherence response may free the miss entry;
- response/data traffic has a reachable, fairly served progress path independent of request pressure;
- a session-epoch change prevents ambiguous old replay and forces explicit protocol resolution;
- reset/link loss cannot silently discard a dirty owner, home transient, accepted request, or completion;
- every admitted coherent read eventually completes, reports poison/error, or enters a declared containment state under the platform's fault assumptions.

## 7. CXL: coherent and memory semantics off package

Compute Express Link (CXL) layers cache/memory protocols with PCIe-based discovery/I/O. Architecturally, devices can expose:

- accelerator functions with coherent access to host memory;
- device-attached memory accessed by the host;
- memory expansion/pooling through switches/fabrics;
- device caches participating under host-managed coherence rules.

The host typically remains a key coherence/management authority. CXL memory is not “slow DRAM with a different connector”; it has fabric/host bridge latency, device controllers, failure domains, poison/RAS, hot-plug/fabric management, and NUMA placement.

For tiered memory, break-even migration follows

$$
N_{reuse}(L_{remote}-L_{local}) > L_{copy}+L_{coherence}+L_{mapping}.
$$

Capacity-only data may remain remote; hot latency-sensitive data may migrate or be replicated under consistency constraints.

## 8. Memory pooling and sharing

Pooling raises utilization by assigning memory capacity dynamically among hosts/devices. Architecture questions:

- allocation granularity and address-map updates;
- fabric manager availability and failover;
- isolation/encryption/key ownership;
- bandwidth oversubscription and QoS;
- poison containment and error attribution;
- coherent versus exclusive ownership;
- migration/quiescence during reallocation;
- topology-aware NUMA placement.

A pooled capacity number is useless without bisection bandwidth and contention policy. Ten hosts cannot each receive peak device bandwidth simultaneously through one oversubscribed switch.

## 9. Clock, reset, power, and discovery

Each chiplet may have independent clocks/voltages/power states. Links need:

- clock-domain crossing and elastic buffering;
- training and lane repair after reset/power-up;
- ordered power-state entry/exit;
- retention or re-discovery of routing/configuration;
- timeout behavior when a peer disappears;
- firmware ownership and version compatibility;
- telemetry for link errors and margins.

Reset is a distributed protocol: stop injection, drain/cancel transactions, resolve dirty coherence state, reset/train link, rediscover capabilities, re-enable routing. Partial reset should not force a full package reset unless the architecture chooses that availability trade.

## 10. Security and trust

Multi-vendor or separately managed chiplets enlarge the trust boundary. Protect:

- identity/authentication and lifecycle state;
- configuration and debug access;
- memory/request address permissions;
- protocol conformance and malformed flits;
- denial of service through credits/priority;
- replay/injection on links;
- data confidentiality/integrity where threat model requires;
- side channels in shared caches/fabrics.

Adapters should validate protocol fields before allocating scarce downstream resources. Rate-limit faulting peers and contain errors to a declared domain.

## 11. Physical/package co-design

Logical topology is constrained by bumps and package routing. A full crossbar among many chiplets may be unroutable; ring/mesh/package switches trade hops against wires. PHY placement fixes die edges and competes with memory interfaces and power delivery.

Thermal gradients affect link timing and stacked memory. Power delivery must handle simultaneous switching on wide die-to-die interfaces. Package escape, microbump pitch, interposer reticle/stitching, bridges, and organic substrate choices change bandwidth density and cost.

Feed package estimates back into architecture before freezing chiplet shapes and link counts.

## 12. Evaluation checklist and counters

Evaluate:

- boundary traffic matrix, burst/fanout, and locality by workload;
- effective bandwidth and tail latency by packet class;
- adapter/credit/reorder occupancy;
- flit overhead and small-message efficiency;
- retry, lane repair, degraded-width performance;
- remote coherence/home transactions and directory storage;
- power/energy per transferred byte including adapters;
- package routing/yield/cost scenarios;
- reset/recovery time and fault containment;
- memory-tier placement/migration benefit.

Simulate protocol and traffic jointly. A link-level bandwidth test cannot reveal coherence serialization or home-node hot spots.

## 13. Numbers to remember

- Partition at boundaries with low latency-critical traffic and high reuse/technology value.
- Chiplet economics combine die yield, package yield, adapter overhead, test, and reuse—not die yield alone.
- Effective link bandwidth multiplies lane rate by encoding/flit/protocol/retry efficiency.
- Credit and retry storage follow bandwidth × round-trip latency.
- Coherence across a resettable/fallible boundary needs explicit dirty-state and transaction recovery.
- CXL memory is a NUMA/fabric tier with management, RAS, security, and contention semantics.

## 14. Worked problems

### Problem 1 — effective link bandwidth

Sixteen lanes each provide 32 Gb/s raw. Combined encoding/flit/protocol efficiency is 82%:

$$
BW=16\times32/8\times0.82=52.48\ \text{GB/s}
$$

per direction if the lanes are unidirectional as modeled. Transaction limits may reduce achieved rate further.

### Problem 2 — credit window

A 50 GB/s link has 80 ns credit round trip:

$$
C\ge50\times10^9\times80\times10^{-9}=4000\ \text{B}.
$$

Roughly 4 KiB of usable outstanding credit is required per fully utilized aggregate path, then partitioned/headroom-adjusted by traffic class.

### Problem 3 — hierarchical directory

An eight-chiplet system with eight cores/chiplet could track 64 core sharers globally. A hierarchical upper directory uses eight chiplet bits; only the owning local directory tracks eight core bits. Storage falls at the global level, but a request may add a local-directory lookup/hop.

## Cross-references

- **Protocols and coherence:** [ACE and CHI](../../01_CPU_Architecture/06_Coherence_and_Consistency/03_ACE_and_CHI.md), [Cache Coherence](../../01_CPU_Architecture/06_Coherence_and_Consistency/01_Cache_Coherence.md).
- **Transport/package:** [Network-on-Chip Architecture](../04_On_Chip_Networks/01_Network_on_Chip.md), [Routing, Flow Control, and Deadlock](../04_On_Chip_Networks/02_Routing_Flow_Control_and_Deadlock.md), [IC Packaging](../../../07_Manufacturing_and_Bringup/02_IC_Packaging.md).
- **Memory/system:** [HBM and Advanced Memory Systems](../../02_GPU_Architecture/02_Memory_System/02_HBM_and_Advanced_Memory_Systems.md), [Full-Chip Modeling](../01_System_Modeling/01_Full_Chip_Modeling.md).

## References

1. UCIe Consortium, [UCIe Specifications](https://www.uciexpress.org/specifications).
2. Compute Express Link Consortium, [CXL Specification](https://computeexpresslink.org/).
3. R. St. Amant et al., “Die-to-Die Interconnects and Chiplet-Based Systems,” IEEE Micro.
4. Arm, [Chiplet System Architecture overview](https://developer.arm.com/community/arm-community-blogs/b/architectures-and-processors-blog/posts/arm-a-profile-architecture-developments-2024).
5. [IC Packaging](../../../07_Manufacturing_and_Bringup/02_IC_Packaging.md) and its primary references.

---

**Navigation:** [System Fabrics index](00_Index.md) · [Interconnect index](../00_Index.md)
