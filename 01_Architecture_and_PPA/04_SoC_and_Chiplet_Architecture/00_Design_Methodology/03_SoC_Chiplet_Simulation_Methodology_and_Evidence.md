# SoC and Chiplet Simulation Methodology and Evidence

> **First-time reader orientation:** SoC simulation composes software execution, CPU/GPU/NPU/device commands, coherence, transaction protocols, NoC packets, memory requests, interrupts, clocks, and power states. The decisive question is not whether every component has a model, but whether their interfaces preserve timing feedback and the workload path needed by the system claim.

> **Abbreviation key — skim now and return as needed:** system on chip (SoC); system-level modeling (SLM); transaction-level modeling (TLM); network on chip (NoC); dynamic random-access memory (DRAM); direct memory access (DMA); operating system (OS); input/output memory management unit (IOMMU); region of interest (ROI); parallel discrete-event simulation (PDES); quality of service (QoS); service-level objective (SLO); power, performance, and area (PPA).

> **Hands off to:** [Full-Chip Modeling](../01_System_Modeling/01_Full_Chip_Modeling.md), [DRAM Simulators](../06_Simulation/01_DRAM_Simulators.md), and architecture-owned CPU/GPU/NPU simulation chapters for their command/instruction boundaries.

---

## 0. Choose the model boundary from the system claim

| Claim | Required boundary |
|---|---|
| software/driver boots and configures devices | functional virtual platform with CPU, memory map, interrupts, devices |
| transaction ordering/QoS | protocol-accurate masters, bridges, buffers, targets, arbitration |
| NoC latency/throughput/deadlock | packet/flit/credit timing and traffic dependencies |
| application time under shared memory | execution/command-driven agents coupled to caches/NoC/controllers/DRAM |
| DRAM policy under fixed offered traffic | standalone physical request trace |
| chiplet link/topology | die placement, packet/protocol overhead, link/PHY latency/bandwidth/retry |
| power/thermal control | activity + clocks/voltages/power states + thermal feedback over time |

Do not use a fixed trace to claim closed-loop application slowdown when slower service would throttle future request injection.

### 0.1 SoC fidelity ladder: pay only for state the claim observes

| Rung | State retained | Typical SoC use | Main blind spot |
|---|---|---|---|
| analytical/queueing | rates, capacities, average service | size links, memory channels, buffers | phase ordering and protocol state |
| untimed functional | software-visible register/memory behavior | firmware, boot, driver correctness | performance and contention |
| loosely timed TLM | transactions plus coarse annotated delay | virtual platform and software throughput | fine arbitration and cycle alignment |
| approximately timed TLM | request/response phases, queues, backpressure | protocol/QoS and coupled device timing | pin/clock details |
| cycle/event model | per-cycle resources, packets, commands | NoC/coherence/DRAM/application timing | implementation wires/gates unless calibrated |
| RTL/emulation | register-transfer implementation | integration, correctness, late performance evidence | slow simulation and limited early flexibility |
| gate/circuit/physical | cells, wires, analog/device behavior | timing, power, signoff, component calibration | application-scale coverage |

Moving downward retains more state and usually reduces simulated target cycles per host second. A practical system study mixes rungs: a functional CPU and device model can boot software while a selected NoC/DRAM path is approximately timed; a detailed CPU interval can use functional peripherals; a calibrated analytical model can sweep thousands of chiplet placements before a few event-level candidates. The boundary is valid only if discarded state cannot change the decision.

The often-quoted speedups are workload and implementation dependent, but the ordering is robust: loosely timed virtual platforms can be hundreds or thousands of times faster than RTL, while application-scale execution-driven cycle models are commonly orders of magnitude slower than native execution. Always measure model throughput on the actual workload and report which components dominate host time.

## 1. Freeze the complete system experiment

Preserve:

