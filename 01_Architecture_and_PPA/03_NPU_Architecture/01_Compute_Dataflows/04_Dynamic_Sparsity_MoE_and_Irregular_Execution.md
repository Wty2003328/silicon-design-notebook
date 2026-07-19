# Dynamic Sparsity, Mixture-of-Experts, and Irregular NPU Execution

```mermaid
flowchart LR
    IN["dense activations / tokens"] --> DET["zero / block / router detection"]
    DET --> META["indices / masks / expert assignments"]
    META --> PACK["compact + bucket + load-balance"]
    PACK --> MOVE["gather / all-to-all / scratchpad placement"]
    MOVE --> EXEC["sparse tensor / vector / expert execution"]
    EXEC --> MERGE["scatter / reduce / restore order"]
    MERGE --> OUT["dense logical output"]
    DET -. "dense fallback if overhead wins" .-> EXEC
```

> **First-time reader orientation:** Dense tensor hardware assumes every processing element receives useful work at a predictable time. Sparse models and mixture-of-experts (MoE) models break that assumption: useful values appear at irregular positions, and different tokens choose different expert networks. The advanced microarchitecture is the machinery that finds, routes, balances, and retires this data-dependent work.

> **Abbreviation key — skim now and return as needed:** neural processing unit (NPU); deep neural network (DNN); large language model (LLM); mixture of experts (MoE); processing element (PE); general matrix multiplication (GEMM); general matrix-vector multiplication (GEMV); sparse matrix–dense matrix multiplication (SpMM); sampled dense–dense matrix multiplication (SDDMM); compressed sparse row (CSR); compressed sparse column (CSC); multiply-accumulate (MAC); network on chip (NoC); high-bandwidth memory (HBM); static random-access memory (SRAM); first in, first out (FIFO); operations per byte (Op/B).

> **Prerequisites:** [Sparsity, Quantization, and Compression](../02_Mapping_and_Memory/02_Sparsity_Quantization_and_Compression.md) for representation and [Systolic and Spatial Dataflows](02_Systolic_Spatial_and_Vector_Dataflows.md) for the dense baseline.
> **Hands off to:** [Transformer and Attention Engines](03_Transformer_and_Attention_Engine_Microarchitecture.md) for attention, [Decoupled Access–Execute](../02_Mapping_and_Memory/03_Decoupled_Access_Execute_and_Scratchpad_Scheduling.md) for queues and scratchpads, and [NoC Routing](../../04_SoC_and_Chiplet_Architecture/04_On_Chip_Networks/02_Routing_Flow_Control_and_Deadlock.md) for transport correctness.

---

## 0. Skipping zeros is not the hard part

If half the operands are zero, the theoretical arithmetic saving is 2×. Real hardware realizes less because it must:

- read and decode position metadata;
- match nonzero coordinates;
- route operands to available PEs;
- compact useful outputs;
- balance variable work across lanes;
- preserve deterministic accumulation and numerical behavior;
- fall back efficiently when a tile is dense.

The central distinction is:

> **Sparsity removes arithmetic but adds control and communication.**

A sparse accelerator wins only when removed data movement and MAC work exceed the metadata, matching, routing, and imbalance cost.

### 0.1 Derive the MoE machinery from eight routed tokens

A dense feed-forward layer sends every token through one weight pair. A mixture-of-experts (MoE) layer first computes router scores, then runs only selected expert feed-forward networks. The apparent saving—one expert instead of all experts—creates a new irregular scheduling problem.

Take eight tokens with top-1 expert choices among four experts:

$$
[e_0,e_3,e_0,e_1,e_3,e_3,e_2,e_3].
$$

A naive implementation sends tokens individually as soon as the router produces them. It performs eight small, poorly utilized matrix–vector launches and repeatedly fetches expert weights. Alternatively, four fixed engines can wait for their assigned tokens, but their work is `[2,1,1,4]`; the layer completes when expert 3 finishes four tokens. Useful assignment efficiency relative to four engines is

