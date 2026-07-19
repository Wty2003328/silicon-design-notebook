# SoC and Chiplet Power, Performance, Area, and Physical Implementation

> **First-time reader orientation:** SoC and chiplet architecture is physical composition. Shared SRAM/DRAM, NoC wires, protocol buffers, clock/power domains, DDR/HBM/CXL/die-to-die PHYs, package routes, thermals, repair, and test can dominate the cost that a logical block diagram omits.

> **Abbreviation key — skim now and return as needed:** system on chip (SoC); static/dynamic random-access memory (SRAM/DRAM); one-transistor one-capacitor (1T1C); network on chip (NoC); input/output (I/O); physical interface (PHY); high-bandwidth memory (HBM); double data rate (DDR); compute express link (CXL); error-correcting code (ECC); power, performance, and area (PPA); process, voltage, and temperature (PVT); dynamic voltage and frequency scaling (DVFS); mean time between failures (MTBF); always-on (AON); clock-domain crossing (CDC); reset-domain crossing (RDC); integrated clock-gating cell (ICG); power-management unit (PMU); Unified Power Format (UPF); Common Power Format (CPF); phase-locked loop (PLL); phase-frequency detector (PFD); voltage-controlled oscillator (VCO); spread-spectrum clocking (SSC); electromagnetic interference (EMI); clock-tree synthesis (CTS); on-chip variation (OCV); power-on reset (PoR).

---

```mermaid
flowchart TD
    B["Blocks, SRAM, NoC, controllers, and PHYs"] --> F["Die and chiplet floorplans"]
    F --> PKG["Package routes, bumps, interposers, and memory placement"]
    PKG --> PDN["Power delivery, clocks, voltage domains, and thermals"]
    PDN --> C["Timing, power, area, reliability, yield, and cost closure"]
    C -->|"violations and calibrated estimates"| B
    C --> R["Tapeout resource ledger and implementation constraints"]
```

The physical hierarchy is coupled: package reach changes PHY power, floorplan distance changes NoC timing, and the resulting power density changes clock and thermal feasibility.

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

### 7.1 Derive domains from independent requirements

A **clock domain** is logic whose sequential state is timed by one clock relationship. A **voltage domain** is logic supplied at one operating voltage. A **power domain** is logic that can be switched on, retained, or off as a unit. They are different partitions: two blocks may share a clock but use different voltage rails, or share a rail while one clock is stopped. Treating the three boundaries as one diagram either adds unnecessary crossings or hides unsafe ones.

Start with the minimum baseline: one rail, one clock, no power gating. It has almost no crossing logic, but an idle accelerator still leaks, a low-speed peripheral pays the compute voltage, and changing the single clock or rail disturbs every block. Those failures derive three separate requirements:

1. independent activity control derives a clock domain and ICG;
2. independent performance/energy operating points derive a voltage domain and level shifters;
3. independent leakage shutdown derives a switchable power domain, isolation, retention or reinitialization, and an AON controller.

```mermaid
flowchart LR
    Base["one rail + one clock"] --> CFail["idle clock tree and flops still toggle"]
    Base --> VFail["slow block pays fast-block voltage"]
    Base --> PFail["idle transistors still leak"]
    CFail --> CD["clock domain: ICG + CDC/RDC contract"]
    VFail --> VD["voltage domain: regulator + level shifters + DVFS state"]
    PFail --> PD["power domain: switch + isolation + retention/reset"]
    CD --> PMU["AON PMU sequences legal transitions"]
    VD --> PMU
    PD --> PMU
```

Partition by a workload and dependency ledger, not by RTL hierarchy. Keep blocks together when they wake together, exchange high-bandwidth single-cycle traffic, share state that would be expensive to retain, or cannot tolerate added level-shifter/isolation delay. Split them when their utilization, required voltage/frequency, retention need, or safety/security authority differs enough to repay the crossings. A very fine partition can lose: every boundary adds control, verification states, placement keepouts, rail routing, wake energy, and timing arcs.

