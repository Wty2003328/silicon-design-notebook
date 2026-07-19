# Central Processing Unit AI Performance Analysis, Profiling, and Research Frontiers

> **First-time reader orientation:** Performance analysis asks which resource limits useful work and how the evidence proves it. Peak operations per second are only one bound. A production request may instead be limited by memory capacity, bandwidth, one dependent cache miss, software overhead, queueing, or a latency objective that forbids large batches.

> **Abbreviation key — skim now and return as needed:** central processing unit (CPU); artificial intelligence (AI); large language model (LLM); floating-point operations (FLOPs); operations per second (OPS); bfloat16 (BF16); integer 8-bit (INT8); not a number (NaN); last-level cache (LLC); non-uniform memory access (NUMA); translation lookaside buffer (TLB); instruction TLB (ITLB); hit-modified cache-to-cache response (HITM); performance monitoring unit (PMU); instructions per cycle (IPC); time to first token (TTFT); time per output token (TPOT); service-level objective (SLO); quality of service (QoS); key-value (KV); mixture of experts (MoE); mean time between failures (MTBF); dynamic voltage and frequency scaling (DVFS).

> **Prerequisites:** [CPU Workloads and Performance Modeling](../00_Design_Methodology/01_CPU_Workloads_Performance_and_DSE.md), [AI Serving](01_End_to_End_AI_Serving_on_CPUs.md), and [AI Operators](02_AI_Operators_on_CPU_Microarchitecture.md).
> **Hands off to:** [CPU Simulation Methodology](../00_Design_Methodology/03_CPU_Simulation_Methodology_and_Evidence.md) and [gem5](../08_Simulation/01_gem5.md) for controlled architectural experiments.

---

## 0. State the claim so it can be disproved

Weak claim: “quantization makes CPU inference faster.”

Research claim: “For decode batch $B\le8$ on a fixed dual-socket platform, a groupwise 4-bit weight layout reduces socket-memory bytes per generated token enough to improve 99th-percentile TPOT under the same model-quality threshold, after including unpacking and NUMA effects.”

The second claim names phase, workload range, mechanism, resource, metric, quality constraint, and confounders. A valid experiment then needs:

- a workload distribution and trace;
- hardware topology, firmware, operating system, runtime, compiler, and model versions;
- a baseline that differs in the intended mechanism;
- direct evidence for the mechanism (bytes, counters, generated code, placement);
- service metrics and correctness/quality;
- uncertainty across repetitions and input variation.

## 1. Count useful work and bytes before measuring time

### 1.1 Operation-count conventions

State whether a fused multiply-add counts as one instruction, one mathematical update, or two FLOPs. State whether integer operations, exponentials, address generation, and sparse skipped work enter the numerator. Peak-hardware marketing conventions may not match algorithmic conventions.

For transformer inference, the rule of thumb “roughly two operations per active parameter per token” captures dense linear layers when multiply and add count separately. It omits attention's sequence-dependent work, normalization, activation, routing, sampling, and embedding access. Use it as a scale estimate, not a complete model.

### 1.2 Byte accounting by boundary

There is no single “bytes moved” value. Distinguish:

- logical tensor bytes requested by the algorithm;
- register-file and bypass traffic;
- L1/L2 transfers including refetch and writeback;
- LLC misses and fills;
- local and remote socket-memory-controller bytes;
- storage, NIC, or accelerator-link traffic.

Arithmetic intensity at boundary $x$ is

$$
I_x=\frac{O_{useful}}{D_x}\quad[\text{operations/byte}].
$$

An operator can be compute-bound relative to DRAM yet L1-bandwidth-bound. Report the boundary whose bandwidth enters the roofline.

## 2. Roofline analysis, with its assumptions visible

For sustainable compute ceiling $P_x$ and bandwidth $B_x$ at a chosen boundary,

$$
P_{attainable}\le\min(P_x,I_xB_x).
$$

The ridge point is

$$
I^*=\frac{P_x}{B_x}.
$$

If measured intensity is below $I^*$, the simple model predicts a bandwidth-bound region. Above it, compute is the first roof. Real kernels lie below the roof because of vector/tile underfill, dependency chains, instruction mix, packing, cache conflicts, TLB walks, load imbalance, synchronization, and runtime overhead.

