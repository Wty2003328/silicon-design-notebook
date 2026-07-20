# 07 — Manufacturing and Bringup: Interview Questions

Consolidated interview Q&A and worked problems from every page in `07_Manufacturing_and_Bringup/`, moved here to keep topic pages focused on concepts. Each section links back to its source page for full context.

---

## Semiconductor Fabrication Process — Senior Engineer Deep Dive

*From [Fabrication_Process.md](../07_Manufacturing_and_Bringup/01_Fabrication_Process.md)*

**Q1: Walk through the major steps in CMOS (Complementary Metal-Oxide-Semiconductor) fabrication.**

STI isolation → well implants → Vt adjust → gate oxide (or high-k deposition) →
gate patterning → halo implant → spacer formation → S/D (source/drain) implant → activation anneal →
silicidation → contact formation → M1 (Cu damascene) → repeat for M2-MN → passivation
→ pad opening. For FinFET (Fin Field-Effect Transistor), add fin patterning before STI (shallow trench isolation) and replace gate with
replacement metal gate (RMG) process.

**Q2: What is the Rayleigh equation and why does it matter?**

$CD_{min} = k_1 \times \lambda / NA$ (where $CD_{min}$ is the minimum printable critical dimension, $k_1$ a process/resist factor, $\lambda$ the exposure wavelength, and $NA$ the numerical aperture). It sets the fundamental resolution limit of lithography. To print
smaller features: reduce wavelength (193nm → 13.5nm EUV (Extreme Ultraviolet)), increase NA (immersion,
high-NA EUV), or reduce k1 (OPC (optical proximity correction), phase shift masks, multi-patterning). At 193nm
immersion, the resolution wall (~36nm half-pitch) drove the need for multi-patterning
and eventually EUV.

**Q3: Explain the difference between DUV (Deep Ultraviolet) and EUV lithography.**

DUV uses 193nm ArF (argon fluoride) excimer laser with refractive optics. EUV uses 13.5nm wavelength from
tin plasma source with reflective optics in vacuum. EUV provides ~14× better resolution,
eliminating the need for multi-patterning on many layers. But EUV tools cost $150M+, have
lower throughput, and have unique challenges (mask defects, pellicle, stochastic effects).

**Q4: What is SADP and why is it preferred over LELE?**

SADP (Self-Aligned Double Patterning) deposits conformal spacers on mandrel sidewalls,
then removes the mandrel. The spacers define features at half the mandrel pitch. Unlike
LELE (Litho-Etch-Litho-Etch; two separate lithography steps), SADP has no overlay error between the two sets of
features because they're self-aligned. This gives better pitch uniformity and is preferred
for critical layers like fins and M1.

**Q5: Why was copper adopted over aluminum for interconnects?**

Cu has ~40% lower resistivity than Al (1.7 vs 2.7 μΩ·cm), which reduces RC (resistance-capacitance) delay and
allows thinner wires. Cu also has ~100× better electromigration resistance than Al. But
Cu cannot be plasma-etched (forms non-volatile etch products), requiring the damascene
process (deposit dielectric → etch trenches → fill Cu → CMP (Chemical-Mechanical Polishing)). Cu also diffuses rapidly
through SiO2, requiring barrier layers (TaN/Ta).

**Q6: What is the dual damascene process?**

It forms both via and trench in a single Cu fill step. Process: deposit dielectric → etch
via holes → etch trenches (aligned to vias) → deposit barrier (TaN/Ta) → deposit Cu seed
→ electroplate Cu → CMP. This is more efficient than single damascene (which requires
separate fill and CMP for via and trench).

**Q7: Explain ALD and why it's critical at advanced nodes.**

Atomic Layer Deposition grows films one atomic layer at a time using alternating
self-limiting precursor pulses. Growth rate is ~1 Å/cycle. It's critical because:
(1) sub-nm thickness control for gate dielectric (HfO2), (2) perfect conformality for
coating 3D structures like FinFET gate trenches, (3) pinhole-free films for barrier layers.
The trade-off is very slow deposition rate.

**Q8: What is the replacement metal gate (RMG) process and why is it used?**

