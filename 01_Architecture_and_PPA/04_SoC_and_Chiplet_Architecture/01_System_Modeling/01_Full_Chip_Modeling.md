# Full-Chip Power and Performance Modeling

> **First-time reader orientation:** A system on chip (SoC) combines processors, accelerators, memories, interconnect, and devices that share power, bandwidth, area, and temperature limits. Power, performance, and area (PPA) cannot be predicted by adding isolated peak numbers: contention and feedback change block behavior. This chapter builds a hierarchy of calibrated block models and coupling equations.

> **Abbreviation key — skim now and return as needed:** central processing unit (CPU); graphics processing unit (GPU); neural processing unit (NPU); system on chip (SoC); register-transfer level (RTL);
> power, performance, and area (PPA); instructions per cycle (IPC); cycles per instruction (CPI); thread-level parallelism (TLP); misses per thousand instructions (MPKI);
> design-space exploration (DSE); out-of-order (OoO); reorder buffer (ROB); miss status holding register (MSHR); arithmetic logic unit (ALU);
> register file (RF); single instruction, multiple threads (SIMT); static random-access memory (SRAM); dynamic random-access memory (DRAM); high-bandwidth memory (HBM);
> double data rate (DDR); level-one cache (L1); level-two cache (L2); last-level cache (LLC); network on chip (NoC);
> quality of service (QoS); direct memory access (DMA); AXI Coherency Extensions (ACE); Coherent Hub Interface (CHI); Modified, Exclusive, Shared, Invalid (MESI);
> Modified, Owned, Exclusive, Shared, Invalid (MOESI); first come, first served (FCFS); dynamic voltage and frequency scaling (DVFS); processing element (PE); multiply-accumulate (MAC);
> general matrix multiplication (GEMM); general-purpose computing on graphics processing units (GPGPU); Peripheral Component Interconnect Express (PCIe); streaming multiprocessor (SM); thermal design power (TDP);
> artificial intelligence (AI); tensor processing unit (TPU); tera floating-point operations per second (TFLOP); 8-bit integer (INT8); physical-layer interface (PHY);
> kilobyte (KB); megabyte (MB); gigabyte (GB); terabyte (TB); gigahertz (GHz).

> **Stage:** 01 · Architecture & PPA (Performance, Power, Area) — the *systematic, hierarchical* model that composes leaf blocks into a **full chip** and answers SoC-level power+performance questions **before RTL (register-transfer level) exists**.
> **Prerequisites:** [SoC/chiplet workload and performance methods](../00_Design_Methodology/01_SoC_Chiplet_Workloads_Performance_and_DSE.md) (fidelity ladder, CPI (cycles per instruction) stack, roofline), [Block_Activity_and_Power](../../../02_Power_and_Low_Power/02_Block_Activity_and_Power.md) (per-block $P=\alpha C V^2 f + P_{leak}$, McPAT/CACTI bottom-up, DPM), [CPU_Architecture](../../01_CPU_Architecture/01_Core_Foundations/01_CPU_Architecture.md), [Memory](../00_Design_Methodology/02_SoC_Chiplet_PPA_and_Physical_Implementation.md).
> **Hands off to:** [GPU_Architecture](../../02_GPU_Architecture/01_Core_Architecture/01_GPU_Architecture.md) and [NPU_Accelerators](../../03_NPU_Architecture/01_Compute_Dataflows/01_NPU_Accelerators.md) (the µarch behind §4), [06_Simulators](00_Index.md) (how every tool named here actually models compute, memory, and time), [Power_Analysis_and_Signoff](../../../02_Power_and_Low_Power/06_Power_Analysis_and_Signoff.md) (the budgeting/governor mechanics at silicon).

---

## 0. Why this page exists

The shallow instinct is: model the CPU core, model the cache, model the DRAM (dynamic random-access memory), add the Watts, and declare the chip understood. That is wrong, and an auditor will catch it in one question, because **a chip is a system, not a bag of blocks.** The number a leaf model gives you is real; what it cannot give you is what happens when you *put the leaves together and run them at once*.

Write the honest full-chip model as a first term plus a correction:

$$
X_{chip} \;=\; \underbrace{\bigoplus_i X_{\text{leaf},i}}_{\text{what a leaf model gives you}} \;+\; \underbrace{X_{\text{coupling}}}_{\text{the whole art of this page}}
$$

The first term is cheap and the sibling pages already produce it — per-block energy and per-block latency from characterized cells and CACTI-fit arrays. The second term is everything this page is about, and it is *not* a small correction. It routinely moves throughput by $1.5\text{–}2\times$ and turns a benchmark's first second into a different machine from its tenth. A model that keeps only the first term does not under-report the chip by a few percent; it answers a different question than the one asked.

Two things go wrong when you keep only $\bigoplus X_{\text{leaf}}$:

1. **The composition operator is wrong for performance.** Power is *extensive* — Watts genuinely sum. Performance is not: throughput is set by the **slowest shared resource** and latencies compose through **overlap and queueing**, never by addition. Summing per-block latencies is a category error, not an approximation.
2. **The operating point is wrong for every block.** Each leaf's energy and delay are functions of $(V, f, T, \text{activity})$. On a real chip those are *not* datasheet constants: a shared power budget sets $f$, a thermal cap sets the sustainable $f$, and contention sets the true activity. A standalone leaf is evaluated at an operating point the assembled chip never actually runs at.

So the full-chip model has three parts, not one — the leaf models, a **contention/overlap layer** that fixes the composition and the contended activity, and a **power/thermal budget layer** that fixes the operating point — and the two upper layers are solved *together* in a control loop (§3). The rest of this page builds those two layers on top of the leaves the sibling pages give you.

**A full-chip model is defined by the questions it must answer** — none of which a leaf model can:

- What is SoC (system-on-chip) power at TDP (thermal design power), delivery losses included?
- What *sustained* clock survives the thermal cap, as opposed to the burst clock the datasheet quotes?
- What memory bandwidth is *actually achieved* once $N$ cores contend for one channel?

If your model cannot answer these, it is a pile of leaf models, not a chip model. That gap — between $\bigoplus X_{\text{leaf}}$ and the truth — is the coupling layer, and closing it is the job.

---

## 1. Composition: push the physics down, keep the composition cheap

Before the coupling, the scaffolding it hangs on: *how* leaves become a chip when nothing is contending yet. The discipline has one governing idea — **compute the physics once, at the bottom, and never again above it.** A per-access SRAM energy or a per-op ALU energy is expensive to derive (it is a circuit-simulation question) but cheap to *reuse*; the architectural model earns its speed by composing pre-computed energies against activity counts, not by re-deriving physics at every level.

### 1.1 The hierarchy and the roll-up rule

You compose strictly bottom-up through named levels, and the rule at every level is the same: **sum of children, plus the glue that only exists at this level.**

| Level | Contents | New cost that first appears here |
|---|---|---|
| **Leaf** | one array/unit (SRAM bank, ALU, FPU, register file, router) | energy/access × access rate; leakage |
| **Block** | a core, a cache slice, one memory controller | pipeline glue, local clock tree |
| **Cluster** | $N$ cores + shared LLC (last-level cache) + local NoC (network-on-chip) | coherence, interconnect, LLC sharing |
| **Chip** | clusters + full uncore (NoC, MC, PHY, PCIe, PMU) | global clock/NoC, uncore floor |
| **Package** | chip + VRM + PDN + board | VRM loss, PDN/IR drop, on-package DRAM/HBM |

The identity every roll-up must satisfy at chip level:

$$
P_{chip} \;=\; \sum_i P_{block,i} \;+\; P_{uncore} \;+\; P_{PDN/VRM}
$$

where $P_{uncore}$ (NoC + MC (memory controller) + PHY (physical-layer interface) + PMU (power-management unit) + I/O) is **not an idle floor you may neglect** — on server parts it is 20–40% of socket power and it *scales with activity* (the PHY and MC burn more under high bandwidth), and $P_{PDN/VRM}$ is the delivery loss $P_{VRM,loss}=P_{load}(1/\eta-1)$ at regulator efficiency $\eta\approx0.85\text{–}0.92$, plus PDN (power-delivery network) $I^2R$ drop ([Signal_Integrity_Reliability](../../../05_Backend_Physical_Design/02_Signal_Integrity_Reliability.md)). These sit *outside* the die-power sum and are exactly what shallow models forget.

The one rule you cannot violate: **power rolls up additively; performance does not.** Watts sum because dissipation is extensive. Latency and bandwidth compose through the bottleneck — the slowest shared resource caps throughput (roofline), and delays overlap or queue rather than add. Never sum latencies across blocks; route them through the coupling layer of §2.

**Why the composition operator differs by quantity — a derivation.** The abstract $\bigoplus_i$ of §0 is not one operator; it is *three*, and which one applies follows from what the quantity physically **is**.

