# SoC and Chiplet Power, Performance, Area, and Physical Implementation

> **First-time reader orientation:** SoC and chiplet architecture is physical composition. Shared SRAM/DRAM, NoC wires, protocol buffers, clock/power domains, DDR/HBM/CXL/die-to-die PHYs, package routes, thermals, repair, and test can dominate the cost that a logical block diagram omits.

> **Abbreviation key — skim now and return as needed:** system on chip (SoC); static/dynamic random-access memory (SRAM/DRAM); one-transistor one-capacitor (1T1C); network on chip (NoC); input/output (I/O); physical interface (PHY); high-bandwidth memory (HBM); double data rate (DDR); compute express link (CXL); error-correcting code (ECC); power, performance, and area (PPA); process, voltage, and temperature (PVT); dynamic voltage and frequency scaling (DVFS); mean time between failures (MTBF).

---

## 0. Build a chip-and-package resource ledger

Inventory:

- compute blocks and local SRAM/cache;
- shared LLC/SRAM/directory/system cache;
- NoC routers, links, buffers, bridges, firewalls, monitors;
- DDR/HBM controllers and PHYs;
- PCIe/CXL/USB/storage/network/display/camera and die-to-die PHYs;
- clock/reset, voltage islands, regulators, isolation/retention, power gates;
- security, safety, reliability, debug, trace, test, repair, fuses;
- pads/microbumps/TSVs/interposer/bridges/package substrate;
- power delivery, heat spreading, keepouts, routing channels, and margin.

Each architecture choice must name its effect on state bits, ports, wires, clock load, power states, verification, and package resources.

## 1. Chip-level PPA equations

$$
P_{dyn}=\sum_j\alpha_jC_jV_j^2f_j,\qquad
P_{leak}=\sum_jV_jI_{leak,j}(T_j,V_j),
$$

$$
A_{die}=\sum_jA_j+A_{NoC/wire}+A_{clock/power/test}+A_{routing\ margin}.
$$

Package/system energy adds off-die links, memory stacks, regulators, and cooling. Average power is not enough: peak current, droop, hotspot temperature, and transient mode changes constrain sustainable performance.

## 2. Shared SRAM and system-cache implementation

A 6T SRAM array stores bits but requires decoder, wordline/bitline, sense/write, banking, ECC/parity, repair, arbitration, and routing. Shared caches additionally store tags, coherence/directory state, replacement, dirty/valid, sharer/owner information, and miss/fill queues.

Large system caches are sliced/distributed because wire delay dominates. Physical address hashing chooses a home slice and shapes NoC traffic. More capacity can reduce DRAM traffic but adds leakage, hit latency, lookup energy, directory area, and die footprint.

ECC parity bits $r$ for $k$ data bits satisfy

$$
2^r\ge k+r+1
$$

for single-error correction; overall parity enables double-error detection. Include correction latency, scrub bandwidth, poison/error reporting, redundancy, and repair.

### 2.1 Choose the memory technology by SoC role

Six-transistor SRAM is fast and logic-process compatible, but its bit-cell area and leakage make very large on-die capacity expensive. **Embedded DRAM (eDRAM)** uses a capacitor-style cell integrated on the logic die, offering higher density and often lower leakage per bit, but it needs sense/restore and refresh machinery, additional process steps, and longer/less uniform access. It has been used for large last-level caches where density and bandwidth outweigh the refresh/process cost. An eDRAM cache model must include refresh interference, row/bank organization, controller/periphery area, and the possibility that its latency differs from SRAM across operating points.

Boot and security state use different memories:

- **mask read-only memory (mask ROM):** dense, low-leakage, and immutable after fabrication; suitable for fixed boot code or tables but any bug requires a new mask;
- **one-time-programmable (OTP) memory/eFuses:** small identity, trim, key, repair, and lifecycle fields programmed after fabrication; include programming circuits, redundancy, sensing margin, access control, and irreversible update rules;
- **embedded flash or non-volatile memory (NVM):** field-updatable firmware/configuration where the process supports it; write voltage/time, endurance, retention, and security dominate rather than read bandwidth;
- **retention SRAM:** preserves selected state while a larger power domain is off; it pays larger cells or an always-on supply, save/restore sequencing, isolation, and retention-voltage verification.

These capacities are usually small compared with shared cache, but they are architecturally critical: the SoC cannot boot, repair lanes, identify itself, or enforce lifecycle security if they fail. Treat visibility, write authority, redundancy, error reporting, and field-update recovery as part of the system contract.

## 3. DRAM 1T1C, sensing, and refresh

A DRAM cell stores charge on a capacitor accessed by one transistor. Reads disturb the stored charge and require sense-amplifier restoration. Cells share long bitlines and sense amplifiers, which creates row activation/precharge behavior and high density.

Architecture consequences:

