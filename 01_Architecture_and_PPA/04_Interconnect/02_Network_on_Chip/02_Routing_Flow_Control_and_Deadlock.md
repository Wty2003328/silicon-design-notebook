# Routing, Flow Control, and Deadlock — Proving the NoC Continues to Move

> **Prerequisites:** [Network-on-Chip Architecture](01_Network_on_Chip.md) (topology, router pipeline, latency), [ACE and CHI](../01_Protocols/02_ACE_and_CHI.md) (coherence message classes), and basic graph theory.
> **Hands off to:** [NoC and Coherence Simulation](../../07_Simulators/03_Memory_and_Interconnect/02_NoC_and_Coherence_Simulation.md) for measurement and [QoS, Ordering, and I/O Coherence](../03_System_Fabrics/01_QoS_Ordering_and_IO_Coherence.md) for policy.

---

## 0. Why this page exists

A NoC can pass every zero-load latency test and still fail permanently under one unlucky combination of requests. Correctness requires proving that packet dependencies cannot form an unbreakable cycle and that mandatory responses can eventually obtain buffers, credits, routes, and arbitration.

```mermaid
flowchart LR
    In["input flits"] --> RC["route compute"]
    RC --> VA["VC allocation"]
    VA --> SA["switch allocation"]
    SA --> X["crossbar"]
    X --> Link["physical link"]
    Credit["downstream credits"] --> SA
    VC["virtual channels"] --> VA
    Escape["deadlock-free escape VC"] --> VA
```

Routing chooses permissible channels; flow control assigns finite storage over time; deadlock analysis proves those choices cannot wait cyclically forever.

## 1. Packets, flits, and wormhole dependency

A packet is divided into flits. In wormhole switching, the head flit acquires channels while trailing flits occupy buffers along the path. If the head blocks, the packet can hold several upstream channels, creating a chain of dependencies.

For packet size $P$ bits, flit width $F$ bits, $H$ hops, router latency $L_r$, and link latency $L_l$, zero-load latency approximates

$$
L_0=H(L_r+L_l)+\left\lceil\frac{P}{F}\right\rceil-1.
$$

Under load add queueing and arbitration. Wider flits reduce serialization but enlarge crossbars, buffers, and links; more buffering absorbs bursts but increases area/leakage and can worsen congestion spreading.

## 2. Credit-based flow control

Each upstream virtual channel (VC) tracks free slots downstream. Sending decrements credit; downstream buffer release eventually returns credit. Correct credit accounting invariant:

$$
0\le credit+inflight+occupied=C_{buffer}
$$

with the exact partition depending on where link pipeline slots are counted.

Round-trip credit latency $L_c$ sets a bandwidth-delay product. To sustain one flit/cycle on a VC without credit bubbles,

$$
B_{VC}\gtrsim L_c
$$

flit slots unless multiple VCs interleave. Over-provisioning buffers can hide credit latency but not downstream service shortage.

On/off flow control uses thresholds rather than per-flit credits, reducing control traffic but requiring headroom for in-flight flits after stop is asserted.

## 3. Virtual channels separate waiting classes

Several logical VCs share one physical link but own independent buffer/allocator state. VCs serve three roles:

1. break protocol/routing dependency cycles;
2. prevent one blocked packet from head-of-line blocking unrelated traffic;
3. implement traffic classes/QoS.

One VC per traffic class is not automatically deadlock-free. The proof depends on how routes transition among VCs and how request/response protocol dependencies map onto them.

VC allocation creates a new resource wait: a packet can hold its input VC while requesting an output VC. Switch allocation then arbitrates among allocated heads. Fairness at both stages is needed for liveness.

## 4. Channel-dependency graph

Create one node per channel resource (often direction × link × VC class). Add edge $c_i\rightarrow c_j$ if a packet may hold $c_i$ while requesting $c_j$. A cycle is a **necessary condition** for routing deadlock under standard wormhole assumptions; an acyclic channel-dependency graph is sufficient to avoid it.

Deterministic dimension-order routing in a 2D mesh orders X channels before Y channels. Since a route never returns from Y to X, the dependency graph is acyclic across dimensions.