$$
\eta=\frac{8}{4\cdot4}=50\%.
$$

The first feature is **grouping**: count tokens per expert, prefix-sum those counts to compute compact-buffer offsets, scatter token records into expert-contiguous batches, execute one batch per expert, then inverse-scatter results to original token order.

| Stage | Concrete state for the eight-token trace | Result |
|---|---|---|
| route | records `(token_id, expert_id, gate_weight)` | token identity survives reordering |
| histogram | counts `[2,1,1,4]` | exact queue/buffer demand is known |
| prefix sum | base offsets `[0,2,3,4]`, end 8 | each expert receives a disjoint range |
| compact scatter | buffer order `[t0,t2 | t3 | t6 | t1,t4,t5,t7]` | contiguous expert batches enable GEMM |
| expert execution | batch shapes 2,1,1,4 | weight tile is reused within each batch |
| inverse scatter | use stored token IDs and gate weights | logical outputs return to `t0…t7` order |

```mermaid
flowchart LR
    T["tokens t0…t7"] --> R["router + top-1<br/>0,3,0,1,3,3,2,3"]
    R --> H["histogram<br/>2,1,1,4"]
    H --> P["prefix offsets<br/>0,2,3,4,8"]
    R --> S["scatter records by offset"]
    P --> S
    S --> Q0["expert 0: t0,t2"]
    S --> Q1["expert 1: t3"]
    S --> Q2["expert 2: t6"]
    S --> Q3["expert 3: t1,t4,t5,t7"]
    Q0 --> G["expert GEMMs / vector epilogues"]
    Q1 --> G
    Q2 --> G
    Q3 --> G
    G --> I["inverse scatter by token ID"]
```

This feature is enabled by more than a top-k comparator. Hardware or software-visible accelerators need histogram counters, a prefix-sum engine or scan sequence, per-expert write cursors, compact-token SRAM, token IDs, gate weights, expert queue descriptors, credits, an inverse map, and a barrier that knows how many assignments must return. With top-2 routing, one token creates two records and its final output cannot retire until both weighted expert results have arrived.

### 0.2 Capacity overflow forces an explicit semantic choice

The average load is $Tk/E=8/4=2$. With capacity factor $c=1.5$, each expert reserves

$$C_e=\lceil1.5\cdot2\rceil=3$$

slots. Expert 3 receives four assignments, so one does not fit. “Capacity factor” becomes hardware behavior at this exact point. The implementation must choose one of four model-visible policies:

- **stall/backpressure:** preserve the route and wait for expert-3 space; increases tail latency and can propagate to the router;
- **spill/second wave:** place the fourth record in an overflow queue and run expert 3 again; preserves semantics but adds launch and weight-residency time;
- **reroute:** use a recorded next-choice expert and its gate rule; changes the selected expert but may be part of the trained model contract;
- **drop:** contribute zero or a residual-only path; cheapest, but explicitly changes model output.

Replay this example with the spill policy. The compact buffer stores the first three expert-3 records in its reserved range and places `t7` in a framed overflow queue. Experts 0–3 run their first batches. Expert-3 weights remain resident, so a second one-token microbatch processes `t7`; its result joins the same inverse-scatter table. The completion barrier expects eight returned records, not merely four expert-done events. If the overflow record is lost, all engines can become idle while the barrier waits forever; if the barrier expects only seven, the layer can retire with a missing token.

### 0.3 Follow imbalance to scheduling and transport features

