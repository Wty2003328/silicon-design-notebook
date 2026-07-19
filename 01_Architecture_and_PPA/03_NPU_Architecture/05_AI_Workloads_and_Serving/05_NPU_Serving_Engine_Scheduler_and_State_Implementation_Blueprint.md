# NPU Serving-Engine, Scheduler, and State Implementation Blueprint

> **Abbreviation key:** neural processing unit (NPU); artificial intelligence (AI); key-value (KV) cache; service-level objective (SLO); time to first token (TTFT); time per output token (TPOT); direct memory access (DMA); mixture of experts (MoE).

## 0. Purpose and design ideology

This chapter reconstructs a serving engine that selects compiled NPU executables, binds dynamic request state, schedules queues, and operates across one or more NPUs. Its design ideology is **shape-aware scheduling with explicit state ownership**. The engine must stay within compiled profiles and device resources while meeting SLOs and preserving KV, event, and command generations.

## 1. Components and request state

Components are request gateway, model/executable registry, admission, shape/profile classifier, scheduler, host preparation, runtime submission/completion, KV/state manager, sampling/output, multi-NPU communication, and health/telemetry.

A request record contains tenant/model/version, input schema and dynamic dimensions, arrival/deadline/priority, phase, prefill offset/chunk, decode position, selected executable/profile, KV/page state and generation, batch/slot, assigned NPU/group, sampling/random state, fallback state, dependencies, cancel epoch, timestamps, and terminal result.

Request states are Accepted → Validated → Admitted → Prepared → Ready → Submitted → Executing → Completing → phase transition or Finalizing → Completed. Cancellation/fault goes through Draining until asynchronous DMA/commands cannot touch reclaimed state.

## 2. Profile and executable dispatch

The classifier maps request/model phase and shapes to a profile whose guards cover dimensions, layout, precision, sequence/batch/token range, and topology. It estimates padding, expected cycles/bytes, workspace/KV, and communication. Profile selection is part of scheduler cost—not a static lookup—because a wide profile may waste work while a narrow profile fragments batches.

Maintain executable availability per worker/group: loaded weights, relocation, warm state, command/event capacity, memory pools, and firmware compatibility. If the exact profile is absent, policy may queue for load/compile, pad, use another group, fall back, or reject. The SLO and quality contract decides; silent fallback is prohibited.

## 3. Admission and resource ledger

Admission reserves or budgets model residency, request/command/event entries, host buffers, DMA/IOMMU mappings, scratchpad/workspace, KV capacity, communication, and output credits. The ledger distinguishes compile-time per-invocation maximum from device-shared pools and persistent state.

For each worker, track capacity totals, reserved, live, pending-free, and failure reserve. Pending-free remains unavailable until device events complete. An NPU with static executable workspace may have lower fragmentation but fewer concurrent profiles; dynamic suballocation improves concurrency at verification cost.

## 4. Scheduler epoch

1. consume completion/fault/cancel events and reconcile resource ledger;
2. publish valid outputs/KV and advance request state;
3. classify queued requests by phase/profile/deadline/tenant/model;
4. enforce decode/real-time service and starvation policy;
5. choose compatible prefill chunks and batch slots;
6. verify executable residency and reserve KV, runtime invocation, command/event, workspace, and communication;
7. prepare tensor bindings, page tables, dynamic dimensions, dependencies, and fallback boundaries;
8. submit to runtime and record invocation generation;
9. schedule next wake from completion, queue arrival, or deadline timer.

On completion, the engine accepts output only when context/invocation/request generations match. It applies fallback results under the same dependency and state contract. Partial command completion is not a request-visible commit unless the executable defines safe checkpoints.

## 5. Batching under compiled profiles

Batch formation is constrained by supported profile and padding. Pack requests with compatible model/version, phase, precision, topology, and shape bucket. Record logical-to-physical slot mapping and masks. A bucket that improves array utilization can delay rare shapes; aging/fallback prevents starvation.

Chunked prefill divides prompt work into supported token profiles. Continuous decode requires an executable capable of variable active masks/sequence lengths or a set of bucketed variants. Rebuilding bindings and device metadata must stay ahead of NPU completion; otherwise the host runtime becomes the bottleneck.