- **Power / energy — additive ($\bigoplus=\sum$).** Energy is *extensive*. The charge drawn from the rail in a cycle is the sum over every switching node, $E_{cyc}=\sum_i E_i$, because charge is conserved and the nodes draw independently from a common supply; dividing by the period gives $P=\sum_i P_i$ **exactly**, with no interaction term. That is why the roll-up is a plain sum and the only subtlety is *remembering* the uncore and delivery terms, not choosing the operator.
- **Throughput — bottleneck ($\bigoplus=\min$).** A chain of resources passing a stream forwards at the rate of its slowest stage, $\Theta=\min_i \Theta_i$. *Proof:* in steady state every stage must process the same number of items per unit time — otherwise a queue between two stages grows without bound, violating stability (Little's law, §2.3, with $\bar N\!\to\!\infty$) — so the common rate cannot exceed the smallest stage capacity, and a work-conserving schedule attains it. This is the roofline $\min(\pi,\beta I)$ generalized from two roofs to a chain.
- **Latency — max-or-sum ($\bigoplus\in[\max,\ \sum]$).** Two operations on the critical path add their times *if serial* and take the $\max$ *if overlapped*; the true value lies between and is fixed by the overlap fraction (§2.3). This is the one operator the naive model gets wrong in **both** directions — it sums when it should $\max$ (missing overlap) and $\max$es when it should sum (missing a serialized dependence).

So "the chip is the sum of its blocks" is true for *exactly one* of the three quantities and false for the other two. The two corrections that make performance compose correctly are therefore named in advance: **overlap** (§2.3, which selects $\max$ vs $\sum$) and **contention** (§2.1–2.2, which inflates each $\Theta_i$ and each latency toward the queue). Up the hierarchy the same three operators recurse — a cluster's memory latency is $t_{\text{core}}+\max(t_{\text{LLC}},\ldots)+t_{\text{queue}}(\rho)$ (sum where serial, $\max$ where overlapped, plus the queue term §2.2 adds at *this* level's shared resource); a pod's step time is $\max(t_{\text{chip}},\,t_{\text{collective}})$ (§4.4) — while power stays a clean $\sum$-plus-glue at every level. **Naive summation fails precisely because it applies the power operator ($\sum$) to throughput and latency.**

Concretely, the leaf→block step is what an architectural power model like **McPAT** does: it maps the µarch onto a few primitive circuit structures (arrays, wires, logic, clocking), pulls each array's per-access energy from an internal CACTI (its cache/array energy model), and composes

$$
P_{core} \;=\; \sum_{u\in\text{units}} E_u\,A_u\,f \;+\; P_{clock} \;+\; \sum_u P_{leak,u}
$$

where $E_u$ = per-event energy (from CACTI-fit arrays / characterized cells), $A_u$ = events per cycle (from the [CPI stack](../00_Design_Methodology/01_SoC_Chiplet_Workloads_Performance_and_DSE.md) counters), $f$ = clock. There is **no new physics at the core level** — the clock tree ($P_{clock}$, typically 30–40% of core dynamic) and leakage ($V\cdot I_{leak}$, scaled by area and temperature) are added, and the array/op energies are *looked up, not recomputed*. That is the whole game restated: push the physics down, keep the composition cheap.

Each per-event energy is itself the $\alpha C V^2$ of a structure (§2.4), so the roll-up is literally *activity × per-event-energy*, and it is worth carrying one to a number. *Worked number — McPAT-style rollup.* Take three structures on a core at $f=3$ GHz: an integer ALU at $E_{\text{ALU}}=3$ pJ/op, a 32 KB L1 at $E_{\text{L1}}=15$ pJ/access, and a register-file read port at $E_{\text{RF}}=1$ pJ/read. Suppose the per-cycle activity counts (from the CPI stack) are $A_{\text{ALU}}=1.0$ op, $A_{\text{L1}}=0.6$ access, $A_{\text{RF}}=2.4$ reads (two source operands + writeback per instruction at IPC $\approx1.5$). Then

$$
\textstyle\sum_u E_u A_u f=(3{\cdot}1.0+15{\cdot}0.6+1{\cdot}2.4)\ \text{pJ/cyc}\times 3\times10^{9}\ \text{cyc/s}=14.4\ \text{pJ/cyc}\times3\times10^{9}=43\ \text{mW}
$$

from just those three units. Add $P_{clock}\approx35\%$ of core dynamic and per-unit leakage and the core power emerges — and *every* term is one (energy × activity) product, characterized **once** (slow, circuit-level) and multiplied **forever** (fast, per-config). The composed number's error is the §1.3 propagation of the inherited $E_u$, not new physics — which is exactly why the architectural rung is instant yet trustworthy to ±20–30%.

### 1.2 Where the numbers come from: the calibration chain

The architectural model is fast enough to sweep hundreds of design points *because* it does not re-derive physics — but that only makes it trustworthy if its inherited per-event energies are anchored to something golden. They are, through a calibration chain in which each level calibrates the one above it:

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk", "nodeSpacing": 60, "rankSpacing": 60, "htmlLabels": false}}}%%
flowchart TD
    SPICE["SPICE (transistor)<br/>golden, slowest"] --> LIB[".lib cell characterization<br/>per-cell E, leakage @ PVT"]
    LIB --> GLP["Gate-level power<br/>PrimeTime-PX, SDF+SAIF"]
    SPICE --> CACTI["CACTI<br/>arrays fit to SPICE"]
    CACTI --> MCPAT["McPAT / architectural<br/>energy × activity, ±20–30%"]
    RTL["RTL power<br/>PowerArtist, SAIF"] --> MCPAT
    classDef gold fill:#fef9c3,stroke:#a16207,color:#000
    classDef tool fill:#dbeafe,stroke:#1d4ed8,color:#000
    class SPICE gold
    class LIB,GLP,CACTI,MCPAT,RTL tool