RMG (gate-last) forms a dummy polysilicon gate first, completes S/D processing and high-
temperature anneal, then removes the dummy gate and deposits the real high-k dielectric
and metal gate at low temperature. This prevents thermal degradation of the high-k/metal
gate stack. Without RMG, the 1050°C S/D anneal would damage the gate stack, causing Vt
shift and reliability problems.

**Q9: How are FinFET source/drains formed differently from planar?**

Planar CMOS uses ion implantation for S/D doping. FinFET fins are too narrow for
implantation (ions would amorphize or damage the fin). Instead, FinFETs use selective
epitaxial growth (SEG): recess the fin in S/D regions, then grow doped epitaxial
Si:P (NMOS) or SiGe:B (PMOS). This also provides strain engineering for mobility boost.

**Q10: What is CMP dishing and erosion? How do you mitigate them?**

Dishing: Soft Cu polishes faster than hard dielectric, causing recessed metal. Worse for
wide lines. Erosion: In dense metal regions, dielectric between lines is over-polished.
Mitigation: (1) Slot wide metals to limit Cu exposure, (2) Insert dummy metal fill for
uniform density, (3) Design rules limiting maximum metal width, (4) Minimum/maximum metal
density per layer (typically 20-80%).

**Q11: What determines defect density and how does it affect yield?**

Defect density D (defects/cm²) comes from particles, pattern errors, and process
excursions. Yield $Y = \exp(-D \times A)$ (with $A$ the die area) for Poisson model. A larger die has lower yield because
each die has more chance of containing a defect. This is why new process nodes launch
with smaller dies first, and why chiplet architectures improve effective yield by using
multiple smaller dies instead of one large monolithic die.

**Q12: What is high-NA EUV and when will it be used?**

High-NA EUV increases the numerical aperture from 0.33 to 0.55, improving resolution
by ~1.7× (CD_min ≈ 8nm half-pitch). It uses anamorphic optics (different magnification
in X and Y). First tools from ASML (EXE:5000 series) are being installed in 2025-2026.
Intel is the first to deploy High-NA EUV at its D1X fab in Oregon, targeting the Intel
14A node. The tools cost $350M+ and require new mask infrastructure. High-NA EUV may
eliminate multi-patterning on most layers at 2nm and below, significantly reducing
process complexity and cost per layer.

**Q13: What is strain engineering in FinFETs?**

SiGe S/D epitaxy in PMOS introduces compressive strain on the Si channel (Ge atoms are
larger than Si), increasing hole mobility by 30-50%. Si:P S/D epitaxy in NMOS introduces
tensile strain, increasing electron mobility by 10-20%. At advanced nodes, stress liners
(SiN) and embedded stressors are also used. Strain engineering is critical — without it,
FinFET performance would be significantly worse.

**Q14: What is the body effect in FinFETs compared to planar?**

In planar bulk CMOS, the body effect increases Vth when VSB > 0 (source above body
potential). In FinFETs, the body effect is much weaker because the thin, fully-depleted
fin is mostly controlled by the gate rather than the substrate. The body is effectively
floating (or weakly coupled). This is advantageous for stacked transistors in logic gates
and for SRAM (Static Random-Access Memory) bit-cell stability.

**Q15: Explain the concept of EOT and why it's used.**

EOT (Equivalent Oxide Thickness) expresses the capacitance of a high-k dielectric in
terms of the SiO2 thickness that would give the same capacitance.
$EOT = t_{physical} \times (k_{SiO2} / k_{high\text{-}k})$. A 2nm HfO2 (k≈20) has $EOT = 2 \times 3.9/20 = 0.39\text{ nm}$.
This gives the gate capacitance of a 0.39nm SiO2 layer but is physically 2nm thick, so
there's no quantum tunneling. EOT allows fair comparison across different dielectric
materials.

**Q16: What causes line-edge roughness (LER) and why is it a problem?**

LER is random variation in the edge position of patterned features, caused by the
stochastic nature of photon absorption and chemical reactions in photoresist. At advanced
nodes where line widths are 10-20nm, LER of 2-3nm represents 10-30% of the line width,
causing significant variation in transistor gate length (and thus Vt, drive current, and
leakage). EUV has worse LER than DUV due to lower photon counts per feature (higher
photon energy → fewer photons per mJ of dose).

**Q17: What are stochastic effects in EUV lithography?**