## 6. KV and persistent-state layout

The KV manager maps logical request/layer/token/head state to NPU-accessible buffers matching executable layout, precision, sharding, alignment, and page-table schema. A block stores allocation/generation, model/profile/layout, owner/prefix, token range, NPU/tier/shard, ready/last-use event, reference/lease, transfer, and eviction.

Profile changes may require state compatibility. If two executables use different KV layout, transition uses a conversion command with destination reservation and ownership commit; direct rebinding is illegal. Prefix reuse key includes model/tokenizer/adapter, token sequence, positional semantics, layout/precision/sharding, and executable compatibility.

Publish state only after device completion. Cancellation marks discard but waits for DMA/commands. Eviction/reclamation respects all reader and transfer events. Finish reserve protects active decode allocation.

## 7. Irregular, fallback, and MoE execution

Dynamic control, unsupported sampling, embeddings, or rare shapes may execute on CPU/GPU fallback. The engine schedules transfers and dependencies explicitly and accounts for fallback capacity/latency. If fallback frequency grows, the plan’s performance assumptions fail; expose it as a primary metric.

MoE scheduling tracks routed token counts, expert capacity/overflow, expert weights/residency, destination NPU, pack/all-to-all/compute/unpack dependencies, and tail expert. A static expert mapping is simple but imbalanced; dynamic placement/migration moves large weights and changes executable/topology assumptions. Capacity policy defines drop, reroute, or overflow expert semantics.

## 8. Multi-NPU and state transfer

Parallel group state includes group/version, ranks/devices, executable/sharding, collective sequence, communication buffers, health, and epoch. All invocations in a group use compatible profile and sequence. Timeout/failure aborts or recovers according to device/runtime support; no rank advances alone.

Disaggregated or migrated KV uses a transfer ledger with source/destination group, layout/precision/shards, token/layer ranges, allocations, source readiness, byte/chunk state, checksum, commit generation, retry, and timeout. Source remains authoritative until destination verification and ownership commit. Duplicate/late transfers are generation-checked.

## 9. Sizing, policies, and faults

Model service curves per executable/profile/phase from calibrated NPU counters and end-to-end timestamps. Use open-loop queueing and capacity ledgers. Goodput is constrained by command/event rate, host preparation, array/vector/memory, external bandwidth, KV capacity, communication, and output.

| Choice | Benefit | Cost |
|---|---|---|
| more shape buckets | utilization | queues, compilation, rare-shape latency |
| padding | batch compatibility | wasted compute/memory |
| static workspaces | predictability | low concurrency/stranded bytes |
| fallback | operator coverage | transfers and variable tail |
| phase separation | isolation | state transfer and stranded capacity |
| preemption | SLO control | checkpoint/restart and device support |

Fault matrix includes executable/profile mismatch, binding failure, out of memory, DMA/translation fault, command timeout, numerical/poison error, device reset, collective failure, fallback failure, state-transfer failure, and output disconnect. Specify state authority, retry idempotence, cleanup, user result, and health transition.

## 10. Invariants, observability, and staged build

Invariants: a request uses compatible model/executable/profile/state layout; every resource reservation is released once after last use; batch slot/request/invocation generations match; KV published ranges reflect completed work; one group owns live state; fallback preserves semantic/version state; collective sequence agrees; accepted work terminates under bounded dependencies.

Trace request → profile decision → batch → invocation → command/DMA/event → array/memory/collective → KV update → sampling/output. Record profile padding, fallback, queue reasons, resource ledger, command gaps, NPU counters, KV, communication, TTFT/TPOT/goodput, failures, and waste.

Build single model/profile/static batch; request states and cancellation; multiple profiles/buckets; continuous batching and KV paging; fallback; multi-NPU; preemption/migration/disaggregation; multi-tenant/overload. Each stage retains a functional reference and ownership audit.

---

← [NPU Compiler/Runtime](04_NPU_Compiler_Runtime_and_Executable_Implementation_Blueprint.md) · next → [NPU AI Stack Verification, Operations, and Deployment](06_NPU_AI_Stack_Verification_Operations_and_Deployment_Blueprint.md)
