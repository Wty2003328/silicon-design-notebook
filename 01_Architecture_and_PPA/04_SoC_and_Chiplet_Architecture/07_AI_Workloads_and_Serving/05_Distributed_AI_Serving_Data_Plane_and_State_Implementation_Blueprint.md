# Distributed AI Serving Data-Plane and State Implementation Blueprint

> **Abbreviation key:** artificial intelligence (AI); central processing unit (CPU); graphics processing unit (GPU); neural processing unit (NPU); key-value (KV) cache; remote direct memory access (RDMA); network interface controller (NIC); service-level objective (SLO); time to first token (TTFT); time per output token (TPOT); mixture of experts (MoE).

## 0. Purpose and design ideology

This chapter reconstructs the request and state data plane across heterogeneous workers. The design ideology is **local fast paths, explicit distributed commits, and end-to-end backpressure**. Hardware links move bytes; the AI stack must define request identity, ownership, consistency, retries, and the point at which compute or state is committed.

## 1. End-to-end component and request schema

~~~mermaid
flowchart LR
    CLI["client"] --> GW["gateway + auth + limits"]
    GW --> ROUTE["model/phase/locality router"]
    ROUTE --> PRE["preprocess / prefill worker group"]
    PRE --> XFER["KV/state-transfer service"]
    XFER --> DEC["decode worker group"]
    DEC --> OUT["sampling / stream / result"]
    STORE["artifact/storage services"] --> PRE
    STORE --> DEC
    FAB["collective + RDMA data plane"] --> PRE
    FAB --> DEC
~~~

A request envelope stores globally unique/idempotency ID, tenant/auth context, model/version/adapter, input schema/payload reference, arrival/deadline/priority, generation parameters and random seed, locality/prefix hint, phase/state handle, trace context, cancel epoch, retry count, output stream credits, and status. Large tensors/state travel by registered buffer handles, not control-message copies.

## 2. Gateway, routing, and admission chain

The gateway authenticates, authorizes, validates size/schema, applies rate/concurrency limits, assigns identity/deadline, and chooses synchronous/stream protocol. It does not admit expensive model state without worker capacity confirmation.

Routing filters endpoint compatibility and health, then scores queue/deadline, prefix/model locality, available KV/workspace, topology, tenant/failure domain, and predicted service. Reservation tokens have endpoint/generation, resource estimate, expiry, and nonce; the worker accepts exactly once or rejects stale/duplicate.

Backpressure propagates from output client → worker output buffers → decode scheduling → KV/admission → router/gateway. Bounded queues at every edge report overload rather than absorbing unbounded work. Separate control/fault progress from bulk data so saturation does not block cleanup.

## 3. Distributed request state machine

States include Received, Routed, Reserved, Preprocessing, Prefilling, HandoffPreparing, Transferring, DecodeReserved, DecodeReady, Decoding, Streaming, Completed, plus Canceling/Failed. Every transition appends or updates an authoritative state record under the chosen durability scope.

The happy-path lifecycle below shows where each commit point makes work authoritative (the five commits are defined immediately after):

~~~mermaid
stateDiagram-v2
    [*] --> Received
    Received --> Routed
    Routed --> Reserved : admission commit
    Reserved --> Preprocessing
    Preprocessing --> Prefilling
    Prefilling --> HandoffPreparing : prefill commit
    HandoffPreparing --> Transferring
    Transferring --> DecodeReserved : handoff commit
    DecodeReserved --> DecodeReady
    DecodeReady --> Decoding
    Decoding --> Decoding : token commit
    Decoding --> Streaming
    Streaming --> Streaming : response commit
    Streaming --> Completed
    Completed --> [*]
    Decoding --> Canceling
    Canceling --> Failed
    Failed --> [*]
    note right of Canceling : any state may enter Canceling or Failed on cancel or fault
~~~

Stateless retry is possible before expensive state creation. After prefill/KV, retry requires an authoritative state owner or recomputation. The protocol defines commit points:

- admission commit: worker accepts responsibility/resources;
- prefill commit: KV ranges are complete at source;
- handoff commit: destination verified state and ownership changed;
- token commit: sampled token and KV step become the next authoritative sequence state;
- response commit: token/chunk is durably/idempotently associated with client progress if required.

Exactly-once network delivery is unrealistic; design idempotent handling with request/transition generations. Duplicate messages return recorded result or are rejected without repeating side effects.

## 4. KV/state transfer protocol

A transfer descriptor records transfer/request/model versions, source/destination group epochs, tensor/layer/token/shard ranges, layout/precision, block list and byte extents, source ready events, destination allocations, route/registration keys, checksums/poison, chunk sequence/acknowledgement, retry, deadline, and commit state.

Protocol:

1. destination reserves compatible capacity and returns handles/epoch;
2. source drains producers and freezes or versions transferred ranges;
3. transport copies chunks using NIC/GPU/NPU/CPU path;
4. destination validates counts/checksums/layout and publishes readiness;
5. coordinator atomically records ownership commit;
6. source releases only after commit and outstanding readers complete;
7. timeout before commit aborts destination and keeps source authoritative; after commit recovery follows destination ownership.