At 13.5nm wavelength, each EUV photon has ~14× more energy than a 193nm photon. So for
the same dose, there are 14× fewer photons per feature. In very small features, the
number of absorbed photons follows Poisson statistics — if only 100 photons form a
feature, the random variation is √100/100 = 10%. This causes stochastic defects: missing
contacts, bridging, line breaks. Mitigation: increase dose (but reduces throughput) or
use chemically-amplified resists with higher sensitivity.

**Q18: Why are there so many metal layers at advanced nodes?**

Lower metal layers have tight pitch (28nm at 7nm) for local connections within standard
cells. Upper layers have relaxed pitch for global signals, power, and clock distribution.
More layers are needed because: (1) transistor density has increased dramatically, (2)
tight-pitch lower metals have higher resistance (limited current capacity), (3) power
grid requires thick upper metals, (4) clock mesh may need dedicated metal layer. 7nm uses
12-15 layers; 3nm may use 15-17.

**Q19: What is the difference between gate-first and gate-last integration?**

Gate-first: Real gate stack (high-k + metal) formed before S/D processing. Simpler flow
but gate materials must withstand ~1050°C anneal. Gate-last (RMG): dummy poly gate formed
first, S/D processing done, then dummy gate replaced with real high-k + metal gate.
More complex flow but avoids thermal damage to gate stack. Gate-last has been standard
since 32nm (Intel) because gate-first couldn't achieve the required Vt and reliability
targets with HKMG (High-K Metal Gate).

**Q20: How does the wafer yield relate to die size and defect density?**

$Y = \exp(-D \times A)$. If defect density D = 0.1/cm² and die area A = 50mm² = 0.5cm²,
Y = exp(-0.05) = 95.1%. But if A = 500mm² = 5cm² (large SoC (System-on-Chip)), Y = exp(-0.5) = 60.7%.
This is why: (1) new processes launch with small dies, (2) chiplets (multiple small dies)
can have much better effective yield than monolithic large dies, (3) yield improvement is
the #1 priority in process development.

**Q21: How is a GAA (Gate-All-Around) nanosheet transistor fabricated differently from a FinFET?**

The key difference is the channel release step. In GAA fabrication, a Si/SiGe superlattice
is grown epitaxially (alternating Si channel layers and sacrificial SiGe layers). After fin
patterning and inner spacer formation, the sacrificial SiGe is selectively etched away
(wet etch with >100:1 selectivity), leaving suspended Si nanosheets. The gate dielectric
and metal gate are then deposited by ALD, wrapping all 4 sides of each nanosheet. This is
fundamentally different from FinFET where the fin is a single vertical structure with the
gate wrapping 3 sides. The GAA process requires: (1) precise epitaxial growth of the
superlattice, (2) highly selective SiGe etch, (3) careful inner spacer formation to
isolate gate from S/D. Samsung was first to production (3nm MBCFET (Multi-Bridge-Channel FET), 2022), followed by
TSMC N2 (2025) and Intel 18A RibbonFET (2025).

**Q22: What is backside power delivery and why is it needed?**

As transistors scale, the frontside metal layers become increasingly congested with both
signal routing and power delivery competing for the same metal tracks. IR (current-resistance) drop on the
power grid worsens because narrower wires have higher resistance. Backside power delivery
solves this by routing VDD/VSS (power and ground) through the back of the wafer: the wafer is thinned,
deep vias (TSV-like) are etched from the backside to reach frontside transistor terminals,
and a power distribution network is formed on the backside using thick metal layers.
Intel PowerVia (first in Intel 18A, 2025) demonstrates up to 50% IR drop reduction and
frees up frontside M0/M1 layers entirely for signal routing, improving cell density.
The trade-off: wafer thinning adds process complexity and cost, and thermal management
becomes more challenging since the backside is no longer available for heat removal.

---

## IC Packaging and Advanced Packaging — Senior Engineer Deep Dive

*From [IC_Packaging.md](../07_Manufacturing_and_Bringup/02_IC_Packaging.md)*

**Q1: Compare wire bonding and flip chip. When would you choose each?**

