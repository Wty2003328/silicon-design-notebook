# End-to-End Artificial-Intelligence Inference and Serving on Neural Processing Units

> **First-time reader orientation:** Inference is the numerical evaluation of a trained model. **Serving** surrounds inference with storage, networking, request admission, batching, state management, scheduling, failure handling, and response delivery. An NPU can execute a kernel quickly while the service remains slow because it is waiting for a checkpoint, tokenizer, host queue, KV-cache page, collective, or another request. This chapter follows the complete path and names the ownership boundary at every step.

> **Abbreviation key — skim now and return as needed:** artificial intelligence (AI); neural processing unit (NPU); central processing unit (CPU); graphics processing unit (GPU); large language model (LLM); intermediate representation (IR); application programming interface (API); key–value (KV) cache; grouped-query attention (GQA); direct memory access (DMA); input-output memory management unit (IOMMU); Peripheral Component Interconnect Express (PCIe); remote procedure call (RPC); network interface card (NIC); high-bandwidth memory (HBM); dynamic random-access memory (DRAM); solid-state drive (SSD); time to first token (TTFT); time per output token (TPOT); inter-token latency (ITL); service-level objective (SLO); quality of service (QoS); tensor parallelism (TP); pipeline parallelism (PP); expert parallelism (EP); mixture of experts (MoE); first in, first out (FIFO); reliability, availability, and serviceability (RAS).

> **Prerequisites:** [AI Workload and Graph Mapping](01_AI_Workload_and_Graph_Mapping_to_NPUs.md), [Transformer and Attention Engine Microarchitecture](../01_Compute_Dataflows/03_Transformer_and_Attention_Engine_Microarchitecture.md), and [Host Interface, Memory Visibility, and Scheduling](../03_System_Integration/01_Host_Interface_Memory_Visibility_and_Scheduling.md).

---

## 0. Separate cold deployment, warm execution, and steady serving

An honest latency report identifies which regime it measures:

| Regime | Included work | Typical purpose |
|---|---|---|
| cold deployment | allocate hosts/devices, fetch model, compile, load weights, initialize runtime | failover, autoscaling, model rollout |
| warm first request | executable present, but caches, pages, clocks, and kernels may be cold | user-visible scale-from-zero behavior |
| steady state | model resident, warm code/data paths, stable arrival process | capacity and efficiency |

Cold-start time can be minutes while one warm device execution is milliseconds. Excluding cold start may be appropriate for a resident service, but it must be declared. For a frequently scaled or multi-model fleet, model load and compilation are part of availability and cost.

## 1. Deployment path: object storage to executable NPU state

~~~mermaid
flowchart LR
    A["object storage / checkpoint shards"] --> B["network + host page cache"]
    B --> C["host DRAM staging"]
    C --> D["deserialize / verify / transform"]
    D --> E["compile or select cached executable"]
    E --> F["allocate device HBM"]
    D --> G["DMA weight shards"]
    G --> F
    F --> H["initialize KV pools + workspaces"]
    H --> I["warm shape buckets / collectives"]
    I --> J["ready endpoint"]
~~~

### 1.1 Checkpoint discovery and integrity

A model package may contain graph/IR, many weight shards, tokenizer assets, quantization scales, sharding metadata, adapter weights, and a manifest. The control plane selects an immutable model version and verifies hashes, schema, precision, and compiler/runtime compatibility before advertising readiness.

For total checkpoint bytes $W$ and effective end-to-end load bandwidth $B_{load}$, a lower bound is

$$
T_{load}\ge\frac{W}{B_{load}},
$$

where $B_{load}$ is the minimum delivered rate across object storage, data-center network, host memory copies/decompression, PCIe or coherent attachment, and device HBM writes. Parallel readers help only until a shared boundary saturates.

### 1.2 Weight transformation and sharding

Stored layout rarely matches the final NPU layout. Deployment may transpose, tile, pack low-bit values, attach scales, sparsify, or partition weights by the device mesh. These transformations consume CPU memory bandwidth and temporary capacity. Pre-transformed artifacts reduce startup work but couple the checkpoint to an accelerator generation, compiler version, topology, and shape/precision policy.