### 2.1 Build a hierarchy, not one roof

Create separate roofs for L1, private mid-level cache, LLC, local memory, remote memory, and accelerator links where relevant. Use sustainable bandwidth measured with access patterns and thread placement similar to the operator. A streaming benchmark with many independent loads is not a valid latency ceiling for pointer-chasing retrieval.

### 2.2 Roofline failure modes

The basic roofline assumes enough independent work to saturate bandwidth and treats all operations as equally executable. It can misdiagnose:

- a serial dependency chain with low achieved bandwidth;
- instruction-front-end or address-generation saturation;
- mixed integer/vector/matrix execution that contends for different ports;
- sparse imbalance where some workers finish early;
- capacity misses that change intensity with cache state;
- tail latency under queueing.

Use roofline to choose a hypothesis, then confirm with stall, port, cache, TLB, and memory-controller evidence.

## 3. Transformer prefill and decode models

Let:

- $P$ be active model parameters per token;
- $q_w$ be stored weight bytes per parameter including scale metadata;
- $S$ be prompt or context length;
- $B$ be active request batch;
- $L$ be layers;
- $H_{kv}$ be KV heads;
- $D_h$ be head dimension;
- $q_{kv}$ be bytes per KV element.

### 3.1 Weight and KV capacity

Approximate weight capacity is

$$
M_w=Pq_w.
$$

KV capacity per request is

$$
M_{KV}=2LH_{kv}D_hS q_{kv}.
$$

The model must include allocator block rounding, prefix sharing, fragmentation, runtime buffers, and page tables when predicting maximum concurrency.

### 3.2 Decode lower bound

At batch one, each token may stream most active weights once. A useful first bandwidth bound is

$$
TPOT_{weights}\ge\frac{Pq_w}{B_{mem,eff}}.
$$

With batch $B$, the same weight stream can serve $B$ token rows in a well-tiled kernel, so ideal weight arithmetic intensity rises roughly with $B$. KV traffic also grows with requests and context, and cache capacity may be exceeded. Therefore batching gain is sublinear and eventually saturates compute or memory bandwidth.

**Illustrative bound:** a 7-billion-parameter model at exactly 4 bits/parameter has 3.5 GB of raw weights before scales and padding. At 200 GB/s sustainable memory bandwidth, the weight-stream lower bound is 17.5 ms/token, or 57 tokens/s, for batch one. This is a roof, not a prediction: metadata, KV reads, non-GEMM operators, remote traffic, and imperfect bandwidth make actual TPOT larger.

### 3.3 Prefill lower bound

Prefill reuses weights across $BS$ token rows. Dense-layer work is roughly $2PBS$ operations, while compulsory weight bytes remain near $Pq_w$ for the phase if each weight is streamed once. Activation and attention traffic grow with $BS$ and $S^2$.

An approximate dense-layer intensity ignoring activations is

$$
I_{prefill,weights}\approx\frac{2PBS}{Pq_w}=\frac{2BS}{q_w}.
$$

This explains why prompt processing can fill matrix units while single-token decode remains bandwidth-shaped. At long sequence, attention and KV writes must be added explicitly; the approximation is not a whole-model formula.

### 3.4 Attention work

For standard multi-head prefill attention, score and value products are approximately proportional to

$$
O_{attn,prefill}\sim4BS^2H_qD_h,
$$

where $H_q$ is query-head count and the factor depends on counting convention. Decode attention is linear in context per new token:

$$
O_{attn,decode}\sim4BSH_qD_h.
$$

KV bytes depend on $H_{kv}$ rather than $H_q$ when K/V are shared across query heads. This distinction is why grouped-query attention can reduce serving memory traffic without proportionally reducing query computation.

## 4. Latency decomposition: TTFT and TPOT

For one generation request,

$$
TTFT=T_{ingress}+T_{queue,p}+T_{prepare}+T_{prefill}+T_{first\ sample}+T_{first\ output}.
$$

For output token $i$,

$$
TPOT_i=T_{queue,d,i}+T_{batch,i}+T_{model,i}+T_{sample,i}+T_{stream,i}.
$$

Inter-token latency and TPOT are sometimes used differently by serving systems; define the measurement boundary. Report median and tail distributions, not just mean. Long pauses matter to interactive users even if average token rate is unchanged.