```

| Level | Tool | Accuracy vs silicon | Speed |
|---|---|---|---|
| Transistor | SPICE | golden (≈1–2%) | slowest |
| Cell library | Liberty `.lib` characterization | ~2–3% | one-time |
| Gate + parasitics | PrimeTime-PX (SDF+SAIF) | ~5–10% | slow |
| RTL | PowerArtist / Joules (SAIF/FSDB) | ~10–20% | medium |
| Architectural | McPAT / Accelergy | ~20–30% *calibrated* (≫ un-cal) | instant |

The ladder is used in **two directions** at opposite ends of a project. **Bottom-up** (pre-silicon) composes energy × activity analytically — this page's mode — for design-space exploration and budgeting before RTL exists. **Top-down** (post-silicon) does the inverse: it takes a *measured* chip power number from RAPL (Running Average Power Limit) / on-die telemetry and decomposes it to blocks via **power proxies** — regressions of block power onto activity counters, fitted to silicon (accuracy ~3–5% per block, far better than bottom-up because it is trained on the real thing). Mature flows run both and reconcile: the proxy is *trained* against the bottom-up model and *validated* against silicon. The mechanics of each tool live in [06_Simulators](00_Index.md); here they are just rungs on the ladder.

### 1.3 What survives summation: the validation theory

Whether a roll-up can be trusted turns on how per-block error behaves under summation, and two error classes behave *oppositely*:

- **Random (uncorrelated) error cancels.** If $N$ blocks each carry independent relative error $\sigma$, the chip-total relative error scales as $\sim\sigma/\sqrt{N}$ — a fleet of $\pm25\%$ blocks can roll up to a $\pm8\text{–}10\%$ chip total. Summation is a variance-averaging operation.
- **Systematic (correlated) error accumulates.** A consistent 15% activity-overestimate in *every* block (e.g. from a continuously-toggling testbench SAIF (Switching Activity Interchange Format)) stays 15% at the chip — it does not wash out. These are the dangerous ones.

*Derivation of the $\sqrt N$.* Write each block as $P_i(1+\varepsilon_i)$ with $\mathbb{E}[\varepsilon_i]=0$, $\operatorname{Var}(\varepsilon_i)=\sigma^2$, the $\varepsilon_i$ independent. The estimated total is $\hat P=\sum_i P_i(1+\varepsilon_i)$, so its absolute error $\sum_i P_i\varepsilon_i$ has variance $\sum_i P_i^2\sigma^2$. For $N$ equal blocks ($P_i=P/N$) the error standard deviation is $\sigma(P/N)\sqrt N=\sigma P/\sqrt N$ — relative error $\sigma/\sqrt N$, the central-limit averaging (e.g. $N=9$ blocks at $\sigma=25\%$ → $25\%/3\approx8\%$). A *correlated* bias $\varepsilon_i=\beta$ (same sign every block) instead gives error $\sum_i P_i\beta=\beta P$, relative error $\beta$ **flat in $N$**: no averaging, because $N$ identical biases grow exactly as fast as the total they inflate. That contrast — $\sigma/\sqrt N$ shrinking vs $\beta$ constant — *is* why relative comparisons are trustworthy and absolute totals are not.

The practical corollary is the single most useful fact about trusting a model: **it is most accurate for *relative* comparisons and least accurate for *absolute* Watts.** Comparing design A to design B in the same model, the systematic error is common-mode and cancels almost entirely; quoting an absolute socket-power number exposes the full systematic bias. So:

| Validation target | Typical acceptance band |
|---|---|
| Per-block, calibrated | ±10–20% |
| Full-chip total, calibrated | ±10–15% |
| Full-chip *un*-calibrated | ±30% or worse |
| Relative (A vs B, same model) | tightest — errors are common-mode |

**Rule:** use the model freely to *rank* design points; validate any absolute total against at least one silicon or gate-level anchor before quoting it to a thermal team.

---

## 2. Why a chip is not the sum of its blocks: five coupling terms

This is the heart of the page — the $X_{\text{coupling}}$ term of §0 made explicit. Five phenomena make the assembled chip deviate from $\bigoplus X_{\text{leaf}}$, each in a specific direction, each with its own theory, and each with an **auditor's red flag** — the tell-tale wrong number a model that omits it will report. Two of the five (contention, queueing) fix the *performance composition operator*; one (overlap) decides whether it is $\max$ or $\Sigma$; two (DVFS, thermal) fix the *operating point* every leaf is evaluated at. The governor of §3 makes them mutually consistent.

### 2.1 Contention — the throughput face of a shared resource

Every shared resource — a DRAM/HBM channel, an LLC, a NoC link, an inter-chip link — has a finite **capacity** $R$ and serves $N$ clients each with **demand** $d_i$. There is one saturation knee, at $\sum_i d_i = R$, and achieved throughput is

$$
\text{achieved} \;=\; \min\!\Big(\textstyle\sum_i d_i,\; R\Big)
$$

- **Below the knee** ($\sum d_i < R$): every client gets its full demand; the resource is not the bottleneck.
- **Above the knee** ($\sum d_i > R$): aggregate throughput clamps at $R$, and each client's share collapses toward $R/N$ under fair arbitration. A core that measured 20 GB/s standalone sees 5 GB/s under an 8-way contended channel — the *same* leaf, a quarter of the bandwidth, purely from company.

*Arbitration decides who loses.* Round-robin / age-based arbiters split near-equally ($R/N$); priority or QoS (quality-of-service) arbiters protect one client at others' expense. The arbitration policy is therefore *part of the contention model* — the same saturated resource yields very different per-client outcomes under round-robin versus strict priority, which is why an MC that favours a latency-critical core, or a NoC with virtual-channel priorities, must be modeled as such and not as a fair split.

*Why $\min()$ is exact, and the share formula.* Work-conservation pins both regimes. A work-conserving server never idles while requests wait, so its output rate is exactly $\min(\text{offered},\text{capacity})=\min(\sum_i d_i,\,R)$ — offered load when it fits, capacity when it does not, with nothing in between (any "achieved" above $R$ would violate the capacity bound; anything below offered-and-under-$R$ would mean the server idled with work pending). Above the knee the capacity is partitioned by the arbiter's weights $w_i$: client $i$ gets $R\,w_i/\sum_j w_j$. Equal weights give the $R/N$ fair split; demand-proportional weights ($w_i=d_i$) give $R\,d_i/\sum_j d_j$ (each keeps its *fraction* of the jam); strict priority gives the top client its full $d_i$ and starves the rest with the remainder. *Worked number.* Eight cores each demanding $d=8$ GB/s share an $R=25.6$ GB/s channel: $\sum d=64>25.6$, so aggregate clamps at $25.6$ and equal arbitration delivers $25.6/8=3.2$ GB/s per core — a core that measured $8$ GB/s standalone keeps just **40%** of it. Give one core strict priority and it holds its full $8$ GB/s, dropping the other seven to $(25.6-8)/7=2.5$ GB/s each: the *same* saturated $R$, a $3.2\times$ per-client spread from arbitration policy alone.

Because a leaf measured alone always runs below its own private knee, **standalone leaf numbers overstate throughput the moment they are composed** — this is the mechanical reason full-chip $\ne \Sigma$ leaves for performance. The contended demand $\sum d_i$ must be evaluated with *every* client present; a per-core number measured in isolation is the wrong input.

> **Auditor's red flag:** a cluster model that reports $N\times$ throughput and feeds each core its *standalone* memory-CPI has no contention layer and is wrong by the queueing gap of §2.2.

### 2.2 Queueing — the latency face of the same resource

The $\min()$ of §2.1 captures throughput and *hides* the latency penalty, which is where naive models lie most. Model the shared resource as a queue with utilization $\rho = \sum_i d_i / R$. Even for the simplest (M/M/1-like) server, mean latency diverges as

$$
L \;\approx\; \frac{L_{\text{service}}}{1-\rho}, \qquad \rho \to 1 \Rightarrow L \to \infty
$$

where $L_{\text{service}}$ = unloaded service time. Latency blows up **super-linearly well before throughput flatlines**: a channel at $\rho = 0.9$ already carries roughly $10\times$ its unloaded service latency while still delivering "90% of peak bandwidth." So "achieved BW = 90% of peak" and "latency is fine" are not the same statement — the last 10% of bandwidth is bought with a latency cliff, and for a latency-sensitive core (one whose ROB cannot cover the stall) that cliff is the real cost. This is why contention and queueing are two faces of one resource: §2.1 is what the resource delivers, §2.2 is what it charges to deliver it, and both are governed by the same $\rho$.

**Deriving the $1/(1-\rho)$ blow-up (M/M/1).** Model the shared resource as a single server with Poisson arrivals at rate $\lambda$ and exponential service at rate $\mu$, so $\rho=\lambda/\mu$ is the utilization and $T_0=1/\mu$ the unloaded service time. Let $p_n$ be the steady-state probability of $n$ requests in the system. In steady state the probability flux across the boundary between states $n$ and $n{+}1$ must balance (a birth–death chain), giving detailed balance $\lambda p_n=\mu p_{n+1}$, hence $p_{n+1}=\rho\,p_n$ and, normalized by $\sum_{n\ge0}p_n=1$, the geometric law $p_n=(1-\rho)\rho^n$. The mean occupancy is

$$
\bar N=\sum_{n\ge0} n\,p_n=(1-\rho)\sum_{n\ge0} n\rho^n=(1-\rho)\frac{\rho}{(1-\rho)^2}=\frac{\rho}{1-\rho}.
$$

Little's law (§2.3), $\bar N=\lambda\bar T$, converts that *population* into a *latency*:

$$
\bar T=\frac{\bar N}{\lambda}=\frac{\rho/(1-\rho)}{\rho\mu}=\frac{1}{\mu(1-\rho)}=\frac{T_0}{1-\rho}=\frac{1}{\mu-\lambda},
$$

where $\bar T$ = mean response time (queue wait + service), $T_0=1/\mu$ = unloaded service time, $\rho=\lambda/\mu$ = utilization. This is the promised $T_0/(1-\rho)$: the service time is unchanged, but the *wait* to reach the server, $\bar T-T_0=\frac{\rho}{1-\rho}T_0$, diverges as $\rho\to1$ — because draining a backlog needs a run of below-average inter-arrival gaps, and such runs get exponentially rarer as arrivals approach capacity.

**The variance correction (M/D/1, Pollaczek–Khinchine).** Exponential service is pessimistic — a DRAM burst or a NoC flit has *nearly deterministic* duration. For any service distribution (M/G/1) the Pollaczek–Khinchine formula gives the mean wait

$$
W_q=\frac{\rho}{1-\rho}\cdot\frac{1+C_v^2}{2}\,T_0,\qquad C_v=\frac{\sigma_S}{T_0},
$$

where $C_v$ = coefficient of variation of the service time (std ÷ mean) and $\sigma_S$ = service-time standard deviation. Exponential service has $C_v=1$, recovering M/M/1's $W_q=\frac{\rho}{1-\rho}T_0$; **deterministic** service (M/D/1) has $C_v=0$, halving it to $W_q=\frac{\rho}{2(1-\rho)}T_0$ — the *half-variance form*. Real memory/NoC service sits between and closer to deterministic, so M/M/1 is an *upper* bound on the queue and M/D/1 a *lower* one; the truth is bracketed by the two.

*Worked number — DRAM/NoC latency at 60% vs 90% load.* Take unloaded service $T_0=80$ ns (a loaded-DRAM read, or a few-hop NoC traversal), response time $\bar T=T_0\big(1+\tfrac{\rho}{1-\rho}\cdot\tfrac{1+C_v^2}{2}\big)$:

| $\rho$ | M/M/1 ($C_v{=}1$): $\bar T=T_0/(1-\rho)$ | M/D/1 ($C_v{=}0$): $\bar T=T_0\big(1+\tfrac{\rho}{2(1-\rho)}\big)$ |
|---|---|---|
| 0.60 | $80/0.40=200$ ns ($2.5\times$) | $80(1+0.75)=140$ ns ($1.75\times$) |
| 0.90 | $80/0.10=800$ ns ($10\times$) | $80(1+4.5)=440$ ns ($5.5\times$) |

Pushing utilization from 60% to 90% — a mere $1.5\times$ more offered load — inflates M/M/1 latency **4×** ($200\to800$ ns) while aggregate bandwidth climbed only from 60% to 90% of peak. That asymmetry is the whole lesson: **the last third of a resource's bandwidth is bought with a 4× latency cliff.** For a core whose ROB (reorder buffer) hides ~300 ns of memory latency, the channel is *invisible* at $\rho=0.6$ (200 ns, covered) and a *stall machine* at $\rho=0.9$ (800 ns, uncovered) — the same channel, the same leaf model, flipped by load alone. The M/D/1 column shows the payoff of a first-ready, first-come, first-served (FR-FCFS) controller that *regularizes* service (turning conflicts into near-deterministic row-hits, lowering $C_v$): it roughly halves the queue term at every $\rho$.

The M/M/1 form is illustrative — real memory controllers are FR-FCFS (first-ready, first-come-first-served) with finite queues — but the $1/(1-\rho)$ blow-up is qualitatively universal, and it is the reason the memory-CPI in a cluster model must be driven by the *contended* channel at its true $\rho$, never the standalone one.

> **Auditor's red flag:** a model that reports achieved bandwidth near peak and unloaded latency in the same breath has priced the throughput but not the queue.

### 2.3 Overlap — max, not sum

A chip does compute, memory movement, and (at pod scale) communication *concurrently* when the hardware and the schedule allow it. The step time is then the **max**, not the sum, of the phase times:

$$
t_{step} = \max\big(t_{comp},\, t_{mem},\, t_{comm}\big)\ \text{(overlapped)} \qquad\text{vs}\qquad t_{step} = t_{comp}+t_{mem}+t_{comm}\ \text{(serial)}
$$

Overlap is what makes the *bottleneck resource — and only it —* set performance: if $t_{mem}$ is hidden under $t_{comp}$ you are compute-bound and memory is free; if the overlap fails, the times add and everything is slower. Real designs live between these bounds, and the **degree of overlap is a first-class model parameter, not a given.**

**Deriving $\max$ vs $\sum$ from the pipeline.** Stream $n$ tiles, each needing compute $c$ and memory-move $m$, through a double-buffered engine (two SRAM tiles: while the array consumes tile $k$, the DMA fills tile $k{+}1$). Serially (one buffer), tile $k$ costs $c+m$ and the run is $n(c+m)$. Double-buffered, after a one-tile prologue fill of $m$ the two resources run *concurrently*: in each slot the array is busy for $c$ and the DMA for $m$, and the slot advances only when **both** finish, so the slot length is $\max(c,m)$ and the run is $m+n\max(c,m)$. Amortized over large $n$,

$$
t_{\text{tile}}\ \xrightarrow{\ n\to\infty\ }\ \max(c,m)\qquad\text{vs.}\qquad c+m\ \ (\text{serial}),
$$

so overlap replaces the *sum* by the *max* — the hidden resource is free up to the point where it *equals* the exposed one. The double-buffering speedup is $\frac{c+m}{\max(c,m)}\in[1,2]$, maximal ($2\times$) exactly when $c=m$ (perfectly balanced) and vanishing when one term dominates (nothing left to hide). This is the chip-scale restatement of the roofline $\max$ of [SoC/chiplet workload and performance methods](../00_Design_Methodology/01_SoC_Chiplet_Workloads_Performance_and_DSE.md): an operator's time is the critical path through concurrent resources, never their sum.

**Partial overlap — the realistic interpolation.** Perfect $\max$ needs the *entire* hidden term to fit under the exposed one; real schedules hide only a fraction $\phi\in[0,1]$ of the smaller phase (finite buffering, imperfect prefetch distance, dependence stalls). With phase times $t_a\ge t_b$, the step interpolates linearly between the two bounds:

$$
t_{\text{step}}(\phi)=t_a+(1-\phi)\,t_b=\underbrace{t_a+t_b}_{\phi=0,\ \text{serial}\ (\sum)}\ \longrightarrow\ \underbrace{t_a}_{\phi=1,\ \text{overlapped}\ (\max)},
$$

where $\phi$ = overlap fraction (0 = none, 1 = smaller phase fully hidden), $t_a$ = larger phase time, $t_b$ = smaller. The sting is that $\phi$ **degrades under contention**: §2.1–2.2 inflate $t_b$ (the memory/comm phase) toward and past $t_a$, at which point even $\phi=1$ can no longer hide it and the step re-expands. Overlap and contention are thus coupled — contention is exactly what turns a hidden term back into an exposed one. *Worked number.* A training microstep has $t_{\text{comp}}=5.0$ ms and $t_{\text{comm}}=3.0$ ms (an AllReduce). Serial: $8.0$ ms. Perfect overlap: $\max(5,3)=5.0$ ms — the comm is free, a $1.6\times$ speedup. At $\phi=0.6$: $5.0+(1-0.6)\cdot3.0=6.2$ ms ($1.29\times$). Now let pod contention (§2.1) inflate the comm to $t_{\text{comm}}=6.0$ ms: even at $\phi=1$ the step is $\max(5,6)=6.0$ ms — the collective has become the *exposed* term, adding compute overlap buys nothing, and the fix is now bandwidth (a better torus embedding, §4.4), not scheduling. One knob — the contended comm time — moved the bottleneck across the $\max$.

Overlap is *conditional* — the $\max()$ holds only when movement is decoupled from consumption:

- **Double-buffering** (NPU §4.4): DMA fills buffer B while the array computes on A. Needs SRAM for *two* tiles; with one, DMA and compute serialize back to the sum and throughput halves.
- **Async DMA / prefetch** (CPU, GPU): loads issued far enough ahead hide memory latency behind compute — *provided* the prefetch distance covers the latency and the MSHRs (miss-status handling registers) do not fill.
- **Comm/compute overlap** (pod §4.4): AllReduce layer $L$ while computing layer $L{-}1$; needs an interleaved schedule and spare link bandwidth.

When a precondition fails, $t_{step}$ slides from $\max$ toward $\Sigma$. So the model must ask two questions, not one: *is it overlapped?* and *is the hidden term still smaller after §2.1–2.2 inflate it?*

**Little's law is the unifying principle.** How much concurrency does hiding latency require? To keep a resource of latency $L$ and bandwidth $B$ busy you need

$$
\text{in-flight work} \;=\; B \times L \quad\text{(the bandwidth–delay product)}
$$

Too few outstanding requests — too few MSHRs, too little prefetch distance, too low GPU occupancy — and the pipe runs dry: you are latency-bound *even below the §2.1 knee*. CPU memory-level parallelism, GPU thread-level parallelism, and NPU double-buffer depth are three names for one mechanism — hold enough in-flight work to keep the bottleneck saturated and thus keep the $\max()$ in force. This also sets up a tension with §2.1–2.2: more in-flight work raises the offered load $\sum d_i$, pushing $\rho \to 1$ where $L$ itself grows and *demands even more* in-flight work. The sweet spot is enough concurrency to hide latency but not so much that the queue explodes.

> **Auditor's red flag:** a model that assumes free overlap (always $\max$) overstates scaling; one that always sums understates it. The overlap fraction is the parameter, and it degrades under contention.

### 2.4 DVFS and turbo — frequency is allocated, not fixed

The sum of every block at its peak $(V,f)$ *exceeds* what the package can dissipate: $\sum P_{block,max} \gg \text{TDP}$. Therefore the per-domain operating point $(V,f)$ is **not a datasheet constant — it is continuously allocated** by a power manager so that $\sum P \le \text{budget}$. Frequency is an *output* of the budget solve, not an input, and this is what §0 meant by "the operating point is wrong for every block": a leaf evaluated at nominal $f$ is evaluated at a frequency the chip may never sustain.

Why the allocation matters so much is the shape of the cost curve. Dynamic power is $P_{dyn} = \alpha C V^2 f$, and along the voltage–frequency curve a higher $f$ needs a higher $V$ (roughly $V \propto f$ in the usable range), so

$$
P_{dyn} \;\propto\; V^2 f \;\propto\; f^3
$$

Dynamic power grows **cubically** with frequency. A small $f$ cut buys a large power cut, and conversely raising the budget buys only a *cube-root* frequency gain — which is exactly why turbo hands the budget to *few* cores (a large $f$ on one core costs what a modest $f$ costs on three) and why sustained all-core clock sits well below single-core burst.

**Deriving $P_{dyn}=\alpha C V^2 f$.** A switching node is a capacitor $C$ charged to $V$ and discharged once per transition. Charging draws $Q=CV$ from the rail at voltage $V$, delivering energy $QV=CV^2$, of which exactly half lands on the capacitor ($\tfrac12CV^2$) and half is burned in the pull-up resistance regardless of its value; the discharge dumps the stored half through the pull-down. So a full charge/discharge cycle dissipates $CV^2$, and a node switching with probability $\alpha$ (the **activity factor**) each clock dissipates $\alpha CV^2$ per cycle, i.e. $P_{dyn}=\alpha CV^2 f$, where $\alpha$ = switching probability/cycle, $C$ = switched capacitance, $V$ = supply, $f$ = clock. Summed over the chip this is precisely the $\sum_u E_u A_u f$ of §1.1 with $E_u=\alpha_u C_u V^2$.

**Why $V$ tracks $f$: the delay law.** A gate's output slews as its drive current charges the next stage: $t_{\text{gate}}\approx C V/I_{\text{on}}$. In the alpha-power (Sakurai) model $I_{\text{on}}\propto(V-V_t)^{\alpha_v}$ with $\alpha_v\in[1,2]$ ($2$ = long-channel square law, $\approx1.3$ modern short-channel), so

$$
f_{\max}\propto\frac{1}{t_{\text{gate}}}\propto\frac{(V-V_t)^{\alpha_v}}{V}\ \xrightarrow[\ V\gg V_t\ ]{}\ V,
$$

where $V_t$ = threshold voltage. Well above threshold the curve is close to *linear* in $V$ (the "$V\propto f$" used above); but as $V\to V_t$ the overdrive $(V-V_t)$ collapses and frequency dies, so the linear approximation — and the cube below — hold only inside the usable DVFS window and **fail at the low-voltage floor**.

**Energy/op $\propto V^2$, power $\propto V^3$.** Divide by throughput. Energy *per cycle* is $P_{dyn}/f=\alpha CV^2$, **frequency-independent**, so the energy to do a fixed amount of work scales as

$$
E_{\text{op}}\propto V^2,\qquad\text{while}\qquad P_{dyn}=\alpha CV^2 f\propto V^3\ \ (\text{since }f\propto V).
$$

That asymmetry is the pivot of DVFS: **slowing down cuts power cubically but energy-per-op only quadratically**, because the work simply takes longer ($t\propto1/f$) and the $f$ in $P=E_{\text{op}}f$ cancels one power of the saving. Lowering $V$ is *always* good for dynamic energy; only the *time* it costs creates a trade.

**The energy–frequency Pareto and race-to-idle.** Total energy to finish a fixed $W$-cycle job is dynamic + leakage over the active window $t=W/f$ (treating $P_{\text{leak}}$ as set mainly by $V_t,T$, hence roughly $V$-independent):

$$
E_{\text{total}}(V)=\underbrace{k\,W\,V^2}_{\text{dynamic},\ \downarrow\text{ with }V}+\underbrace{P_{\text{leak}}\,\frac{W}{f}}_{\text{leakage},\ \propto 1/V,\ \uparrow\text{ as you slow}},
$$

where $k$ absorbs $\alpha C$ per cycle and $f\propto V$. The terms pull opposite ways; setting $dE_{\text{total}}/dV=2kWV-P_{\text{leak}}W/(aV^2)=0$ (with $f=aV$) gives an interior **energy-optimal voltage** $V^\star=(P_{\text{leak}}/2ka)^{1/3}\propto P_{\text{leak}}^{1/3}$: below it, slowing further *costs* energy. This is the quantitative **race-to-idle vs pace-to-deadline** split — above $V^\star$ (leakage small) energy keeps dropping as you slow, so *pace-to-deadline* wins; below it (leaky node) the minimum is behind you and *race-to-idle* (run fast, finish, power-gate so leakage stops) wins. *Worked number.* A kernel runs at $(V,f)=(1.0\text{ V},3.0\text{ GHz})$ drawing $P_{dyn}=10$ W, finishing in $1.0$ s. Slow it to $2.4$ GHz ($0.8\times$), so $V\approx0.8$ V and $P_{dyn}=10\cdot0.8^3=5.12$ W — a **20% $f$-cut nearly halves dynamic power** — but it now takes $1.25$ s, and its *dynamic* energy drops $10\to5.12\cdot1.25=6.4$ J (a $3.6$ J saving, the $V^2=0.64$ energy/op ratio). Whether that wins depends on leakage over the deadline $D=1.25$ s: the fast run finishes early and power-gates for the spare $0.25$ s, so it leaks only for $1.0$ s while the paced run leaks the whole $1.25$ s. Race-to-idle wins when the paced run's extra leakage $P_{\text{leak}}\cdot0.25$ exceeds the $3.6$ J dynamic saving — i.e. $P_{\text{leak}}>14.4$ W. So at $P_{\text{leak}}=4$ W pacing wins (fast+gate $(10{+}4)\cdot1.0=14$ J vs paced $(5.12{+}4)\cdot1.25=11.4$ J), while at $P_{\text{leak}}=20$ W race-to-idle wins (fast+gate $(10{+}20)\cdot1.0=30$ J vs paced $(5.12{+}20)\cdot1.25=31.4$ J). The crossover is $V^\star$ in disguise, and it slides toward race-to-idle as the node gets leakier — exactly the advanced-node trend.

Every vendor mechanism is the same idea — measure power, compare to a cap, actuate $(V,f)$ — under different names:

| Mechanism | Vendor | What it does |
|---|---|---|
| Turbo Boost | Intel | raises few-core $f$ above base while others idle, until $\sum P =$ budget |
| Precision Boost / PBO | AMD | opportunistic $f$ up to power/thermal/current limits |
| GPU Boost | NVIDIA | highest clock s.t. $P\le$ cap *and* $T\le T_{limit}$ (`nvidia-smi -pl`) |
| RAPL | Intel | HW enforces PL1 (sustained, $\tau\sim$ tens of s) and PL2 (burst, $\tau\sim$ ms); throttles $f$ when the running-average power exceeds the limit |

The two-timescale RAPL model (PL2 above PL1 for a window $\tau$) is not an implementation quirk — it is the chip *spending thermal capacitance* (§2.5), and it is why burst clock exceeds sustained clock.

A second consequence is the classic **race-to-idle vs pace-to-deadline** split, two opposite ways to spend a budget:

- **Race-to-idle:** run at max $f$, finish fast, drop to a deep idle state. Wins when idle (leakage-dominated) power is very low and there is a long idle tail to amortize — you pay the high active power only briefly.
- **Pace-to-deadline:** run at the *lowest* $f$ that still meets the deadline, exploiting $P\sim f^3$ so that stretching the work cuts energy super-linearly. Wins when there is a hard deadline and idle power is *not* negligible (leakage would burn during the idle tail anyway).

Which wins is the dynamic-power-management break-even of [Block_Activity_and_Power](../../../02_Power_and_Low_Power/02_Block_Activity_and_Power.md): race-to-idle pays only when $E_{\text{saved by idling}} > E_{\text{extra to run fast}} + E_{\text{transition}}$. On leaky advanced nodes (high static power) race-to-idle is increasingly favored; on low-leakage or deadline-bound work, pace-to-deadline.

> **Auditor's red flag:** quoting throughput at peak $f$ across all cores at once ignores that the budget cannot fund it — the all-core sustained $f$ is the frequency at which $\sum P = \text{TDP}$, always lower.

### 2.5 Thermal — power is temperature, and temperature is throttle

The steady-state thermal model is Ohm's law for heat — power is the "current," temperature rise the "voltage," thermal resistance the "resistance":

$$
T_j \;=\; T_{amb} + P\cdot R_{\theta ja}
$$

where $T_j$ = junction temperature, $T_{amb}$ = ambient, $P$ = dissipated power, $R_{\theta ja}$ = junction-to-ambient thermal resistance (°C/W), the *series* sum of die→case, the TIM (thermal interface material), and heatsink→air. $R_{\theta ja}$ ranges from ~0.1–0.5 °C/W for a big server part under a large heatsink to >10 °C/W for a small package with poor cooling — and note it is the *cooling solution's* number, not the die's. This single equation says: at fixed cooling, $T_j$ rises *linearly* with power, so **power is temperature.**

The transient model adds thermal capacitance $C_\theta$ (the mass that must be heated), giving a first-order RC approach to steady state:

$$
T_j(t) = T_\infty + (T_0 - T_\infty)\,e^{-t/\tau_\theta}, \qquad \tau_\theta = R_\theta C_\theta
$$

The **thermal time constant** $\tau_\theta$ spans orders of magnitude within one package: the tiny die heats in **~ms** (small $C_\theta$), while the heatsink/package has $\tau_\theta$ of **~seconds to tens of seconds** (large $C_\theta$). That span *is* why burst beats sustained: for a few ms the die can dissipate above steady-state TDP because the heatsink has not warmed yet — precisely the headroom RAPL PL2 and Turbo (§2.4) exploit. When $T_j$ reaches $T_{j,max}$ (commonly ~95–105 °C for logic), the governor cuts $(V,f)$ to force $P$ down, so the **sustained clock is the frequency at which $T_j$ settles *at* $T_{j,max}$** — necessarily below the burst clock the die hits before the package warms.

**Deriving the transient from an energy balance.** The RC form is not analogy-by-assertion; it is the heat-balance ODE. Power $P$ flows in; heat leaves to ambient through $R_\theta$ at rate $(T_j-T_{amb})/R_\theta$; the surplus heats the mass $C_\theta$ (J/°C):

$$
C_\theta\frac{dT_j}{dt}=P-\frac{T_j-T_{amb}}{R_\theta}.
$$

Setting $dT_j/dt=0$ returns the algebraic law $T_j=T_{amb}+PR_\theta$; for constant $P$ the linear ODE solves to $T_j(t)=T_\infty+(T_0-T_\infty)e^{-t/\tau_\theta}$ with $T_\infty=T_{amb}+PR_\theta$ and $\tau_\theta=R_\theta C_\theta$ — the quoted transient, now with $\tau_\theta$ *derived* as the product that makes the exponent dimensionless ($[\text{°C/W}]\cdot[\text{J/°C}]=[\text{s}]$).

**The turbo-duration formula.** Start at $T_0$ and apply a burst power whose steady target $T_\infty=T_{amb}+P_{\text{burst}}R_\theta$ *exceeds* the limit $T_{j,max}$. The die climbs the exponential and trips when $T_j(t)=T_{j,max}$; solving for $t$,

$$
t_{\text{turbo}}=\tau_\theta\,\ln\!\frac{T_\infty-T_0}{T_\infty-T_{j,max}},
$$

where $T_0$ = temperature at burst onset, $T_\infty$ = the (unreachable) steady target of the burst, $T_{j,max}$ = throttle temperature. The burst lasts *longer* the larger the thermal mass ($\tau_\theta$) and the more headroom $T_{j,max}-T_0$ — which is exactly why the governor spends the die's small-$\tau$ capacitance first and the heatsink's large-$\tau$ capacitance second, and why PL2/Turbo windows are set to the package $\tau_\theta$ rather than chosen arbitrarily. *Worked number — how long can it boost?* (illustrative) In-case ambient $T_{amb}=45$ °C, cooling $R_\theta=0.30$ °C/W, limit $T_{j,max}=100$ °C. At a baseline $P_0=150$ W the junction settles at $T_0=45+150\cdot0.30=90$ °C (10 °C of headroom). Burst to $P_{\text{burst}}=250$ W: its steady target would be $T_\infty=45+250\cdot0.30=120$ °C — above the limit, so the burst is *thermally unsustainable* and runs only until the die hits 100 °C. With heatsink/package $C_\theta=100$ J/°C, $\tau_\theta=0.30\cdot100=30$ s, so

$$
t_{\text{turbo}}=30\,\ln\!\frac{120-90}{120-100}=30\,\ln\frac{30}{20}=30\cdot0.405=12.2\text{ s}.
$$

The part holds 250 W for ~12 s before throttling — the concrete meaning of "PL2 for $\tau\sim$ tens of seconds, then PL1." Halve the thermal mass and the boost halves to ~6 s; start hotter ($T_0=95$ °C) and it collapses to $30\ln(25/20)=6.7$ s. The **sustained** ceiling is then the fixed point of the throttle loop — the $P(f)$ at which $T_{amb}+P(f)R_\theta=T_{j,max}$: solving $45+0.30\,P=100$ gives $P_{\text{sust}}=183$ W, and the governor lowers $f$ until draw sits at that 183 W ceiling.

The deepest coupling is a positive feedback. Subthreshold leakage rises *exponentially* with temperature — roughly **doubling every ~10 °C** — closing a loop:

$$
T_j\uparrow \;\Rightarrow\; P_{leak}\uparrow \;\Rightarrow\; P_{total}\uparrow \;\Rightarrow\; T_j\uparrow
$$

Usually the loop gain is below 1 — cooling removes heat faster than leakage adds it — so it merely inflates steady-state $T_j$ and leakage, which is already reason enough that **leakage must be modeled at the operating temperature, not at 25 °C.** But if the gain exceeds 1 (weak cooling, high $R_\theta$, a leaky node, high $V$) it becomes **thermal runaway**: $T$ and leakage diverge until the part hard-throttles or trips protection. This makes the thermal and leakage models *mutually coupled* — you cannot solve one without the other — and it is the deepest reason perf, power, and thermal are a single co-model (§3).

> **Auditor's red flag:** reporting peak/burst clock as if it were sustained — the same error as ignoring the power budget (§2.4), because thermal *is* a budget, expressed in °C over $\tau_\theta$.

---

## 3. The system loop: one fixed point, not two models

The five terms of §2 do not act in sequence; they close a loop. Trace the dependencies: a performance number depends on the clock $f$; $f$ is set by the power budget (§2.4) and the thermal cap (§2.5); the power drawn depends on activity, which depends on the *achieved* performance after contention (§2.1–2.2) and overlap (§2.3); and leakage — a power term — depends on temperature, which depends on power (§2.5). **Every arrow points at every other.** There is no "compute performance first, then estimate power" — it is one **fixed-point solve** for a self-consistent operating point.

The full-chip model is therefore four parts wired in a loop:

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk", "nodeSpacing": 60, "rankSpacing": 60, "htmlLabels": false}}}%%
flowchart TD
    GOV["Governor / control loop (§2.4)<br/>DVFS + throttle: pick V,f per domain,<br/>gate idle blocks (RAPL / Boost)"]
    PWR["Power / thermal layer (§2.5)<br/>Tj = Tamb + P·Rθ (+ transient Cθ)<br/>P ≤ TDP/RAPL cap; leakage(T) loop"]
    CON["Contention / overlap layer (§2.1–2.3)<br/>achieved = min(Σd, R); t = max(·), not Σ"]
    LEAF["Leaf models (§1)<br/>CACTI / McPAT / DRAMPower / DSENT"]
    GOV -->|"sets V, f"| PWR
    PWR -->|config| CON
    CON -->|config| LEAF
    LEAF -->|"per-event energy × activity"| CON
    CON -->|"activity @ (V,f)"| PWR
    PWR -->|"P, Tj  (P = Σ leaves)"| GOV
    classDef g fill:#fde68a,stroke:#b45309,color:#000
    classDef p fill:#fee2e2,stroke:#b91c1c,color:#000
    classDef c fill:#dcfce7,stroke:#15803d,color:#000
    classDef l fill:#dbeafe,stroke:#1d4ed8,color:#000
    class GOV g
    class PWR p
    class CON c
    class LEAF l
```