For $D$ devices, a naïve loader that downloads the complete $W$ bytes to every host creates $DW$ network traffic even if each device keeps only $W/D$. Shard-aware storage and peer distribution can reduce source pressure, but peer broadcast adds topology and fault-recovery complexity.

### 1.3 Executable loading and warm-up

An NPU executable normally contains device commands/code plus metadata for constants, buffers, shape guards, and collectives. The runtime:

1. validates architecture and runtime versions;
2. reserves device and host-pinned memory;
3. loads executable code and constants;
4. creates command/completion queues and device contexts;
5. establishes collective groups and communication buffers;
6. runs representative shape buckets to populate caches and detect failures;
7. exposes the endpoint only after all required shards are ready.

Warm-up must cover the executable variants used by production. Warming only the maximum batch does not prove that a small decode bucket, a ragged attention path, or an MoE overflow path is resident and correct.

## 2. Request path from the network to the NPU

~~~mermaid
sequenceDiagram
    participant C as Client
    participant F as Frontend / router
    participant H as Host serving process
    participant R as NPU runtime
    participant N as NPU device(s)
    C->>F: RPC request
    F->>H: route model/version/tenant
    H->>H: parse, tokenize, validate
    H->>H: admission + KV reservation
    H->>R: enqueue prefill descriptor
    R->>N: DMA inputs + launch executable
    N-->>R: prefill completion + KV state
    loop each accepted output step
        H->>R: enqueue decode batch
        R->>N: launch decode + collectives
        N-->>R: logits / selected token
        R-->>H: completion
        H->>H: sample, stop check, schedule next
        H-->>C: streamed token
    end
    H->>H: release KV and accounting state
~~~

This sequence is a logical view. Implementations may sample on-device, use persistent device loops, or let the runtime enqueue future steps. The invariant is that request state, device state, and returned tokens remain associated with the correct model/version/tenant through batching and cancellation.

## 3. Frontend, tokenization, and admission

### 3.1 Network and RPC boundary

The frontend terminates transport, authenticates, enforces quotas, selects a model version, and routes to a replica. Request latency can include queueing at a load balancer and cross-rack transfer before the serving process observes the request. Payloads for text are small, but multimodal inputs, embeddings, or diffusion conditioning can be large enough for serialization and network copies to matter.

### 3.2 Preprocessing and tokenization

Text tokenization, image decode/resize, feature normalization, audio framing, or retrieval augmentation often runs on CPUs. These stages can be parallel pipelines, but their output order and model-version compatibility matter. Measure preprocessing separately and include it in end-to-end latency if the API accepts raw inputs.

### 3.3 Admission control is a capacity proof

Admission checks whether accepting a request can preserve memory and latency targets. For an autoregressive request it estimates:

- prompt tokens and allowed output tokens;
- KV-cache bytes over the possible lifetime;
- prefill work and deadline;
- available sequence slots/pages;
- tenant priority and cancellation policy;
- device/replica compatibility for adapters and quantization.

Accepting every request until memory is exhausted converts a predictable overload into allocation failure or tail-latency collapse. Rejection, backpressure, or queue limits are part of the architecture.

## 4. Scheduler state and batching

### 4.1 Static versus continuous batching

Static batching waits for a fixed group and runs it to completion. Continuous or iteration-level batching rebuilds the active set at step boundaries: finished sequences leave, new work enters, and remaining sequences continue. It improves occupancy under variable output lengths but requires indirect KV addressing and per-sequence metadata. Orca is a foundational primary example of iteration-level scheduling for generative serving [1].

At a scheduling instant, the engine chooses a batch $\mathcal B$ under constraints such as

$$
\sum_{i\in\mathcal B} KV_i\le C_{KV},\qquad
T_{pred}(\mathcal B)\le D_{earliest}-t_{now},
$$

where $KV_i$ is the predicted incremental/reserved KV bytes for request $i$, $C_{KV}$ is currently allocatable KV capacity after weight/workspace/margin reservations, $T_{pred}(\mathcal B)$ is predicted batch service duration, $D_{earliest}$ is the earliest absolute deadline in the candidate batch, and $t_{now}$ is the scheduling timestamp. The deadline inequality therefore compares duration with remaining slack. Add sequence-count, token-count, workspace, shape-bucket, and priority limits. Maximizing batch size alone can violate the earliest deadline or starve short requests.