### 4.1 Critical path and overlap

If tokenization and retrieval overlap, TTFT includes their maximum plus synchronization, not their sum. If model execution and output streaming overlap across requests, throughput can improve without reducing one request's compute latency. Use a dependency-aware trace:

$$
T_{critical}=\max_{p\in\mathcal P}\sum_{e\in p}t_e,
$$

where $\mathcal P$ is the set of dependency paths through the request graph.

### 4.2 Tail amplification

Tail events include page faults, remote NUMA placement, thermal/frequency changes, allocator locks, retrieval miss paths, batch interference, operating-system preemption, and background model work. A median-only microbenchmark systematically hides them. Record enough request attributes to explain tail cohorts without logging private content.

## 5. Batch-size effects from kernel to service

Let per-iteration model time at batch $B$ be $T_m(B)$. Token throughput is

$$
X_{tok}(B)=\frac{B}{T_m(B)}.
$$

But an arriving request may wait $T_{form}(B)$ for a batch and then share the iteration. A service objective constrains

$$
T_{queue}+T_{form}(B)+T_m(B)\le T_{SLO}.
$$

The throughput-optimal batch can violate latency. Dynamic batching chooses a maximum wait and a shape based on queue state. Continuous batching avoids waiting for all sequences to finish but creates variable matrix sizes and KV allocation churn.

### 5.1 Prefill/decode interference

A long prefill can delay decode iterations when both share the same CPU cores, caches, and memory channels. Chunking prefill limits each scheduling quantum but may reduce GEMM efficiency and reread data. Separating core pools or sockets isolates latency but may strand capacity and require KV transfer. The choice should be modeled using measured phase service curves, not assumed from accelerator studies.

### 5.2 Goodput

For request set $R$, define indicator $g_r=1$ only if request $r$ meets all latency and correctness objectives. Over interval $T$,

$$
G=\frac{\sum_{r\in R}g_r}{T}.
$$

This prevents a system from claiming success by completing more requests while missing most deadlines. State percentile target and admission policy; rejecting requests can otherwise make SLO attainment look artificially good.

## 6. Queueing theory for serving capacity

### 6.1 Utilization and the knee

For arrival rate $\lambda$, mean service time $E[S]$, and $k$ equivalent workers,

$$
\rho=\frac{\lambda E[S]}{k}.
$$

Queueing grows sharply as $\rho$ approaches one. The exact tail depends on arrival and service distributions, scheduling, batching, and worker heterogeneity.

For the simple M/M/1 model, the two “M” terms mean memoryless Poisson arrivals and independent, identically distributed exponential service times; “1” means one first-come, first-served server. With arrivals independent of service and stable utilization $0\le\rho<1$,

$$
E[T]=\frac{E[S]}{1-\rho}.
$$

AI service times are not generally exponential, batching violates independent per-request service, and a CPU pool is rarely one identical server. Use this as saturation intuition, not a final predictor.

### 6.2 Variability matters

Kingman's approximation applies to a GI/G/1 queue: independent renewal interarrivals with a general distribution, independent and identically distributed general service times, one first-come, first-served server, finite arrival/service variances, and $\rho<1$. It is a heavy-traffic mean-wait approximation, most useful near—not at—saturation:

$$
E[W_q]\approx\frac{\rho}{1-\rho}\frac{c_a^2+c_s^2}{2}E[S],
$$

where $c_a$ and $c_s$ are coefficients of variation for interarrival and service time. Variable prompt/output lengths raise $c_s$; bursty traffic raises $c_a$. Length-aware scheduling can reduce head-of-line blocking but risks starving long requests and may require prediction.

### 6.3 Little's law as a conservation check

For stable systems,

$$
N=\lambda T,
$$

where $N$ is average requests in the system and $T$ average residence time. If logs violate this substantially over a stable interval, arrival/completion boundaries or dropped requests are likely miscounted.

## 7. CPU microarchitecture performance model

A cycle stack can partition execution into retiring work and lost opportunity:

$$
C=C_{retire}+C_{frontend}+C_{bad\ speculation}+C_{backend\ core}+C_{memory}+C_{sync}+C_{idle}.
$$

This partition is conceptual; real events overlap and vendor top-down methodologies use specific counter definitions. Do not add overlapping raw stall events as if mutually exclusive.