| Observed failure | Feature introduced | Enabling descriptors/state/dataflow | PPA/complexity and losing case |
|---|---|---|---|
| per-token launches underfill matrix engine | expert grouping and compact batches | token record, histogram, prefix offsets, compact/inverse buffers | scan/scatter latency and SRAM; loses for tiny batches or low-latency single-token service |
| hot expert sets layer time | split batch or replicate hot expert | sub-batch descriptors, replica/version table, merge count | duplicate weights and traffic; cold experts strand replica capacity |
| expert weights exceed local memory | expert cache and prefetch | object-size tags, residency/refcount, router-informed prefetch, miss queue | megabyte fills, fragmentation, wrong prediction wastes bandwidth |
| remote experts cause all-to-all bursts | hierarchical placement and credits | destination chip/core, token packet, virtual channel, receive capacity | NoC/link buffers and synchronization; local imbalance can be cheaper than remote traffic |
| arbitrary sparse products unbalance PEs | overdecomposed tasks + work queues | coordinate/output-owner tags, queue/crossbar, reduction arrival count | routing/queue energy and changed accumulation order |
| dense tiles pay sparse decode overhead | density-aware dense fallback | density/format field, conversion descriptor, common output layout/scale | estimator/conversion overhead; wrong threshold oscillates modes |

Replication illustrates the trade-off. Splitting expert 3 across two engines reduces its four-token critical batch to two tokens per replica, but only if both replicas possess the same weight version. Loading that expert twice can cost more than the saved compute for one small batch. Admission should compare weight-transfer time with expected future assignments, and counters must attribute whether a “replica hit” actually reused resident weights.

### 0.4 Replay malformed routing and a remote fault

If a router emits expert ID 5 when only IDs 0–3 exist, bounds validation must fault before indexing histogram or destination tables. Otherwise one malformed score can corrupt queue metadata or route a packet outside its security allocation. The terminal record identifies request, layer, token, expert ID, and routing epoch; no success barrier fires for that layer.

If expert 3 is remote and one token packet is retried after a link error, the receiver deduplicates by `(request, layer, token_id, expert_id, routing_epoch)`. A credit reserves queue capacity before transmission. The sender retains the record until an acknowledgement establishes ownership transfer; reset increments the epoch and quarantines late packets. Replaying without identity can execute a token twice, while dropping the original before acknowledgement can lose it. Expert computation may proceed for already received tokens, but final inverse scatter waits for exactly the assignment count declared by the router, or terminates with the documented fault policy.

### 0.5 Trace-derived counters and assertions

Record router/top-k cycles; assignment count; expert histogram and its max/mean/variance; histogram/scan/scatter time; compact and overflow bytes; microbatch-size distribution; matrix utilization per expert; expert-weight residency hits, fills, and wasted prefetches; local/remote token bytes; credit and queue stalls; replicas used; spill/reroute/drop count; inverse-scatter wait; and layer critical expert. A single average expert load hides the tail that determines completion.

Assertions should guarantee that histogram sum equals the number of assignment records; prefix ranges are disjoint, monotonic, and within capacity; every accepted assignment occupies exactly one normal, overflow, reroute, or dropped-policy state; token/expert/epoch identity survives compaction and transport; top-k results produce exactly the declared number of returns; no expert queue accepts without credit; duplicate remote packets cannot duplicate computation or accumulation; inverse scatter writes the correct token and gate contribution; and the layer emits success only after every required assignment reaches one terminal state.

## 1. Structured versus unstructured sparsity

**Structured sparsity** removes fixed groups—blocks, channels, or a pattern such as two nonzeros in every four weights. Hardware can decode the pattern with small masks and route remaining values through regular datapaths.

**Unstructured sparsity** permits arbitrary zero positions. It offers more pruning freedom but requires coordinates or run lengths and creates irregular matches.

For density $d$, value width $b_v$, and metadata cost $b_m$ per retained value, compressed bytes relative to dense bytes are approximately

$$
R_{bytes}=d\frac{b_v+b_m}{b_v}.
$$

Compression saves bandwidth only when $R_{bytes}<1$, or

$$
d<\frac{b_v}{b_v+b_m}.
$$

With 8-bit values and 4-bit average metadata, density must be below $8/12=66.7\%$ before capacity falls at all. Decode and routing energy demand an even lower break-even density.