- software images, firmware/kernel/drivers, binaries, libraries, inputs, command lines;
- CPU/GPU/NPU/device configurations and mappings;
- memory map, address translation/page placement, coherence domains;
- NoC topology/routing/link width/buffers/clock and protocol settings;
- DDR/HBM device/controller/address map/timing;
- DMA/I/O/display/camera/storage/network traffic and interrupt models;
- clock/voltage/power/thermal policies and initial state;
- use-case arrivals, priorities, QoS/SLO, ROI/warm-up, seeds, and completion checks.

A component version without interface configuration is insufficient. The resolved system topology and all clock/time-unit conversions belong in provenance.

## 2. Compose models through explicit contracts

For every interface define:

- request/response fields, IDs, ordering, attributes, byte enables, atomic/coherence meaning;
- address domain (virtual, I/O virtual, physical, local/interleaved) and mapping;
- timing protocol: blocking/nonblocking, acceptance, backpressure, credits, retries;
- clock domain and conversion to global event time;
- buffer ownership and lifetime;
- error, cancellation, reset, and power-state behavior;
- statistics boundary and byte/event accounting.

Zero-time combinational cycles across models cause nondeterminism or impossible same-cycle feedback. Define event phases or minimum latency.

## 3. From software work to system events

~~~text
application / service request
 -> CPU instructions and runtime/driver calls
 -> device descriptors, kernel launches, DMA, interrupts
 -> virtual/I/O address translation and protection
 -> cache/coherence or noncoherent transaction
 -> protocol message and NoC packet/flits
 -> target cache/device/memory-controller request
 -> DRAM commands or device work
 -> response/completion/interrupt
 -> dependent software or device work
~~~

At each stage, timing can change later work. A full-system execution-driven model preserves the loop; a command trace freezes host/device decisions; a packet trace freezes protocol traffic; a DRAM trace freezes hierarchy filtering and often arrival time. State exactly where the artifact enters.

### 3.1 The discrete-event engine beneath a composed SoC

An event is a timestamped consequence such as “credit arrives,” “DRAM read completes,” “interrupt becomes pending,” or “DMA descriptor becomes ready.” A minimal engine maintains a priority queue ordered by simulated time and deterministic tie-break priority:

~~~text
elaborate components and connect interfaces
schedule initial clock/device/software events
while completion condition is false:
    event = pop earliest (time, phase, sequence)
    now = event.time
    apply event to component state
    emit counters and schedule consequences at time >= now
~~~

The causality invariant is that an event never schedules a consequence in the past. Zero-delay chains need a secondary phase or **delta-cycle** order so that a component can update state, notify dependents, and reach a stable fixed point without advancing simulated time. Cross-model adapters must preserve this order; directly calling back through a zero-time cycle can produce nondeterminism or infinite delta iteration.

Contention is not a constant added afterward. If a NoC output grants one flit/cycle, a DRAM bank accepts commands only when timing guards allow, or a bridge has eight credits, simultaneous requests queue behind those finite resources. Their arrival-to-grant gaps *become* queueing latency. Removing queues, credits, or backpressure structurally removes saturation from the model.

### 3.2 Why detailed simulation is slow

Let $n_{ev}$ be host-side events processed per simulated target cycle and $i_{ev}$ be average host instructions/work per event. Model work per target cycle is roughly

$$
w\propto n_{ev}i_{ev}.
$$

Pin toggles, per-flit routers, coherence transients, cache probes, and DRAM command guards raise $n_{ev}$; dynamic allocation, complex data structures, callbacks, and four-state values raise $i_{ev}$. Event skipping helps only during idle periods. Temporal decoupling reduces synchronization events but may hide short feedback. Trace replay removes functional execution work but can freeze the path. Parallel discrete-event simulation divides work but adds synchronization and lookahead constraints.

Optimize only after measuring host-time attribution. If 70% of runtime is detailed CPU execution, abstracting an already-cheap peripheral does little. If the study concerns NoC tail latency, replacing per-flit events with a fixed link delay destroys the observable being measured even if it runs faster.

## 4. TLM and virtual platforms

TLM represents transactions rather than pins. Useful abstraction levels include:

- untimed functional calls for software correctness;
- approximately timed transactions with annotated delay;
- loosely timed models with temporal decoupling;
- cycle/protocol-accurate bridges or selected hotspots.

Temporal decoupling lets initiators run ahead before synchronizing, improving speed but reducing observation of fine contention/order. Set quantum below the smallest interaction that affects the claim. Functional device models can boot software while detailed NoC/DRAM models are swapped in for selected paths.

## 5. NoC and coherence event simulation

A cache miss may create request, snoop/forward, data, and acknowledgment messages. Packetization converts messages into flits:

$$
N_{flit}=\left\lceil\frac{B_{header}+B_{payload}}{B_{flit}}\right\rceil.
$$

The model tracks injection, route/virtual-channel allocation, switch/link traversal, credit/backpressure, ejection, reassembly, destination protocol state, and dependent responses. Keep levels distinct:

- CPU/device transaction latency;
- coherence transaction/message count;
- packet/flit network latency and occupancy;
- end-to-end miss/command completion.

Average packet latency is not cache-miss latency because a miss can wait before injection, require multiple packets, and wait at the destination.

Validate protocol safety, absence of orphan/duplicate responses, credit conservation, and liveness/deadlock under finite buffers.

## 6. DRAM trace point and replay

A real memory-controller trace usually comes from

~~~text
dynamic load/store or DMA
 -> translation
 -> caches/coherence/prefetch/writeback filtering
 -> physical controller request
~~~

A standalone request should include physical address, read/write/atomic type, source/QoS where relevant, size, and arrival cycle/gap. Feeding CPU virtual loads directly to DRAM invents traffic and uses the wrong bank bits.

At arrival $a_i$, the controller accepts or backpressures, maps channel/rank/bank/row/column, queues the request, issues legal commands under timing/refresh/turnaround rules, schedules data-bus use, and records completion $d_i$. Raw outputs include accepted/completed requests, command counts, row hits/conflicts, queue occupancy, bytes, and $d_i-a_i$.

Fixed traces are valid for policy comparison under the same offered stream. They do not reproduce core/device throttling if changed memory timing would delay future arrivals. Couple agents for application/SLO claims.

## 7. Chiplet and package-level modeling

Add die-to-die packetization/protocol, bridge buffering, serialization, link/PHY latency, retry/error, clock crossing, topology/routing, and package placement. Track useful payload and wire bytes separately. A link trace may freeze coherence/home placement; execution-driven coupling is required if remote latency changes cache sharing or work scheduling.

Thermal/power simulation consumes per-block activity over time, converts to power, solves temperature, then may feed leakage/DVFS/throttling back into timing. Define update intervals and convergence; too-large intervals miss bursts, too-small intervals waste runtime or create numerical oscillation.

## 8. Parallel discrete-event simulation

Large many-core/datacenter/chiplet systems partition events across logical processes. PDES improves scale but requires conservative/optimistic synchronization, lookahead, deterministic ordering, and careful cross-partition latency. Parallel speedup does not justify causality violations. Record partitioning because it can affect reproducibility and runtime.

## 9. Measurement phases

1. boot/initialize/load software and devices;
2. fast-forward to use-case state;
3. warm architectural and queue states;
4. reset statistics;
5. run a complete transaction/frame/request/job ROI;
6. drain only the work included by the metric;
7. verify software output, device completions, interrupt counts, and no timeout/deadlock.

Asynchronous work must complete before ROI end. For service workloads, warm arrival queues and report tail percentiles. For power-state studies, include transition energy/time and initial state.

## 10. Raw counters to SoC metrics

$$
L_i=d_i-a_i,\qquad
\bar L=\frac{1}{N}\sum_iL_i,\qquad
BW=\frac{\sum_iB_i}{t_{end}-t_{start}},
$$

plus percentile latency, deadline misses, fairness, queue occupancy, NoC flits/hops, cache/coherence transactions, DRAM commands/row hits, device utilization, energy, and thermal/power-state residency.