### 7.1 Compute-limited signatures

Evidence may include:

- high utilization of vector/matrix execution slots;
- expected vector/tile instructions in disassembly;
- low memory-controller utilization relative to sustainable bandwidth;
- performance scaling with additional compute resources but not memory channels;
- predictable gain when increasing arithmetic throughput or improving tile fill.

Low IPC alone does not prove a compute limit; multi-cycle tile instructions may retire few instructions while doing much work.

### 7.2 Bandwidth-limited signatures

- local memory bandwidth near the access-pattern-specific ceiling;
- performance tracks weight precision or memory-channel count;
- additional compute cores provide little gain after saturation;
- LLC miss traffic matches tensor-byte accounting;
- batch size raises operations per weight byte until another roof becomes active.

### 7.3 Latency/parallelism-limited signatures

- many cache/TLB misses but low total bandwidth;
- long dependency chains and few outstanding misses;
- software prefetch or multiple concurrent queries raises bandwidth and performance;
- graph traversal or sparse lookup dominates rather than streaming tensors.

### 7.4 Frontend and software-overhead signatures

- small kernels, high branch pressure, instruction-cache misses, or frequent runtime calls;
- time between operator calls comparable to operator execution;
- lock contention, futex waits, context switches, or allocator activity;
- speedup from fusion, graph capture, persistent workers, or larger work quanta without changing arithmetic.

## 8. A layered profiling workflow

### 8.1 Level 1: service trace

Capture request arrival, admission, preparation, prefill, every decode iteration, sampling, output, cancellation, and completion. Include batch size, prompt/output lengths, model version, worker/socket, and state capacity. This establishes TTFT, TPOT, queueing, and tail cohorts.

### 8.2 Level 2: operator trace

Record operator shapes, data types, layout, primitive/kernel selection, thread count, packing, scratch allocation, and duration. Separate one-time compilation/packing from steady state. Check that operator sums reconcile with model intervals after accounting for overlap.

### 8.3 Level 3: generated code and ISA

Inspect disassembly or compiler reports:

- was the intended vector, dot-product, or matrix path selected?
- are tails masked, scalarized, or padded?
- are loads contiguous, gathered, or repeatedly converted?
- did fusion cause spills or excessive code size?
- are tile configurations or packed formats reused?

### 8.4 Level 4: PMU and uncore counters

Use the CPU PMU for retired instructions, cycles, branches, cache/TLB events, and execution activity. Use uncore/system counters for memory-controller and inter-socket traffic. Counter names and semantics are microarchitecture-specific; verify the vendor event guide and multiplexing error.

Recommended derived views include:

| Question | Evidence |
|---|---|
| Is the intended work executing? | retired vector/tile events, disassembly, useful operation count |
| Is memory local? | local/remote bytes, page placement, thread affinity |
| Is bandwidth saturated? | per-controller bytes/time versus measured sustainable roof |
| Are misses independent? | outstanding requests, miss latency, achieved bandwidth |
| Is translation important? | TLB misses, completed page walks, walk cycles |
| Is coherence interfering? | line invalidations, HITM/cache-to-cache events, shared-line ownership |
| Is the frontend limiting? | fetch/decode starvation, instruction-cache/ITLB misses, branch recovery |

### 8.5 Level 5: controlled perturbations

Change one resource or mechanism:

- vary batch, sequence length, precision, and operator shape;
- vary cores, SMT, sockets, memory channels, page size, and NUMA policy;
- pin data local versus deliberately remote;
- disable packing/fusion/prefetch or switch kernel paths;
- sweep arrival rate and background interference;
- compare warm, cold-cache, and cold-process conditions.

A true bottleneck should respond predictably. If halving weight bytes does not change memory traffic or time, the assumed weight-bandwidth mechanism is incomplete.

## 9. Measurement discipline

### 9.1 Control the platform

Record processor model/stepping, socket/core topology, memory population and speed, firmware, microcode, SMT, power limits, DVFS/turbo policy, page policy, kernel, compiler, libraries, model/checkpoint hash, and command line. Monitor frequency, temperature, throttling, and corrected errors during runs.

### 9.2 Warm-up and steady state