### 7.2 One accelerator shutdown, with the state that makes it safe

Assume an NPU domain has an ingress queue, DMA engine, scratchpad, compute array, and completion queue. The naive action—drop its supply when software requests idle—can lose a DMA write, strand a coherent request, or drive an unknown value into an AON interrupt controller. A safe implementation adds this explicit state:

- `admit_enable`, which prevents new commands;
- accepted/outstanding counters for DMA, fabric, and completion writes;
- a `quiesce_req/quiesce_ack` handshake;
- retained context bits or a declared reinitialization image;
- isolation controls with a specified clamp value;
- clock-gate, reset, power-switch, and power-good controls;
- a transition state machine and timeout/error record in the AON PMU.

```mermaid
stateDiagram-v2
    [*] --> ON
    ON --> DRAIN: idle_request / admit_enable=0
    DRAIN --> SAVE: queues_empty && outstanding=0
    SAVE --> ISOLATE: retention_save_ack
    ISOLATE --> OFF: outputs_clamped; clock_off; switch_off
    OFF --> POWERING: wake_request / switch_on
    POWERING --> RESTORE: power_good && clocks_stable
    RESTORE --> ON: reset_released; restore_done; isolate=0; admit_enable=1
    DRAIN --> ON: abort_before_isolation
    SAVE --> FAULT: save_timeout
    POWERING --> FAULT: power_good_timeout
```

```wavedrom
{ "signal": [
  { "name": "idle_req",       "wave": "0.1......0" },
  { "name": "admit_enable",   "wave": "1..0.....1" },
  { "name": "outstanding",    "wave": "=.=.=0....", "data": ["3", "2", "1"] },
  { "name": "retention_save", "wave": "0....10..0" },
  { "name": "isolate",        "wave": "0.....1.0." },
  { "name": "clock_enable",   "wave": "1......0.1" },
  { "name": "power_good",     "wave": "1.......01" },
  { "name": "wake_req",       "wave": "0.......10" }
] }
```

The ordering is the mechanism: stop admission, drain accepted effects to their promised ordering point, save retained state, clamp outputs, stop the clock, and only then remove power. Wake reverses physical dependencies: establish the rail, wait for `power_good`, stabilize clock/reset, restore or reinitialize, re-establish protocol credits and identities, then remove isolation and reopen admission. If a save or power-good timeout occurs, the controller enters a diagnosable fault state; it must not guess that state was retained.

### 7.3 Clock-domain and reset-domain crossings are protocols

A single-bit level that changes slowly can cross through a two-flop synchronizer, accepting latency and a nonzero metastability probability. A pulse needs pulse stretching, a toggle protocol, or request/acknowledge so it cannot disappear between destination edges. Multi-bit data needs a bundled-data handshake or an asynchronous FIFO; synchronizing each data bit independently can assemble a word that never existed. A reset crossing has a related rule: asynchronous assertion may force a safe state immediately, but deassertion is synchronized separately in every receiving clock domain so flops do not leave reset on different edges.

```mermaid
flowchart TD
    S["source-domain event"] --> Q{"payload kind?"}
    Q -->|"stable one-bit level"| Sync["two-flop synchronizer"]
    Q -->|"pulse / must not lose"| Hand["toggle or req/ack handshake"]
    Q -->|"multi-bit stream"| FIFO["asynchronous FIFO: Gray pointers + synchronized status"]
    Q -->|"reset release"| Reset["asynchronous assert, per-domain synchronized deassert"]
    Sync --> Obs["latency + MTBF evidence"]
    Hand --> Obs
    FIFO --> Obs
    Reset --> Obs
```

The PPA trade is concrete. More synchronizer stages improve MTBF but add latency. Deeper asynchronous FIFOs absorb clock-ratio bursts but cost SRAM/flops and increase backpressure delay. A generated clock with a known phase relation may use a constrained synchronous crossing; an unrelated or stoppable clock must not inherit that assumption.