### 4.2 Prefill/decode interference

Large prefills are compute-intensive and long; decode steps are shorter and latency-sensitive. If both share the same matrix array, HBM, or command queue, an uninterruptible prefill can create a decode bubble. Policies include:

- chunk prefill into bounded token blocks;
- reserve device/core/time capacity for decode;
- use separate queues with deadline-aware arbitration;
- place prefill and decode on different devices;
- preempt only at compiler-defined safe points.

Chunking introduces extra boundaries and may reduce attention reuse. The right chunk size balances decode tail latency against prefill efficiency.

### 4.3 Shape bucketing and executable dispatch

Compiled NPUs often use a finite set of batch/token/sequence buckets. The scheduler maps the active batch to the smallest legal executable, pads inactive lanes, and supplies masks/page metadata. A bucket can improve launch regularity while wasting operations. Record both logical batch tokens and executed padded tokens.

### 4.4 Host scheduling must stay ahead of the device

Let host preparation time per step be $T_h$ and device step time $T_d$. With one serial host thread and no lookahead, sustainable operation requires $T_h\le T_d$. Otherwise the NPU sees launch gaps even if its kernels are efficient. Pinned buffers, prebuilt descriptors, asynchronous completion, parallel tokenization, and device-resident control reduce this risk.

## 5. Prefill path in detail

Prefill transforms input tokens into initial hidden states and KV entries:

1. token IDs and position/attention metadata are staged;
2. embeddings are gathered;
3. each Transformer layer runs normalization, projections, attention, output projection, and feed-forward work;
4. K/V tensors are appended to request-owned cache pages;
5. final logits are produced for the first generation decision;
6. the sampler chooses the first output token and the request joins decode.

Prefill generally exposes large GEMMs and can be compute-bound. However, long-context attention, KV writes, embedding gathers, or multi-device collectives can become limiting. Time to first token decomposes as

$$
TTFT=T_{front}+T_{preprocess}+T_{queue}+T_{prefill}+T_{sample}+T_{stream}.
$$

For multi-device prefill, $T_{prefill}$ includes collectives and stage bubbles on the critical path. Reporting device kernel time as TTFT omits the service.

## 6. KV-cache allocation, addressing, and lifetime

For $L$ layers, $H_{KV}$ KV heads, head dimension $d_h$, cached tokens $S$, and element bytes $b$, uncompressed capacity per sequence is

$$
C_{KV}=2LH_{KV}d_hSb.
$$

The factor two is for keys and values. Add page metadata, alignment, replication, speculative branches, and allocator reserve.

### 6.1 Contiguous versus paged allocation

Contiguous allocation is simple and DMA-friendly but requires predicting maximum sequence length and suffers external fragmentation. Paged allocation reserves fixed-size blocks on demand and maps logical token blocks to physical pages. It reduces stranded capacity and enables sharing/branching, but adds:

- page tables or block lists per request;
- gather address generation;
- partially filled last-page waste;
- metadata traffic and cache pressure;
- reference counting and generation checks.

For page capacity $P$ tokens and request length $S$, internal fragmentation is at most $P-1$ token slots per independently allocated sequence, before prefix sharing.

### 6.2 Prefix reuse and copy-on-write

Requests sharing an identical prompt prefix may reference common KV pages. The service must key the cache by model, weights/adapters, tokenizer, positions, precision, and any state affecting hidden values. A branch that appends new tokens uses copy-on-write semantics so one request cannot modify another's prefix.

### 6.3 Eviction, swapping, and remote KV

When HBM is insufficient, the engine may reject, evict recomputable prefixes, or move KV to host/remote memory. Moving $B_{KV}$ bytes across delivered bandwidth $B_m$ adds at least $B_{KV}/B_m$ plus software and translation latency. A slow tier is useful only if transfer can be overlapped or is cheaper than recomputation under the request's SLO.

## 7. Decode loop in detail

For each active sequence:

1. consume the most recent token and position;
2. run embedding and per-layer projections;
3. read prior KV pages, compute attention, and append new K/V;
4. execute feed-forward/MoE work and collectives;
5. produce logits;
6. apply constraints, penalties, and sampling or greedy selection;
7. test stop conditions;
8. stream the token and either release or requeue the sequence.

Decode often reads a large fraction of weights for one token batch and all prior KV entries needed by attention. Weight traffic is amortized across the batch, while KV traffic grows with active context. This creates two different batching effects:

- larger batch improves weight reuse and array utilization;
- more/longer sequences increase KV bytes and scheduler state.

Time per output token can be written

$$
TPOT\approx T_{schedule}+T_{host/device}+T_{decode\ critical\ path}+T_{sample}+T_{stream}.
$$

For one request, end-to-end generation latency is approximately

$$
T_{request}=TTFT+(N_{out}-1)\,ITL_{avg}+T_{finalize},
$$

but percentiles must be computed from per-request traces rather than substituting averages into this expression.

## 8. Sampling and speculative decoding

Sampling may include temperature scaling, repetition penalties, masking, top-k/top-p selection, random-number generation, and token decoding. It can run on a CPU, vector engine, or dedicated unit. CPU sampling creates a device-to-host dependency every step unless logits processing is reduced or moved on-device.