Warm-up may include page faults, code generation, weight packing, cache fill, branch training, memory allocation, and frequency transition. Report cold-start separately when it matters. For steady-state claims, define a convergence rule rather than discarding an arbitrary number of iterations.

### 9.3 Statistics

Use enough independent runs and requests to characterize the target percentile. Report median, percentiles, dispersion or confidence intervals, and outliers with cause where known. Paired tests using the same request trace reduce workload noise. Do not average rates with an arithmetic mean when intervals differ; aggregate total work over total time.

### 9.4 Correctness and quality

Performance alternatives must preserve an explicit contract:

- model output tolerance or task metric;
- tokenizer and prompt equivalence;
- quantization calibration and evaluation set;
- sampling seed/policy or distributional criterion;
- retrieval recall/precision;
- numerical overflow, NaN, and long-context behavior.

Approximate work skipped because quality changed is not a free microarchitectural speedup.

## 10. Energy, power, and cost

Energy per useful token is

$$
E_{token}=\frac{\int_{t_0}^{t_1}P(t)dt}{N_{good\ tokens}}.
$$

Include idle and memory power over the service interval. Race-to-idle may reduce energy if a faster kernel lets resources enter low-power states; turbo may increase power more than it reduces time. Batching often improves tokens/joule but can violate latency.

Total-cost analysis includes server amortization, memory capacity, network/storage, power delivery, cooling, software/operator cost, and stranded headroom. A CPU solution may win small or irregular workloads through reuse of general-purpose capacity even when an accelerator has higher dense peak; at high sustained dense load, specialized hardware may dominate. Compare at fixed SLO, quality, availability, and deployment scale.

## 11. Simulation and analytical co-design

Full LLM serving is usually too slow for naïve cycle-accurate simulation. Use a calibrated hierarchy:

1. Measure real operator shapes and service traces.
2. Use analytical models to identify active roofs and sensitive parameters.
3. Simulate representative regions or microkernels for a proposed cache, predictor, ISA, or memory change.
4. Feed simulated service times into a queueing/discrete-event serving model.
5. Validate the composition against a real baseline and report error by phase.

Do not apply a simulated 10% GEMM speedup to whole-request latency without Amdahl's law and phase interference:

$$
S_{total}=\frac{1}{(1-f)+f/S_{kernel}},
$$

where $f$ is the baseline fraction actually accelerated. If decode remains memory-bound, more matrix peak may not produce $S_{kernel}>1$ at all.

## 12. Research frontiers

### 12.1 Memory-centric CPU inference

**Question:** How should cache, prefetch, memory channels, compression, and near-memory logic evolve for weight- and KV-streaming decode?

Research needs models that include decompression energy, metadata, random KV blocks, multi-tenant interference, and capacity. Wider compute without proportional byte supply may help prefill but leave decode unchanged.

### 12.2 Adaptive heterogeneous execution

**Question:** Can a runtime place operators or phases on scalar CPU, vector/matrix CPU, GPU, NPU, or near-memory engines using live queue, topology, power, and SLO state?

The hard part is transfer and synchronization hysteresis. An optimal isolated operator placement may cause end-to-end oscillation or state movement. Control policies need stability analysis and counterfactual evaluation.

### 12.3 Sparse and MoE co-design

**Question:** Which predictor, cache, network, and scheduling support makes dynamic sparsity useful rather than metadata-bound?

Open issues include expert locality prediction, load imbalance, compact sparse formats, indirect prefetch, gather/scatter throughput, quality-aware routing, and NUMA/external-memory placement.

### 12.4 Quantization and numerical architecture

**Question:** What precision, group size, accumulator, and scaling contract minimizes bytes and energy under task-quality constraints?

Research must co-design training/calibration, ISA primitives, packed layout, exception/rounding behavior, and verification. A new format without compiler and kernel support is not an architecture result.

### 12.5 Virtual memory for model and KV state

**Question:** Can page sizes, translation caches, block allocators, prefix sharing, compression, and tiering be unified without excessive fragmentation or fault latency?

CPU virtual-memory mechanisms offer mature protection and sharing, but model runtimes add a second logical paging layer. Cross-layer hints could improve placement while risking policy coupling and isolation leaks.

### 12.6 QoS and secure multi-tenancy

**Question:** How can shared caches, predictors, memory bandwidth, SMT contexts, and matrix units provide predictable tail latency without wasting capacity?