### 7.4 Voltage transitions need a closed-loop protocol

For a DVFS change, frequency and voltage cannot move in arbitrary order. When increasing performance, raise voltage and wait for regulator/clock qualification before increasing frequency. When decreasing performance, lower frequency first, then reduce voltage. During the transition, block or tolerate work according to the clock/rail specification. The PMU needs requested and actual operating points, transition-in-progress state, acknowledgements from regulator and clock generator, timeout handling, and thermal/current limit overrides.

Break-even for entering a lower-power state is not its advertised leakage alone. If transition energy is $E_{tr}$, active-state power is $P_{on}$, sleep power is $P_{sleep}$, and transition latency is acceptable, the energy break-even idle time is approximately

$$
t_{BE}=\frac{E_{tr}}{P_{on}-P_{sleep}}.
$$

Predicting an idle interval shorter than $t_{BE}$ wastes energy; a longer interval may still lose if wake latency violates a service-level objective. Retaining more state shortens wake but adds retention-cell area, an AON rail load, save/restore verification, and leakage.

### 7.5 UPF/CPF is executable power intent, not the architecture itself

UPF and CPF describe power domains, supply networks and states, power switches, isolation, retention, level shifting, and related control intent so tools can insert/check implementation cells and power-aware behavior. They do not invent quiescence, completion semantics, safe clamp values, or the software-visible state machine. Those must exist in the architecture before power intent is written.

Use one traceable flow:

```mermaid
flowchart LR
    Req["architectural power-state table"] --> RTL["RTL: PMU FSM, quiesce, save/restore, controls"]
    Req --> Intent["UPF/CPF: domains, supplies, states, switches, isolation, retention, level shifting"]
    RTL --> PA["power-aware RTL simulation"]
    Intent --> PA
    PA --> Syn["synthesis: insert/map low-power cells"]
    Syn --> Equiv["power-aware equivalence + structural checks"]
    Equiv --> PnR["placement/CTS/routing: rails, switches, AON routes, IR drop"]
    PnR --> Gate["gate simulation + static timing across modes"]
    Gate --> Signoff["scenario coverage, power integrity, wake sequence, fault evidence"]
```

The architecture-to-intent handoff must name, for every crossing and state: source/destination domain, legal supply states, signal direction, clamp value and when it becomes active, level-shifter direction, retained registers and retention supply, save/restore protocol, reset behavior, and ownership of each control. Tools can then detect an unisolated crossing or missing level shifter. They cannot decide whether a response channel must clamp to `VALID=0` or whether the fabric must synthesize an error response instead; that is a protocol decision.

Power-aware verification injects supply-off corruption and replays the shutdown/wake trace. Check that no request is accepted after admission closes; every earlier accepted request completes or is explicitly aborted; isolation precedes corruption; retained state survives allowed states; non-retained state returns to reset; no clock toggles an unpowered domain; wake does not release output before state and protocol credits are valid; and every timeout reaches a bounded, observable recovery. Structural UPF/CPF checks and a clean CDC report are necessary, but neither proves the transaction-level drain invariant.

A mode table should name:

- allowed domain clocks/voltages;
- active/retained/off blocks;
- isolation and reset state;
- memory retention/flush/coherence actions;
- wake triggers and maximum latency;
- inrush/current and thermal restrictions.

These are architecture-visible because they affect latency, correctness, and software policy.

### 7.6 Clock generation: multiply a shared reference, per domain

