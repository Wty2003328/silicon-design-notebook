# GPU Simulators — GPGPU-Sim and Accel-Sim

> **Prerequisites:** [Simulation_Methodology](01_Simulation_Methodology.md) (execution- vs trace-driven §4, the event engine §3, ROI/warm-up/sampling §5), [Performance_Modeling_and_DSE §9](../01_Modeling/01_Performance_Modeling_and_DSE.md) (the GPU roofline/occupancy model these tools make executable).
> **Hands off to:** [Full_Chip_Modeling §3](../01_Modeling/02_Full_Chip_Modeling.md) (GPU as a chip in the perf→power→thermal ladder), and the companion **AI-infra notebook** ([silicon-to-serving](https://github.com/Wty2003328/silicon-to-serving)) for GPU *microarchitecture* depth (SM internals, warp execution, tensor cores) that this page cites but does not re-derive.

---

## 0. Why this page exists

A GPU number — "1.9 IPC," "achieved 82% of HBM peak," "230 W" — is only as trustworthy as the model that produced it, and GPUs stress the modeling problem harder than CPUs: performance is a *throughput* emergent property of thousands of warps contending for L1/L2/HBM and an on-chip crossbar, not a per-instruction latency ([Performance_Modeling_and_DSE §9](../01_Modeling/01_Performance_Modeling_and_DSE.md)). This page explains the two open simulators that dominate architecture research — **GPGPU-Sim** (the execution-driven original) and **Accel-Sim** (the modern trace-driven framework that wraps GPGPU-Sim as its timing model) — at the level of *how the model works and how a CUDA kernel becomes an IPC/bandwidth number*, with the validation story that says whether to believe it (Accel-Sim lands within **~15% of real silicon** on cycles).

The scope line, per the [brief's](../01_Modeling/02_Full_Chip_Modeling.md) cross-link discipline: **microarchitecture depth (what an SM *is*) lives in the companion AI-infra notebook; this page is about the *simulator* — its paradigm, its timing model, and how it turns a workload into a validated statistic.**

---

## 1. Two simulators, two paradigms

| | **GPGPU-Sim 3.x/4.0** | **Accel-Sim** (ISPASS 2020) |
|---|---|---|
| Front-end | **execution-driven**: functionally executes the kernel | **trace-driven** (SASS) *or* execution-driven (PTX) |
| ISA modeled | **PTX** (virtual ISA) or **PTXPlus** (≈SASS for GT200) | **SASS** machine ISA, via an ISA-independent IR |
| Timing model | its own SIMT-core model | **wraps GPGPU-Sim 4.0** as the timing back-end |
| Power | **GPUWattch** | **AccelWattch** |
| Validated to | GT200, Fermi (≈98% correlation, PTXPlus) | Kepler→Turing (**15% cycle MAE**, 1.00 corr on Volta); Ampere configs shipped |

The relationship is the key fact: **Accel-Sim is not a new timing model — it is a new *front-end and validation harness* around GPGPU-Sim's timing core.** GPGPU-Sim answers "run this CUDA and time it"; Accel-Sim answers "replay this *real SASS trace* — including closed-source cuBLAS/cuDNN kernels — through a tuned, silicon-validated version of that same timing core." Sections 2–3 describe the shared timing model; §5 is the paradigm trade-off that motivated Accel-Sim.

---

## 2. The SIMT core timing model

The heart of both tools is the **SIMT core** (GPGPU-Sim calls it a *shader core*; it models one NVIDIA SM). It executes **warps** — lock-stepped groups of 32 threads — through a pipeline whose stages are the load-bearing structures (mechanism, not signal list):

- **Fetch / decode.** A round-robin arbiter reads the I-cache into a per-warp **instruction buffer** (I-Buffer, ~2 entries/warp). The model tracks I-cache misses like any cache.
- **Warp scheduler.** Each cycle a scheduler picks a *ready* warp to issue from (round-robin LRR, greedy-then-oldest **GTO**, or two-level). Multiple schedulers per core. Accel-Sim's **sub-core model** (Volta+) gives each scheduler its **own register-file slice and execution units** — a partitioned SM, which is why post-Volta occupancy and issue behavior only match hardware under the sub-core model.
- **Scoreboard.** Tracks pending register writes to enforce **RAW/WAW** hazards: a warp is *not ready* to issue if its operands are still in flight. This is how the model serializes dependent instructions without a full OoO window — GPUs hide latency with *other warps*, not reordering within one.
- **SIMT stack (divergence).** Per-warp stack that handles branch divergence via **post-dominator reconvergence**: on a divergent branch it pushes entries `(target PC, active mask, reconvergence PC)` and executes each path with a reduced active mask, re-merging at the immediate post-dominator. **Divergence directly lowers throughput** because masked-off lanes do no work — and the model counts exactly that.
- **Operand collector.** Models the register file as **single-ported banks** with an arbiter that spaces accesses to avoid bank conflicts — a real throughput limiter the naive "registers are free" model misses.
- **Execution units.** Separate pipelines — SP/INT (ALU), SFU (transcendental), LD/ST, and **Tensor Cores** (§4) — each with a configured latency and initiation interval. The units and their counts come from a config file (per-GPU).
- **Writeback** frees scoreboard entries and wakes dependent warps.

**The point of this model is the same as an OoO core model's ([OoO_Execution](../02_CPU/03_OoO_Execution.md)): throughput is set by whichever structure saturates first** — not enough eligible warps (low occupancy)? scheduler starved on scoreboard stalls? operand-collector bank conflicts? execution-unit initiation-interval limited? The simulator tells you *which*, which is the whole reason to run it over a roofline estimate.

---

## 3. Coalescing and the cache / HBM / interconnect contention layer

Memory is where GPU performance is usually won or lost, and where the "achieved bandwidth" number is *born* (as an output of contention, exactly like the [DRAM page §8](03_DRAM_Simulators.md)):

- **Coalescer.** A warp's 32 per-lane addresses are merged into the fewest memory transactions. GPGPU-Sim (GT200 era) coalesces at half-warp granularity; **Accel-Sim models a sub-warp, sectored coalescer on 32-byte sectors**, matching the sector structure reverse-engineered on Pascal/Volta/Turing. A perfectly coalesced access touches one line; a scattered one explodes into many transactions — the model counts the *post-coalescer* access count, which is what actually hits the caches.
- **L1 / L2.** Both sectored (32 B sectors within 128 B lines) with MSHRs bounding outstanding misses. Accel-Sim adds an **adaptive, streaming L1** and an **IPOLY-hashed L2** that scatters addresses across L2 slices to avoid *partition camping* (a few hot slices serializing traffic) — the GPU cousin of the DRAM bank-hashing in [DRAM_Simulators §5](03_DRAM_Simulators.md). Write-back + write-allocate with sub-sector write merging to conserve DRAM bandwidth.
- **Interconnect.** GPGPU-Sim routes SM↔L2-slice traffic through **BookSim** (a detailed virtual-channel NoC simulator; separate request/response networks to avoid protocol deadlock); Accel-Sim uses a configurable crossbar with set flit size and bandwidth. **This is the contention layer** — the shared resource where two SMs' memory streams serialize (the [Network_on_Chip](../04_Interconnect/03_Network_on_Chip.md) theory, in-package).
- **DRAM (GDDR/HBM).** A cycle-level GDDR/HBM controller with row buffers, JEDEC-style timing, and FIFO or FR-FCFS scheduling — the same kind of model as the [DRAM_Simulators](03_DRAM_Simulators.md) page, just wider and hotter. Achieved HBM bandwidth is an *output* of this stack.

The queueing intuition from [Simulation_Methodology §7](01_Simulation_Methodology.md) applies at every one of these shared resources: latency stays flat until utilization nears saturation, then climbs as $\sim 1/(1-\rho)$. **Accel-Sim's tuning of this layer is exactly what closed the accuracy gap** — it reaches ~82% of theoretical HBM bandwidth and ~85% of L1 bandwidth on microbenchmarks, versus GPGPU-Sim 3.x's ~62% and ~33%, because the older coalescer/cache/interconnect models under-delivered bytes.

---

## 4. Tensor Core modeling

Tensor Cores (the matrix-multiply-accumulate units behind modern GEMM/AI throughput) are modeled as **execution units with a matrix-op latency and initiation interval**, at two levels of fidelity:

- **PTX / execution-driven**: the abstract **WMMA** (warp matrix-multiply-accumulate) instruction — one coarse op the timing model charges a latency for.
- **SASS / trace-driven**: the fine-grained **HMMA** instructions the compiler actually emits (Accel-Sim ships Volta/Turing ISA defs; a Turing config models, e.g., 8 HMMA tensor pipes/SM). This captures the real instruction count and issue pattern.

Interestingly, the paper notes the *abstract* WMMA sometimes matches hardware better than fine-grained HMMA for certain tile sizes — a reminder that "closer to the metal" is not automatically "more accurate," it just moves where the error lives. **Deep Tensor-Core / systolic modeling and the AI-kernel view belong to the companion AI-infra notebook**; here the point is only *how the simulator accounts for them* — as configured EU latency/throughput driving the same §2 pipeline.

---

## 5. Trace-driven vs execution-driven — Accel-Sim's key design point

This is the pivotal trade-off ([Simulation_Methodology §4](01_Simulation_Methodology.md)), and Accel-Sim's central bet.

**Execution-driven (GPGPU-Sim, PTX).** The simulator functionally executes the kernel, so it has real data values and control flow, and can model anything data-dependent (divergence on computed values, atomics, inter-block synchronization). But it simulates the **virtual ISA (PTX)** with its naive infinite-register model — *not* the machine code that actually runs — so it misses real register allocation, instruction scheduling, and compiler optimization, and it **cannot run closed-source kernels** (cuBLAS/cuDNN) at all.

**Trace-driven (Accel-Sim, SASS).** Instrument a *real* execution once with **NVBit** and record a per-warp SASS trace: PC, active mask, register operands, execution-unit assignment, and memory addresses. Replay that into the timing model. This:

1. captures the **actual machine ISA** — real register allocation, scheduling, and optimizations the PTX model fabricates away;
2. runs **closed-source, hand-tuned libraries** (cuDNN, cuBLAS, CUTLASS) that have no open PTX to execute;
3. is **~4.3× faster** (≈12.5 K warp-instructions/s) because it skips functional emulation of every scalar thread;
4. is portable via an **ISA-independent IR** with 1:1 SASS correspondence, so a new GPU generation is an "ISA-def file" (opcode→EU map) rather than a rewrite.

The cost — the standard trace-driven caveat — is a **frozen path**: a SASS trace captured on one GPU cannot reveal wrong-path or data-dependent behavior for a *different* configuration, and timing-dependent effects (spin-waits, some atomics/synchronization) are not faithfully re-timed. **The rule holds exactly as in the methodology page: trace-driven SASS is sound because a kernel's instruction/address stream is a faithful stimulus for the *timing* model, and it is the only way to reach real compiler output and closed-source libraries — but keep PTX execution-driven mode for studies where timing feeds back into which instructions run.** Traces are large (6.2 TB per GPU generation for the validation suite), tamed to ~317 GB by **base+stride compression** of the regular address/register streams.

---

## 6. How a CUDA kernel or SASS trace becomes IPC and achieved bandwidth

The stat pipeline, end to end, and *how each number is computed* ([Simulation_Methodology §5](01_Simulation_Methodology.md)):

**Execution-driven (GPGPU-Sim).** `nvcc` compiles CUDA/OpenCL to PTX+SASS in the fat binary → GPGPU-Sim extracts PTX (or `cuobjdump`'s SASS, optionally converted to PTXPlus) → a **functional** pass executes threads to produce correct state and the dynamic instruction stream → that same stream drives the **timing** pass, accumulating cycles through the §2–3 model.

**Trace-driven (Accel-Sim).** NVBit records the SASS trace on real hardware → the trace front-end feeds warp-instructions into GPGPU-Sim 4.0's timing model → cycles accumulate identically.

Either way, the reported statistics are computed as:

$$\text{IPC} = \frac{\text{warp-instructions executed}}{\text{cycles}}, \qquad \text{BW}_{\text{achieved}} = \frac{\text{bytes moved (post-coalesce)}}{\text{cycles}/f}, \qquad \text{occupancy} = \frac{\overline{\text{active warps}}}{\text{warps}_{\max}/\text{SM}}$$

where $f$ = core clock and *warp-instructions* (machine-instruction count) is used for cross-simulator fairness. Accel-Sim deliberately emits counters with **1:1 correspondence to NVIDIA profiler (nvprof/Nsight) counters** — L1/L2 reads, hits, hit rates, DRAM read/write counts, per-unit stalls — so validation is a *counter-by-counter* comparison against silicon, not a single-number hand-wave. As always, the honest report attaches an ROI, warm-up, and (for sampled runs) a confidence interval ([Simulation_Methodology §5](01_Simulation_Methodology.md)).

---

## 7. Power — GPUWattch and AccelWattch

Power reuses the perf model's **activity counts** and multiplies by **per-event energy** — the same activity×energy principle as McPAT/DRAMPower ([Full_Chip_Modeling §1.7](../01_Modeling/02_Full_Chip_Modeling.md)):

$$P_{\text{dyn}} = \sum_i a_i \cdot \frac{E_i}{T_{\text{elapsed}}}, \qquad P_{\text{total}} = \underbrace{\beta C f^3}_{\text{dynamic, DVFS}} + \underbrace{\tau f}_{\text{static}} + \underbrace{P_{\text{const}}}_{\text{fans/aux}}$$

where $a_i$ = accesses to component $i$ (from the timing sim), $E_i$ = energy per access, $T_{\text{elapsed}}$ = runtime, and $f$ = clock.

- **GPUWattch** (integrated in GPGPU-Sim) was the original; it is badly miscalibrated for modern parts — **219% MAPE on Volta** — largely because it mis-modeled constant/static power under DVFS.
- **AccelWattch** (MICRO 2021) fixes this with an explicit **constant + static + dynamic** decomposition (including chip-/SM-/lane-level power-gating for leakage) and the cubic-in-$f$ DVFS form above. Its 22 dynamic-component energies are fit by quadratic programming over 102 microbenchmarks. It runs in four modes — **SASS-SIM, PTX-SIM, HW (hardware counters only), and HYBRID** — so you can power a component from counters even without a full timing model.
- **Validation (Volta GV100):** AccelWattch SASS-SIM reaches **9.2% ± 3.1% MAPE**, Pearson $r$ ≈ 0.83–0.91 across 26 kernels — a **22–24× accuracy gain over GPUWattch** — and transfers to unseen architectures at 11% (Pascal TITAN X) and 13% (Turing RTX 2060S) MAPE.

---

## 8. Supported GPUs and the validation story

The reason to trust an Accel-Sim number is its **counter-level correlation to real silicon**:

| Config | Cycle MAE | Correlation | Note |
|---|---|---|---|
| Volta (Quadro V100), **SASS trace** | **15%** | **1.00** | the headline result |
| Volta, PTX execution-driven | 34% | 0.98 | virtual ISA costs accuracy |
| GPGPU-Sim 3.x (prior art) | **94%** | — | Accel-Sim cuts cycle error 79 pp |
| Deepbench (closed-source cuDNN/cuBLAS) | 33% | 0.95 | texture/local-load modeling gaps |

Per-metric on Volta the agreement is tight: dynamic **instruction count 1% MAE** (vs 27% for GPGPU-Sim 3.x), L2-reads 0.03 NRMSE (vs 2.67), DRAM-reads 0.92 NRMSE (vs 5.69) — i.e. it gets not just cycles but the *component activity* that drives power. **Generations:** the ISPASS 2020 paper validates **Kepler (TITAN), Pascal (TITAN X), Volta (V100), Turing (RTX 2060)**; the framework has since shipped tuned/tested **Ampere** configs (SM80 A100, SM86 RTX 3070) and is the standard vehicle for Volta-through-Ampere-class studies. The honest reading of "15%": it is a **mean absolute error on execution cycles with correlation ≈ 1.0**, so the model *ranks* configurations essentially perfectly even where it is 15% off in absolute cycles — which, per [Simulation_Methodology §8](01_Simulation_Methodology.md), is exactly the property that matters for design-space exploration.

---

## Numbers to memorize

| Quantity | Value | Why it matters |
|---|---|---|
| Warp width | 32 threads (SIMT lock-step) | the unit the timing model advances |
| Accel-Sim cycle accuracy (Volta SASS) | **15% MAE, 1.00 corr** | the "trust to ~15%" headline |
| PTX vs SASS accuracy (Volta) | 34% vs 15% MAE | why Accel-Sim went trace-driven |
| GPGPU-Sim 3.x → Accel-Sim | 94% → 15% cycle error | 79 pp improvement |
| Trace-driven speedup | ~4.3× (≈12.5 K warp-inst/s) | skips scalar functional emulation |
| Achieved BW (Accel-Sim vs GPGPU-Sim 3.x) | HBM 82% vs 62%, L1 85% vs 33% | contention layer is the accuracy lever |
| AccelWattch power (Volta SASS-SIM) | **9.2% MAPE** (GPUWattch: 219%) | activity×energy, done right |
| IPC definition | warp-instructions / cycles | machine-inst count, for fairness |
| Validated generations | Kepler→Turing (paper); Ampere (configs) | the coverage envelope |

---

## Cross-references

- **Down the stack:** [DRAM_Simulators](03_DRAM_Simulators.md) (the GDDR/HBM tier behind the GPU memory system — same model, wider), [Network_on_Chip](../04_Interconnect/03_Network_on_Chip.md) (the SM↔L2 interconnect/BookSim contention layer), [OoO_Execution](../02_CPU/03_OoO_Execution.md) (the "which structure saturates first" framing, borrowed for warps).
- **Up the stack:** [Simulation_Methodology](01_Simulation_Methodology.md) (execution-vs-trace §4, workload→number §5, validation §8), [Performance_Modeling_and_DSE §9](../01_Modeling/01_Performance_Modeling_and_DSE.md) (the GPU roofline/occupancy model this makes executable) and [§11 NeuSim](../01_Modeling/01_Performance_Modeling_and_DSE.md) (the operator-level NPU analogue), [Full_Chip_Modeling §3](../01_Modeling/02_Full_Chip_Modeling.md) (GPU in the chip-level perf→power→thermal ladder), [Root Index](../../Index.md).
- **Companion notebook:** GPU **microarchitecture** depth — SM internals, warp execution, Tensor Cores, CUDA kernels — lives in the AI-infra notebook, [silicon-to-serving](https://github.com/Wty2003328/silicon-to-serving); this page deliberately models the *simulator*, not the silicon.

---

## References

- Bakhoda, Yuan, Fung, Wong, Aamodt. *Analyzing CUDA Workloads Using a Detailed GPU Simulator* (GPGPU-Sim). ISPASS 2009. [[project]](http://gpgpu-sim.org/)
- GPGPU-Sim manual (SIMT core, memory model, PTXPlus). [[manual]](http://gpgpu-sim.org/manual/index.php/Main_Page)
- Leng et al. *GPUWattch: Enabling Energy Optimizations in GPGPUs.* ISCA 2013.
- Khairy, Shen, Aamodt, Rogers. *Accel-Sim: An Extensible Simulation Framework for Validated GPU Modeling.* ISCA 2020. [[pdf]](https://mkhairy.github.io/Docs/Accel-Sim.pdf) · [[site]](https://accel-sim.github.io/) · [[GitHub]](https://github.com/accel-sim/accel-sim-framework)
- Kandiah, Peverelle, Khairy, et al. *AccelWattch: A Power Modeling Framework for Modern GPUs.* MICRO 2021. [[pdf]](https://paragon.cs.northwestern.edu/papers/2021-MICRO-AccelWattch-Kandiah.pdf)
