# CPU Serving Runtime, Scheduler, and State Implementation Blueprint

> **Abbreviation key:** central processing unit (CPU); artificial intelligence (AI); key-value (KV) cache; large language model (LLM); service-level objective (SLO); time to first token (TTFT); time per output token (TPOT); non-uniform memory access (NUMA); remote procedure call (RPC); graphics processing unit (GPU); neural processing unit (NPU).

## 0. Purpose and design ideology

This chapter specifies the stateful service around CPU-only or CPU-coordinated inference. The design ideology is **admit only work whose state and service can be owned, then make every transition explicit**. Raw throughput is secondary to goodput: completed, correct requests satisfying latency and isolation objectives.

## 1. Service components and interfaces

~~~mermaid
flowchart LR
    GW["network/RPC gateway"] --> ADM["validation + admission"]
    ADM --> Q["tenant/model queues"]
    Q --> SCH["batch/phase scheduler"]
    SCH --> EXEC["CPU plan or device runtime"]
    EXEC --> KV["KV/persistent-state manager"]
    EXEC --> OUT["sampling/postprocess/stream"]
    REG["model registry + residency"] --> SCH
    CAP["capacity + telemetry"] -. controls .-> ADM
    CAP -. controls .-> SCH
~~~

Define request submission, streaming response, cancellation, health/readiness, model load/unload, and administrative configuration interfaces. Submission validates authentication/tenant, model/version, input schema, length/shape bounds, generation parameters, deadline/priority, idempotency identifier, and resource quota before allocating expensive state.

## 2. Request and model state schemas

A request record stores request/tenant/model/version identity, arrival/deadline, input and tokenizer schema version, prompt/features, generated output, phase, prefill progress, decode position, sampling state and random seed, KV allocation/generation, assigned worker/device/NUMA node, batch slot, dependencies, cancel epoch, retry count, timestamps, error, and terminal delivery status.

Use a state machine such as:

$$Accepted\rightarrow Admitted\rightarrow Preprocessing\rightarrow ReadyPrefill\rightarrow Prefilling\rightarrow ReadyDecode\leftrightarrow Decoding\rightarrow Finalizing\rightarrow Completed.$$

From any live state, cancellation/fault may lead through Draining to Canceled/Failed. A transition owns its side effects and is committed once. Network disconnect does not automatically mean compute cancellation; policy defines whether results may be cached, discarded, or continued.

A model-residency record stores artifact/plan hashes, tokenizer/feature version, NUMA/device placement, packed-weight objects, workspace/KV pool reservations, warm state, active references, load/drain state, health, and rollback predecessor. New requests bind to an immutable model version; rolling update never changes semantics beneath an active request.

## 3. Admission as a capacity proof

Admission checks input limits, model readiness, tenant quota, queue/deadline feasibility, core/device budget, KV and workspace headroom, output/network backpressure, and failure reserve. Maintain distinct hard capacity (bytes/entries), rate capacity (compute/memory/network per second), and latency risk.

For an LLM with $L$ layers, $H_{KV}$ key/value heads, head dimension $D$, element bytes $b$, and total live tokens $T$, base KV use is

$$M_{KV}=2LH_{KV}DbT,$$

plus allocator pages, metadata, prefix copy-on-write, speculative branches, and safety margin. Reject or queue before allocation crosses a high-water mark that leaves no space for in-flight requests to finish.

Admission under overload may reject, delay with a bounded queue, degrade optional work, route elsewhere, or select a smaller/quantized model if the contract permits. It must not accept unlimited work and rely on timeouts: timed-out work can continue consuming CPU/device state and worsen overload.

## 4. Scheduler loop in causal order

The runtime is a single decision loop: one scheduler thread advances every request so that step order itself enforces the safety rules — reclaim before admitting, reserve before publishing, and publish before mutating shared state. At each scheduling epoch:

1. process completions, faults, cancellations, and freed KV/output capacity;
2. update per-request phase/progress and deadline slack;
3. admit queued work within resource and SLO budgets;
4. choose prefill chunks and decode requests under token/sequence/byte/workspace budgets;
5. form shape-compatible batches and select an execution plan;
6. reserve batch slots, KV pages, workspace, CPU teams/device queue entries, and output credits atomically;
7. publish the batch task with request IDs and generations;
8. on completion, scatter outputs to matching requests, update KV and sampling state, stream ready tokens, and release resources;
9. record decisions and schedule the next epoch.

Never form a batch and then discover that one request cannot allocate KV; reservation precedes publication or has a complete rollback. A delayed completion uses request/batch generation to avoid writing into a reassigned slot.

~~~mermaid
flowchart TD
    EPOCH(["epoch start"]) --> REAP["reap completions, faults, cancels<br/>free KV and output capacity"]
    REAP --> UPD["update phase, progress<br/>and deadline slack"]
    UPD --> ADM["admit queued work<br/>within resource and SLO budgets"]
    ADM --> SEL["select prefill chunks<br/>and decode requests"]
    SEL --> FORM["form shape-compatible batch<br/>select execution plan"]
    FORM --> RES{"reserve slots, KV, workspace,<br/>output credits atomically?"}
    RES -- "no" --> ROLL["roll back<br/>requeue request"]
    RES -- "yes" --> PUB["publish batch task<br/>with request IDs and generations"]
    PUB --> STEP["execute step<br/>CPU plan or device runtime"]
    STEP --> DONE["on completion scatter outputs<br/>append KV, stream tokens, release"]
    ROLL --> NEXT["record decisions<br/>schedule next epoch"]
    DONE --> NEXT
    NEXT --> EPOCH
    RES -. "reserve pages" .-> KVM[("KV / state<br/>manager")]
    DONE -. "append and free" .-> KVM