Every clock domain of §7.1 needs a stable clock at its operating frequency, but the package delivers only a slow, spectrally clean **crystal reference** — tens of megahertz, accurate to parts per million, present at one pin. You cannot filter a 40 MHz reference up to a 3 GHz core clock; you must *build* a local oscillator and discipline it against the reference with a **phase-locked loop (PLL)**: a **phase-frequency detector (PFD)** measures phase error, a charge pump and loop filter integrate it into a control voltage, a **voltage-controlled oscillator (VCO)** converts that voltage to frequency, and a $\div N$ feedback divider closes the loop so that at lock $f_{out}=(N/M)f_{ref}$. The loop dynamics, jitter budget, ring-versus-LC VCO choice, and fractional-$N$ synthesis are derived in [the frontend clock page](../../../03_Frontend_RTL_and_Verification/05_PLL_DLL_and_Clock_Distribution.md); this page owns the *integration* decisions: how many PLLs, fed by what reference, with what modulation.

**Distribute the slow reference, regenerate the fast clock locally.** The clock is the one net guaranteed to switch every cycle, so its dynamic power is $P=\alpha C V^2 f$ with activity $\alpha=1$ (the $P_{dyn}$ of §1). Fanning a multi-gigahertz clock across a two-centimetre die is therefore the worst possible net: maximum $f$, die-spanning $C$, and skew/jitter accumulating over centimetres. Distributing the 50 MHz reference instead drops $f$ by ~60×, cutting that net's power ~60× for the same capacitance, and letting each domain *regenerate* its own gigahertz clock keeps every fast net short and local. This is the structural reason SoCs place a PLL per domain (or per cluster) rather than one central PLL fanning a fast clock everywhere.

- **Per-domain PLL vs one shared PLL + dividers.** Independent PLLs let each domain retune $N$ for its own DVFS operating point (§7.4) and confine a lock failure to one domain — at the cost of one analog block, one lock event, and one reference-tree load each. A single PLL with integer dividers is smaller but ties every domain to one frequency plan and one lock.
- **Spread-spectrum clocking (SSC).** To meet an **electromagnetic-interference (EMI)** limit, dither $N$ (fractional-$N$) so the carrier energy spreads over a band instead of a sharp tone — typically ~0.5% down-spread, triangular, at ~30–33 kHz — lowering the peak spectral line by ~10–15 dB. The cost: added tracking jitter, and every synchronous receiver of that clock (and any crossing to a non-spread domain, §7.3) must tolerate the modulation; a clock feeding an exact-rate SerDes reference must not be spread.

*Worked number — lock time is a boot cost.* Lock time is a settling time, $t_{lock}\approx(1/\zeta\omega_n)\ln(\Delta f_{init}/f_{tol})\sim\text{few}/f_{bw}$, and sampling stability caps the loop bandwidth at $f_{bw}\lesssim f_{ref}/10$. A 50 MHz reference with $f_{bw}=f_{ref}/15\approx3.3$ MHz locks in $\sim5/f_{bw}\approx1.5\ \mu\text{s}$; a 25 MHz reference with a conservative $f_{bw}=f_{ref}/20=1.25$ MHz stretches to $\sim4\ \mu\text{s}$. Released together, eight per-domain PLLs cost only a few microseconds — negligible beside DDR training (Full-Chip Modeling §7) — but PLL-lock gates every domain that depends on that clock, which is why it is an explicit reset-release milestone (§7.8).

**Trade-off — when the simpler option wins.** A microcontroller or small SoC whose logic runs at one frequency and one voltage needs exactly one PLL and a few dividers; per-domain PLLs would add analog area, power, and lock events for nothing. Reach for per-domain PLLs only when domains genuinely need independent frequency/DVFS or fault isolation, and for SSC only when a radiated-emissions limit bites and every downstream consumer tolerates the modulation.

```mermaid
flowchart LR
    XTAL["crystal reference<br/>25-50 MHz, ppm-accurate"] --> RBUF["buffered reference tree<br/>slow net, low power"]
    RBUF --> P0["PLL: CPU domain<br/>xN0, 3.0 GHz"]
    RBUF --> P1["PLL: NPU domain<br/>xN1, 1.4 GHz"]
    RBUF --> P2["PLL: NoC/SoC domain<br/>xN2, 1.0 GHz"]
    RBUF --> P3["PLL: peripheral + SSC<br/>xN3, 0.3 GHz"]
    P0 --> C0["local clock tree"]
    P1 --> C1["local clock tree"]
    P2 --> C2["local clock tree"]
    P3 --> C3["local clock tree"]
```