```mermaid
flowchart LR
    Xp["X channels"] --> Yp["Y channels"]
    Xn["other X direction class"] --> Yp
    Yp -. "no legal dependency back to X" .-> Stop["acyclic order"]
```

Turn-model routing forbids selected turns to break cycles while retaining some adaptivity. The proof must include all topology wraparound links, local injection/ejection, and VC transitions.

## 5. Adaptive routing and escape resources

Adaptive routing chooses among minimal or nonminimal paths based on congestion/faults. Flexibility adds dependency edges and can reintroduce cycles. Duato-style designs provide:

- adaptive VCs for performance;
- a connected deadlock-free escape subnetwork;
- a rule that any packet can eventually enter the escape network and follow its restrictions.

The escape path must have reserved/obtainable buffers and fair arbitration. If adaptive traffic can permanently consume escape resources, the proof is hollow.

Nonminimal routing (for example, Valiant-style intermediate routing) balances adversarial traffic at extra hops. It requires enough VC classes to prevent phase transitions from making channel cycles.

## 6. Protocol deadlock versus network deadlock

Even an acyclic routing graph can deadlock at the protocol level:

- requests occupy all buffers;
- progress requires responses;
- responses cannot enter because requests consumed shared resources.

Coherence may have request, snoop, response, data, and retry dependencies. Map classes onto separate virtual networks/buffers according to the protocol dependency graph. Reserve space or provide consumption guarantees for messages that release resources.

Example rule: a node receiving a snoop must be able to accept it and generate a response without waiting for a resource held by the original request. If response injection needs an SQ/cache resource blocked by that request, endpoint microarchitecture participates in the cycle.

## 7. Livelock and starvation

Deadlock is no movement; livelock is movement without destination progress; starvation is one packet waiting indefinitely while others proceed.

Causes:

- adaptive deflection repeatedly reroutes a packet;
- priority traffic continuously wins arbitration;
- retries collide in phase;
- age resets when a packet changes VC/router;
- requestor-specific throttles prevent mandatory responses.

Mitigations:

- monotonically increasing age and oldest-first escape;
- bounded deflection/nonminimal hops;
- fair round-robin or weighted arbitration with aging;
- randomized/exponential retry backoff;
- reserved response/maintenance service;
- formal liveness assumptions on sinks and credits.

Fairness needs a bound when real-time/QoS matters. “Eventually” is insufficient if deadline traffic can wait milliseconds.

## 8. Multicast and reduction dependencies

Coherence snoops and accelerator collectives may replicate or combine packets. A multicast head can reserve several outputs. Atomic multi-output allocation risks deadlock or low utilization; incremental replication needs per-branch state and replay safety.

Design choices:

- source replication into unicasts;
- router replication with branch masks;
- tree-based multicast/reduction;
- destination acknowledgements aggregated in network/home nodes.

Replication amplifies offered load. If average fanout is $K$, a logical message may consume $K$ destination paths plus responses. Model link-level flit-hops, not logical packet count.

## 9. Congestion control

Backpressure is necessary but can spread congestion trees. Control options:

- injection throttling based on local/global occupancy;
- source rate/token control by class/requestor;
- adaptive path selection;
- separating short control from long data packets;
- age-aware arbitration;
- congestion notification to cores/prefetchers;
- admission limits on outstanding transactions.

Stability requires offered load below sustainable service for every cut. Bisection bandwidth gives a topology bound:

$$
\lambda_{cross}\bar{P}<BW_{bisection}.
$$

No router optimization can overcome a workload whose required traffic exceeds the cut.

## 10. Fault-aware routing

Disabled links/routers change the dependency graph. A rerouting algorithm proven for a healthy mesh may deadlock after faults. Approaches:

- recompute acyclic routing tables for the remaining topology;
- preserve an escape spanning tree;
- use additional VCs/classes for detours;
- isolate unreachable partitions and report failure;
- drain or epoch-flush traffic before changing routes.

Dynamic route updates must avoid mixing old/new routes into a transient cyclic dependency. Version routes and quiesce/drain or prove compatibility across versions.