~~~

## 5. Batching and phase policy

Static batching maximizes regularity but waits for the slowest request. Continuous batching changes membership after each decode iteration. Chunked prefill bounds the long compute burst from a large prompt so decode deadlines receive service.

The scheduler budgets useful tokens, compute, memory bytes, and expected time. Prefill is commonly compute/reuse friendly; decode at small batch is weight/KV bandwidth sensitive. Co-scheduling improves utilization but causes interference. Separate CPU core pools or accelerators isolate phases at the cost of stranded capacity and state transfer.

Policies include first-come-first-served, earliest deadline, shortest predicted remaining work, tenant-weighted deficit, and model/NUMA affinity. Deadline policy needs service-time estimates; affinity reduces weight/KV movement but can create hot workers. State maximum starvation or age promotion.

## 6. KV, prefix, and temporary-state manager

A KV block record stores block/generation, owner request or shared-prefix object, model/layer/token range, layout/precision, memory tier/NUMA/device location, page-table mapping, reference count, dirty/transfer state, last use, readiness event, checksum/poison, and eviction state.

Allocation reserves blocks before execution. Append publishes newly written token ranges only after completion. Prefix sharing uses immutable shared blocks and copy-on-write for divergent continuations. A prefix-cache key includes model/version, tokenizer version, exact token sequence or collision-safe digest, positional/attention semantics, layout/precision, and adapter context. Reference counting plus generations prevents use-after-free; leases or epoch reclamation protect asynchronous users.

Eviction first removes unreferenced prefixes, then policy-selected live request state only if preemption/swap is supported. Swapping to remote NUMA, host storage, or another device accounts for transfer time and source/destination ownership. Capacity is not reclaimed until all readers/transfers acknowledge.

## 7. Sampling, streaming, and exactly-once side effects

Sampling state includes logits processing parameters, random generator state, repetition constraints, and speculative branch state. Deterministic replay requires defined random seed and operation order. CPU sampling can overlap next device step but must synchronize before the token determines later input.

The output path applies token acceptance, detokenization/feature decode, safety/postprocessing if part of the contract, and streaming with bounded buffers. Slow clients create backpressure. Policy may pause decode, buffer to a limit, or cancel; unlimited output buffering violates memory isolation.

Idempotency concerns externally visible billing/logging/response operations. Compute may be retried only if its state transition is repeatable or guarded by generation/commit records. A token must not be billed or streamed twice after worker retry.

## 8. CPU-only and accelerator-coordinated modes

CPU-only workers schedule thread teams and NUMA-local kernels directly. Accelerator workers prepare descriptors, submit asynchronously, and keep CPU completion/sampling/streaming ahead of device work. Maintain separate queues for host preparation, device execution, and completion to expose starvation.

For heterogeneous fallback, pin one request’s state to a compatible plan/layout unless migration explicitly converts it. Moving a live request between CPU, GPU, and NPU may require KV layout/precision transformation and changes numerical behavior; model it as a versioned state-transfer protocol.

## 9. Fault, cancellation, and overload behavior

Failure matrix entries include network disconnect, malformed input, model-load failure, out-of-memory, kernel error, device reset, worker crash, state corruption, deadline expiry, and dependency service loss. For each specify detector, contained scope, state validity, retry/idempotence, cleanup, client status, and telemetry.

Cancellation increments the request epoch, removes not-yet-published work, marks in-flight tasks discard-on-return, releases resources only when no asynchronous user remains, and terminates the stream once. Watchdogs inspect the oldest queue/task and progress counters; a global timeout without resource snapshot is hard to debug.

## 10. Sizing, trade-offs, and invariants

Little’s law $N=\lambda W$ relates mean live requests to arrival rate and mean time, but size queues/KV for tail and burst distributions. Safe offered load is below the knee where queue delay consumes deadline slack. Measure open-loop arrivals; a closed-loop client hides overload by slowing submissions.

| Choice | Benefit | Cost |
|---|---|---|
| large batches | weight reuse and throughput | TPOT, queueing, KV/workspace |
| prefix caching | avoids repeated prefill | capacity, lookup integrity, version invalidation |
| request preemption | protects deadlines | swap/recompute cost and complex ownership |
| NUMA/model affinity | locality | imbalance and failover movement |
| separate phase pools | isolation | stranded capacity and transfer |
| aggressive admission | utilization | nonlinear tail collapse and cleanup waste |

Invariants: each request has one live owner and one terminal result; each batch slot maps to one request generation; KV ranges are published only after valid compute; shared prefixes are immutable; cancellation/failed epochs cannot mutate current state; quota accounting includes queued, executing, swapped, and buffered work; and accepted work either completes or returns a defined terminal failure under bounded dependencies.

## 11. Staged implementation and evidence

Build one model, one worker, static batch, contiguous state, and synchronous output. Add explicit request states and cancellations; bounded queues and admission; continuous batching; paged KV/prefix; asynchronous accelerator submission; multi-model/tenant fairness; preemption/migration; and distributed routing in that order.

Trace request, scheduler epoch, batch, plan, task, KV block, device submission, sampling, and stream identities. Metrics include queue time by state, admitted/rejected reason, batch composition, prefill/decode work, KV occupancy/fragmentation, plan/fallback, CPU/device time, cancellation waste, output backpressure, TTFT/TPOT/goodput, and deadline misses.

---

← [CPU AI Software Stack](04_CPU_AI_Software_Stack_Implementation_Blueprint.md) · next → [CPU AI Stack Verification, Operations, and Deployment](06_CPU_AI_Stack_Verification_Operations_and_Deployment_Blueprint.md)