- an activated row resides in a row buffer;
- row hits avoid another activate/precharge and are faster/more efficient;
- banks allow parallel open rows;
- charge leakage requires refresh;
- timing constraints protect sensing, restoration, power, and disturbance limits.

Refresh consumes command/bank time and energy, with stronger effects at high temperature/density. RowHammer mitigation, ECC, sparing/repair, and patrol/scrub policies add traffic/state. The SoC memory model must include the device organization and controller policy that produce delivered service.

### 3.1 From capacitor charge to a controller-visible access

A stored one is only a small charge $Q=C_{cell}V$. When the access transistor connects the cell capacitor to a much larger precharged bitline capacitance $C_{BL}$, charge sharing produces a small voltage deviation:

$$
\Delta V_{BL}\approx\frac{C_{cell}}{C_{cell}+C_{BL}}\left(V_{cell}-V_{pre}\right).
$$

The sense amplifier detects and amplifies that small difference to full logic levels while restoring the destructive read. Long bitlines improve density because many cells share a sense amplifier, but they increase $C_{BL}$, reduce the initial signal, and slow sensing. DRAM organization is therefore a density/latency trade, not simply “a slow SRAM.”

At the command level, **ACTIVATE** raises a row into the bank's sense-amplifier row buffer; **READ** or **WRITE** selects columns; **PRECHARGE** closes the row. Timing names describe minimum separations: $t_{RCD}$ from activate to column access, column-access latency before read data, $t_{RAS}$ for minimum active time, and $t_{RP}$ to precharge. A row hit can issue a column command without another activation, whereas a conflict must precharge and activate a different row. Bank groups, data-bus turnaround, power limits on closely spaced activations, and refresh constrain parallelism even when different addresses appear independent.

Refresh overhead has a simple lower-bound intuition. If a rank requires $N_{ref}$ refresh operations per retention interval $T_{ret}$ and each blocks relevant resources for $t_{RFC}$, the blocked-time fraction is approximately

$$
\eta_{refresh}\approx\frac{N_{ref}t_{RFC}}{T_{ret}},
$$

before counting request reordering disruption. Higher temperature can shorten retention requirements; denser devices tend to have longer refresh commands. Per-bank refresh reduces the scope of each interruption but adds scheduling constraints. Fine-grained and temperature-compensated policies change both availability and worst-case latency.

### 3.2 Device reliability changes SoC traffic and state

RowHammer is repeated activation-induced disturbance of nearby rows. Mitigations may count activations, refresh neighbors, throttle suspect rows, remap addresses, or rely on device-managed tracking. Every option consumes counters, table storage, commands, bandwidth, power, or latency and must be included in worst-case quality-of-service analysis. On-die ECC can repair internal defects without exposing correction detail to the controller, but it does not replace end-to-end ECC across the controller, PHY, package, and memory device. The error contract must state what is corrected, detected, retried, poisoned, logged, or surfaced to software.

Capacity also includes yield mechanisms. Redundant rows/columns, post-package repair, lane repair, ECC bits, and bad-region retirement make usable capacity smaller than raw manufactured bits. These are SoC concerns because firmware-visible capacity, boot-time training, telemetry, retirement policy, and serviceability cross the device/controller/software boundary.

## 4. NoC physical cost

A router contains input buffers, virtual-channel state, route/allocator logic, crossbar, output/credit state, clocking, and link interfaces. Approximate area/power scales with ports $P$, virtual channels $V_c$, buffer depth $D$, link width $W$, and frequency:

$$
A_{router}\sim A_{buffers}(PV_cDW)+A_{crossbar}(P^2W)+A_{alloc}(P,V_c),
$$

with technology/topology-dependent constants. Wider links reduce serialization but increase wires, crossbar, register, clock, and repeater load. Long links may require pipeline stages, altering credit round-trip and latency.

Floorplan determines physical hop length; a topology drawn as uniform one-cycle links may not be implementable across a large die.

## 5. Protocol and bridge state is real silicon

AXI/CHI/CXL/coherence bridges store transaction IDs, ordering domains, outstanding requests, write data, snoop state, retry, credits, error/security attributes, and clock-domain crossings. Supporting more outstanding work expands buffers and comparison/order logic. Width conversion and asynchronous crossings add FIFOs and latency.

Protocol correctness can force resources that a bandwidth spreadsheet omits: separate response channels, deadlock-breaking buffers, snoop filters/directories, barriers, and retry queues.

## 6. PHY, package, and chiplet costs

Off-die bandwidth needs serializers/deserializers, clock-data recovery or forwarded clocks, equalization, training, calibration, termination, electrostatic-discharge protection, bumps/pads, package routes, and protocol controllers. Energy per transferred bit is

$$
E_{link}=M_{wire}e_{bit}+E_{fixed/training/retry}.
$$

