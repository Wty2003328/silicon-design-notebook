# 05 — Backend Physical Design: Interview Questions

Consolidated interview Q&A and worked problems from every page in `05_Backend_Physical_Design/`, moved here to keep topic pages focused on concepts. Each section links back to its source page for full context.

---

## Physical Design (PnR) — Senior Engineer Deep Dive

*From [Physical_Design.md](../05_Backend_Physical_Design/01_Physical_Design.md)*

### Q1: Walk me through the complete ASIC PD flow from netlist to GDSII.

**A:** The flow begins with a synthesized gate-level netlist, SDC (Synopsys Design Constraints) timing constraints, technology LEF (Library Exchange Format) files, and standard cell/macro libraries. First, floorplanning establishes the die size (cell area / utilization), places macros, defines IO pin locations, and creates the power grid (VDD/VSS — power and ground — rings and stripes). Next, placement distributes standard cells using analytical global placement followed by legalization to snap cells to rows. The tool optimizes for timing, congestion, and power. Clock Tree Synthesis builds a balanced buffer tree for each clock domain, targeting minimum skew and insertion delay, using CTS-specific cells and NDR (Non-Default Rules) routing rules. Post-CTS, hold time fixing is performed. Routing proceeds in phases: global routing (GCell-level paths), track assignment, and detail routing (DRC-clean geometries). Post-route optimization fixes remaining timing/DRC issues. Chip finishing adds filler cells, metal fill for CMP (Chemical-Mechanical Polishing) density, and via optimization. Signoff runs include STA (Static Timing Analysis; PrimeTime, MCMM (Multi-Corner Multi-Mode) with POCV (Parametric On-Chip Variation)), DRC/LVS (Calibre/ICV), IR (current-resistance) drop/EM (electromigration) analysis (RedHawk), and extraction correlation. After all signoff clean, GDSII (Graphic Data System II) is streamed out.

---

### Q2: How do you estimate die size? What utilization would you target for a high-performance CPU block vs a low-power IoT chip?

**A:** Die size = (total standard cell area / utilization) + macro area + IO ring area. For a high-performance CPU targeting >2GHz, I'd use 60-65% utilization to provide ample routing resources and minimize congestion — critical paths need direct routes without detours. For a low-power IoT (Internet of Things) chip running at 100MHz, 75-80% utilization is acceptable since timing is relaxed and area is the primary cost driver. The remaining whitespace in both cases accommodates routing tracks, buffer insertion during CTS and optimization, decap cells, and filler cells.

---

### Q3: You have a block with 20 large SRAM macros. Describe your macro placement strategy.

**A:** First, analyze fly-lines to understand dataflow between macros and logic. Group macros that share buses (e.g., cache data arrays accessed by the same controller). Place macros along the periphery to maximize contiguous standard cell area in the center. Orient macros so signal pins face the logic — for example, SRAM (Static Random-Access Memory) data pins toward the datapath, address pins toward the address decoder. Leave minimum 10-20um channels between macros for routing. Apply halos (5-10um) around each macro to prevent standard cells from crowding macro pins. Place hard blockages over macros and partial blockages (50-60% utilization) in channels. Run trial placement and routing to verify no congestion hotspots in channels. If macros have dedicated power, ensure power grid straps extend into macro channels with adequate via stacks.

---

### Q4: Explain the difference between static and dynamic IR drop. Which is harder to fix?

**A:** Static IR drop is the DC voltage drop caused by average current flowing through the resistive power grid: V_drop = I_avg × R_grid. It produces a steady-state voltage map. Dynamic IR drop is the transient voltage drop caused by sudden current surges during simultaneous switching (e.g., many FFs (flip-flops) capturing on the same clock edge). It includes both resistive (IR) and inductive (Ldi/dt) components and can be 2-5x worse than static. Dynamic is harder to fix because it depends on switching patterns, which vary with functional scenarios. Fixes include: adding more power straps (reduces R), inserting decap cells near hotspots (local charge reservoir), staggering clock edges, reducing simultaneous switching activity, and ensuring adequate package-level decoupling capacitance.