Wire bonding is cheaper, peripheral-only I/O, higher inductance (~1 nH/wire), limited
to ~500 pins. Flip chip has area-array I/O, lower inductance (~0.1 nH/bump), supports
10,000+ connections, better thermal (die face-down). Choose wire bond for low-cost,
low-pin-count packages (sensors, microcontrollers). Choose flip chip for high-performance
(GPUs, CPUs, SoCs) or high-pin-count applications.

**Q2: What is a TSV? What are its key parameters?**

A Through-Silicon Via is a vertical copper connection through a silicon wafer. Key
parameters: diameter (5-10 μm interposer, 1-5 μm 3D IC), depth (50-100 μm interposer,
5-50 μm 3D), aspect ratio (5:1 to 20:1), resistance (~10-50 mΩ), capacitance (~10-50 fF),
inductance (~10-50 pH). TSVs enable vertical integration in 2.5D/3D IC packaging with
much better electrical performance than wire bonds.

**Q3: What is CoWoS and how does it work?**

TSMC CoWoS (Chip-on-Wafer-on-Substrate) is a 2.5D packaging technology. Multiple dies
(e.g., GPU + HBM (High Bandwidth Memory) stacks) are mounted on a silicon interposer with fine-pitch RDL (redistribution layer; 0.4-2μm
L/S (line/space)) for die-to-die connections, and TSVs for vertical connection to the package substrate
below. CoWoS-S uses a standard silicon interposer, CoWoS-R uses RDL-only (organic),
and CoWoS-L uses a larger interposer with embedded silicon bridges.

**Q4: Why are chiplets becoming popular? What are the trade-offs?**

Chiplets improve effective yield (small dies yield better than one large die), enable
mixing process nodes (3nm logic + 7nm I/O), reduce design cost (reuse chiplets across
products), and overcome the reticle size limit. Trade-offs: die-to-die interface adds
latency (2-5 ns), bandwidth is lower than on-chip wires, packaging cost increases,
design complexity (multi-die floor planning, power delivery, thermal).

**Q5: Explain HBM and why it's critical for AI accelerators.**

