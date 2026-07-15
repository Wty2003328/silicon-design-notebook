# Accelerator & DNN/NPU Simulators — Mapping, Dataflow, and Cycles

> **Prerequisites:** [Simulation_Methodology](01_Simulation_Methodology.md) (the paradigm/fidelity vocabulary this page instantiates for accelerators), [Performance_Modeling_and_DSE §9–11](../01_Performance_Modeling_and_DSE.md) (systolic-array cycle model, dataflow taxonomy, the NeuSim worked example this page extends).
> **Hands off to:** [Full_Chip_Modeling §4](../02_Full_Chip_Modeling.md) (composing a PE→array→chip→pod NPU model with power/thermal), the companion AI-infra notebook for GPU/TPU microarchitecture.

---

## 0. Why this page exists

A DNN accelerator has almost no branches, no speculation, and a tiny instruction set — so the CPU-simulator machinery ([Simulation_Methodology](01_Simulation_Methodology.md)) is the wrong tool. What decides an NPU's performance and energy is instead a **mapping problem**: given a layer (a loop nest over tensor dimensions) and a fixed array of MACs (multiply-accumulate units) with a small SRAM hierarchy, *which loop tiling, loop order, and spatial unrolling* you pick sets the data-movement volume, the array utilization, and therefore the latency and — dominantly — the energy. Accelerator simulators are built around that mapping problem, and they split into three paradigms that answer different questions at different cost. This page explains the shared mechanism (how a layer plus a mapping becomes a latency and an energy number), the dataflow taxonomy that mapping expresses, and the tools — analytical (Timeloop/Accelergy, MAESTRO), cycle-accurate (SCALE-Sim v3), and operator-level industrial (NeuSim, ONNXim) — and when each is the right one.

The single habit this page installs: **before you believe an accelerator PPA number, ask whether it came from an analytical mapping model, a cycle-accurate array model, or an operator-level graph model — because each is blind to a different thing.**

---

## 1. The three paradigms and when to use each

Accelerator simulation is stratified not by "how detailed" alone but by *what unit of work* the model reasons about. Three rungs, mapping onto the general fidelity ladder of [Simulation_Methodology §2](01_Simulation_Methodology.md):

| Paradigm | Unit of reasoning | What it computes | What it is blind to | Speed | Representative tools |
|---|---|---|---|---|---|
| **Analytical mapping / dataflow** | one layer + one mapping | access counts per memory level → energy; MACs → ideal cycles; searches the mapping space | cycle-level contention, DRAM scheduling, cross-layer/runtime effects | ~$10^4$–$10^6$ mappings/s | Timeloop+Accelergy, MAESTRO |
| **Cycle-accurate (array)** | one layer, cycle by cycle | array fill/drain + SRAM/DRAM stalls → real cycle counts, stall breakdown, achieved bandwidth | whole-graph scheduling, multi-tenant/runtime, exact RTL corner cases | ~KIPS–MIPS | SCALE-Sim v3 |
| **Operator-level industrial** | the whole model graph, per operator | per-operator time/energy, multi-core/multi-chip contention, perf→power→carbon→SLO | intra-operator cycle detail (compute is a closed-form or table) | fast (graph-scale, distributable) | NeuSim, ONNXim |

The rule mirrors the CPU rule ("use the fastest model that distinguishes the choices you are deciding between"), specialized for accelerators:

- **Choosing a dataflow or a buffer hierarchy, or minimizing energy per inference?** Analytical mapping — you need to search *millions* of mappings, and energy (not cycles) is the objective, so a closed-form access-count model is both sufficient and the only thing fast enough.
- **Sizing the array or the SRAM, or asking "am I compute- or memory-stall-bound on this layer?"** Cycle-accurate — only a time-stepped model produces *stall cycles* and *achieved DRAM bandwidth*, which is exactly where analytical models are optimistic (they assume the mapping's data arrives on time).
- **Planning a datacenter NPU, an LLM (large language model) serving deployment, or a multi-tenant chip?** Operator-level — the questions are whole-graph (parallelism strategy, batch, KV-cache traffic), multi-chip (collectives over the interconnect), and system (power, carbon, SLO), none of which a single-layer model sees. This is the [NeuSim §11](../01_Performance_Modeling_and_DSE.md) rung.

**These are complementary, not competing.** A realistic flow uses Timeloop to pick the mapping, SCALE-Sim to check the mapping's real stall behavior on the chosen array/SRAM, and an operator-level tool to roll the per-layer results into a graph- and pod-level PPA number.

---

## 2. From a layer + mapping to latency and energy — the shared mechanism

Every tool on this page, analytical or cycle-accurate, is a different realization of the same five-step reduction. Understanding it once demystifies all of them.

**Step 1 — the layer is a loop nest.** A convolution (or, after `im2col`, a GEMM — general matrix multiply) is a perfect nest over tensor dimensions. For a 2-D conv with batch $N$, output channels $K$, input channels $C$, output height/width $P\times Q$, filter $R\times S$, the total useful work is

$$\text{MACs} = N\,K\,C\,P\,Q\,R\,S,$$

with no data-dependent control flow — **the entire compute is known statically from the layer's shape.** That is why accelerator performance is an offline *analysis*, not an *execution*.

**Step 2 — the mapping tiles and unrolls the nest.** A mapping assigns, at each level of the memory/compute hierarchy (DRAM → global SRAM → PE-local register file → the MAC), a *tile size* for each loop, a *loop order* (permutation), and which loops are unrolled *spatially* across the PE array (the parallelism) versus *temporally* (streamed in time). The mapping is the accelerator's "program."

**Step 3 — reuse and access counts fall out of the mapping.** For each tensor $t$ (weights, inputs, partial sums) and each storage level $\ell$, the number of accesses is set by how many times the loops *outside* level $\ell$ re-fetch the tile held *inside* it:

$$\text{accesses}_{t,\ell} = (\text{iterations of loops above }\ell)\times(\text{tile footprint of }t\text{ at }\ell), \qquad \text{reuse}_{t,\ell} = \frac{\text{MACs touching }t}{\text{accesses}_{t,\ell}}.$$

A tensor that stays resident while many outer iterations reuse it has high reuse and few accesses; a poorly ordered nest re-streams it from a distant, expensive level. This is the whole game — **the mapping is chosen to keep the reused tensor in the cheapest level.**

**Step 4 — latency.** Ideal cycles are the useful MACs divided by the array's parallel capacity, inflated by utilization loss and pipeline fill/drain:

$$\text{cycles} \approx \frac{\text{MACs}}{P_{\text{MAC}}\cdot U} + \text{fill/drain} + \text{stall}_{\text{mem}}, \qquad U = U_{\text{spatial}}\cdot U_{\text{temporal}},$$

where $P_{\text{MAC}}$ = number of MAC units, $U_{\text{spatial}}$ = fraction of the array actually mapped (edge/quantization loss when a dimension does not fill the array), $U_{\text{temporal}}$ = fill/drain amortization $K/(K+2D)$ for a $D\times D$ array streaming a reduction of length $K$ ([Perf_Modeling §10.1](../01_Performance_Modeling_and_DSE.md)), and $\text{stall}_{\text{mem}}$ = cycles the array is starved because SRAM/DRAM could not deliver operands. **Analytical tools drop $\text{stall}_{\text{mem}}$ (they assume perfect delivery); cycle-accurate tools compute it — that difference is the whole reason both paradigms exist.**

**Step 5 — energy.** Energy is the sum over every action of its count times its per-action cost:

$$E = \sum_{\ell}\sum_{t} \text{accesses}_{t,\ell}\cdot e_{\ell} \;+\; \text{MACs}\cdot e_{\text{MAC}},$$

where $e_\ell$ = energy per access at level $\ell$ (a register read is ~$10^2\times$ cheaper than a DRAM access) and $e_{\text{MAC}}$ = energy per MAC. Because $e_{\text{DRAM}} \gg e_{\text{SRAM}} \gg e_{\text{RF}}$, **energy is dominated by where the accesses land, which the mapping controls — so the mapping is far more an energy decision than a cycle decision.** This is the accelerator analogue of the activity × per-event-energy model that McPAT applies to CPUs ([Full_Chip_Modeling §1.5](../02_Full_Chip_Modeling.md)); Accelergy (§4) is literally that model for accelerators.

The two families on this page are exactly the two ways to run steps 3–5: **analytical** tools compute the access counts in closed form from the loop bounds and search over mappings; **cycle-accurate** tools step the array in time and let the access pattern and the stalls emerge.

---

## 3. The dataflow taxonomy and why it is mostly an energy lever

A *dataflow* is the equivalence class of mappings that hold the same tensor stationary in the PEs. The three canonical ones — plus input-stationary, which several tools implement — are summarized as a reuse table in [Perf_Modeling §10.2](../01_Performance_Modeling_and_DSE.md); here is the mechanism behind that table, because it is what the mapping tools are searching over.

- **Weight-stationary (WS).** Each weight is loaded into a PE register and held while a stream of activations flows past, so a weight is read from SRAM once and reused across the whole activation stream. Minimizes *weight* movement. This is the TPU-style GEMM dataflow; it wins when there are many activations per weight (large batch or large spatial map).
- **Output-stationary (OS).** Each output/partial-sum accumulates *in place* in a PE accumulator across the entire $K$ reduction, so partial sums are never spilled to and reloaded from SRAM. Minimizes *partial-sum* movement (which is read-modify-write traffic, the most expensive kind). Wins when the reduction dimension $K$ is long.
- **Row-stationary (RS).** Eyeriss's dataflow: a 1-D convolution primitive (one filter row × one ifmap row → one psum row) is kept resident in each PE, and the 2-D convolution is tiled across the PE array so that weights, inputs, *and* partial sums are all reused locally. It balances all three reuse types rather than maximizing one, which is why Eyeriss reported it **1.4×–2.5× more energy-efficient than WS/OS/no-local-reuse on AlexNet** — a scoped, workload-specific result, not a global optimum.

The load-bearing point: **the dataflow changes cycles only modestly (via utilization) but changes energy substantially (via which level absorbs the accesses).** Two mappings can have identical MAC counts and near-identical cycle counts yet differ 2–3× in energy because one keeps psums in accumulators and the other spills them to SRAM. That is precisely why an analytical *energy* model (§4–5) is the primary dataflow-selection tool and a cycle model (§6) is the confirmation step, not the other way around.

---

## 4. Analytical mapping + energy — Timeloop and Accelergy

**Timeloop** (MIT, Parashar et al., ISPASS 2019) is the reference *mapper*. It describes the accelerator not as RTL but as an abstract **hierarchy of storage and compute levels** (DRAM, global buffer, PE-local buffers, the arithmetic units), each with a capacity, a fanout (spatial parallelism), and a bandwidth. Given a layer's dimensions and that hardware template, Timeloop constructs the **mapspace** — every legal combination of per-level tile sizes, loop permutations, and spatial partitionings — and searches it (exhaustively for small spaces, or by random/heuristic sampling for large ones). For each candidate mapping it evaluates steps 3–4 of §2 analytically: it computes, per tensor and per level, the exact number of reads/writes/updates (the "action counts") and the cycle count under an idealized bandwidth model, then keeps the mapping that minimizes the chosen objective (energy, energy-delay product, or cycles). **Its output is the optimal mapping plus a full per-level access-count breakdown** — the numerator of every energy term.

**Accelergy** (MIT, Wu et al., ICCAD 2019) is the *energy/area* back end that consumes those action counts. It holds an **Energy Reference Table (ERT)**: a per-component table of energy-per-action (a 64 KB SRAM read, a MAC, a NoC hop) generated by **plug-in estimators** — CACTI for SRAM/buffers, Aladdin-style models or technology plug-ins for logic — at a specified technology node. Energy is then the plain dot product

$$E = \sum_{\text{component }c}\ \sum_{\text{action }a} N_{c,a}\cdot e_{c,a},$$

where $N_{c,a}$ = action count from Timeloop and $e_{c,a}$ = the ERT entry. Accelergy also composes *compound components* (e.g., a PE = registers + MAC + local control) so the library matches the architecture's real hierarchy. **This is the McPAT pattern — activity × per-event energy — for accelerators**, and it is why the pair is the standard mapping-and-energy loop: Timeloop supplies activity, Accelergy supplies the physics.

The paradigm's boundary is explicit in step 4: because Timeloop's timing uses an *idealized* bandwidth model, it reports the cycles a mapping *would* take if every level delivered operands on schedule. It cannot tell you that a real DRAM channel with FR-FCFS scheduling and refresh will stall the array 18% of the time — that is the cycle-accurate rung's job (§6). Use Timeloop+Accelergy to *choose the mapping and rank the energy*; do not quote its cycle count as an achieved latency for a memory-bound layer.

---

## 5. Analytical data-centric reuse — MAESTRO

**MAESTRO** (Georgia Tech, Kwon et al., MICRO 2019) attacks the same analytical problem from the opposite direction. Where Timeloop is *loop-nest-centric* (you describe the nested loops and tiling, and reuse is derived), MAESTRO is **data-centric**: you describe how each tensor dimension is *distributed over space and time* with a small set of directives, and it derives everything else. The directives are:

- **`TemporalMap(size, offset)`** — a dimension is tiled and streamed *in time* within each PE (temporal reuse from the PE's local buffer).
- **`SpatialMap(size, offset)`** — a dimension is partitioned *across PEs in space* (spatial reuse via multicast/forwarding on the NoC).
- **`Cluster(size)`** — groups PEs into a hierarchy, so maps can be nested (e.g., spatial across clusters, then spatial within a cluster).

From this data-centric description plus a hardware config (number of PEs, NoC bandwidth, buffer sizes), MAESTRO computes, in closed form: the **temporal and spatial reuse** of each tensor; the resulting **buffer-size requirement** at each level; the **NoC bandwidth requirement** (spatial multicast reduces it, but demands a matching network); the **PE utilization**; and a **roofline runtime** (whichever of compute or the buffer/NoC bandwidth is the binding constraint). It emits latency, energy, and hardware-cost estimates fast enough to sweep enormous spaces — the paper reports searching 480M designs to find 2.5M valid ones at ~0.17M designs/s.

Why keep both MAESTRO and Timeloop in the toolbox? They *frame the same analytical model differently, and the framing changes what is easy to express.* Data-centric directives make spatial reuse and NoC/buffer sizing first-class outputs, which is ideal when the question is "what interconnect bandwidth and buffer capacity does this dataflow need?" Loop-centric mapspaces make exhaustive mapping search and per-level action counts first-class, which is ideal for energy-optimal mapping. Both are analytical (no cycle stepping), both are blind to runtime contention, and both are orders of magnitude faster than §6.

---

## 6. Cycle-accurate systolic — SCALE-Sim v3

**SCALE-Sim** (originally ARM/Georgia Tech, Samajdar et al., ISPASS 2020; **v3**: Ramachandran et al., ISPASS 2025, [arXiv:2504.15377](https://arxiv.org/abs/2504.15377)) is the standard **cycle-accurate** model of a systolic array, and it exists precisely to compute the $\text{stall}_{\text{mem}}$ term the analytical tools drop. It represents a layer as a GEMM, the array as a $D_r\times D_c$ grid of MACs running a chosen dataflow — **weight-, output-, or input-stationary** — and it time-steps the array's *demand* for operands against the SRAM's ability to supply them. Each cycle it advances the systolic wavefront (paying the fill/drain latency of [Perf_Modeling §10.1](../01_Performance_Modeling_and_DSE.md)), reads the required IFMAP/filter operands from the on-chip SRAM buffers, writes OFMAP results, and — when a buffer must be refilled from DRAM and the DRAM bandwidth cannot keep up — **stalls the array**. Total cycles are therefore compute cycles plus emergent memory stalls, not an idealized quotient.

Its outputs are the ones only a cycle model produces: **total cycles, compute cycles, stall cycles, array mapping efficiency (spatial utilization), compute utilization (including stalls), and per-buffer SRAM and DRAM traffic and achieved bandwidth.** Feed it a layer (or a full network, layer by layer, end-to-end) plus the array dimensions, dataflow, SRAM sizes, and memory bandwidth; read out where the cycles went.

Version 3 is the one to cite because it closes the gaps that made v1/v2 optimistic on real systems:

- **Cycle-accurate DRAM via Ramulator** — replaces v2's flat bandwidth assumption with a real bank/rank/channel + JEDEC-timing + FR-FCFS model ([DDR_Controller](../10_DDR_Controller.md)), so $\text{stall}_{\text{mem}}$ reflects actual DRAM scheduling, not an average.
- **Multi-core** — spatio-temporal partitioning of a layer across multiple arrays with a shared hierarchical memory, so contention for shared SRAM/DRAM is modeled.
- **Sparsity** — layer-wise and row-wise sparse GEMM, so the MAC count and traffic reflect skipped zeros.
- **Energy/power via Accelergy** — the same action-count × ERT overlay as §4, so v3 emits energy alongside cycles.

The v3 paper's headline is a direct lesson in *why* you need the cycle rung: on ViT-base a 128×128 array is 6.53× faster than a 32×32 by latency, but the 32×32 is **2.86× more energy-efficient** — and, separately, output-stationary shows 30.1% lower *execution* cycles than weight-stationary once DRAM stalls are counted versus only 21% lower *compute* cycles. **The ranking flips depending on whether you look at compute cycles, execution cycles, or energy — which is unknowable from an analytical cycle count alone.**

---

## 7. Operator-level, graph-scale — NeuSim and ONNXim

The single-layer tools above answer "how good is this mapping on this array?" They do not answer "how does a 70B-parameter model, sharded tensor/pipeline/data-parallel across a 64-chip pod, meet a latency SLO at what power and carbon?" That is the **operator-level industrial** rung, whose unit of reasoning is the whole model graph and whose scope is the chip, the pod, and the datacenter.

**NeuSim** (UIUC PlatformX) is the notebook's worked reference for this rung, and it is treated in full in [Performance_Modeling_and_DSE §11](../01_Performance_Modeling_and_DSE.md) — the four-stage perf→power→carbon→SLO flow, the component models (SA, VU, SRAM, HBM, ICI), the config-swept Ray-distributed DSE, and the ReGate power-gating/DVFS knobs. **That treatment is not repeated here; what matters for this page is where NeuSim sits in the paradigm taxonomy.** It is *not* cycle-accurate: per operator, compute time comes from the closed-form systolic/vector models of [Perf_Modeling §10](../01_Performance_Modeling_and_DSE.md), not a time-stepped array. Its value is entirely at the graph-and-system scope that the single-layer tools cannot reach:

- It emits **per-operator** execution time, FLOPS, and memory traffic, and **classifies each operator SA-/VU-/HBM-bound** automatically — the §2.4 bottleneck taxonomy as a direct output rather than a hand calculation.
- It models the **inter-chip interconnect (ICI)** and collectives (AllReduce over a 2D/3D torus), so multi-chip parallelism strategies (TP/PP/DP/EP) are first-class.
- It overlays **power, carbon (embodied + operational), and SLO filtering**, making it a datacenter-scale DSE tool, not a microarchitecture tool.

So NeuSim complements, rather than competes with, §4–6: you would still use Timeloop/SCALE-Sim to characterize *one* operator on *one* array, then feed those per-operator characterizations into a NeuSim-style graph/pod model. It is the accelerator analogue of "leaf models composed into a full chip" ([Full_Chip_Modeling §4](../02_Full_Chip_Modeling.md)), operator-level and carbon-aware.

**ONNXim** (POSTECH, Ham et al., IEEE Computer Architecture Letters 2024, [arXiv:2406.08051](https://arxiv.org/abs/2406.08051)) occupies the same operator-level rung but with a sharper focus: **fast, cycle-level multi-core NPU simulation for DNN inference serving**, especially multi-tenant. Its inputs are ONNX model graphs (framework-agnostic, no per-kernel reimplementation). Its key modeling choice is a *deliberate asymmetry* of fidelity: it **abstracts compute** (a tensor tile from on-chip scratchpad has a deterministic compute latency, so the array is not time-stepped internally) but keeps **cycle-level DRAM and NoC** models, because in a multi-tenant NPU running several models at once the thing that actually determines performance is *contention* for shared memory and interconnect — exactly what a functional or purely analytical model would miss ([Simulation_Methodology §3](01_Simulation_Methodology.md): "contention emerges from shared event resources"). That asymmetry is why it reports up to **384× faster** simulation than Accel-Sim while still capturing the interference that matters. Use ONNXim when the question is scheduling/interference across concurrent DNNs on a many-core NPU; use SCALE-Sim when the question is one array's internal stall behavior.

---

## Numbers to memorize

| Quantity | Value | Why it matters |
|---|---|---|
| Accelerator MAC count | $N\,K\,C\,P\,Q\,R\,S$ (static) | compute is known offline → performance is analysis, not execution |
| Energy per access ordering | $e_{\text{DRAM}} \gg e_{\text{SRAM}} \gg e_{\text{RF}}$ (~$10^2$–$10^3\times$ spread) | why the mapping is an *energy* decision |
| Fill/drain amortization | $K/(K+2D)$ for a $D\times D$ array | small reductions waste the pipeline; drives large-$K$ tiling |
| Row-stationary vs others (Eyeriss/AlexNet) | 1.4×–2.5× energy-efficiency | dataflow is mostly an energy, not a cycle, lever |
| SCALE-Sim v3 array-size trade (ViT-base) | 128×128 is 6.53× faster; 32×32 is 2.86× more energy-efficient | latency-only metrics mis-rank designs |
| OS vs WS with DRAM stalls (SCALE-Sim v3) | 30.1% fewer execution cycles vs 21% fewer compute cycles | stalls change the ranking → need the cycle rung |
| ONNXim vs Accel-Sim | up to 384× faster | abstract compute, keep cycle-level DRAM/NoC for contention |
| Analytical mapping search rate (MAESTRO) | ~0.17M designs/s | why analytical models own dataflow/mapping DSE |

---

## Cross-references

- **Down the stack:** [Simulation_Methodology](01_Simulation_Methodology.md) (functional/timing/physical, trace vs execution, discrete-event contention — the paradigm vocabulary), [DDR_Controller](../10_DDR_Controller.md) (the FR-FCFS/JEDEC DRAM model SCALE-Sim v3 pulls in via Ramulator), [Memory §12b](../09_Memory.md) (compute-in-memory, the accelerator substrate NeuroSim models — see [Other_Architecture_Simulators](06_Other_Architecture_Simulators.md)).
- **Up the stack:** [Performance_Modeling_and_DSE §9–11](../01_Performance_Modeling_and_DSE.md) (systolic/occupancy/roofline kernels, dataflow table, and the full NeuSim worked example this page extends), [Full_Chip_Modeling §4](../02_Full_Chip_Modeling.md) (PE→array→core→chip→pod composition with power/thermal; the tool map that places these simulators), [Block_Activity_and_Power](../../02_Power_and_Low_Power/02_Block_Activity_and_Power.md) (the activity × per-event-energy method Accelergy applies).
- **Sibling pages (this folder):** [Other_Architecture_Simulators](06_Other_Architecture_Simulators.md) (manycore/CPU, NoC, CIM/neuromorphic, datacenter, chiplet, RISC-V), and the gem5 / DRAM / GPU per-tool pages.

## References

- A. Parashar et al., "Timeloop: A Systematic Approach to DNN Accelerator Evaluation," ISPASS 2019 — [timeloop.csail.mit.edu](https://timeloop.csail.mit.edu/), [paper PDF](https://accelergy.mit.edu/timeloop.pdf).
- Y. N. Wu, J. S. Emer, V. Sze, "Accelergy: An Architecture-Level Energy Estimation Methodology for Accelerator Designs," ICCAD 2019 — [accelergy.mit.edu](https://accelergy.mit.edu/), [Energy Reference Table docs](https://timeloop.csail.mit.edu/v4/parsing_and_intermediate_files/energy-and-area-reference-tables).
- H. Kwon et al., "Understanding Reuse, Performance, and Hardware Cost of DNN Dataflows: A Data-Centric Approach (MAESTRO)," MICRO 2019 — [arXiv:1805.02566](https://arxiv.org/abs/1805.02566), [maestro.ece.gatech.edu](https://maestro.ece.gatech.edu/).
- R. Ramachandran et al., "SCALE-Sim v3: A modular cycle-accurate systolic accelerator simulator for end-to-end system analysis," ISPASS 2025 — [arXiv:2504.15377](https://arxiv.org/abs/2504.15377), [github.com/scalesim-project/scale-sim-v3](https://github.com/scalesim-project/scale-sim-v3).
- A. Samajdar et al., "A Systematic Methodology for Characterizing Scalability of DNN Accelerators using SCALE-Sim," ISPASS 2020 — [github.com/scalesim-project/SCALE-Sim](https://github.com/scalesim-project/SCALE-Sim).
- H. Ham et al., "ONNXim: A Fast, Cycle-level Multi-core NPU Simulator," IEEE CAL 2024 — [arXiv:2406.08051](https://arxiv.org/abs/2406.08051), [github.com/PSAL-POSTECH/ONNXim](https://github.com/PSAL-POSTECH/ONNXim).
- NeuSim (UIUC PlatformX) — [github.com/platformxlab/NeuSim](https://github.com/platformxlab/NeuSim) (full treatment in [Performance_Modeling_and_DSE §11](../01_Performance_Modeling_and_DSE.md)).
- Y.-H. Chen, J. Emer, V. Sze, "Eyeriss: A Spatial Architecture for Energy-Efficient Dataflow for CNNs (row-stationary)," ISCA 2016.