---

### Q5: What is useful skew in CTS? When would you use it, and what are the risks?

**A:** Useful skew intentionally unbalances the clock tree to borrow time from a slack-rich stage and give it to a slack-poor stage. If a critical path has 100ps of negative setup slack, and the downstream path has 500ps of positive slack, adding 200ps of skew to the capturing FF (delaying its clock arrival) gives the critical path 200ps more time while taking 200ps from the downstream path (which can afford it). The risk is that useful skew tightens hold timing on the skewed paths — the data must still be stable for the hold time after the (now delayed) clock arrives. This typically requires inserting hold-fix buffers, which add area and power. The CTS tool must jointly optimize setup and hold across the design to find the globally optimal skew schedule.

---

### Q6: How do NDR (Non-Default Rules) help clock quality? What's the area cost?

**A:** NDR for clock nets typically specifies 2x width and 2x spacing. Double width cuts wire resistance in half, reducing RC (resistance-capacitance) delay and making clock delay less sensitive to process variation (less skew variation). Double spacing reduces coupling capacitance to adjacent signal wires, reducing crosstalk-induced jitter. Multi-cut vias are also mandated for reliability. The cost is significant: a 2W2S clock wire occupies roughly 4x the routing resources of a standard wire. For a large design where clock nets span the full die, this can consume 10-20% of available routing tracks on clock-heavy layers. The trade-off is worthwhile because clock quality directly impacts all timing paths in the design.

---

### Q7: Explain the antenna effect. How does the router fix antenna violations?

**A:** During fabrication, metal layers are processed sequentially (bottom-up). When a metal layer is being plasma-etched, the exposed metal acts as an antenna, collecting charge. If this metal is connected to a transistor gate through lower metals, the accumulated charge can damage the thin gate oxide. The antenna ratio (metal area / gate area) must stay below a foundry-specified limit per layer. The router fixes violations by: (1) inserting protection diodes near the gate to discharge charge (most common), (2) breaking the long wire by jumping to a higher metal layer (which is processed later, breaking the charge path), or (3) rearranging the routing topology to reduce the single-layer wire length connected to the gate. Diode insertion is simplest but requires placement of diode cells, which may need whitespace near the gate.

---

### Q8: What is the difference between DRC and LVS? Can a design pass DRC but fail LVS?

**A:** DRC (Design Rule Check) checks geometric rules — spacing, width, enclosure, density — ensuring the layout is manufacturable. LVS (Layout Versus Schematic) verifies that the manufactured circuit (extracted from layout geometry) matches the intended schematic. Yes, a design can pass DRC but fail LVS. For example, two nets could be properly spaced (DRC clean) but a missing via causes an open in one net, meaning the extracted circuit has a disconnected node that doesn't match the schematic. Conversely, a short between two wires violates both DRC (spacing) and LVS (incorrect connectivity), but some shorts may be through substrate or other non-obvious paths that DRC doesn't directly check.

---

### Q9: Explain AOCV and POCV. Why are they better than flat OCV?

**A:** Flat OCV (On-Chip Variation) applies a fixed derate (e.g., 5%) to all cells, assuming worst-case variation on every stage. This is pessimistic for deep paths because statistical variations tend to average out over many stages. AOCV (Advanced OCV) uses depth-dependent derates: a 2-stage path might get 8% derate, while a 20-stage path gets only 3%. This reduces pessimism and allows higher operating frequencies (or lower voltage). POCV (Parametric OCV) goes further by treating each cell's delay as a distribution (mean + sigma). Path delay sigma grows as sqrt(sum of variances), which grows much slower than linearly. For a 100-stage path, POCV's relative variation is ~10x smaller than flat OCV's, enabling 5-10% better timing margins. POCV is the preferred method at 7nm and below.