Partitioning and throttling require feedback signals, timescales, and threat models. Performance isolation and side-channel resistance overlap but are not identical. Evaluate adversarial co-runners and mitigation overhead.

### 12.7 Serving-aware branch and data speculation

Tokenization, parsers, retrieval, allocators, and runtimes remain branch- and memory-latency-sensitive even when tensor kernels dominate operations. Research opportunities include confidence-aware prefetch, safe speculation across tenant boundaries, value/locality prediction for irregular graph search, and phase-aware predictor state. Improvements must count transient leakage and wasted-energy cost.

### 12.8 Reliability and reproducibility

Large memory footprints increase exposure to memory faults; long-running services face software and hardware aging, model reload, and partial failure. Research needs fault containment for weight/KV corruption, efficient checksums or error-correcting codes, deterministic recovery, and public artifacts that reproduce both performance and output quality.

## 13. Worked research plan

**Hypothesis:** CPU decode is limited by local weight bandwidth at small batch, and groupwise 4-bit packing improves goodput until unpack execution becomes the roof.

1. **Workload:** fixed model/checkpoint, real prompt/output length distribution, arrival-rate sweep, quality threshold, TTFT/TPOT SLO.
2. **Alternatives:** BF16, INT8, and 4-bit weight-only kernels with documented groups/layouts; same scheduler and tokenizer.
3. **Model:** predict bytes/token, unpack operations, ridge points, KV capacity, and queueing knee.
4. **Controls:** one socket then two; local then remote placement; batch sweep; warm steady state; fixed and production frequency policies.
5. **Evidence:** generated ISA, operator times, PMU, memory-controller bytes, remote-link bytes, page walks, frequency/power, request trace.
6. **Validation:** compare predicted and measured bytes/time; verify quality; explain residual and tails.
7. **Decision:** identify crossover batch/sequence ranges and whether added ISA/cache/decompression support would move the roof.

This plan can produce an architecture insight. Reporting one `tokens/s` comparison cannot.

## 14. Final audit checklist

- Are useful operations, bytes, and precision conventions explicit?
- Does every roof use sustainable bandwidth/compute for the actual access pattern?
- Are prefill, decode, attention, retrieval, and sampling separated?
- Are TTFT, TPOT, throughput, goodput, capacity, quality, and energy all defined?
- Are batch formation and queueing included rather than hidden outside kernel timing?
- Are CPU topology, page placement, thread affinity, and frequency recorded?
- Do PMU/uncore events directly support the proposed mechanism?
- Were counter overlap, multiplexing, and skid limitations considered?
- Do controlled perturbations move performance as the model predicts?
- Are cold start, steady state, failure, and adversarial interference distinct experiments?
- Can another researcher reproduce artifacts and derive the reported result?

## References

1. S. Williams, A. Waterman, and D. Patterson, [“Roofline: An Insightful Visual Performance Model for Multicore Architectures,”](https://escholarship.org/uc/item/8st6q26b) *Communications of the ACM*, 2009.
2. D. Patterson and J. Hennessy, *Computer Organization and Design* and *Computer Architecture: A Quantitative Approach*, quantitative-performance chapters.
3. Linux kernel, [`perf` userspace tool documentation](https://docs.kernel.org/userspace-api/perf_ring_buffer.html) and processor-vendor PMU event guides.
4. MLCommons, [MLPerf Inference policies and benchmark repository](https://github.com/mlcommons/inference), for scenario, accuracy, and measurement discipline.
5. W. Kwon et al., [“Efficient Memory Management for Large Language Model Serving with PagedAttention,”](https://doi.org/10.1145/3600006.3613165) SOSP 2023.
6. Y. Zhong et al., [“DistServe: Disaggregating Prefill and Decoding for Goodput-optimized Large Language Model Serving,”](https://www.usenix.org/conference/osdi24/presentation/zhong-yinmin) OSDI 2024.
7. Intel, [64 and IA-32 Architectures Optimization Reference Manual](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html), for microarchitecture-specific optimization and event interpretation.

---

← [AI Operators on CPU Microarchitecture](02_AI_Operators_on_CPU_Microarchitecture.md) · [AI Workloads and Serving index](00_Index.md) · [CPU Architecture](../00_Index.md)