## 2. Sparse storage and decode front end

CSR and CSC store nonzero values, their column or row indices, and pointers marking each row or column. Block-sparse formats store coordinates per tile rather than per value. Bitmap formats spend one bit per candidate position and are attractive at moderate density.

A sparse decode front end contains:

- metadata and value fetch queues;
- pointer walkers or bitmap scanners;
- prefix-sum or population-count logic to locate packed values;
- bounds and malformed-stream checks;
- small FIFOs decoupling variable-rate decode from compute;
- tile-density detection for sparse-versus-dense dispatch.

Metadata and values must remain synchronized across retries. A dropped metadata word can misassociate every later value, so designs use packet lengths, tile identifiers, checksums, and end markers rather than an unframed stream.

## 3. Matching sparse operands

Sparse–dense multiplication only needs coordinates from the sparse operand. Sparse–sparse multiplication must find matching reduction coordinates from both operands.

Common matching microarchitectures include:

- merge two sorted index streams;
- intersect bitmaps;
- hash or associative lookup;
- distribute nonzeros by coordinate and merge partial products;
- convert one operand to a dense local tile when density is high.

The matcher produces a variable number of products per cycle. Elastic FIFOs isolate it from the multiplier array. If average matcher output is $\lambda_m$ products/cycle and compute accepts $\mu_c$, stable execution requires $\lambda_m<\mu_c$; burst capacity handles local variation but cannot fix an average mismatch.

## 4. Flexible distribution and reduction

A dense systolic array assumes each operand advances to a predictable neighbor. Irregular sparsity needs a more flexible interconnect:

- multicast one activation to PEs holding matching weights;
- unicast uncommon values;
- route partial sums to the owner of an output coordinate;
- merge multiple products targeting the same output;
- bypass idle PEs or steal work.

Eyeriss v2 uses a hierarchical mesh that can change between unicast, multicast, and broadcast behavior. SIGMA uses flexible distribution and reduction networks to support irregular GEMM. The recurring trade-off is regular-wire efficiency versus mapping flexibility.

A full crossbar can route anything but scales poorly in area and energy. Multi-stage networks or hierarchical clusters reduce cost, at the price of blocking and path-dependent latency. Queue depth and credit flow control become part of the PE utilization equation.

## 5. Load balance: useful work, not stored elements

Equal numbers of nonzeros do not guarantee equal work. In a matrix product, a nonzero that matches many values produces more MACs than one that matches none. An accurate work estimator counts expected matches or output updates.

Scheduling options include:

- static partition by row, column, head, or block;
- split heavy rows into several tasks;
- global or hierarchical work queues;
- work stealing among PE clusters;
- overdecomposition into many small tiles;
- dynamic dense fallback for high-density tiles.

If PE $i$ receives work $w_i$, ideal balanced time is $\sum_i w_i/P$. Actual time is at least $\max_i w_i$, so load-balance efficiency is

$$
\eta_{bal}=\frac{\sum_i w_i}{P\max_i w_i}.
$$

Arithmetic sparsity of 80% can still underperform dense hardware if one PE receives most surviving work.

## 6. Mixture-of-experts as dynamic sparsity

An MoE layer contains many expert subnetworks but routes each token to only a small top-$k$ subset. It is sparse activation at expert granularity.

The pipeline is:

1. a router computes expert scores for each token;
2. top-$k$ selection chooses experts;
3. tokens are counted and grouped by expert;
4. a prefix sum assigns compacted buffer offsets;
5. tokens travel to local or remote expert engines;
6. each expert executes a batched FFN;
7. outputs return and are weighted/reordered into token order.

This adds hardware needs absent from a dense FFN:

- top-k reduction;
- histogram and prefix-sum engines;
- token compaction/scatter and inverse gather;
- per-expert queues and capacity limits;
- all-to-all transport across cores or chips;
- dynamic batching and expert cache policy;
- overflow and dropped-token behavior defined by the model.