---

### Q10: How do you handle crosstalk in routing? What's the timing impact?

**A:** Crosstalk is managed through SI-aware routing and extraction. The router uses shielding (inserting VSS/VDD wires between sensitive nets), spacing (widening the gap between aggressors and victims), and layer assignment (moving sensitive nets away from aggressors). Timing impact: opposite-direction switching makes the victim slower (coupling capacitance effectively doubles per Miller effect), while same-direction switching makes it faster. In STA, we analyze both for setup (add slow-down SI (signal integrity) delay) and hold (add speed-up SI benefit). The extraction tool reports coupling capacitances in SPEF (Standard Parasitic Exchange Format), and the timer computes worst-case aggressor alignment. Fixing priorities: clock nets first (jitter), then setup-critical data paths, then hold-critical paths.

---

### Q11: What is metal fill? Why is it needed, and what's the timing impact?

**A:** Metal fill (dummy fill) adds non-functional metal shapes to meet CMP density requirements. Without it, regions with low metal density polish differently during CMP, causing dishing (metal thinning) and erosion, leading to thickness variation and potentially shorts or opens. Foundries require 20-80% density per layer per window. The timing impact is that fill shapes add parasitic capacitance to nearby signal wires, slightly increasing delay. Timing-aware fill placement keeps fill shapes away from timing-critical nets (with a defined keep-out distance). Post-fill extraction and STA re-run are mandatory for signoff.

---

### Q12: Describe the difference between H-tree and balanced CTS. When would you choose each?

**A:** H-tree uses a recursive symmetric topology where each branch has exactly equal wire length, providing inherently zero skew (in theory). It works best for regular structures (SRAM arrays, FPGAs (Field-Programmable Gate Arrays)) where sinks are uniformly distributed. Balanced CTS (standard approach in ASIC (Application-Specific Integrated Circuit) tools) builds a buffer tree bottom-up, clustering nearby sinks and inserting buffers to equalize delays. It handles irregular sink distributions much better and is the default for digital ASIC designs. A clock mesh provides the lowest skew (<5ps) by using a grid of wires with multiple drivers, but at 3-5x the power cost of tree-based CTS. Choose H-tree for array structures, balanced CTS for general ASIC logic, and mesh for high-performance processors where skew budget is extremely tight.

---

### Q13: What is Black's equation? How does EM impact PD decisions?

**A:** Black's equation models electromigration MTTF (Mean Time To Failure): MTTF = A × J^(-n) × exp(Ea/(kT)), where J is current density, n ≈ 1-2, Ea is activation energy, T is temperature. Higher current density and temperature exponentially reduce wire lifetime. PD (Physical Design) impact: power straps must be wide enough to keep J below EM limits (typically 1 MA/cm^2 for DC). Signal wires carry less current but long-running high-toggle nets (clocks) need checking. Multi-cut vias distribute current across multiple via cuts. At the end, EM analysis (RedHawk/Voltus) generates a current density map and flags violations. Fixes include widening wires, adding parallel straps, reducing load (downsizing driver fanout), or adding more via cuts.

---

### Q14: How does fin quantization affect physical design at 7nm and below?

**A:** In FinFET (Fin Field-Effect Transistor) technology, transistor drive strength is quantized by the number of fins (1, 2, 3, etc. fins), unlike planar CMOS (Complementary Metal-Oxide-Semiconductor) where width is continuous. Standard cell height is defined in terms of fin pitches — common heights are 6T, 7.5T, 9T (tracks). A 6T cell has fewer routing tracks (fewer M2 tracks available above the cell), saving area but making routing harder. A 9T cell provides more tracks, easing routing and enabling higher performance but using more area. The PD engineer must choose the cell library height based on the block's requirements: 6T for area-critical mobile chips, 7.5T for balanced designs, 9T for high-frequency server chips. Additionally, cell sizing options are discrete (1-fin steps), meaning timing optimization has coarser granularity than planar CMOS.