Control flows *down* (the governor sets $(V,f)$ and gating); computed quantities flow *back up* (leaf energies become achieved throughput and true activity, which become $P$ and $T_j$). The solve iterates: pick $(V,f)$ → leaves emit activity → contention/overlap turn activity into achieved performance *and* the true (contended) activity counts → those give $P$ → $P$ gives $T_j$ (with the leakage–$T$ loop) → the governor checks $P\le\text{budget}$ and $T_j\le T_{j,max}$, adjusts $(V,f)$, and repeats until nothing moves. Only at that self-consistent point is a perf *or* power number meaningful — quoting either before the loop closes is quoting a transient the chip passes through, not a state it sits in.

On real silicon this loop *is* the governor, running in hardware/firmware at millisecond granularity: measure (RAPL counters, on-die current and thermal sensors) → compare to budget/limit → actuate $(V,f)$ and gating → repeat. In a model it is a solver iterating the four boxes to convergence. Either way, **this loop is the precise reason a leaf-only model is not a chip model** — it has the bottom box and none of the three above it.

---

## 4. One equation, three architectures

Everything above is architecture-independent, which is the point: CPU, GPU, and NPU are the *same* composition and the *same* five coupling terms with different labels. This section is deliberately compact — it carries the load-bearing numbers and vendor data, not a re-derivation. For the microarchitecture behind each, see [GPU_Architecture](../../02_GPU_Architecture/01_Core_Architecture/01_GPU_Architecture.md) and [NPU_Accelerators](../../03_NPU_Architecture/01_Compute_Dataflows/01_NPU_Accelerators.md); for how each is *simulated*, [06_Simulators](00_Index.md).