### 7.7 Clock distribution: bound global skew to a fraction of the period

A regenerated clock is worthless if it reaches endpoints at different instants. The enemy is not delay but *difference* in delay. **Insertion delay** — source-to-endpoint latency, common to all endpoints — merely shifts the whole clock and cancels out of every register-to-register path. **Skew** — the arrival difference between a launch flop and its capture flop — lands directly in the timing budget:

$$
T_{clk}\ge t_{cq}+t_{comb}+t_{setup}-\delta,\qquad t_{hold}\le t_{cq}+t_{comb,\min}-\delta,
$$

where $\delta=t_{capture}-t_{launch}$ has an uncertain sign per path: positive skew relaxes setup but tightens hold. Because the sign is not guaranteed, **unbudgeted skew is stolen straight from the cycle.**

**Why a balanced tree bounds skew.** An H-tree branches recursively in H shapes so every leaf is geometrically equidistant from the root — matched wire length gives matched delay and near-zero *nominal* skew by construction; a spine/fishbone is the tool-balanced form that **clock-tree synthesis (CTS)** builds; a mesh shorts endpoints together from many drive points so local variation averages out. A balanced tree's residual skew is not nominal but comes from **on-chip variation (OCV)**: the "same" 20 ps buffer is 18 ps on one branch and 22 ps on another, bounded in signoff by opposed derating of the launch and capture paths. The topology detail, OCV, and common-path pessimism removal are on [the frontend clock page](../../../03_Frontend_RTL_and_Verification/05_PLL_DLL_and_Clock_Distribution.md); here the load-bearing quantity is the *budget*.

**The skew–power trade.** Driving skew toward zero costs power. A mesh minimizes the divergent (non-common) clock length, making it the least OCV-sensitive topology — but it is a die-spanning continuous grid switching every cycle, so its capacitance, and thus its $\alpha{=}1$ clock power, dwarfs a tree's. Cutting skew by moving from a balanced tree to a mesh can push the clock network from ~25% to ~40% of a domain's dynamic power. Skew is bought in Watts.

*Worked number — skew as a fraction of the period.* A 2.5 GHz domain has $T=400$ ps. A CTS skew target of 5% is 20 ps; combine it (root-sum-square) with ~12 ps of PLL period jitter (§7.6) and ~6 ps of duty/OCV margin and the total clock uncertainty is $\sqrt{20^2+12^2+6^2}\approx24$ ps — ~6% of the period, subtracted from every path before a gate is placed. Tighten skew to 2% (8 ps) with a mesh and uncertainty falls to ~15 ps (~3.8%), recovering ~9 ps of cycle at the mesh's power cost. On a 400 MHz peripheral domain ($T=2.5$ ns) the same 20 ps skew is 0.8% of the period and irrelevant.

**Trade-off — when the simpler option wins.** For nearly every domain an automated spine/tree meets a 5% skew target at far lower power and metal, and CTS closes it without a mesh. Reserve the mesh (or a mesh-fed-by-H-tree hybrid) for the single top-bin, highest-frequency domain where OCV and skew actually dominate closure; at modest frequencies skew is a negligible fraction of a long period and a plain buffered tree is correct.

### 7.8 Reset distribution: assert asynchronously, release synchronously, in dependency order

At cold power-on every flip-flop holds an unknown value; the reset network must force a known safe state before the first useful edge, or blocks latch garbage. Three failures make a naive reset unsafe, and each dictates part of the discipline. (1) A *synchronous-only* reset cannot initialize a domain whose clock is stopped or whose PLL has not locked — precisely the cold-start condition — so assertion must be **asynchronous**. (2) A carelessly *released* reset drives flops near the release edge metastable and lets different flops leave reset on different edges, so deassertion must be **synchronous** per domain. (3) A consumer released before its producer sees garbage, so release must be **sequenced in dependency order.**