## 11. Verification strategy

### Safety

- credits never underflow/overflow and conserve buffer capacity;
- flits of a packet preserve order and identity;
- tail releases each reserved VC exactly once;
- route outputs are legal and eventually reach the destination;
- multicast replicas/acks are neither lost nor duplicated;
- no message crosses security/QoS domains incorrectly.

### Liveness

- every accepted packet reaches a sink under stated sink/fairness assumptions;
- mandatory responses have an independent progress path;
- retry state cannot oscillate indefinitely;
- escape VC is reachable and fairly served;
- route reconfiguration drains all old-version packets.

Use formal analysis on small parameterizations to find dependency cycles and credit bugs; use random/adversarial simulation for saturation, hotspot, transpose, tornado, and protocol traffic.

## 12. Counters and plots

- injection/acceptance/ejection rate by virtual network and requestor;
- per-link utilization and flit-hop count;
- VC occupancy, credit stalls, allocation stalls;
- packet latency distribution split into serialization, hops, and queueing;
- age/starvation maximums;
- adaptive versus escape route use;
- deadlock watchdog state and blocked-resource chain;
- multicast fanout and replication load;
- saturation curve (latency versus offered load).

Always plot latency against offered/accepted throughput. One point below saturation cannot validate flow control.

## 13. Numbers to remember

- Wormhole packets hold channels while their heads wait, creating dependency chains.
- Credit round-trip sets a buffer bandwidth-delay product.
- VCs are logical resources for deadlock avoidance, head-of-line isolation, and QoS.
- An acyclic channel-dependency graph proves routing deadlock freedom under its assumptions.
- Protocol deadlock can exist even when routing is deadlock-free.
- Adaptive routing needs a reachable, fairly served deadlock-free escape path.

## 14. Worked problems

### Problem 1 — serialization and zero-load latency

A 256-bit packet crosses six hops on a 128-bit link; each hop has 1-cycle router and 1-cycle link latency:

$$
L_0=6(1+1)+\lceil256/128\rceil-1=13\ \text{cycles}.
$$

Queueing under load is additional.

### Problem 2 — credit buffer depth

Downstream credit round trip is 7 cycles and the link sends one flit/cycle. One VC needs at least 7 effective slots/in-flight credits to avoid bubbles. Four VCs with two slots each may collectively fill the link under mixed traffic, but one long packet on a single VC can still bubble.

### Problem 3 — protocol cycle

Requests and responses share every endpoint input slot. All slots fill with requests waiting for invalidation responses, so responses cannot be accepted. Routing continues elsewhere but protocol progress stops. Reserve response capacity or separate virtual networks and ensure response generation does not wait on request-held resources.

## Cross-references

- **Topology/router:** [Network-on-Chip Architecture](01_Network_on_Chip.md).
- **Protocol classes:** [ACE and CHI](../01_Protocols/02_ACE_and_CHI.md), [Cache Coherence](../../03_Memory/03_Coherence_and_Consistency/01_Cache_Coherence.md).
- **Evaluation/policy:** [NoC and Coherence Simulation](../../07_Simulators/03_Memory_and_Interconnect/02_NoC_and_Coherence_Simulation.md), [QoS, Ordering, and I/O Coherence](../03_System_Fabrics/01_QoS_Ordering_and_IO_Coherence.md).

## References

1. W. Dally and B. Towles, *Principles and Practices of Interconnection Networks*.
2. J. Duato, “A Necessary and Sufficient Condition for Deadlock-Free Adaptive Routing in Wormhole Networks,” TPDS 1995.
3. C. Glass and L. Ni, “The Turn Model for Adaptive Routing,” ISCA 1992.
4. gem5, [Garnet 2.0 On-Chip Network Model](https://www.gem5.org/documentation/general_docs/ruby/garnet-2/).
5. L. Peh and W. Dally, “A Delay Model and Speculative Architecture for Pipelined Routers,” HPCA 2001.

---

**Navigation:** [Network on Chip index](00_Index.md) · [Interconnect index](../00_Index.md)