Speculative decoding uses a cheaper draft model to propose $k$ tokens, then a target model verifies them in one pass while preserving the target distribution through a correction rule; Leviathan et al. provide a foundational exact formulation [2]. If accepted prefix length is random variable $A\in[0,k]$, useful tokens per target verification are approximately $1+E[A]$ (depending on the algorithm's correction step). The speedup is bounded by

$$
Speedup\lesssim\frac{(1+E[A])T_{target,1}}
{T_{draft,k}+T_{target,verify(k)}+T_{coord}}.
$$

It helps when acceptance is high and verification exploits NPU parallelism. It can hurt when draft/target traffic competes for HBM, KV branches consume capacity, or synchronization dominates. Quality equivalence and sampling semantics must be verified, not assumed.

## 9. Non-LLM serving paths

### 9.1 Encoder, vision, and ranking models

These often perform one forward pass per request. Dynamic batching collects requests until a size or deadline threshold, preprocessing feeds dense tensors, the NPU executes, and postprocessing returns classes, embeddings, or scores. Tail latency commonly depends on batch-wait policy and input preprocessing rather than KV state.

### 9.2 Recommendation

A request may fetch sparse features/embeddings from a distributed store, run dense NPU layers, and join results. The critical path spans networked lookup and NPU compute. Co-batching dense work without coordinating sparse arrival can leave the NPU waiting for stragglers.

### 9.3 Diffusion and iterative models

Diffusion serves a sequence of denoising steps. It reuses model weights but carries latent state between iterations. Scheduler policy can batch requests at the same step/shape; guidance or variable step counts create divergence. Latency is the dependency chain over all iterations, not one denoiser invocation.

## 10. Multi-NPU execution and collectives

### 10.1 Tensor parallelism

Partitioning weight columns or rows distributes GEMM but introduces all-gather, all-reduce, or reduce-scatter. A ring all-reduce of message $M$ across $N$ devices has a first-order time

$$
T_{ring}\approx2(N-1)L_{step}+\frac{2(N-1)}{N}\frac{M}{B_{eff}}.
$$

The $2(N-1)$ steps and $\frac{2(N-1)}{N}M$ transferred bytes come from a reduce-scatter followed by an all-gather, each phase using $N-1$ steps that move one $M/N$ chunk per step.

Small decode collectives can be latency-dominated. The launch, synchronization, routing, and protocol terms cannot be hidden by quoting link peak bandwidth.

### 10.2 Pipeline parallelism

Layer stages fit different devices. For $K$ stages and microbatch count $m$, an ideal balanced pipeline has a fill/drain fraction roughly $(K-1)/(m+K-1)$ ($K-1$ bubble steps to fill and drain, out of $m+K-1$ total pipeline steps). Serving has small and irregular microbatches, so bubbles and stage imbalance can be large. A failed or slow stage stalls the entire request.

### 10.3 Expert parallelism

MoE dispatch performs all-to-all token exchange, expert compute, and reverse exchange. Message sizes are routing-dependent; one hot expert can bottleneck its destination. Capacity padding can regularize shapes but wastes communication and compute.

### 10.4 Topology-aware placement

The logical device mesh should align high-volume collectives with high-bandwidth, low-contention links. Crossing a rack or scale-out boundary can change bandwidth and latency by an order of magnitude relative to on-package/board links. The placement record must include physical endpoints, routes, competing tenants, and collective algorithm.

## 11. Disaggregated prefill and decode

Prefill and decode favor different resources, so a service may place them on different NPU pools. A prefill worker builds KV state; a decode worker receives or remotely accesses that state and generates tokens. DistServe is a primary system study of this phase disaggregation and its goodput trade-off [3].

The handoff transfers approximately the produced KV bytes plus metadata:

$$
T_{handoff}\ge L_{setup}+\frac{B_{KV}}{B_{net,eff}}+T_{register/map}+T_{queue}.
$$

Disaggregation is useful when independent scaling and reduced phase interference outweigh handoff cost. Research questions include:

- direct NPU-to-NPU transfer versus staging through host DRAM;
- eager streaming of layer/chunk KV versus transfer after full prefill;
- overlap between prefill tail and decode setup;
- routing a request to a decode worker with capacity and topology locality;
- failure recovery when the only KV copy is in transit;
- encryption, isolation, and version compatibility across pools.

Disaggregation shifts the bottleneck rather than removing it. TTFT includes handoff, while decode throughput may improve through better batching and resource specialization.

## 12. Runtime queues and host–device interaction

The serving process writes descriptors to a submission ring, updates a doorbell, and later consumes completion records. The runtime maps request buffers through an IOMMU or device memory context, pins/translates pages, and programs DMA. Correctness requires:

- descriptor writes visible before the doorbell;
- device reads complete before input reuse;
- result/KV writes visible before completion;
- completions tagged with context, request, generation, and status;
- canceled/reset contexts unable to corrupt reassigned memory;
- queue credits preventing overflow;
- watchdog and fault paths that release or quarantine resources safely.

Submission batching reduces doorbells and interrupts but increases waiting. Polling reduces completion latency at CPU-power cost; interrupts save CPU but add wake-up latency. These trade-offs matter for small decode steps.

## 13. Queueing, overload, and tail latency

Let arrival rate be $\lambda$ requests/s and sustainable service rate be $\mu$. Utilization is $\rho=\lambda/\mu$. As $\rho$ approaches one, queueing latency and sensitivity to service-time variance rise sharply. Real services are not simple single-server queues with memoryless arrivals and service: batching couples requests, lengths are heavy-tailed, and multi-device jobs require synchronized resources. Still, the utilization warning remains fundamental.

Little's law relates average in-flight requests $L$, arrival rate, and average time in system $W$:

$$
L=\lambda W.
$$

Use it as a conservation check, not as a tail model. Tail-SLO analysis should replay or generate the real arrival process, input/output length distribution, priorities, cancellations, and failure events. Report percentile TTFT and TPOT separately; their causes differ.

Policies for overload include admission rejection, deadline-aware scheduling, bounded queues, replica autoscaling, quality/length limits, and load shedding. An architecture result at an unconstrained offline batch does not establish online SLO capacity.

## 14. Cancellation, failures, and rolling updates

Cancellation can arrive while work is queued, executing, communicating, or streaming. The service marks the request dead, prevents future scheduling, safely ignores or aborts outstanding commands, and releases KV pages only after no device/communication operation can reference them.

Failure domains include process, host, NPU, link, rack, model shard, and control plane. Multi-device inference may require all shards; one failure can invalidate the whole replica. Recovery choices are:

- retry from prompt on another replica;
- restore/reconstruct KV state;
- keep redundant KV/checkpoint state;
- return an error within a bounded time.

Rolling updates keep old and new model versions isolated while draining requests. Prefix/KV caches cannot be silently shared across incompatible weights, quantization, tokenizer, or executable layouts.

## 15. End-to-end latency and throughput ledger

For a unary model:

$$
T_{e2e}=T_{network,in}+T_{front}+T_{pre}+T_{queue}+T_{host/device}+T_{NPU}+T_{post}+T_{network,out}.
$$

For an autoregressive model, preserve a timestamped ledger:

| Timestamp/event | Reveals |
|---|---|
| request received | external arrival process |
| preprocessing complete | CPU/input cost |
| admitted/KV reserved | admission and allocation cost |
| prefill scheduled/submitted/start/end | queue, host gap, device prefill |
| first token sampled/sent | sampling and network contribution to TTFT |
| each decode scheduled/start/end/sent | batching, device, and streaming ITL |
| request complete/KV freed | finalization and lifetime |

Throughput must state the unit: requests/s, input tokens/s, output tokens/s, total tokens/s, samples/s, or useful operations/s. “Tokens/s” without input/output distinction can hide a workload mix change.

## 16. Worked serving analysis

Assume a model has 32 layers, 8 KV heads, head dimension 128, 2-byte KV values, and an active request at 16,384 cached tokens. Its KV capacity is

$$
2\times32\times8\times128\times2\times16384=2\ \text{GiB}.
$$

Forty such sequences need about 80 GiB before allocator reserve and metadata. On a 96-GiB device, admitting 40 because the model weights “fit” leaves little workspace or fragmentation margin. If each decode step reads the full resident KV once, the theoretical KV traffic alone is enormous; actual attention sharding, head sharing, precision, and caching determine delivered bytes. This is why admission, GQA, paging, and context distribution are architecture inputs—not serving afterthoughts.

Now suppose prefill produces 2 GiB ($2\times2^{30}$ bytes) of KV and a disaggregated link delivers 50 decimal GB/s ($50\times10^9$ bytes/s). Transfer alone is at least $2\times2^{30}/(50\times10^9)\approx42.9$ ms, before setup and queueing. If local TTFT is 80 ms, disaggregation adds a material fraction unless transfer overlaps prefill or enables enough queue/resource benefit to compensate.

## 17. Serving audit checklist

- Does the measured boundary include raw-input preprocessing and network transport?
- Are cold load, warm first request, and steady state separated?
- Are weight/checkpoint transformations and executable variants versioned?
- Does admission reserve worst-case or policy-bounded KV/workspace capacity?
- Are prefill and decode queues, interference, and chunking visible?
- Are logical and padded tokens both counted?
- Is host scheduling fast enough to keep the NPU fed?
- Are KV layout, pages, metadata, sharing, eviction, and cancellation modeled?
- Are sampling and stop logic included in per-token dependencies?
- Are collective algorithms, topology, and delivered bandwidth recorded?
- Does disaggregation include KV transfer, setup, routing, and failure ownership?
- Are TTFT, TPOT/ITL, total latency, throughput, and percentiles computed per request?
- Is the arrival/length distribution representative of the claimed SLO?

## 18. Evidence chain, failure boundaries, and open questions

| Workload phase | Mechanism | Theory/bound | Observable evidence | Validation and failure boundary |
|---|---|---|---|---|
| cold model deployment | sharded read, transform, compile/cache, HBM DMA, warm-up | slowest-boundary load bandwidth and temporary capacity | per-shard storage/network/host/DMA timestamps, bytes, executable-cache hit, HBM occupancy | cold-start repetitions with empty/warm caches; fails under source throttling, shard skew, extra transforms, or version mismatch |
| prefill | large GEMMs, fused attention, KV append, collectives | compute/attention/communication critical path | queue/submission/start/end events, array/vector counters, HBM/KV bytes, collective trace | prompt-length sweep at fixed batch; fails when chunking, long attention, KV writes, or collectives change the bottleneck |
| decode | continuous batch, weight reuse, paged KV gather, sampling | weight/KV rooflines plus host/device dependency | logical/padded batch, weight/KV bytes, page metadata, host gaps, per-token timestamps | context/batch sweep with identical output policy; fails when KV, vector/sampling, padding, or queue delay dominates |
| multi-NPU inference | tensor/pipeline/expert sharding and collectives | latency/bandwidth collective model plus stage imbalance | message bytes/steps/routes, link occupancy, synchronization, stage timeline | vary device count/topology and isolate links; fails when small-message latency, hot experts, bubbles, or topology contention dominates |
| prefill/decode disaggregation | KV ownership transfer between pools | $B_{KV}/B_{net,eff}$ plus setup/queue and overlap | per-page readiness, transfer bytes/rate, source/destination queue, TTFT/TPOT | compare local and disaggregated service at fixed load/SLO; fails when KV transfer and failure recovery exceed specialization benefit |
| online SLO | admission, deadline-aware batching, bounded queues | queueing conservation and empirical per-request percentiles | arrival/admission/schedule/token timestamps, rejection/cancellation, queue depth | replay representative burst/length distribution; fails near saturation or when hidden load shedding changes accepted work |

Open research questions include hardware-visible preemption points for chunked prefill, safe device-resident scheduling, KV compression with bounded quality/error, remote-KV consistency and security, joint network/device scheduling for disaggregation, speculative-decoding state that avoids KV copying, and tail-aware compilation that selects kernels for deadlines rather than mean device time.

## References

1. G.-I. Yu et al., “Orca: A Distributed Serving System for Transformer-Based Generative Models,” OSDI 2022 — [USENIX](https://www.usenix.org/conference/osdi22/presentation/yu).
2. Y. Leviathan, M. Kalman, and Y. Matias, “Fast Inference from Transformers via Speculative Decoding,” ICML 2023 — [PMLR](https://proceedings.mlr.press/v202/leviathan23a.html).
3. Y. Zhong et al., “DistServe: Disaggregating Prefill and Decoding for Goodput-optimized Large Language Model Serving,” OSDI 2024 — [USENIX](https://www.usenix.org/conference/osdi24/presentation/zhong-yinmin).
4. W. Kwon et al., “Efficient Memory Management for Large Language Model Serving with PagedAttention,” SOSP 2023 — [ACM proceedings](https://doi.org/10.1145/3600006.3613165).
5. Z. Li et al., “AlpaServe: Statistical Multiplexing with Model Parallelism for Deep Learning Serving,” OSDI 2023 — [USENIX](https://www.usenix.org/conference/osdi23/presentation/li-zhouhan).
6. P. Barham et al., “Pathways: Asynchronous Distributed Dataflow for ML,” MLSys 2022 — [Google Research](https://research.google/pubs/pathways-asynchronous-distributed-dataflow-for-ml/).
7. MLCommons, [MLPerf Inference documentation](https://docs.mlcommons.org/inference/) — load generation, accuracy, scenarios, and result computation.
8. OpenXLA, [PJRT Uniform Device API](https://openxla.org/xla/pjrt) — framework/runtime device boundary.
9. Google Cloud, [Building production AI on Cloud TPUs with JAX](https://docs.cloud.google.com/tpu/docs/jax-ai-stack) — primary example of native serialized models, TPU runtime serving, batching, paged attention, and profiling.

## Cross-references

- [Host Interface, Memory Visibility, and Scheduling](../03_System_Integration/01_Host_Interface_Memory_Visibility_and_Scheduling.md) expands queue, IOMMU, context, preemption, and fault mechanics.
- [Decoupled Access–Execute](../02_Mapping_and_Memory/03_Decoupled_Access_Execute_and_Scratchpad_Scheduling.md) explains descriptor and event-driven overlap inside the NPU.
- [Transformer and Attention Engines](../01_Compute_Dataflows/03_Transformer_and_Attention_Engine_Microarchitecture.md) owns prefill/decode datapaths and KV-cache microarchitecture.
- [Performance, Compiler, Profiling, and Research Methodology](03_Performance_Compiler_Profiling_and_Research_Methodology.md) turns this path into measurable, falsifiable evidence.

---

← [AI Workload and Graph Mapping](01_AI_Workload_and_Graph_Mapping_to_NPUs.md) · [AI Workloads and Serving index](00_Index.md) · next → [Performance, Compiler, Profiling, and Research Methodology](03_Performance_Compiler_Profiling_and_Research_Methodology.md)