**Mechanism — the per-domain reset synchronizer.** The common asynchronous reset drives the asynchronous clear of a two-flop chain (the first flop's D tied to logic 1) clocked by the *destination* clock:

```tikz
\usepackage{circuitikz}
\begin{document}
\begin{circuitikz}[american,thick,scale=0.85,transform shape]
  \tikzset{ff/.style={draw,minimum width=1.5cm,minimum height=1.7cm,align=center,font=\small}}
  \node[ff] (ff1) at (0,0) {DFF};
  \node[ff] (ff2) at (3.8,0) {DFF};
  \node[left,font=\footnotesize]  at (ff1.west){D}; \node[right,font=\footnotesize] at (ff1.east){Q};
  \node[left,font=\footnotesize]  at (ff2.west){D}; \node[right,font=\footnotesize] at (ff2.east){Q};
  \draw (ff1.west) -- ++(-1.2,0) node[left]{\texttt{1}};
  \draw[->] (ff1.east) -- (ff2.west);
  \draw[->] (ff2.east) -- ++(1.5,0) node[right]{\texttt{rst\_sync\_n}};
  \draw (ff1.south) -- (0,-1.7) coordinate (c1);
  \draw (ff2.south) -- (3.8,-1.7) coordinate (c2);
  \draw (c1) -- (c2);
  \draw (c1) -- (-2.5,-1.7) node[left]{\texttt{clk\_dst}};
  \draw (ff1.north) -- (0,1.7) coordinate (r1);
  \draw (ff2.north) -- (3.8,1.7) coordinate (r2);
  \draw (r1) -- (r2);
  \draw (r1) -- (-2.5,1.7) node[left]{$\overline{\texttt{arst}}$ clear};
\end{circuitikz}
\end{document}
```

Tying $D_1$ high and feeding the asynchronous reset to both clears (the open circles mark the active-low clear inputs) makes assertion immediate and clock-independent — the clears force both $Q\to0$ the instant reset asserts, even with a dead clock — while deassertion propagates through the two flops on successive destination-clock edges, so every flop in the domain leaves reset on one common, clean edge.

**Why synchronous deassert is mandatory.** A flop's reset pin carries two timing checks analogous to setup/hold: **recovery time** $t_{rec}$ (reset must be stably deasserted *before* the active clock edge) and **removal time** $t_{rem}$ (reset must stay asserted *until after* it). If an asynchronous deassertion lands in the window $[-t_{rem},\,t_{rec}]$ around a clock edge, the flop can go metastable on the *release* edge exactly as a data violation would; and because different flops see the deassertion at slightly different times, some leave reset a cycle later than others, so the domain starts inconsistent. Synchronizing the deassertion through the two-flop chain aligns release to a common edge and gives the usual resolution — with per-stage resolution time $t_r$ the release-edge failure rate follows the same $\mathrm{MTBF}\propto e^{t_r/\tau}/(f_{clk}f_{event})$ of §9. Reset deassertion happens roughly once per boot, so $f_{event}$ is tiny and two stages are astronomically safe; a domain whose reset toggles often (per-domain power cycling) is the case that actually needs the MTBF margin.

**Reset distribution has its own skew.** The synchronized release is itself distributed through a buffered reset tree with insertion delay and *reset skew*, just like the clock. *Worked number — reset-tree depth vs frequency.* Suppose the synchronized reset is distributed with 120 ps of insertion skew. In a 2.5 GHz domain ($T=400$ ps) one release cycle is 400 ps $>$ 120 ps, so every flop captures the same release edge — safe with a plain reset tree. Move the identical 120 ps skew into a 5 GHz domain ($T=200$ ps): now skew is 60% of a cycle and, with clock skew added, the release can be caught on two different edges. The fix is the same discipline as the clock — balance the reset tree to a few percent of the period ($\le$~10 ps) or add a held/pipelined release stage. Reset is not exempt from §7.7 once the period shrinks.

**Sequencing — RDC and the reset controller.** **Reset-domain crossing (RDC)** gives *each* clock domain its own synchronizer off the common asynchronous source, so assertion is simultaneous but each domain releases on its own clock; chaining the synchronizers enforces producer-before-consumer order. Above the domains, the always-on **power-on-reset (PoR)**/reset controller releases resets in dependency order, each step gated by success evidence and a watchdog timeout that drops to a defined fallback:

```mermaid
flowchart TD
    POK["rails power-good (AON first)"] --> CLK["clocks + PLL lock (§7.6)"]
    CLK --> MEM["memories: BIST, repair, retention restore"]
    MEM --> LOG["core logic out of reset"]
    LOG --> IF["interfaces/links: credits, training, epochs"]
    POK -.->|"power-good timeout"| FB["fault / safe fallback"]
    CLK -.->|"lock timeout"| FB
    MEM -.->|"BIST/repair fail"| FB
    IF -.->|"train fail"| FB
```

The ordering *is* the mechanism: a block is released only after everything it depends on is stable — a clock before the logic it times, a memory's built-in self-test and repair before the logic that reads it, a link's credits before the traffic that uses it. Releasing out of order is the reset analogue of an unsequenced power-up (§7.2). The full power-on milestone chain — rails → boot ROM → PLLs → DDR training → firmware → application cores — is [Full-chip boot and power-on sequencing (Full-Chip Modeling §7)](../01_System_Modeling/01_Full_Chip_Modeling.md).

**Trade-off — when the simpler option wins.** Async-assert/sync-deassert is standard because it initializes without a clock yet releases cleanly. A *fully synchronous* reset removes the removal-metastability problem and is friendliest to timing and design-for-test, but it cannot initialize a domain before its clock/PLL is live, so it fails at cold power-on — usable only for a block guaranteed a running clock. A *fully asynchronous* reset (async assert **and** async deassert) is simplest to route but reintroduces the removal race, tolerable only for a single-flop or quasi-static block. For sequencing, a central always-on reset controller gives one auditable order with per-step fallback and is worth it for any multi-domain SoC; a tiny fixed-order design can hardwire local release chains — but the boot/AON path and the default-slave liveness (Full-Chip Modeling §8) must stay hardwired even so.

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
- [Low-Power Architecture and Domain Partitioning](../../../02_Power_and_Low_Power/03_Low_Power_Architecture_and_Domain_Partitioning.md).
- [UPF and CPF Power Intent](../../../02_Power_and_Low_Power/05_UPF_and_CPF_Power_Intent.md).
- [Clock Generation and Distribution — PLL, DLL, and the Clock Network](../../../03_Frontend_RTL_and_Verification/05_PLL_DLL_and_Clock_Distribution.md) (the PLL loop dynamics, jitter budget, and H-tree/mesh/CTS behind §7.6–§7.7).
- [Async Design and CDC](../../../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) (the reset synchronizer and RDC of §7.8, and the CDC structures of §7.3).
- [Full-Chip Integration, Verification, and Bring-up Blueprint](../08_Implementation_Blueprints/03_Full_Chip_Integration_Verification_and_Bringup_Blueprint.md) (the boot/reset dependency graph and first-silicon gates for §7.8).

## References

1. N. Weste and D. Harris, *CMOS VLSI Design*.
2. W. Dally and B. Towles, *Principles and Practices of Interconnection Networks*.
3. JEDEC DDR/HBM standards and memory reliability literature.
4. UCIe/CXL/PCIe specifications and contemporary chiplet/package literature.

---

← [SoC/Chiplet Workloads and DSE](01_SoC_Chiplet_Workloads_Performance_and_DSE.md) · next → [SoC/Chiplet Simulation Methodology and Evidence](03_SoC_Chiplet_Simulation_Methodology_and_Evidence.md)
