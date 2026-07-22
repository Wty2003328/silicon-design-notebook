# Heterogeneous AI Platform and Control-Plane Implementation Blueprint

> **Abbreviation key:** artificial intelligence (AI); central processing unit (CPU); graphics processing unit (GPU); neural processing unit (NPU); system on chip (SoC); high-bandwidth memory (HBM); input-output memory management unit (IOMMU); network interface controller (NIC); service-level objective (SLO); application programming interface (API).

## 0. Purpose and design ideology

This chapter reconstructs the platform that makes CPU, GPU, NPU, memory, storage, and fabric usable as one AI system. The control plane discovers resources, validates and places model artifacts, creates execution groups, configures memory/network/security, routes requests, and manages lifecycle. The design ideology is **separate desired state from observed state, and make placement a constraint proof rather than a label**.

## 1. Control-plane components

~~~mermaid
flowchart LR
    REG["model/artifact registry"] --> DEP["deployment controller"]
    INV["hardware/topology inventory"] --> PLC["placement solver"]
    POL["tenant/SLO/security policy"] --> PLC
    DEP --> PLC
    PLC --> CFG["versioned worker-group specification"]
    CFG --> AG["node/device agents"]
    AG --> CPU["CPU services"]
    AG --> GPU["GPU workers"]
    AG --> NPU["NPU workers"]
    AG --> MEM["memory/storage/network resources"]
    TEL["health/capacity telemetry"] --> INV
    TEL --> DEP
~~~

Components include model/artifact registry, hardware/topology inventory, deployment and placement controllers, node/device agents, configuration/secrets service, admission/router control, capacity/autoscaling, health/fault membership, and telemetry/catalog. The fast request/data path remains outside controllers; control unavailability must not immediately stop already configured workers.

## 2. Desired-state object schemas

A model-version object stores immutable artifact/tokenizer/feature hashes, graph/compiler/engine variants by architecture, precision/quality, weight/KV/state formats, resource profiles, supported shapes, security/provenance, and rollback predecessor.

The registry preserves the source semantic-graph and intermediate-representation (IR) identities used to build every CPU, GPU, or NPU executable variant. The control plane does not rewrite the IR; the architecture-owned compiler does. It does require a lineage record from source artifact and compiler/pass configuration to target executable, shape guards, numerical/quality result, and compatibility. Placement may select among validated variants but cannot substitute an unvalidated compiler product.

A deployment object stores model/version, replica/worker-group count, hardware classes, parallelism topology, phase roles, memory tiers, SLO/quality, traffic policy, tenant/quota, update strategy, failure domains, configuration, and artifact references.

A worker-group specification stores unique generation, node/chiplet/device membership, CPU core/NUMA allocation, GPU/NPU partition, HBM/DDR reservations, IOMMU/security context, scale-up/scale-out interfaces, parallel groups/ranks, executables, state layout, queue/KV/workspace budgets, network endpoints, health/readiness checks, and drain policy.

Observed status is separate: applied generation, loaded/warm plans, allocated resources, topology/link health, free capacity, active requests/state, errors, and timestamps. Controllers reconcile desired and observed state idempotently.

## 3. Hardware and topology inventory

Inventory records CPU sockets/cores/NUMA memory, GPU/NPU devices/partitions, local HBM/DDR, interconnect links and bandwidth/latency/failure domain, NIC/storage paths, IOMMU/virtualization, power/thermal limits, firmware/driver/capability, health, and allocation. Stable device identities survive agent restart; epochs distinguish reset/replacement.

Represent topology as graph nodes/resources and directed edges with usable bandwidth, latency, ordering/coherence semantics, contention group, and health. Placement cost uses the traffic matrix from tensor/collective/KV/model-load flows. A nominal device count without link graph cannot choose tensor/expert groups or state-transfer route.

## 4. Placement solver

Hard constraints include target compatibility, capacity for weights/KV/workspace/communication and rollout reserve, group size/topology, security/tenant isolation, device health, failure-domain policy, power/thermal, and required storage/network reachability. Soft objectives include communication cost, model/prefix affinity, load balance, fragmentation, energy/cost, and migration/startup time.

Placement steps:

1. filter eligible hardware by artifact/executable/capability/security;
2. form topology-compatible device groups;
3. check persistent/transient capacity and failure/rollout headroom;
4. estimate compute/memory/communication service by phase/profile;
5. score locality, contention, failure domain, energy, and current load;
6. choose group and generate full resource specification;
7. reserve atomically or roll back all partial reservations;
8. ask agents to load/configure/warm; publish routable readiness only after checks.

Record constraint failures and score components. A placement policy that cannot explain why a group was chosen cannot be audited or debugged.

## 5. Artifact distribution and model residency

The registry stores manifests and immutable shards; distribution resolves nearest trusted cache/storage, reserves staging memory, streams with bounded queues, verifies checksum/signature, transforms/packs only under a versioned build, loads to host/device, relocates executables, forms parallel groups, and warms representative paths.