### 4.1 The hierarchies line up

| CPU | GPU | NPU | Shared glue that appears at this level |
|---|---|---|---|
| core | SM (streaming multiprocessor) | systolic array + vector unit | pipeline/warp/dataflow control, local clock |
| cluster (N cores + LLC) | GPC (SM cluster) | NPU core (array + SRAM + DMA) | intra-cluster crossbar, regional clock domain |
| SoC (+ DDR) | full GPU (+ L2, HBM) | chip (multi-core + HBM + ICI) | shared memory pool, uncore, chip-wide power cap |
| — | — | pod (multi-chip torus) | inter-chip network, collectives |

The roll-up rule (§1.1), the calibration chain (§1.2), and the loop (§3) are identical at every row. What differs is *which* coupling term dominates: the CPU foregrounds contention on one DRAM channel, the GPU foregrounds the HBM-bandwidth knee under a power cap, and the NPU foregrounds overlap — on-chip (double-buffering) and across the pod (collectives).

### 4.2 CPU: the DRAM channel is the textbook contention instance

Climb core → cluster → SoC and the coupling appears exactly on schedule. At the **cluster**, a shared LLC means co-runners evict each other's lines, so *effective* per-core miss rate (MPKI, misses per kilo-instruction) rises as capacity splits $N$ ways, and coherence traffic (snoops, invalidations, writebacks under MESI/MOESI, or point-to-point messages under a directory protocol like CHI, [ACE_and_CHI](../../01_CPU_Architecture/06_Coherence_and_Consistency/03_ACE_and_CHI.md)) adds NoC energy that scales with the miss rate. Sub-linear scaling follows directly — Amdahl *and* contention:

$$
\text{Speedup}(N) \;=\; \frac{1}{(1-p) + p/N + C(N)}
$$

where $C(N)$ is the contention term that *grows* with $N$ (LLC eviction, NoC and channel queueing, §2.1–2.2). This is why 8 memory-heavy cores deliver ~4–5× rather than 8×, while a cache-fitting compute phase approaches Amdahl's ceiling.

**From Amdahl to the Universal Scalability Law.** Amdahl caps speedup with a *serial* fraction but predicts monotone gains; real machines get *worse* past a point, which needs a second penalty. The **USL** (Gunther) writes the $N$-way speedup as

$$
S(N)=\frac{N}{1+\sigma(N-1)+\kappa N(N-1)},
$$

where $\sigma$ = **contention** coefficient (the serialized/queued fraction a growing crowd must take turns on, §2.1–2.2) and $\kappa$ = **coherency** coefficient (the cost of keeping $N$ workers mutually consistent: snoops, invalidations, barriers). The growth orders differ: $\sigma(N-1)$ is *linear* (a shared queue serves one at a time, so the wait grows with the number waiting), while $\kappa N(N-1)$ is *quadratic* — it counts the $\binom{N}{2}$ **pairs** that must be kept coherent, since point-to-point consistency traffic is all-pairs. Setting $\kappa=0$ recovers Amdahl with $\sigma\leftrightarrow(1-p)$ (as $N\to\infty$, $S\to1/\sigma$, the serial ceiling); the heuristic $C(N)$ above *is* $\sigma(N-1)+\kappa N(N-1)$ made explicit.