Layer-pipelined transfer commits independently usable layer ranges only if decode executable can consume partial readiness safely. Layout conversion is a named compute stage with new destination format identity.

## 5. Collective and parallel execution data plane

Worker groups have membership/version, rank mapping, topology, collective sequence, buffer ownership, timeouts, and fault policy. Tensor, pipeline, expert, and data parallelism create different communication and synchronization. The runtime submits collectives with dependency events and stable sequence IDs; all ranks agree.

For MoE, route records token-to-expert assignment, capacity/overflow, source/destination, packed offsets, dispatch collective, expert compute, and return scatter. Backpressure from a hot expert must bound upstream routing buffers; dropping or overflow-expert behavior follows model semantics.

Pipeline stages attach microbatch/request identity and activation generation. A downstream timeout cannot cause upstream buffer reuse until cancellation/abort propagates.

## 6. Memory tiers and remote state

State may reside in device HBM, host DDR, peer device, CXL-attached memory, or remote service/storage. Each tier advertises capacity, registration, bandwidth/latency distribution, access granule, consistency, failure, and cost. A state-location record stores authoritative tier, replicas/cache copies, dirty/version, leases/readers, migration, and eviction.

Caching remote KV/prefix requires coherence at the software object level. Immutable prefix replicas are simple. Mutable decode state has one writer; readers use committed versions. Eviction revokes mapping/leases and waits for in-flight operations before reuse.

## 7. Scheduling, fairness, and overload across pools

Global routing and local schedulers share capacity signals but avoid per-token centralized decisions. Workers export bounded summaries: queue/phase work, KV free/reserved, model/prefix residency, service curves, health, and generation. Signals are stale; reservation closes the race.

Admission budgets end-to-end deadline across gateway, queue, prefill, transfer, decode iterations, and output. Prefill pool, transfer fabric, and decode pool each need headroom. Overproducing prefill KV when decode is saturated only shifts the queue into expensive state.

Policies use tenant weighted fairness, deadlines, phase priorities, locality, and failure reserve. Backpressure can stop prefill, limit prompt length/chunks, reject new work, or reroute. Always protect resources required to finish admitted decode and cleanup failed transfers.

## 8. Failure and recovery matrix

Failures include gateway/router, source/destination worker, device, collective rank, NIC/link, state-transfer service, metadata store, client disconnect, and network partition. For each define detector/timeout, authoritative state, in-flight side effects, retry/idempotence, cleanup, failover/recompute, user status, and telemetry.

Leases/heartbeats detect failed owners but do not by themselves prove device writes stopped. Reuse requires fencing via group/context epoch, transport revoke, and device reset/quiescence as appropriate. Split-brain prevention is more important than fast retry.

For source failure before handoff commit, recompute prefill or recover from durable/replicated state. Destination failure after commit needs destination replica/checkpoint or request failure/recompute; source cannot assume old state remained valid. Define policy by cost and SLO.

## 9. Sizing and trade-offs

Transfer lower bound is setup/queue plus bytes/effective bandwidth; end-to-end uses the critical path and overlap. Size registered buffers and descriptors from rate × occupied time plus burst/failure. State capacity includes live KV, handoff duplicate, prefix, communication, fragmentation, and failover/rollout reserve.

| Choice | Benefit | Cost |
|---|---|---|
| coupled prefill/decode | no handoff, simple ownership | phase interference and inflexible capacity |
| disaggregated pools | independent scaling/isolation | transfer, distributed commit, failure |
| replicated state | failover/read locality | capacity and consistency |
| recompute on failure | simple storage | TTFT/compute and cascading load |
| centralized state store | coordination/recovery | latency and availability bottleneck |
| locality routing | reuse | imbalance/stale affinity |

## 10. Invariants, observability, and staged construction

Invariants: one authoritative owner/version for mutable request state; reservations and transitions are idempotent; destination cannot consume before verified commit; source cannot release before ownership commit/readers finish; group/routing epochs reject stale traffic; output/token side effects commit once; quotas include in-flight duplicates; backpressure remains bounded and cleanup traffic progresses.

Trace gateway → routing/reservation → worker scheduler → kernels/commands/collectives → KV blocks/transfers/commit → sampling/output. Metrics include queue/reservation, phase work, capacity, transfer bytes/chunks/retries, collective sequence/imbalance, state ownership/lease, routing locality, backpressure, TTFT/TPOT/goodput, faults/recompute/waste.

Build one coupled worker; routed stateless replicas; stateful request IDs/cancellation; multi-device collectives; explicit KV manager; destination reservation/copy/commit; disaggregated pools; multi-tier cache; failure fencing/recovery; multi-tenant overload.

---

← [AI Platform Control Plane](04_Heterogeneous_AI_Platform_and_Control_Plane_Implementation_Blueprint.md) · next → [AI Platform Verification, Operations, and Deployment](06_AI_Platform_Verification_Operations_and_Deployment_Blueprint.md)