## 7. Expert imbalance and capacity

If $T$ tokens each choose $k$ experts among $E$, average assignments per expert are $Tk/E$. Real routers are not uniform. A hot expert can determine layer time while other engines idle.

Systems use a **capacity factor** $c>1$, reserving roughly

$$
C_e=\left\lceil c\frac{Tk}{E}\right\rceil
$$

token slots per expert. Larger $c$ reduces overflow but increases buffer capacity and worst-case communication. Hardware must implement the chosen overflow policy: drop, reroute, spill, or stall. Silently overwriting a full expert queue changes the neural network.

Dynamic schedulers can split hot experts across replicated engines, merge cold experts onto one engine, or delay a batch until enough same-expert tokens form an efficient GEMM. These improve utilization but add token latency and ordering state.

## 8. Expert weights and memory hierarchy

MoE reduces active compute per token but increases total parameter capacity. All expert weights may not fit in local HBM or SRAM. The hierarchy may:

- keep popular experts resident;
- prefetch predicted experts after router scores are partially known;
- replicate hot experts across chips;
- shard large experts;
- stream cold experts from slower memory;
- use near-memory compute for low-intensity GEMV-like experts.

Expert caching resembles a cache but with much larger objects and software-visible scheduling. Replacement cost is not a cache-line miss; it can be megabytes of weights. Admission should consider expected future token count, transfer time, and residency opportunity.

If expert weights take $T_w$ to load and then serve $n$ tokens at per-token saved time $\Delta T$, prefetch/admission breaks even only when

$$
n\Delta T>T_w.
$$

## 9. Sparse attention

Sparse attention removes score connections between token pairs. Hardware typically performs:

- SDDMM to compute selected $QK^T$ entries;
- masking or pruning;
- sparse softmax reductions;
- SpMM to multiply sparse probabilities by V.

The sparse pattern may be static (window/block), learned but fixed per layer, or dynamic per input. Static block sparsity maps most efficiently because tiles align with memory bursts and matrix units. Fine-grained dynamic sparsity saves more arithmetic but requires runtime index generation and irregular reduction.

SpAtten prunes tokens and heads; Sanger co-designs a regular sparse pattern and reconfigurable datapath. The general architecture lesson is to make the sparsity granularity match the transport and compute granularity. A theoretically sparse pattern that produces one useful value per cache line is a bandwidth loss.

## 10. Dense fallback and mode selection

Sparse hardware should not force every tile through metadata decode. A density estimator or compiler hint can choose:

- dense systolic path;
- structured-sparse tensor path;
- unstructured sparse path;
- vector/scalar path for tiny fragments.

Let dense time be $T_d$ and sparse time

$$
T_s=T_{meta}+dT_{compute}+T_{imbalance}+T_{route}.
$$

Choose sparse only when $T_s<T_d$. The threshold varies with tile shape, value width, metadata cache hit rate, and NoC pressure, so a fixed global density threshold is rarely optimal.

## 11. Correctness, determinism, and numerical order

Dynamic work distribution can change accumulation order. Floating-point addition is not associative, so two legal schedules may produce slightly different answers. Architecture must specify whether results are deterministic, reproducible within a tolerance, or unconstrained within the numeric format.

Verify:

1. metadata decodes to exactly the intended coordinates;
2. every surviving product contributes once to the correct output;
3. zero-skipped work truly has a zero contribution under quantization and scaling rules;
4. expert compaction and inverse gather preserve token identity;
5. overflow follows the model's defined policy;
6. credits prevent network and expert-queue overflow;
7. partial sums cannot deadlock while waiting for each other;
8. sparse/dense mode transitions use identical numerical scales and layouts;
9. malformed descriptors cannot read outside the assigned memory context.