HBM stacks 8-12 DRAM (Dynamic Random-Access Memory) dies vertically using TSVs, providing a 1024-bit wide interface
(vs DDR5's 64 bits). This gives enormous bandwidth (>1 TB/s per stack). AI workloads
are memory-bandwidth limited — transformers need to move massive weight matrices
between memory and compute. HBM provides 5-10× the bandwidth of GDDR6 in a compact
form factor, mounted on the same interposer as the GPU/accelerator die.

**Q6: What is hybrid bonding and why is it revolutionary?**

Hybrid bonding directly connects Cu pads between two dies without solder bumps. At 1-10 μm
pitch (vs 40-55 μm for μbumps), it enables 100× more connections per area. This gives
unprecedented inter-die bandwidth and lowest latency. Used in AMD V-Cache (3D SRAM cache),
Sony image sensors, and planned for future chiplet-to-chiplet connections. The key
challenge is surface preparation — requires atomic-level flatness for reliable bonding.

**Q7: What is UCIe (Universal Chiplet Interconnect Express) and how does it compare to proprietary die-to-die links?**

UCIe is an open standard for chiplet interconnect, defining physical, link, and protocol
layers. It supports both standard package (100-130 μm pitch) and advanced package
(25-55 μm pitch). Benefits: interoperability between vendors' chiplets, standard
verification methodology. Compared to proprietary links (AMD Infinity Fabric, Apple
UltraFusion): UCIe may have slightly higher overhead but enables a chiplet ecosystem
where third-party chiplets can be integrated.

**Q8: What are the thermal challenges in 3D IC stacking?**

In 3D stacking, the top die's heat must conduct through the bottom die to reach the
heat sink. This creates a higher thermal resistance path. The bonding interface (solder,
underfill, or hybrid bond) adds thermal resistance. Result: top die runs hotter, which
increases leakage and reduces reliability. Solutions: thermal TSVs (Cu-filled vias for
heat conduction), power-aware floorplanning, splitting high-power blocks across layers,
and advanced cooling (microfluidic channels).

**Q9: What is the yield advantage of chiplets?**

For Poisson defect model: $Y = \exp(-D \times A)$. A monolithic 500mm² die at D=0.1/cm² yields
60.7%. Two 250mm² chiplets each yield 77.9%, and the combined good-pair probability
is 77.9%² = 60.7% — same if you need both to work. BUT: with testing, you only combine
known-good dies (KGD), so yield $= Y_{chiplet} \times (1 - r_{esc}) \approx 77.9\%$ (with $r_{esc}$ the test-escape rate). The real
advantage grows with more chiplets and higher defect density.

**Q10: How does power delivery work in a multi-die package?**

Power flows: VRM (Voltage Regulator Module) → PCB (Printed Circuit Board) → BGA (Ball Grid Array) balls → package substrate → C4 bumps → interposer (if 2.5D)
→ μbumps → die. Each die has its own power domain (possibly different voltages). The
power delivery network (PDN) impedance must be extremely low (< 0.2 mΩ for 200A at
35mV drop). This requires massive parallelism in bumps and vias, multi-level decoupling
(on-die MOS (metal-oxide-semiconductor) caps, on-package MIM (Metal-Insulator-Metal) caps, on-board MLCCs (Multi-Layer Ceramic Capacitors)), and careful PDN design with
full-wave electromagnetic simulation.

**Q11: What is the difference between 2.5D and 3D IC?**

2.5D: Dies placed side-by-side on an interposer (silicon or organic). Dies don't stack
vertically. Inter-die connections through interposer RDL. Example: GPU + HBM on CoWoS.
3D: Dies stacked vertically, connected by TSVs or hybrid bonds through the silicon.
Much higher connection density but worse thermal and more complex. Example: AMD V-Cache,
HBM stacks. Many "3D" products are actually 2.5D with 3D memory stacks.

**Q12: What is fan-out packaging and when is it used?**

Fan-out WLP (Wafer-Level Packaging) uses a reconstituted wafer where dies are embedded in mold compound with an
RDL that extends beyond the die boundary. This allows more I/O than the die area alone
would permit. TSMC InFO (Integrated Fan-Out) is a leading fan-out technology, used in Apple's A-series chips.
Fan-out is cheaper than silicon interposer for applications that don't need the extreme
die-to-die bandwidth of CoWoS. It's ideal for mobile SoCs and mid-range packages.

**Q13: What challenges exist at <10μm hybrid bonding pitch?**

At sub-10μm pitch: (1) Alignment accuracy must be <500nm (extremely tight for D2W).
(2) Surface cleanliness — a single particle can cause bond failure. (3) Pad size is so
small that Cu grain structure matters for bond quality. (4) Thermal expansion mismatch
can cause misalignment during anneal. (5) Testing at this density is challenging — can't
probe individual bonds. (6) Design rules for keep-out and dummy patterns become complex.

---

## Tapeout and Post-Silicon Bring-up

*From [Tapeout_and_Post_Silicon_Bringup.md](../07_Manufacturing_and_Bringup/03_Tapeout_and_Post_Silicon_Bringup.md)*

**Q: First silicon draws huge current at power-on. First hypotheses?** A short — power-to-ground bridging that [LVS](../06_Signoff/03_Physical_Verification_DRC_LVS.md) should have caught (or in an analog/IO area it didn't cover), a missing well tie, or a bus-contention/X-state driving fight (often a reset bug leaving outputs enabled). Drop voltage, use thermal imaging to localize the hot spot, scan-dump if it'll come up enough.

**Q: Why is DFT (Design-for-Test) critical for bring-up, not just production test?** Because silicon has no internal visibility. Scan lets you freeze and read out all flop state to see *where* a non-booting chip died; trace buffers capture internal activity. Without good DFT, debugging first silicon is nearly impossible — you're guessing at a black box.

**Q: What's a shmoo plot and what does it tell you?** A 2-D pass/fail map over voltage × frequency (often × temperature). It reveals the real operating region, the actual fmax/Vmin vs STA (Static Timing Analysis)'s prediction, and the *shape* of failures — a clean diagonal edge suggests a speed path; a ragged or unexpected boundary suggests an electrical/SI (signal integrity) or marginal-timing issue.

**Q: A functional bug is found in silicon. Options?** If it can be fixed by changing logic reachable with spare cells, do a **metal-layer ECO (Engineering Change Order)** (cheap, weeks). If it needs base-layer changes, it's a **full respin** (expensive, months). Sometimes you can **work around it in firmware/software** and ship, fixing it in the next spin — often the fastest path to revenue.