Residency states Registered → Fetching → Verified → Transforming → Loading → GroupForming → Warming → Ready, with Failed and Draining/Unloaded. A group is not routable when only some ranks have weights or collectives. Loading consumes storage, PCIe/fabric, host memory, HBM, CPU transformation, and temporary duplicate capacity; placement includes this transient peak.

~~~mermaid
stateDiagram-v2
    [*] --> Registered
    Registered --> Fetching
    Fetching --> Verified
    Verified --> Transforming: versioned build
    Transforming --> Loading
    Loading --> GroupForming
    GroupForming --> Warming
    Warming --> Ready: all ranks warm
    Ready --> Draining: scale-in or rollout
    Draining --> Unloaded
    Unloaded --> [*]
    Verified --> Failed: bad checksum or signature
    Loading --> Failed: resource error
    Failed --> [*]
~~~

Content-addressed caches key source and transformed artifacts by all semantic/layout/compiler/target inputs. Eviction respects active references and rollout predecessor.

## 6. Unified runtime and driver boundary

The platform API abstracts discovery, memory import/export, executable load, group creation, submission/events, profiling, faults, and reset, but does not erase device-specific semantics. Capability queries reveal supported layouts/precision/queues/coherence/state transfer.

Cross-device buffer handles contain allocation/generation, owner device/context, address/registration, size/layout/precision, readiness/last-use events, sharing/permissions, transport route, and revoke epoch. Transfer/collective commands name source/destination group generations and completion visibility. CPU, GPU, and NPU adapters convert to their native runtime while preserving common ownership/error identity.

## 7. Router and service discovery control

The controller publishes endpoint sets with model/version, phase/role, supported profiles, topology/group, locality keys, capacity signals, health, and generation. Routers consume snapshots and continue during brief controller loss. Endpoint removal first stops new assignment, then drains or transfers state; stale routing epochs are rejected.

Routing policies combine model/phase compatibility, prefix/KV affinity, queue/deadline, available memory, failure domain, tenant, and locality. Report why each routing decision or rejection occurred. Load-only routing can destroy prefix affinity; affinity-only routing can overload a worker.

## 8. Capacity and autoscaling

Maintain demand by model/phase/shape/tenant and service curves by hardware/executable/configuration. Required groups account for offered load, target utilization below the queueing knee, SLO distribution, worker failure, rollout overlap, load/warm delay, and burst forecast.

Scale-out states Planned → Reserving → Loading → Warming → Ready; scale-in uses DrainRequested → Unroutable → StateDrained → Unloaded → Released. Hysteresis and minimum dwell avoid oscillation. For stateful decode, scale-in may wait much longer than stateless replicas; plan prefill/decode pools separately.

## 9. Security and multi-tenancy

Control objects are authenticated, authorized, signed, and auditable. Enforce tenant/model/device/memory/network quotas; IOMMU/context isolation; encrypted or trusted artifact transport; secrets rotation; signed code/executables; debug/profiler access; zeroization; and restricted topology/health disclosure.

Separate control-plane identities from request identities but link them in protected audit. A compromised agent cannot self-declare readiness or capacity without attested/validated checks under the threat model.

## 10. Trade-offs and invariants

| Choice | Benefit | Cost |
|---|---|---|
| centralized placement | global optimization | availability/scale bottleneck |
| hierarchical controllers | scale/failure isolation | stale views and coordination |
| common runtime abstraction | portability | lowest-common-denominator risk |
| preloaded replicas | low latency | stranded expensive memory |
| just-in-time loading | utilization | cold latency/storage bursts |
| fine device partitioning | multi-tenancy | fragmentation and interference |

Invariants: desired-state generations are monotonic; one resource allocation belongs to one live deployment lease; a routable endpoint has compatible loaded/warm artifacts and healthy group; placement never exceeds hard capacity/security/topology; artifact identity is immutable; stale agent/routing epochs cannot mutate current resources; control retries are idempotent; teardown drains/revokes before reuse.

## 11. Staged implementation and observability

Build static inventory and manually placed single-device worker; versioned artifact registry; agent reconciliation; topology-aware groups; automated placement/reservations; health/readiness and routing snapshots; drain/rollout; capacity/autoscaling; heterogeneous/multi-tenant placement and failures.

Trace deployment generation → placement decision → reservation → artifact stages → group formation → readiness → routing. Metrics include constraint failures, resource fragmentation, load/warm time, artifact cache, topology health, ready versus desired capacity, routing reasons, drain time, control reconciliation, stale epochs, and rollout/failure reserve.

---

← [AI Serving Performance/Research](03_AI_Serving_Performance_Analysis_and_Research_Methodology.md) · next → [Distributed AI Serving Data Plane and State](05_Distributed_AI_Serving_Data_Plane_and_State_Implementation_Blueprint.md)
