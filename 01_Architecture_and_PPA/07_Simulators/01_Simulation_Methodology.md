# Simulation Methodology — How Architectural Simulators Actually Work

> **Prerequisites:** [Performance_Modeling_and_DSE](../01_Modeling/01_Performance_Modeling_and_DSE.md) (the analytical leaf models a simulator makes executable), [Full_Chip_Modeling](../01_Modeling/02_Full_Chip_Modeling.md) (the tool/fidelity ladder this page explains conceptually).
> **Hands off to:** the per-tool pages in this folder — gem5, DRAM simulators, GPU simulators, accelerator/DNN simulators, and analytical models.

---

## 0. Why this page exists

Every performance or power number in this notebook that is not measured on real silicon comes out of a *model that was run* — a simulator. An architect who quotes "IPC 2.1" or "achieved bandwidth 480 GB/s" without knowing **what kind of model produced it** cannot say whether the number is trustworthy to 5% or to 50%. This page is about the machinery: what a simulator abstracts, what it computes cycle by cycle, how a benchmark is turned into a statistic, and where the error comes from. The specific tools (gem5, DRAMSim3, GPGPU-Sim, NeuSim) are instances of the ideas here; each gets its own page.

The single most important habit: **before you believe a simulated number, ask which rung of the fidelity ladder it came from and what that rung is allowed to model.**

---

## 1. The three questions every simulator answers

A processor simulator is a program that reproduces, at some chosen fidelity, *what a target machine would do when it runs a workload*. It answers up to three separable questions, and simulators differ mainly in **which of the three they model and how faithfully**:

1. **Functional — "what is the result?"** Execute the instructions so that architectural state (registers, memory) ends up correct. This is what an ISA emulator (QEMU, gem5 "atomic", Spike for RISC-V) does. It gives you the *right answer* and the *dynamic instruction count*, but **no timing**.
2. **Timing — "how long does it take?"** Model the microarchitecture — pipelines, caches, branch predictor, memory system — so you can count *cycles*, not just instructions. This is where IPC/CPI comes from.
3. **Physical — "what does it cost?"** Overlay energy, power, area, temperature onto the timed activity. This is the McPAT / Accelergy / DRAMPower / HotSpot layer covered in [Full_Chip_Modeling](../01_Modeling/02_Full_Chip_Modeling.md).

The clean architecture — used by gem5, GPGPU-Sim, and most modern tools — **separates functional from timing** (the "execute-in-execute" or "timing-directed" split): a functional core guarantees correctness while a timing model, driven by the same instruction stream, accounts for when each event happens. Keeping them separate is what lets one functional model feed many timing models.

---

## 2. The fidelity–speed tradeoff (the master curve)

Simulation fidelity and simulation speed trade off across roughly six orders of magnitude. Knowing where a tool sits tells you both its error bar and its cost.

| Level | What it models | Typical slowdown (host/target) | Error on cycles | Example |
|---|---|---|---|---|
| Native / functional emulation | correctness only, no time | 1–10× | n/a (no timing) | QEMU, Spike, gem5-atomic |
| Analytical / mechanistic | closed-form CPI from a few parameters | ~instant | 5–20% | interval model, roofline |
| Event-driven cycle-approximate | components + queueing, not every latch | 1k–10k× | 5–15% | gem5-O3, GPGPU-Sim, Ramulator |
| Cycle-accurate (µarch) | every pipeline stage, validated to a real core | 10k–100k× | <5% vs that core | vendor-internal, some gem5 configs |
| RTL simulation | the actual gates/registers | 10⁶–10⁷× | exact (it *is* the design) | Verilator/VCS on the RTL |
| Emulation / FPGA prototype | RTL in hardware | 10–1000× | exact, fast | Palladium, FireSim |