Useful counters include decoded density, metadata bytes, matcher utilization, PE imbalance, dense-fallback rate, expert histogram, overflow count, token all-to-all bytes, hot-expert replication hits, and sparse queue backpressure.

## 12. Worked examples

**1 — Metadata break-even.** INT8 values use 8 bits and each retained value averages 6 metadata bits. At density $d=40\%$, compressed size is $0.4(8+6)/8=0.70$ of dense—30% fewer bytes. At $d=70\%$, it is $1.225$ of dense, so compression increases capacity even before decode energy.

**2 — Balance loss.** Eight PE clusters receive work `[100, 100, 100, 100, 100, 100, 100, 900]`. Total is 1600, ideal time 200, actual at least 900. $\eta_{bal}=1600/(8\times900)=22.2\%$. Even though all 1600 operations are useful, imbalance wastes nearly four-fifths of peak capacity.

**3 — MoE capacity.** 4096 tokens choose top-2 among 64 experts. Average is 128 assignments/expert. With capacity factor 1.25, each expert buffer reserves 160 token slots. If one expert receives 230 assignments, 70 must be rerouted, spilled, dropped, or stall the layer—the hardware cannot treat capacity factor as merely a software statistic.

## Numbers to remember

| Quantity | Typical scale | Why it matters |
|---|---:|---|
| sparse metadata | bits to several bytes per nonzero | sets bandwidth break-even density |
| structured pattern | small fixed groups or blocks | maps efficiently to regular tensor hardware |
| MoE top-k | commonly a small subset of experts | sparse compute but dynamic routing |
| capacity factor | above 1 | trades overflow risk for buffer/communication cost |
| balance efficiency | $\sum w_i/(P\max w_i)$ | useful-work utilization after skew |
| sparse win condition | saved compute + bytes > metadata + routing + imbalance | prevents “zero count” from being mistaken for speedup |

## Cross-references

- [Sparsity, Quantization, and Compression](../02_Mapping_and_Memory/02_Sparsity_Quantization_and_Compression.md) explains formats before this execution machinery.
- [Transformer and Attention Engines](03_Transformer_and_Attention_Engine_Microarchitecture.md) supplies the dense/fused attention baseline.
- [Network-on-Chip](../../04_SoC_and_Chiplet_Architecture/04_On_Chip_Networks/01_Network_on_Chip.md) covers the routing fabric used by dynamic tokens and operands.
- [GPU Operand Delivery](../../02_GPU_Architecture/01_Core_Architecture/03_Operand_Collectors_Register_Files_and_Scoreboards.md) provides a throughput-processor comparison for elastic operand collection.

## References

1. Y.-H. Chen et al., “Eyeriss v2: A Flexible Accelerator for Emerging Deep Neural Networks,” JETCAS 2019 — [paper](https://eems.mit.edu/wp-content/uploads/2019/04/2019_jetcas_eyerissv2.pdf).
2. E. Qin et al., “SIGMA: A Sparse and Irregular GEMM Accelerator with Flexible Interconnects for DNN Training,” HPCA 2020 — [DOI](https://doi.org/10.1109/HPCA47549.2020.00015).
3. H. Wang, Z. Zhang, and S. Han, “SpAtten,” HPCA 2021 — [paper](https://arxiv.org/abs/2012.09852).
4. L. Lu et al., “Sanger: A Co-Design Framework for Enabling Sparse Attention using Reconfigurable Architecture,” MICRO 2021 — [paper](https://liqianglu-zju.github.io/files/conference/2021/MICRO_2021_Sanger.pdf).
5. S. Rajbhandari et al., “DeepSpeed-MoE,” 2022 — [paper](https://arxiv.org/abs/2201.05596).

---

← [Transformer and Attention Engines](03_Transformer_and_Attention_Engine_Microarchitecture.md) · [Compute Dataflows index](00_Index.md) · next → [Mapping and Memory](../02_Mapping_and_Memory/00_Index.md)