Chiplet partitioning can improve yield and mix process nodes, but adds die-to-die PHY area/power/latency, package/interposer cost, coherency/protocol overhead, test/known-good-die requirements, power-delivery complexity, and thermal coupling.

Yield intuition for defect density $D_0$ and die area $A$ begins with

$$
Y\approx e^{-D_0A},
$$

though real models include clustering and redundancy. Smaller chiplets may improve per-die yield while package assembly yield/cost becomes important.

## 7. Clock, voltage, and power-state architecture

Multiple domains require clock/reset crossings, synchronizers/FIFOs, isolation, retention, level shifters, and sequencing. DVFS saves power only if workload slack and transition latency support it. Power-gating saves leakage but costs wake energy/time and state retention/reinitialization.

A mode table should name:

- allowed domain clocks/voltages;
- active/retained/off blocks;
- isolation and reset state;
- memory retention/flush/coherence actions;
- wake triggers and maximum latency;
- inrush/current and thermal restrictions.

These are architecture-visible because they affect latency, correctness, and software policy.

## 8. Thermal and power delivery

For block power $P_i$ and thermal resistance/coupling matrix $R_{\theta}$,

$$
\Delta\mathbf{T}=R_{\theta}\mathbf{P}
$$

is a useful linear first approximation. Chiplets/3D stacks require vertical and lateral coupling; HBM and PHY hotspots can heat compute or memory. Temperature feeds leakage and timing, closing a performance-power-thermal loop.

Power delivery must sustain average and transient current with acceptable droop. Wide simultaneous switching in NoC/compute/PHY can constrain boost. Include regulator efficiency, package/board loss, decoupling, bump/current density, and power-grid area.

## 9. Reliability, repair, and observability

Architectural resources include ECC/parity, retries, watchdogs, error containment, fault isolation, spare links/rows, cache/SM/core disable, thermal sensors, performance monitors, and trace buffers. Recovery changes traffic/timing and must be included in safety/availability use cases.

For synchronizer metastability, MTBF grows exponentially with resolution time and degrades with source/destination event rates. Clock-domain crossings need explicit structures and verification, not assumed zero-time connections.

## 10. Early uncertainty and calibration

Use ranges for macro availability, link/PHY estimates, routing utilization, clock tree, activity, package cost, and thermal conditions. Report both parametric and model-form uncertainty. Sensitivity

$$
S_x=\frac{\partial\ln Y}{\partial\ln x}
$$

identifies what to refine. Calibrate block estimates against memory compilers, synthesized fabrics/bridges, prior silicon, PHY vendor data, and package/thermal models.

## 11. Worked chiplet trade

A 600 mm² monolithic compute die is partitioned into four 140 mm² compute chiplets plus a 90 mm² I/O die. Compute-die yield may improve, but evaluate:

- four die-to-die PHY/control blocks and package routes;
- added remote-cache/memory latency and wire bytes;
- package/interposer area, assembly and known-good-die test;
- I/O-die bottleneck and power delivery;
- thermal distribution and process-node costs.

If the workload keeps most traffic local, partitioning can win cost/yield. If coherence and shared-memory traffic repeatedly cross the package, PHY energy/latency and I/O-die contention can erase it. The traffic-placement model and physical package estimate must be solved together.

## 12. SoC/chiplet PPA checklist

- Include shared cache/directory, NoC, protocol buffers, PHYs, clocks/power/test, and routing margin.
- Use physical distances and achievable link/cache latencies.
- Price DRAM refresh/RAS and memory-controller/PHY/package resources.
- Model sustained temperature, leakage, power delivery, and transitions.
- Include chiplet assembly/yield/test and cross-die traffic.
- Define reliability/recovery and observability overhead.
- Carry calibrated ranges; do not present early area/power as signoff precision.

## Cross-references

- [Full-Chip Modeling](../01_System_Modeling/01_Full_Chip_Modeling.md).
- [DDR Controller](../02_Shared_Memory/01_DDR_Controller.md).
- [Routing, Flow Control, and Deadlock](../04_On_Chip_Networks/02_Routing_Flow_Control_and_Deadlock.md).
- [Chiplets, CXL, and Die-to-Die](../05_IO_and_Chiplets/02_Chiplets_CXL_and_Die_to_Die.md).

## References

1. N. Weste and D. Harris, *CMOS VLSI Design*.
2. W. Dally and B. Towles, *Principles and Practices of Interconnection Networks*.
3. JEDEC DDR/HBM standards and memory reliability literature.
4. UCIe/CXL/PCIe specifications and contemporary chiplet/package literature.

---

← [SoC/Chiplet Workloads and DSE](01_SoC_Chiplet_Workloads_Performance_and_DSE.md) · next → [SoC/Chiplet Simulation Methodology and Evidence](03_SoC_Chiplet_Simulation_Methodology_and_Evidence.md)