Two lessons. First, **"cycle-accurate" is a claim about a specific target**, not a universal virtue — an O3 model validated against a Cortex-A76 is not cycle-accurate for a Neoverse-V2. Second, the reason architects live mostly on the middle two rungs (analytical + event-driven) is the **speed budget**: a design-space exploration (DSE) sweep of thousands of configurations cannot afford RTL, so you push physics down into pre-characterized per-event costs (CACTI energies, DRAM timings) and keep the *composition* fast — the theme of [Full_Chip_Modeling §2](../01_Modeling/02_Full_Chip_Modeling.md).

---

## 3. The event-driven engine — the heart of a timing simulator

Cycle-approximate and cycle-accurate timing simulators are almost always **discrete-event simulators**. The core is not "loop over cycles and update everything" (that is a cycle-*stepped* engine, simpler but slower); it is a **priority queue of events keyed by the tick they fire**:

```
event_queue = min-heap keyed by tick
while not empty and tick < end:
    (tick, event) = pop-min          # jump straight to the next thing that happens
    event.process()                  # may schedule new events at tick + latency
```

The key idea is **time skipping**: if nothing happens between cycle 100 and cycle 137 (a load is in flight to DRAM), the engine jumps directly to 137 instead of simulating 37 empty cycles. Latencies are expressed as *"schedule this event `+N` ticks from now"* — a cache miss schedules its fill event at `now + t_DRAM`; a pipeline stage schedules the next stage at `now + 1`. This is why a memory-bound workload can simulate *faster* than a compute-bound one of the same length: fewer events per unit of simulated time.

Consequences an architect should internalize:

- **Latency is data, not control flow.** Every modeled latency (cache hit time, DRAM `t_RCD`, NoC hop, NVLink transfer) is a `+N` on a scheduled event. Changing a latency parameter is changing an addend, which is why sensitivity sweeps are cheap and why a wrong latency constant silently biases every downstream number.
- **Contention emerges from shared event resources.** Two requests contend not because the model has an explicit "contention formula" but because they compete for the same port/bank/queue whose events serialize. A model with no shared resource *cannot* produce queueing delay — it will report the sum of standalone latencies and be optimistic (the auditor's first probe, [Full_Chip_Modeling §5.1](../01_Modeling/02_Full_Chip_Modeling.md)).
- **Determinism and reproducibility** come from a fixed tie-breaking rule for events at the same tick. Non-determinism in parallel simulators (multiple event queues) is a real correctness hazard.

---

## 4. Trace-driven vs execution-driven — where the instruction stream comes from

A timing model needs a stream of operations to time. There are two ways to feed it, and the choice determines what the simulator can and cannot capture.

**Execution-driven.** The functional model runs the real program and hands each instruction to the timing model as it goes. This is the gold standard because it captures **path effects**: mispredicted-branch wrong-path instructions that pollute caches, data-dependent control flow, spin-loops that change with timing, and the feedback where timing affects which instructions even execute (e.g., a lock acquired in a different order). gem5, GPGPU-Sim, and Accel-Sim in execution mode work this way.

**Trace-driven.** You record a trace once (instructions, addresses) and replay it into the timing model. Much faster and repeatable, and it decouples workload capture from model iteration — but it **freezes the path**. A trace taken on one machine cannot show wrong-path pollution for a *different* branch predictor, and it mishandles anything where timing changes the instruction stream (multithreaded races, most spin-waiting). Trace-driven is excellent for memory-system and NoC studies (addresses are what matter) and dangerous for core-microarchitecture studies (path matters). Accel-Sim's trace mode and most DRAM-simulator front-ends are trace-driven for exactly this reason: for a DRAM channel, the *address/time stream* is a faithful stimulus, but for an OoO core the path is the point.

The rule: **trace-driven is sound when the thing you are studying does not feed back into the instruction path; execution-driven is required when it does.**

---

## 5. How a workload becomes a number — the part people get wrong

Running "the benchmark" end-to-end at cycle-approximate fidelity is usually infeasible (a 1-trillion-instruction SPEC run at 10 000× slowdown is months of wall-clock). So the reported number is almost always the product of a **sampling and warm-up methodology**, and most simulation mistakes live here, not in the model.

1. **Region of interest (ROI).** You skip initialization and measure only the steady-state region — often via magic instructions / `m5_work_begin` markers. Reporting whole-program numbers when the ROI is a kernel is a classic error.
2. **Warm-up.** When you fast-forward (functional-only) to the ROI, the caches, TLBs, and branch predictor are *cold*. Measuring immediately overstates miss rates. You must run a **warm-up window** (functionally warming caches, or detailed-warming for a few million instructions) before you start counting. Under-warming is the most common silent bias in student and paper results.
3. **Sampling.** Instead of all N instructions, measure representative slices:
   - **SimPoint** clusters the program's execution into phases using basic-block vectors (BBVs) and simulates one representative *simpoint* per phase, then weights them. It exploits the fact that program behavior is *phasic and recurrent*.
   - **SMARTS / statistical sampling** takes many small, uniformly spaced detailed samples with functional warming between them and reports a mean with a **confidence interval** — the number comes with $\pm$ error and an $n$, like any Monte-Carlo estimate.
4. **The statistic.** Cycles are accumulated over the measured region; then

$$\text{CPI} = \frac{\text{cycles}}{\text{instructions}}, \qquad \text{IPC} = \frac{1}{\text{CPI}}, \qquad \text{achieved BW} = \frac{\text{bytes moved}}{\text{cycles}/f}.$$

A subtle point: **speedups must be summarized with the geometric mean** across benchmarks (ratios), while rates at a fixed workload use the arithmetic mean — mixing these is a real reporting bug.

The auditor's checklist for any simulated result: *What was the ROI? How long was warm-up? What sampling method and how many samples? Is there a confidence interval? Geometric or arithmetic mean?* A number without these is a number without an error bar.

---

## 6. How compute is modeled — the timing of a core

A timing core model does not re-simulate the ALU's logic; it models the **structural and hazard constraints that decide when an instruction can advance**. Two canonical fidelities:

- **In-order** model: an instruction issues when its operands are ready and its functional unit is free; a miss/hazard stalls the pipe. Cheap; adequate for simple cores and for first-order studies.
- **Out-of-order (O3)** model: the machine that matters for big cores. The simulator maintains the *structures that impose the limits* — a reorder buffer (ROB) bounding how far ahead retirement can lag, physical registers and a rename map, reservation-station/IQ slots, load/store queues, and issue-port counts. Each cycle it (conceptually) fetches up to width, renames against free physical registers, wakes up ready instructions, selects up to issue-width for execution, and retires in order from the ROB head. **The point of the model is that throughput is set by whichever structure saturates first** — you learn *why* IPC is 1.8 not 4 (ROB-full? LSQ-full? a long dependence chain? issue-port pressure?), which is exactly the conceptual "what is stored and why" framing this notebook prefers over signal lists. See [OoO_Execution](../02_CPU/03_OoO_Execution.md) for the structures themselves.

The **interval / mechanistic model** (Karkhanis–Smith) is the analytical dual: instead of simulating the window, it says a balanced OoO core issues at its ideal width *between* "miss events" (branch mispredicts, last-level-cache misses), and each miss event punches a hole in the issue rate of a characterizable width. CPI becomes a base term plus $\sum(\text{miss rate}\times\text{penalty})$ — closed-form, instant, and good to ~10–20%. It is why you can reason about a core on a whiteboard; the event-driven O3 model is the same story made executable and 10% more accurate.

---

## 7. How the memory system is modeled — usually the dominant term

For most modern workloads the memory system decides performance, so its model deserves the most scrutiny. Fidelity rungs:

- **Fixed-latency** ("simple/classic" memory): every access costs a constant $t$. Fast, and wrong whenever the memory system is contended — it has *no queueing term*, so it cannot show bandwidth saturation.
- **Cache hierarchy model**: per-level capacity/associativity/latency, MSHRs bounding outstanding misses, coherence (the gem5 Ruby model is a full coherence-protocol state machine). This produces the miss-rate and MLP effects that fixed-latency cannot.
- **Cycle-level DRAM model** (Ramulator, DRAMSim3): models banks/ranks/channels, the JEDEC timing constraints ($t_{RCD}, t_{RP}, t_{RAS}, t_{FAW}, t_{WTR}, \dots$), the row-buffer, refresh, and the **scheduler** (FR-FCFS: first-ready, first-come-first-served, which reorders to hit open rows). Achieved bandwidth is an *output* of this contention, not an input — a memory-bound number that came from a fixed-latency model is not credible.

The theoretical backbone across all of it is **queueing**: a shared resource of service rate $\mu$ under offered load $\lambda$ has utilization $\rho=\lambda/\mu$, and mean latency grows as $\sim 1/(1-\rho)$ — latency runs to the knee as you approach saturation. Every credible memory/NoC simulator is, at bottom, a structural realization of that curve; a model that reports latency flat with load has no queue and is optimistic. (DRAM device/controller detail: [Memory](../03_Memory/03_Memory.md), [DDR_Controller](../03_Memory/04_DDR_Controller.md).)

---

## 8. Validation, calibration, and the error budget

A simulator is a *hypothesis about a machine*; it is only as good as its validation.

- **Validation** compares simulated statistics against real hardware performance counters on the same binaries. The honest figure of merit is not a single average error but the **error distribution and the correlation of trends** — a model that is 15% off in absolute IPC but ranks configurations correctly is often *more useful for DSE* than one that is 5% off but mis-orders two designs.
- **Calibration** fits free parameters (latencies, penalties, McPAT's technology knobs) to close the gap; an *un-calibrated* architectural model can be 2× off, a calibrated one ~20–30% (see the [Full_Chip_Modeling §1.7 accuracy ladder](../01_Modeling/02_Full_Chip_Modeling.md)).
- **Composed error**: when you chain models (perf → power → thermal), errors compound. A ±20% power model feeding a thermal model feeding a DVFS governor can swing the final throughput materially — which is why the perf/power/thermal loop must be *co-validated*, not validated stage by stage in isolation.

The mature stance: quote simulated numbers with their provenance and error class, use the model for *relative* comparisons whenever possible (trends survive calibration error better than absolutes), and reserve absolute claims for validated, calibrated configurations.

---

## 9. Numbers to memorize

| Quantity | Value | Why it matters |
|---|---|---|
| Functional-only slowdown | 1–10× | correctness/ICount is cheap |
| Event-driven cycle-approximate slowdown | 10³–10⁴× | the working rung for µarch studies |
| RTL simulation slowdown | 10⁶–10⁷× | why you don't DSE on RTL |
| Calibrated architectural perf/power error | ~20–30% | the DSE-rung error bar |
| Cycle-accurate (validated-to-target) error | <5% | the sign-off-ish rung |
| Queueing latency law | $\sim 1/(1-\rho)$ | why latency explodes near saturation |
| CPI (interval model) | base $+\sum(\text{miss rate}\times\text{penalty})$ | closed-form core performance |
| Summary statistic | geomean for ratios, arithmetic for rates | the reporting-correctness rule |

---

## Cross-references

- **Down the stack:** [Performance_Modeling_and_DSE](../01_Modeling/01_Performance_Modeling_and_DSE.md) (analytical kernels + the NeuSim worked example), [OoO_Execution](../02_CPU/03_OoO_Execution.md) (the structures an O3 timing model tracks), [Memory](../03_Memory/03_Memory.md) / [DDR_Controller](../03_Memory/04_DDR_Controller.md) (what DRAM simulators encode).
- **Up the stack:** [Full_Chip_Modeling](../01_Modeling/02_Full_Chip_Modeling.md) (composing leaf models into a chip; the full tool/fidelity ladder and the perf→power→thermal loop).
- **Sibling pages (this folder):** per-tool deep dives on gem5, DRAM simulators, GPU simulators, accelerator/DNN simulators, and analytical models.