**Why there is a peak — and it retrogrades.** The quadratic term makes adding cores eventually *lose*. Maximize $S(N)$: with $D(N)=1+\sigma(N-1)+\kappa N(N-1)$, $\frac{dS}{dN}=\frac{D-ND'}{D^2}$, and $D-ND'=(1-\sigma)-\kappa N^2$ (the linear and constant pieces cancel), zero at

$$
N^\star=\sqrt{\frac{1-\sigma}{\kappa}}.
$$

Below $N^\star$ the numerator's $N$ outruns the denominator; above it the $\kappa N^2$ coherency cost dominates and **throughput falls as cores are added** — the *retrograde* regime, the quantitative reason a chip has an optimal core count rather than an unbounded one. (Amdahl, $\kappa=0$, has $N^\star\to\infty$: it saturates but never retrogrades, so it cannot even express the observed peak.) *Worked number.* Take $\sigma=0.08$ (8% contention), $\kappa=0.001$ (0.1% coherency). Peak at $N^\star=\sqrt{(1-0.08)/0.001}=\sqrt{920}\approx30$ cores, where $S(30)=\frac{30}{1+0.08\cdot29+0.001\cdot30\cdot29}=\frac{30}{1+2.32+0.87}=\frac{30}{4.19}=7.2\times$. Doubling to $N=64$ *regresses* to $S(64)=\frac{64}{1+0.08\cdot63+0.001\cdot64\cdot63}=\frac{64}{1+5.04+4.03}=\frac{64}{10.07}=6.4\times$ — **more than double the cores, less speedup**, the excess eaten by the $O(N^2)$ coherency term. The "~4–5× on 8 cores" above is the same curve at $N=8$: $S(8)=\frac{8}{1+0.08\cdot7+0.001\cdot8\cdot7}=\frac{8}{1.62}=4.9\times$. Past $N^\star$ the only fixes are lowering $\kappa$ (larger private caches, NUCA, relaxed consistency) or $\sigma$ (less sharing) — attacking the *coefficients*, never $N$.

At the **SoC**, the DRAM channel is the canonical shared resource. Its *performance* and its *energy* are modeled by different tools — a distinction an auditor watches for: performance (achieved BW, latency, scheduling) from a cycle-level channel model (Ramulator / DRAMSim3), energy from a state-integrator (DRAMPower), which computes $\sum_{\text{states}}(\text{time in state}\times \text{IDD}\times V_{DD})$. Two device facts survive from the parameter detail (which lives in [DDR_Controller](../02_Shared_Memory/01_DDR_Controller.md) and [Memory](../00_Design_Methodology/02_SoC_Chiplet_PPA_and_Physical_Implementation.md)):

- **Achieved BW is access-pattern-bound, not just peak.** A row-buffer **hit** costs only $t_{CAS}$; a **miss** adds an activate; a **conflict** adds a precharge *and* an activate. FR-FCFS scheduling reorders to favour row-hits, so a streaming, well-interleaved stream reaches ~80–90% of peak while a conflict-heavy random stream falls to ~40–50% — the *same* channel, half the bandwidth, from access pattern alone.
- **Refresh is an unavoidable off-the-top tax:** $\approx t_{RFC}/t_{REFI}$, so **~4–5% on DDR4** (~350 ns / 7.8 µs) but **~8–10% on DDR5** ($t_{REFI}$ halved to ~3.9 µs while $t_{RFC}$ held), worsening at higher density.

The **interaction** is §2.1–2.2 in the flesh: the channel is shared across all cores, so as they contend the MC queue saturates, aggregate BW stays near peak (FR-FCFS) but *per-core* BW divides toward $R/N$ and *per-request latency* balloons via queueing. A core that saw 20 GB/s standalone may see 5 GB/s under 8-core load — which is why the cluster model's memory-CPI *must* be driven by the contended channel, never the standalone one.

### 4.3 GPU: the HBM roofline knee, under a power cap

The GPU re-runs the hierarchy with a *throughput* philosophy — thousands of threads, wide SIMT (single-instruction, multiple-thread) lanes, memory bandwidth as the first-class shared resource — and hides latency not with OoO speculation but with **thread-level parallelism**: many resident warps, and on a stall the scheduler switches to a ready warp for free. **Occupancy** (resident warps ÷ hardware max) quantifies the available TLP, but it is a *ceiling on activity, not activity itself* — a 100%-occupant kernel can still be issue-starved if every warp is blocked on the same saturated memory system. That is Little's law (§2.3): occupancy only helps if enough warps are *eligible* to keep $B\times L$ bytes in flight.

The dominant coupling is HBM (high-bandwidth memory) bandwidth shared across all SMs — the §2.1 knee, at GPU scale:

$$
B_{achieved} = \min\!\Big(\textstyle\sum_i \text{demand}_i,\; B_{HBM}\Big), \qquad N^\star = \frac{B_{HBM}}{\text{demand per SM}}
$$

($N^\star$ is just the §2.1 knee in units of SMs: set $\sum_i\text{demand}_i=N\,d=B_{HBM}$ and solve. Past it, $\partial B_{achieved}/\partial N=0$ — the marginal SM adds compute *demand* but zero delivered *bytes*, so it buys only queueing and power.) At per-SM demand 30 GB/s and $B_{HBM}=3$ TB/s the knee is $N^\star = 100$ active SMs; on a ~130-SM die, all-SM streaming pushes $\sum\text{demand}=3.9$ TB/s past $B_{HBM}$, so achieved saturates at 3 TB/s and per-SM effective BW collapses from 30 → ~23 GB/s while every SM's memory-CPI rises together. Past the knee, *adding SMs adds no throughput* — only contention and power. Data-movement energy is a real uncore line, not a floor: **HBM3 ≈ 0.29 pJ/bit** at the interface, **NVLink ≈ 1.3 pJ/bit** (~5× more efficient than **PCIe Gen5 ≈ 6–7 pJ/bit**), so a multi-GPU AllReduce over NVLink is a quantifiable energy term.

The GPU is where **both** coupling families bite at once: the *contention* layer collapses per-SM BW (above), and the *budget* layer (§2.4–2.5) runs GPU Boost — the highest clock with $P\le\text{cap}$ and $T\le T_{limit}$. These interact perversely: a memory-bound phase draws *less* core power, so Boost may *raise* the clock — which buys nothing, because the bottleneck is HBM, not compute; conversely a compute-bound tensor-core phase hits the power cap and throttles the very compute rate that defined it. A correct model co-solves BW-sharing and the power-capped clock in one loop (§3).

> **Auditor's red flag:** reporting peak TFLOPs × peak clock × all SMs ignores *both* layers — the HBM knee and the power cap.

### 4.4 NPU: overlap on-chip, collectives across the pod

The NPU (neural processing unit; TPU-class dataflow accelerator) climbs one level higher — PE/MAC → systolic array → core → chip → **pod** — because AI systems are multi-chip, and its philosophy inverts the GPU's: instead of many latency-hiding threads, a **deterministic dataflow** streams operands through a fixed array, and the whole game is *overlap* — keep the array fed.

**On-chip, overlap is §2.3 as the single most important modeling fact.** An NPU core wraps the array with a large on-chip SRAM (static RAM) and DMA (direct-memory-access) engines; with the SRAM double-buffered, DMA fills buffer B while the array computes on A, so

$$
t_{tile} = \max\big(t_{compute},\ t_{DMA}\big)\ \text{(overlapped)} \qquad\text{NOT}\qquad t_{compute}+t_{DMA}\ \text{(serial)}
$$

Which term dominates classifies the tile — SA-bound (compute-limited), VU-bound (too much elementwise vs GEMM), or HBM-bound (DMA-limited, low arithmetic intensity) — and *arithmetic intensity decides the flip*, exactly the roofline. A concrete tile: a $128\times128$ INT8 weight sub-tile is 16 KB, streaming in $t_{DMA}\approx 16\text{ KB}/1.2\text{ TB/s}\approx 13$ ns; if $t_{compute}\approx 90$ ns ($K{=}128$ at ~1.4 GHz) then DMA is fully hidden and the tile is SA-bound. But the overlap is *conditional* (§2.3): the buffer must hold *two* tiles (single-buffered, DMA and compute serialize back to the sum and throughput halves) *and* HBM BW must satisfy $t_{DMA}\le t_{compute}$. When sibling cores contend for HBM (§2.1), the per-core $t_{DMA}$ grows and tiles that were SA-bound standalone flip HBM-bound — the NPU's contended-vs-standalone regime change. A leakage note peculiar to this class: because the dataflow schedule is *known ahead of time*, fine-grained gating can follow the compute wavefront with near-zero mis-wake cost (unlike speculative CPU gating), which matters because **30–72% of NPU energy is static** on weakly-managed designs (ReGate, MICRO 2025).

**Across the pod, overlap becomes collectives, and the network is the bottleneck.** Hundreds–thousands of chips wire into a torus (2D = 4 neighbours, TPU v2/v3-class; 3D = 6 neighbours, e.g. $4\times4\times4$ cubes on TPU v4+), and the dominant operation is **AllReduce** for gradient/activation sync. Ring-AllReduce runtime (bandwidth-optimal, Patarasuk & Yuan 2009):

$$
t_{AllReduce} \approx \underbrace{\frac{2(N-1)}{N}\cdot\frac{S}{B_{link}}}_{\text{bandwidth term}} + \underbrace{2(N-1)\,\ell}_{\text{latency term}}
$$

where $N$ = chips in the ring, $S$ = payload bytes, $B_{link}$ = per-link ICI (inter-chip interconnect) bandwidth, $\ell$ = per-hop latency. *Why $2(N-1)/N$* (sketch; full derivation in [SoC/chiplet workload and performance methods](../00_Design_Methodology/01_SoC_Chiplet_Workloads_Performance_and_DSE.md)): split the payload into $N$ chunks of $S/N$. Reduce-scatter runs $N-1$ steps, each chip sending one chunk per step until each owns one fully-reduced chunk — $(N-1)\tfrac{S}{N}$ bytes/chip; all-gather circulates those finished chunks in another $N-1$ steps — another $(N-1)\tfrac{S}{N}$; each chip sends and receives simultaneously, so wall-time is $\frac{2(N-1)}{N}\frac{S}{B_{link}}$. As $N\to\infty$ the bandwidth term $\to 2S/B_{link}$ — *independent of $N$*, the whole point of ring-AllReduce, since bigger rings use proportionally smaller chunks — while the latency term grows linearly, so at large $N$ or small $S$ latency dominates and hierarchical/tree collectives win. Worked: $N=256$, $S=256$ MB (bf16 shard), $B_{link}=1.2$ TB/s gives $t_{AllReduce}\approx 425\ \mu s$; against a ~5 ms per-step compute this is easily hidden — *compute-bound*, good scaling. The saving grace is again §2.3 overlap: AllReduce layer $L$ while the backward pass still computes layer $L{-}1$, so the effective step is $\max(t_{compute}, t_{comm})$ — but *only if* the schedule interleaves independent comm and compute *and* the ICI links are not already saturated by other tensor/pipeline-parallel traffic (the pod-scale contention layer). A model assuming free overlap overstates scaling.

> **Auditor's red flag:** classifying a tile's regime (or a pod's scaling) from *standalone* bandwidth — the regime is set by how many siblings contend *right now*, and by whether the overlap preconditions actually hold.

---

## 5. Numbers to remember

Order-of-magnitude anchors for back-of-envelope full-chip sanity checks. **State the basis; these are public-literature ballparks, not any specific silicon — use vendor/tool numbers for a real quote.**

| Quantity | Value | Basis / note |
|---|---|---|
| Uncore share of socket power | **~20–40%** | server RAPL package/uncore split; scales with BW, not a floor (§1.1, §4.2) |
| VRM efficiency $\eta$ | **~0.85–0.92** | delivery loss $P(1/\eta-1)$ ⇒ ~9–18% lost in regulator (§1.1) |
| DRAM refresh BW loss | **~4–5% DDR4, ~8–10% DDR5** | $t_{RFC}/t_{REFI}$ (~350 ns / 7.8 µs; ~295–410 ns / 3.9 µs); worse at high density (§4.2) |
| Clock-tree share of core dynamic | **~30–40%** | McPAT / [Block_Activity](../../../02_Power_and_Low_Power/02_Block_Activity_and_Power.md) (§1.1) |
| Calibrated arch-model accuracy | **±20–30%** abs, tighter relative | inherits CACTI/cell energies; relative error is common-mode (§1.2–1.3) |
| Random vs systematic block error | **$\sigma/\sqrt{N}$ cancels; systematic stays** | why relative $\gg$ absolute trust (§1.3) |
| HBM3 interface energy | **~0.29 pJ/bit** | NVIDIA/WikiChip class; order-of-mag (§4.3) |
| NVLink / PCIe Gen5 energy | **~1.3 / ~6–7 pJ/bit** | NVLink ~5× more efficient; comm baseline (§4.3) |
| NoC energy | **~a few pJ/flit-hop** | DSENT/Orion, node-dependent (§4.2) |
| NPU static (leakage) fraction | **~30–72%** | ReGate, MICRO 2025 (arXiv:2508.02536) (§4.4) |
| Dynamic power vs frequency | **$P\sim f^3$** over the DVFS range | $V\propto f$ ⇒ small $f$ cut, large power cut (§2.4) |
| Energy/op vs voltage | **$E_{op}\propto V^2$** (energy/cycle $=\alpha CV^2$, $f$-indep.) | slow-down cuts power cubically, energy quadratically (§2.4) |
| Energy-optimal voltage | **$V^\star\propto P_{leak}^{1/3}$**; leakier ⇒ race-to-idle | pace-to-deadline above $V^\star$, race below (§2.4) |
| RAPL windows | **PL1 $\tau\sim$ tens of s, PL2 $\tau\sim$ ms** | burst > sustained via thermal $C_\theta$ (§2.4–2.5) |
| Junction-to-ambient $R_{\theta ja}$ | **~0.1–0.5 °C/W** (server+heatsink) to **>10 °C/W** (small pkg) | it's the cooling's number, not the die's (§2.5) |
| Leakage vs temperature | **~2× per +10 °C** | subthreshold $T$-dependence; drives runaway loop (§2.5) |
| Die vs package thermal time constant | **~ms (die) vs ~s (heatsink)** | $\tau_\theta=R_\theta C_\theta$; enables burst > sustained (§2.5) |
| Turbo/boost duration | **$\tau_\theta\ln\frac{T_\infty-T_0}{T_\infty-T_{j,max}}$** | thermal mass + headroom set the window (§2.5) |
| $T_{j,max}$ (logic throttle point) | **~95–105 °C** | sustained clock settles here (§2.5) |
| Queueing blow-up, $\rho=0.6\to0.9$ | **~2.5× → ~10×** unloaded (M/M/1); M/D/1 halves the queue | $\bar T=T_0/(1-\rho)$; det. service $C_v{=}0$ (§2.2) |
| Ring-AllReduce bandwidth factor | **$2(N-1)/N \to 2$** | Patarasuk 2009; per-link cost $\to 2S/B_{link}$ (§4.4) |
| Multicore speedup (mem-heavy) | **~4–5× on 8 cores** | Amdahl + contention $C(N)$, not 8× (§4.2) |
| USL scalability peak | **$N^\star=\sqrt{(1-\sigma)/\kappa}$**, retrograde beyond | coherency $O(N^2)$ caps core count (§4.2) |
| Double-buffer speedup | **$\le 2\times$**, max at $t_{comp}=t_{mem}$ | $(c{+}m)/\max(c,m)$; overlap replaces $\sum$ by $\max$ (§2.3) |

---

## 6. The tool map

A full-chip model is a *pipeline of tools*: a performance/timing tool emits activity, a power tool turns activity into a per-block power map, and a thermal tool turns the power map + floorplan into temperatures — which feed the governor (§3). Read a row as "at this layer, use these three." The **mechanics of each tool** — how it models compute, memory, and time, and where its error comes from — live in [06_Simulators](00_Index.md); this table is only the *which-tool-where* index.

| Modeling layer | Performance / timing | Power / energy | Thermal |
|---|---|---|---|
| **Leaf (arrays, cells)** | — (analytical) | **CACTI** (array E), Accelergy | (feeds up) |
| **Block / core (CPU)** | **gem5** (cycle/CPI), Sniper | **McPAT** (energy × activity) | HotSpot (per-block map) |
| **DRAM channel** | **Ramulator**, DRAMSim3 | **DRAMPower** (IDD × time) | HotSpot (DRAM layer) |
| **NoC / interconnect** | gem5-Ruby (latency-under-load) | **DSENT**, Orion (pJ/flit-hop) | HotSpot |
| **GPU (SM→chip)** | **GPGPU-Sim / Accel-Sim** | **GPUWattch / AccelWattch** | HotSpot |
| **NPU (PE→chip)** | **SCALE-Sim**, ONNXim | Accelergy/Timeloop, NeuSim | HotSpot |
| **NPU pod (multi-chip)** | **NeuSim** (ICI, collectives) | NeuSim / composed per-chip | system-level |
| **Thermal (all)** | (consumes power map) | (consumes power map) | **HotSpot** (RC grid); HotSniper, CoMeT (2.5D/3D) |
| **DSE / orchestration** | roofline + fidelity ladder ([SoC/chiplet workload and performance methods](../00_Design_Methodology/01_SoC_Chiplet_Workloads_Performance_and_DSE.md)) | budget solve (§2.4) | budget solve (§2.5) |

The columns are *different tools on purpose* — the auditor's distinction of §4.2: Ramulator gives bandwidth, DRAMPower gives energy; never conflate a timing tool with a power tool. The thermal column is what §2.5 adds; HotSpot is the standard, consuming the power column's output plus a floorplan → per-block $T_j(t)$. For a full co-modeled DSE loop, an orchestration layer drives the perf→power→thermal→governor fixed point of §3 across a design space.

---

## Cross-references

- **Down the stack (what this composes):** [Block_Activity_and_Power](../../../02_Power_and_Low_Power/02_Block_Activity_and_Power.md) (the leaf energies, McPAT/CACTI bottom-up, DPM/gating, clock-tree share), [Power_Fundamentals](../../../02_Power_and_Low_Power/01_Power_Fundamentals.md) (the $\alpha CV^2f$ charge-accounting, the alpha-power delay law, and the DVFS energy/op $\propto V^2$ derivation behind §2.4), [SoC/chiplet workload and performance methods](../00_Design_Methodology/01_SoC_Chiplet_Workloads_Performance_and_DSE.md) (the CPI stack and roofline that supply activity and the bottleneck), [06_Simulators](00_Index.md) (how every tool in §6 actually models time, memory, and energy — [SoC/chiplet analytical models](../00_Design_Methodology/01_SoC_Chiplet_Workloads_Performance_and_DSE.md) for the M/M/1–M/D/1 queueing, Little's-law, roofline, and USL duals of §2 and §4.2), [Signal_Integrity_Reliability](../../../05_Backend_Physical_Design/02_Signal_Integrity_Reliability.md) (the PDN/IR-drop delivery loss of §1.1).
- **Up the stack (what builds on this):** [GPU_Architecture](../../02_GPU_Architecture/01_Core_Architecture/01_GPU_Architecture.md) and [NPU_Accelerators](../../03_NPU_Architecture/01_Compute_Dataflows/01_NPU_Accelerators.md) (the µarch behind §4's power/perf roll-ups), [Power_Analysis_and_Signoff](../../../02_Power_and_Low_Power/06_Power_Analysis_and_Signoff.md) and [Power_Reduction_Techniques](../../../02_Power_and_Low_Power/04_Power_Reduction_Techniques.md) (the budgeting/DVFS/gating mechanics at silicon).
- **Adjacent / device detail:** [Memory](../00_Design_Methodology/02_SoC_Chiplet_PPA_and_Physical_Implementation.md) & [DDR_Controller](../02_Shared_Memory/01_DDR_Controller.md) (the DRAM timing/IDD parameters §4.2 abstracts), [Network_on_Chip](../04_On_Chip_Networks/01_Network_on_Chip.md) & [ACE_and_CHI](../../01_CPU_Architecture/06_Coherence_and_Consistency/03_ACE_and_CHI.md) (the NoC/coherence fabric of §4.2), [Cache_Microarchitecture](../../01_CPU_Architecture/04_Cache_Hierarchy/01_Cache_Microarchitecture.md) (the LLC whose sharing drives cluster contention), [Cache_Coherence](../../01_CPU_Architecture/06_Coherence_and_Consistency/01_Cache_Coherence.md) (the invalidation, ownership-transfer, directory, and false-sharing costs behind that contention).

---

## References

1. Li, S. et al., "McPAT: An Integrated Power, Area, and Timing Modeling Framework for Multicore and Manycore Architectures," *MICRO*, 2009. The architectural energy×activity roll-up of §1.1.
2. Muralimanohar, N., Balasubramonian, R., and Jouppi, N.P., "CACTI 6.0: A Tool to Model Large Caches," HP Labs, 2009. The array energies §1 pushes down.
3. Williams, S., Waterman, A., and Patterson, D., "Roofline: An Insightful Visual Performance Model for Multicore Architectures," *CACM*, 52(4), 2009. The bottleneck/overlap model of §2.3.
4. Gunther, N.J., *Guerrilla Capacity Planning*, Springer, 2007. The Universal Scalability Law — Amdahl's serial term plus the $\sigma$-contention and $\kappa$-coherency terms, and the retrograde peak $N^\star=\sqrt{(1-\sigma)/\kappa}$ of §4.2.
5. Chandrasekar, K. et al., "DRAMPower: Open-Source DRAM Power & Energy Estimation Tool," 2012. The IDD × time-in-state integral of §4.2.
6. Skadron, K. et al., "Temperature-Aware Microarchitecture" (HotSpot), *ISCA*, 2003. The RC thermal model of §2.5.
7. Patarasuk, P. and Yuan, X., "Bandwidth Optimal All-reduce Algorithms for Clusters of Workstations," *JPDC*, 69(2), 2009. The ring-AllReduce runtime of §4.4.
8. Zhang, Y. et al., "ReGate: Fine-Grained Power Gating for NPUs," *MICRO*, 2025 (arXiv:2508.02536). The 30–72% static-energy finding of §4.4.
9. Intel, *Running Average Power Limit (RAPL)* and *Turbo Boost* documentation; NVIDIA GPU Boost / `nvidia-smi` power-cap docs; AMD Precision Boost. The governor mechanisms of §2.4.
10. Kleinrock, L., *Queueing Systems, Volume 1: Theory*, Wiley, 1975. The M/M/1 birth–death response time $T_0/(1-\rho)$ and the Pollaczek–Khinchine mean-wait formula (M/D/1 half-variance) behind the §2.2 queueing derivation.
11. Sakurai, T. and Newton, A.R., "Alpha-power law MOSFET model and its applications to CMOS inverter delay," *IEEE JSSC*, 25(2), 1990. The $f\propto(V-V_t)^{\alpha_v}/V$ delay law behind the §2.4 DVFS cube.