---

### Q15: Walk me through a timing ECO flow for fixing a setup violation found in signoff STA.

**A:** Start by analyzing the violation in PrimeTime: identify the failing path, the amount of negative slack, and which stages contribute the most delay. Strategies in priority order: (1) Swap cells to faster VT (e.g., SVT (standard threshold voltage) → LVT (low threshold voltage)), which increases leakage but reduces delay — check that the leakage budget isn't exceeded. (2) Upsize the driving cell (e.g., BUFX2 → BUFX8) to reduce stage delay — check that load on the previous stage doesn't worsen. (3) Insert a buffer to break a long wire into two shorter segments, reducing RC delay. (4) Reroute the critical net to a higher (thicker) metal layer for lower resistance. After the ECO (Engineering Change Order), run incremental placement (if cells were added), incremental routing, and re-extract parasitics. Re-run STA to verify the fix doesn't create new violations. Then re-run DRC/LVS on the affected region.

---

### Q16: What is hierarchical PD? When would you use it vs flat implementation?

**A:** Hierarchical PD partitions a large SoC (System-on-Chip) into blocks, each implemented independently and then integrated at the top level. Use it when the design exceeds 100-200M gates (flat tools struggle with runtime and memory) or when multiple teams work in parallel with different schedules. Each block gets a timing budget (interface constraints at its boundary) and is implemented as a separate PnR (Place-and-Route) run. The top level stitches blocks together using their LEF abstracts (physical outlines + pin locations). Challenges include interface timing budgeting (if wrong, blocks don't integrate cleanly), cross-block clock tree balancing, power grid continuity, and feedthrough signals. Flat implementation gives globally optimal results but doesn't scale beyond ~200M gates with current tools. Most modern SoCs use a mix: large regular blocks (CPU, GPU) are hierarchical, while smaller glue logic is flat.

---

### Q17: Explain double patterning coloring constraints. What happens if there's a coloring conflict?

**A:** In LELE (Litho-Etch-Litho-Etch) double patterning, features on one metal layer are split onto two photomasks (two colors). Adjacent features must be on different colors because same-color features need wider spacing (single-exposure resolution limit). A coloring conflict occurs when three features are mutually close — you can't assign colors to all three without violating same-color spacing. Example: three parallel wires each within minimum space of each other form a conflict. The fix requires the router to either: increase spacing for at least one pair (to allow same-color assignment), jog one wire to create distance, or re-route entirely. Some tools support pre-coloring critical nets. DRC checks include color-aware rules (same-color spacing vs different-color spacing) and are mandatory at 7nm and below.

---

### Q18: How do you verify that the power grid is adequate before starting placement?

**A:** After floorplanning and power grid creation, run early IR drop analysis using simplified models. Use a tool like RedHawk or Voltus with estimated power consumption (from synthesis power reports or activity assumptions). Check: (1) Static IR drop at every node is within budget (typically <5% VDD). (2) EM current density on every strap and via is below limits. (3) Power grid mesh has adequate connectivity (no broken straps or missing vias). Also check: ring resistance is low enough, via arrays at strap intersections have sufficient cuts, and decap cell plan provides enough local charge storage. If violations exist, iterate on strap width, pitch, number of metal layers used, and ring dimensions before committing to placement.

---

### Q19: What is CCD (Concurrent Clock and Data) optimization?

**A:** Traditional CTS builds the clock tree first, then freezes it and optimizes data paths. CCD allows the tool to simultaneously adjust both clock tree delays and data path delays during optimization. By selectively adding or removing clock tree delay at specific endpoints (useful skew), CCD can relax critical setup paths without requiring data path changes. This provides 5-10% better frequency compared to fixed-clock-then-optimize flows. The cost is longer runtime and increased hold buffer insertion (since skewing the clock creates new hold violations). CCD is available in ICC2 (as "clock concurrent optimization") and Innovus (as "CCOpt"). It's essential for high-frequency designs where traditional CTS + data optimization can't close timing.

---

### Q20: What are the key considerations for 3D IC physical design?

**A:** (1) TSV (through-silicon via) planning: TSVs are 5-10um diameter with 5-15um keep-out zones due to mechanical stress, consuming significant area. Need to carefully budget TSV count and locations. (2) Thermal: bottom die has higher thermal resistance (heat must pass through top die or interposer), requiring thermal-aware placement. (3) Power delivery: power must reach both dies through TSVs or micro-bumps; IR drop analysis becomes 3D. (4) Timing: TSV delay (~1-5ps) and capacitance (~50-200fF) must be modeled in STA. (5) Testing: access to bottom die is through top die; need test strategies for each die independently and combined. (6) Partitioning: deciding which logic goes on which die to minimize TSV count while meeting thermal and timing requirements. (7) Alignment: micro-bump pitch (40-100um) limits the density of inter-die connections.

---

### Q21: Describe a routing congestion scenario you've encountered and how you resolved it.

**A:** [Framework answer] A common scenario: a block with 16 SRAM macros has severe congestion in the channels between macros. The GRC (global routing congestion) map shows 150% overflow on horizontal M4 in the channel regions. Root cause: all data bus wires (256-bit bus) must route through narrow channels between macros. Resolution: (1) Widened channels by moving macros further apart (traded 3% area increase). (2) Applied cell padding of 2 sites in congested regions. (3) Added routing blockages on M2/M3 in the channels to push routing to higher layers. (4) Reoriented two macros so data pins faced different directions, distributing routing demand. (5) Used placement density constraints to prevent cells from filling the channels. Result: overflow reduced to 0%, with only 1% frequency impact from slightly longer wires.

---

### Q22: What is the difference between a halo and a blockage in macro placement?

**A:** A halo is a relative keep-out zone around a macro — it moves with the macro if the macro is repositioned. It's specified as a distance from each macro edge (e.g., 5um on all sides). It prevents standard cells from being placed too close to the macro, ensuring adequate routing channels for macro pins. A blockage is an absolute coordinate-based region where placement or routing is prohibited. Blockages don't move with any object. Hard placement blockages completely prevent cell placement; soft blockages discourage but allow placement if necessary; partial blockages limit density (e.g., 50% utilization). Routing blockages prevent routing on specific layers in a region. In practice, halos are used around macros, while blockages are used for fixed regions (analog areas, special routing reservations).

---

### Q23: How does scan chain reordering interact with placement?

**A:** Scan chains are inserted during DFT (Design for Testability) synthesis, creating a serial shift register through all scan FFs. Initially, the chain order is arbitrary, meaning physically distant FFs may be adjacent in the chain, creating long scan routing. Scan chain reordering (during or after placement) rearranges the chain to follow physical proximity, reducing total scan wire length by 30-60%. This improves congestion (fewer long wires) and hold timing in scan mode (shorter wires = less delay variation). The tool groups FFs in the same scan chain that are physically nearby and chains them in a serpentine pattern across the block. Constraint: must maintain the same number of scan chains and chain lengths. Tools like ICC2 and Innovus perform this automatically as part of the placement flow.

---

### Q24: How would you handle a scenario where post-route STA shows 200ps WNS on a path that was clean post-CTS?

**A:** This 200ps degradation from post-CTS to post-route indicates significant routing issues. Debug steps: (1) Check if the path has unexpected wire detours — congestion may have caused the router to take longer paths. (2) Check crosstalk: SI analysis may show coupling delay from adjacent aggressors. (3) Check extraction: compare post-CTS estimated wire delay vs post-route extracted delay. (4) Look at specific stages: identify which cell-to-cell segments degraded most. Fixes depend on root cause: if congestion-driven detour, apply routing guides or NDR for the critical net; if crosstalk, shield the victim net or add spacing; if extraction mismatch, the pre-route estimate was too optimistic (adjust RC estimation during placement). For 200ps, likely need a combination: route the critical net on higher metals (lower R), shield it, and possibly add a pipeline register (if architecture allows) or run CCD to redistribute the timing budget.

---

### Q25: Explain the sign-off flow: what checks must pass before tapeout?

**A:** All of the following must be clean (zero violations) for tapeout:

1. **STA**: All setup and hold timing met across all MCMM scenarios (10-30 corners × modes). POCV/AOCV derating applied. SI-aware analysis. Max transition and max capacitance clean.
2. **DRC**: Zero violations on all layers, including multi-patterning (LELE/SADP) rules, density checks, and antenna rules.
3. **LVS**: Layout matches schematic with zero errors. All devices match, all nets connected correctly.
4. **ERC (Electrical Rule Check)**: No floating gates, no well connectivity issues, ESD (Electrostatic Discharge) path complete.
5. **IR drop**: Static IR < 5% VDD at all nodes. Dynamic IR within budget for worst-case switching scenarios.
6. **EM**: All wires and vias within current density limits per Black's equation for target lifetime (e.g., 10-year reliability).
7. **Metal density**: All layers within foundry-specified density range (with dummy fill).
8. **Formal verification (LEC)**: Post-layout netlist matches post-synthesis netlist.

Additionally: power analysis confirms total power within package/board thermal limits, and the GDSII file passes foundry's incoming inspection checks.

---

### Q26: What is the impact of VDD scaling on physical design at advanced nodes?

**A:** Lower VDD (e.g., 0.5V at 3nm vs 1.0V at 28nm) impacts PD in several ways: (1) Reduced noise margins — 5% IR drop budget at 0.5V is only 25mV, demanding a denser power grid with more metal consumed for power straps. (2) Slower gates — lower VDD increases gate delay, requiring either higher performance cells (more leakage) or additional pipeline stages. (3) Increased sensitivity to variation — at 0.5V, a 25mV VT variation is 5% of VDD (vs 2.5% at 1.0V), making OCV derating more aggressive. (4) SRAM Vmin concerns — SRAMs have minimum operating voltage that may be higher than logic Vmin, requiring separate SRAM power domains or assist circuits. (5) Leakage becomes a larger fraction of total power, making VT optimization (multi-VT) critical in placement/optimization.

---

## Signal Integrity and Reliability — Senior Engineer Deep Dive

*From [Signal_Integrity_Reliability.md](../05_Backend_Physical_Design/02_Signal_Integrity_Reliability.md)*

**Q1: What is crosstalk and what causes it?**

Crosstalk is unintended signal coupling between adjacent wires through parasitic coupling
capacitance. When an aggressor wire switches, current flows through the coupling
capacitance and induces a voltage change on the neighboring victim wire. At advanced nodes
(7nm and below), coupling capacitance can be 50-80% of total wire capacitance because wire
spacing has shrunk much faster than wire height.

**Q2: Explain the Miller effect in crosstalk delay.**

When aggressor and victim switch in opposite directions, the voltage change across the
coupling capacitance is 2×VDD, making the effective coupling capacitance 2×Cc (Miller
factor = 2). This additional load slows the victim. When they switch in the same direction,
the effective coupling is 0 (Miller factor = 0), speeding up the victim. The worst case
for setup timing is opposite switching; for hold timing, it's same-direction switching.

**Q3: How does SI-aware STA work?**

PrimeTime SI reads the design with coupling capacitances from parasitic extraction. For
each data path, it identifies aggressor nets whose switching windows overlap with the
victim. For setup analysis, it assumes worst-case (opposite switching) on data paths and
best-case (same switching) on clock paths. It uses timing windows to filter out physically
impossible aggressor-victim combinations, avoiding excessive pessimism.

**Q4: Write Black's equation and explain each term.**

$MTTF = A \times J^{-n} \times \exp(E_a/(kT))$. A is a material constant, J is current density (higher
J → shorter life), n is the current density exponent (1 for void nucleation, 2 for void
growth), Ea is activation energy (higher → more EM-resistant, Cu ~0.7-0.9 eV vs Al
~0.5-0.7 eV), k is Boltzmann constant, T is temperature (exponential dependence — 10°C
increase can halve lifetime).

**Q5: What is the difference between static and dynamic IR drop?**

Static IR drop is the average voltage drop due to DC current flowing through the resistive
power grid — it's constant over time. Dynamic IR drop is the instantaneous voltage drop
during switching events (especially clock edges), which can be much worse due to high peak
currents and package inductance (Ldi/dt). Dynamic IR drop is typically 2-3× worse than
static and causes localized timing failures.

**Q6: How do you fix IR drop violations?**

(1) Widen power straps to reduce grid resistance. (2) Add more power vias (especially
multi-cut vias). (3) Insert decoupling capacitor cells near hotspot regions. (4) Add more
power pads/bumps for shorter current paths. (5) Redistribute high-power cells to avoid
current concentration. (6) Increase metal layer allocation for power grid.

**Q7: What is self-heating EM and why is it a concern at advanced nodes?**

At advanced nodes, metal wires are so thin and surrounded by low-k dielectric (poor
thermal conductor) that Joule heating significantly raises the wire temperature. This
elevated temperature is fed back into Black's equation, dramatically reducing EM lifetime.
A wire might pass EM at the nominal temperature but fail when self-heating adds 10-30°C.
Modern EM signoff tools (RedHawk, Voltus) include self-heating models.

**Q8: How do you design the power grid for an SoC?**

Start from power budget (total current per domain). Size top-metal straps to carry the
required current within EM limits. Create a mesh with redundant paths. Use via arrays at
strap intersections. Size lower-metal connections to standard cells. Run static IR drop
analysis and iterate strap width/pitch. Add decap cells to reduce dynamic IR drop. Verify
EM on all grid elements. Typical guideline: power grid uses 10-20% of routing resources.

**Q9: What is NBTI and how does it affect timing signoff?**

Negative Bias Temperature Instability affects PMOS (P-channel MOS) transistors during normal operation
(VGS, the gate-to-source voltage, < 0). Holes trapped in the gate oxide increase Vth over time, degrading drive
current and slowing the transistor. After 10 years at 125°C, ΔVth can be 30-50 mV.
Timing signoff must use "aged" library models that include this degradation, ensuring the
chip still meets timing at end-of-life.

**Q10: How do you prevent EM in clock networks?**

Clock nets carry bidirectional current (switching 0→1→0 every cycle) at high frequency.
Use AC/RMS (root mean square) EM limits (higher than DC limits). Ensure clock buffers/inverters have adequate
drive strength (reduces current per wire). Use NDR (double-width, double-spacing) for
clock wires — reduces current density and resistance. Multi-cut vias at every clock buffer
connection. Monitor via EM especially carefully (single-cut vias are weakest links).

**Q11: What is thermal runaway?**

When a chip's leakage power creates a positive feedback loop: higher temperature →
exponentially more leakage → more heat → higher temperature. If heat generation exceeds
dissipation capacity, the temperature rises uncontrollably until the chip fails. Prevention
requires thermal sensors, DVFS (Dynamic Voltage and Frequency Scaling) throttling, and proper heat sink design. It's especially
dangerous during stress testing or in systems with inadequate cooling.

**Q12: How does crosstalk affect hold timing?**

For hold analysis, the concern is data arriving too fast at the capture register. Crosstalk
that speeds up the data path (same-direction switching, Miller factor = 0) makes data
arrive earlier, potentially causing hold violations. Simultaneously, crosstalk that slows
the clock path (opposite-direction switching on clock wire) makes the capture clock arrive
later. Both effects reduce hold slack. This is why hold violations can appear after routing
even if they weren't present post-synthesis.

**Q13: What is antenna effect and how is it related to reliability?**

During metal etching, long metal lines connected to thin gate oxide accumulate charge from
the plasma process. The charge-to-gate-oxide-area ratio (antenna ratio) must stay within
limits to prevent Fowler-Nordheim tunneling damage to the oxide. If the antenna ratio
exceeds the process limit, the oxide degrades, causing Vth shift and TDDB (Time-Dependent Dielectric Breakdown) failure. Fixes:
add protection diodes, break long wires across metal layers, reroute.

**Q14: What is the difference between DC EM and AC EM limits?**

DC EM applies to wires carrying unidirectional current (e.g., power grid where current
always flows from pad to cells). AC EM applies to wires carrying bidirectional current
(e.g., signal and clock nets that switch between 0 and VDD). AC EM limits are typically
2-3× higher than DC limits because the alternating current direction allows partial
healing (atoms migrate back). The relevant metric for AC EM is the RMS current, not
average or peak.

**Q15: How do you calculate the required decoupling capacitance?**

$C = I_{peak} \times \Delta t / \Delta V_{allowed}$. Determine peak current from switching activity analysis
(e.g., clock edge current), the duration of the current pulse (typically ~100ps for
clock edge), and the allowed voltage drop (typically 5-10% of VDD). Place decap cells
within ~50-100μm of hotspots (beyond that, wire resistance limits effectiveness). Verify
with dynamic IR drop simulation and iterate.

**Q16: What is via EM and why are multi-cut vias important?**

Single-cut vias have very limited current-carrying capacity (0.1-0.2 mA per cut) due
to their small cross-sectional area. If a signal or power net exceeds this limit, EM
causes void formation at the via interface → open circuit. Multi-cut vias (2, 4, or
more cuts in parallel) divide the current, keeping each via below its EM limit. DFM (Design for Manufacturability)
rules recommend multi-cut vias wherever space permits. In power grid, via arrays (many
cuts at strap intersections) are mandatory.

**Q17: What is the noise budget and how is it allocated?**

The total noise margin (~0.35V at VDD=0.7V) must accommodate all noise sources: crosstalk
coupling (~40%), power supply noise / IR drop (~30%), and process variation (~30%). Each
source is allocated a budget, and the design must keep each noise type within its budget.
If one source exceeds its allocation (e.g., severe IR drop), the margin for others shrinks,
potentially causing failures. Noise budget management is critical for low-voltage designs.

**Q18: What tools are used for reliability signoff?**

EM/IR: RedHawk-SC (Synopsys), Voltus (Cadence) — analyze power grid integrity, current
density, voltage drop. Aging: MOSRA (MOS Reliability Analysis) in PrimeTime (Synopsys), aging models in Tempus
(Cadence) — BTI (Bias Temperature Instability) and HCI (Hot Carrier Injection) degradation analysis. ESD: commercial ESD checkers integrated
with DRC tools (Calibre, IC Validator). Thermal: ANSYS RedHawk-SC Electrothermal,
Voltus-Fi, Cadence Celsius — chip and package thermal analysis.

**Q19: How does temperature inversion affect reliability analysis?**

At low VDD (< 0.8V), the fastest corner shifts from high-temperature to low-temperature
(because Vth reduction at high T is more beneficial than mobility loss). This means hold
timing must be checked at low temperature (fast corner), not just high temperature. For
reliability, this complicates analysis because aging and EM models assume worst-case at
high temperature, but timing margins may be worse at low temperature.

**Q20: What is stress migration and how does it differ from EM?**

Stress migration (SM) is void formation in metal due to mechanical stress, not electrical
current. Thermal cycling creates stress between metal and dielectric (different thermal
expansion coefficients). Voids form at stress concentration points (via interfaces, line
ends). Unlike EM, SM occurs even without current flow — it can affect idle metal lines.
SM is checked separately from EM and has its own lifetime models based on stress gradient
and temperature cycling range.