Define boundaries: useful payload, protocol bytes, flit bytes, cache-line bytes, DRAM-burst bytes, and PHY wire bytes are all different. Do not add component latencies that already overlap in an end-to-end event schedule.

## 11. Conservation and validation

Check:

- requests accepted = completed + outstanding + canceled under explicit rules;
- credits/buffers/messages/bytes conserve across interfaces;
- every coherence transaction reaches a legal stable outcome;
- DMA/device completions match descriptors and software observations;
- cache/controller/DRAM bytes reconcile with documented amplification;
- clock/time conversion yields consistent elapsed time;
- power totals equal component activity × cost plus state leakage;
- no early limit, deadlock timeout, or unverified output.

Validate bottom-up and end-to-end: protocol microtests, NoC traffic patterns, DRAM command traces, device sequences, then representative coupled use cases against RTL/emulation/hardware.

### 11.1 SoC composition error budget

Component accuracy does not guarantee system accuracy. A CPU, NoC, and DRAM model can each match isolated tests yet produce the wrong coupled result if their adapter loses backpressure, changes address mapping, double-counts latency, ignores clock-domain crossings, or uses inconsistent byte/transaction definitions. Treat each interface contract as its own validation object.

Separate workload concurrency error, component parameter error, interface/composition error, sampling error, and structural omission. Tail latency is especially sensitive to rare alignment of bursts, refresh, coherence, and power-state transitions; an average-rate traffic generator cannot validate a percentile or deadline claim. A fixed open-loop trace also cannot reproduce throttling that changes later injection.

Use conservation checks to localize mistakes, then compare intermediate distributions: injection and service rates, queue occupancy, hop count, row-buffer state, retry/coherence traffic, and device utilization. When a design difference is smaller than the observed cross-use-case or validation residual, label it unresolved and escalate the relevant component/interface fidelity rather than tuning unrelated blocks.

## 12. SoC/chiplet tool map

| Tool family | Primary use |
|---|---|
| SystemC/TLM/virtual platform | firmware/OS/driver and transaction composition |
| gem5/full-system CPU simulators | software execution, caches/coherence/devices |
| SST/many-component PDES | large heterogeneous/many-node composition |
| Garnet/BookSim-style NoC | router/flit/credit timing and topology |
| Ramulator/DRAMSim3 | controller/DRAM request timing |
| DRAMPower | command-trace memory power |
| ns-3/OMNeT++-style network models | larger communication/network systems |
| RTL/emulation/FPGA | implementation-specific integration and software validation |

Use adapters that preserve address, ordering, backpressure, clocks, and statistics. A tool name does not prove these contracts are correct.

## 13. Reproducible result package

Preserve software/artifact hashes, full topology/configuration, trace points/hashes, clock/time mappings, commands/seeds, ROI/warm-up, raw event/counter logs, extraction/formulas, validation, and known model gaps. A plotted end-to-end point must be traceable through every component boundary.

## Cross-references

- [Full-Chip Modeling](../01_System_Modeling/01_Full_Chip_Modeling.md) builds the performance-power-thermal composition loop.
- [Network on Chip](../04_On_Chip_Networks/01_Network_on_Chip.md) and [Routing/Flow Control/Deadlock](../04_On_Chip_Networks/02_Routing_Flow_Control_and_Deadlock.md).
- [DRAM Simulators](../06_Simulation/01_DRAM_Simulators.md) provides tool-specific controller/device depth.

## References

1. IEEE 1666 SystemC and TLM-2.0 documentation.
2. gem5, SST, BookSim/Garnet, Ramulator, DRAMSim3, and DRAMPower official documentation/papers.
3. W. Dally and B. Towles, *Principles and Practices of Interconnection Networks*.
4. Parallel discrete-event simulation literature for conservative/optimistic synchronization.

---

← [SoC/Chiplet PPA and Physical Implementation](02_SoC_Chiplet_PPA_and_Physical_Implementation.md) · [SoC/chiplet book index](../00_Index.md) · next → [System Modeling](../01_System_Modeling/00_Index.md)
