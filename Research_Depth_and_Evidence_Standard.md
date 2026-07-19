# Research-Depth and Evidence Standard

This notebook serves two readers at once:

1. a first-time reader who needs every unfamiliar concept and abbreviation introduced before it is used; and
2. a research or advanced engineering candidate who must derive performance, challenge assumptions, design experiments, and defend conclusions with evidence.

“Detailed” therefore does not mean accumulating product facts. A research-depth page must connect:

$$
\text{workload} \rightarrow \text{hardware mechanism} \rightarrow
\text{theoretical model} \rightarrow \text{observable evidence} \rightarrow
\text{validated conclusion and trade-off}.
$$

---

## 1. Required layers of a substantive technical chapter

### 1.1 Orientation and vocabulary

- State why the topic exists and what problem it solves.
- Define every abbreviation and specialized term at first occurrence.
- Distinguish nearby concepts that beginners commonly confuse.
- State prerequisites and ownership boundaries so the reader knows what is assumed and what is deferred.

### 1.2 Contract and abstraction boundary

Name inputs, outputs, state, invariants, timing/ordering assumptions, failure behavior, and what the model deliberately omits. For a simulator, state which artifacts enter and how they become events. For a protocol, state what each endpoint may assume. For a microarchitecture, state the architectural behavior that must remain unchanged.

### 1.3 Mechanism in causal order

Explain what happens cycle by cycle, transaction by transaction, or stage by stage. Do not replace causality with a feature list. A strong mechanism section answers:

- what creates an event;
- where it waits and why;
- what resource serves it;
- what state changes;
- how completion or failure is observed;
- what downstream work becomes enabled.

### 1.4 Theory and assumptions

Derive the governing model and define every symbol, unit, and assumption. Include lower/upper bounds, limiting cases, and where the approximation fails. Useful models include throughput/latency laws, occupancy, roofline, queueing, working-set/capacity, communication, reliability, energy, and area/timing scaling.

An equation without assumptions is not a theory section. A number substituted into an unexplained formula is not a derivation.

### 1.5 Implementation consequences

Connect the abstract mechanism to queues, arrays, pipelines, state machines, networks, memories, clocks, physical placement, software/runtime policy, and verification. Explain why an apparently attractive technique costs area, energy, timing margin, complexity, or generality.

### 1.6 Measurement and simulation

For every important conclusion, identify:

- the metric and exact timestamp or counting boundary;
- the counter, trace, waveform, or simulator output that observes it;
- how raw events are aggregated into the reported result;
- calibration and validation against a stronger model or real system;
- sources of uncertainty, bias, and sampling error.

### 1.7 Worked reasoning

Include at least one worked example that changes a design decision. It should expose units, intermediate values, bottleneck classification, and sensitivity—not only the final answer.

### 1.8 Failure modes and research questions

State when the mechanism loses, what invalidates the model, how an implementation fails, and which questions remain open. Research preparation requires understanding the boundary of current knowledge, not just the accepted design.

### 1.9 Implementation-reconstruction contract

A chapter that claims implementation depth must let a reader reconstruct a defensible block specification without copying an existing design. Source code is not required, but naming components is not enough. The chapter must provide the following artifacts in prose, diagrams, tables, or equations:

1. **Requirements and external contract:** supported operations, legal inputs, outputs, ordering, precision, exceptions, privilege/security behavior, and explicit non-goals.
2. **Ownership decomposition:** which block owns every piece of state and which behavior belongs to an upstream or downstream block.
3. **State schema:** the fields stored in each queue, table, cache entry, descriptor, checkpoint, or protocol state, including allocation and reclamation conditions.
4. **Interface protocol:** request/response payloads, identity tags, ready/valid or credit behavior, backpressure, retry, cancellation, reset, and clock/power-domain assumptions.
5. **Causal control flow:** the normal path and exceptional paths, including arbitration, hazards, replay, flush, timeout, fault, and recovery.
6. **Resource and sizing model:** bandwidth, latency, capacity, associativity, queue depth, banking, port count, metadata cost, and at least one calculation connecting workload demand to a design choice.
7. **Policy choices:** the replacement, scheduling, routing, prediction, admission, or quality-of-service policy; alternatives; why one was selected; and workloads for which that choice loses.
8. **Invariants and progress:** properties that must never be violated, plus the mechanism that prevents deadlock, livelock, starvation, data loss, duplication, or imprecise state.
9. **Physical consequences:** critical paths, high-fanout structures, placement-locality needs, power/activity sources, clocking, and where pipelining changes visible behavior.
10. **Observability and verification:** counters, traces, assertions, reference models, directed corner cases, constrained-random tests, coverage targets, and performance-calibration experiments.
11. **Bring-up and evolution:** a minimum viable implementation, staged feature-enablement order, debug escape hatches, compatibility/versioning rules, and a path to scale the design.

The practical test is whether a reader can produce three reviewable documents from the chapter: a block diagram with ownership boundaries, a state-and-interface specification with invariants, and a verification/bring-up plan. If essential choices remain hidden behind phrases such as “the scheduler handles it” or “the cache returns the data,” the chapter is not implementation-reconstructable.

---

## 2. Claim and source discipline

Classify claims mentally before writing:

| Claim type | Example | Evidence expectation |
|---|---|---|
| physical/theoretical | dynamic energy scales with switched capacitance and voltage squared | textbook or foundational paper plus derivation |
| standard/protocol | legal transaction, command, timing, or state | current official standard/specification |
| implementation | a named design uses a structure | official architecture guide, paper, code, or measured reverse engineering |
| measured result | speedup, latency, bandwidth, power | reproducible experiment with workload/system/configuration |
| projection/roadmap | future product capability or adoption | dated primary announcement, clearly labeled as projected |
| inference | mechanism likely causes an observed result | explicit reasoning and alternative explanations |

Source preference:

1. standards and official specifications;
2. peer-reviewed or clearly identified research papers;
3. maintained source code and official technical documentation;
4. vendor architecture/programming guides;
5. reproducible measurements and public datasets;
6. secondary summaries only for orientation or to locate primary evidence.

Time-sensitive product, software, roadmap, and market claims must be dated and rechecked. Never convert an announcement into a shipping fact, a theoretical peak into delivered performance, or a vendor benchmark into a universal result.

---

## 3. Performance-result contract

Every performance number must state enough context to reproduce its meaning:

- workload/model, input distribution, precision, quality/accuracy target;
- hardware topology and relevant clocks/power mode;
- software, compiler, runtime, driver, and important configuration;
- batch/concurrency/parallelism and memory placement;
- metric definition and included/excluded stages;
- warm-up, run duration, repetitions, percentile/confidence treatment;
- baseline and whether it was tuned comparably;
- limiting resource and evidence supporting that classification.

Do not compare results if their quality, scope, workload, or metric boundary differs without explaining the normalization.

---

## 4. Simulator-result contract

A simulator chapter must trace:

1. source workload and inputs;
2. compiler/graph transformation and executable artifact;
3. loader/runtime initialization;
4. dynamic instructions, operators, memory requests, packets, or events;
5. timing/resource-contention model;
6. warm-up, sampling, and region-of-interest policy;
7. raw counters/events;
8. formulas and aggregation into final metrics;
9. calibration/validation targets;
10. error budget and model-validity boundary.

Trace-driven replay must state which feedback is broken. Analytical models must state which distributions or correlations they discard. Cycle accuracy is not automatic correctness; a detailed model with wrong inputs is still wrong.

---

## 5. Cross-layer AI-workload requirement

AI coverage in each architecture book must be architecture-specific:

- **CPU:** preprocessing, retrieval, orchestration, vector/matrix kernels, memory/NUMA behavior, accelerator feeding, and CPU-only inference.
- **GPU:** graph/runtime-to-kernel lowering, SIMT/tensor execution, HBM/KV behavior, batching, prefill/decode, multi-GPU communication, and profiler evidence.
- **NPU:** graph compilation, tiling/dataflow/scratchpad/DMA scheduling, precision/sparsity, dynamic/fallback behavior, serving queues, and multi-NPU execution.
- **SoC/chiplet:** complete request/data path across CPU, accelerator, memory, NoC, chiplet links, storage, NIC, and scale-out network, including SLO/tail/failure behavior.

Coverage must include both operator mapping and the complete serving or application path. A peak-TOPS/FLOPS comparison is insufficient.

### 5.1 AI-stack implementation-reconstruction contract

The implementation-reconstruction requirement applies above the chip as well as inside it. An AI chapter must make the software stack reconstructable when that stack is central to the topic. Source code is optional, but the reader must be able to derive the services, schemas, interfaces, state machines, policies, tests, and rollout plan. Cover:

1. **Model and data artifacts:** checkpoint/tensor format, manifest, integrity, version, shapes/layouts/precision, tokenizer or feature schema, and compatibility.
2. **Semantic graph and IR:** graph capture/import, intermediate-representation fields, side effects, aliasing, dynamic shapes/control flow, decomposition, and fallback boundaries.
3. **Compiler decisions:** legality conditions and order for fusion, layout, quantization, sparsity, tiling, parallelization, kernel selection/generation, memory planning, and autotuning.
4. **Executable contract:** code objects, engines, target/version requirements, shape profiles, constants/weights, resource metadata, relocation, and reproducible build identity.
5. **Runtime and driver boundary:** contexts, streams/queues/events, memory allocation/registration, submission, synchronization, faults, cancellation, reset, and completion visibility.
6. **Persistent and temporary state:** weights, activations, workspaces, key-value state, prefix cache, communication buffers, ownership, generation, lifetime, eviction, migration, and reclamation.
7. **Request scheduler:** request schema and state machine, admission proof, batching/chunking, fairness/deadlines, preemption, overload, backpressure, and terminal states.
8. **Distributed execution:** topology discovery, placement, parallel groups, collectives, state transfer, consistency, retry, failure membership, and recovery.
9. **Numerical and semantic correctness:** precision/rounding, quality target, deterministic boundary, differential references, metamorphic tests, and fallback equivalence.
10. **Observability:** stable identities across request, graph node, kernel/command, memory transfer, collective, and response; exact metrics, traces, logs, counters, and conservation checks.
11. **Security and multi-tenancy:** authentication/authorization boundary, memory isolation, quotas, model/data confidentiality, debug access, side-channel assumptions, and zeroization.
12. **Deployment and operations:** artifact registry, configuration schema, compatibility matrix, cold/warm start, canary, drain, rolling update, rollback, incident snapshot, capacity model, and disaster recovery.

The practical review asks the reader to produce an AI-stack component diagram, artifact/interface schemas, compiler and request state machines, memory-ownership ledger, scheduler pseudocode in prose, failure matrix, validation plan, telemetry schema, and deployment runbook. Describing a sequence as “the framework compiles the model” or “the server batches requests” without these decisions does not meet the standard.

---

## 6. Review rubric

Score each dimension from 0 to 3:

| Dimension | 0 | 1 | 2 | 3 |
|---|---|---|---|---|
| newcomer accessibility | unexplained jargon | partial glossary | definitions and prerequisites | concepts built progressively with contrasts/examples |
| mechanism | feature list | qualitative sketch | causal stages and state | cycle/event detail with invariants/failures |
| theory | absent | formula quoted | derived model with assumptions | bounds, sensitivity, limiting cases, failure region |
| implementation | absent | components named | concrete structures/flow | PPA/timing/software/verification trade-offs |
| reconstructability | no build path | block list only | contracts, state, interfaces, and sizing | complete policy/invariant/verification/bring-up specification |
| AI-stack implementation, when applicable | framework/server named | major layers listed | artifacts, IR/compiler/runtime, scheduler and state | distributed failure, telemetry, security, deployment and rollback are reconstructable |
| evidence | unsupported | references only | counters/experiments identified | reproducible chain, validation, uncertainty |
| worked reasoning | absent | toy arithmetic | decision-relevant example | multiple regimes/sensitivity and interpretation |
| research readiness | summary only | common questions | open problems/trade-offs | hypotheses, experimental design, generalization limits |

A substantive research-preparation chapter should reach at least level 2 in every dimension and level 3 in the dimensions central to its purpose.

---

## 7. Audit workflow

1. Inventory pages, ownership, cross-links, and prerequisites.
2. Build a layer ledger: for each layer, record its contract, state owner, interfaces, policies, invariants, physical cost, evidence, and build/verification gate.
3. Walk at least one request, model artifact, instruction, transaction, tensor tile, or packet through every layer. For an AI stack, include graph/IR transformation, executable selection, runtime submission, persistent state, serving policy, distributed handoff, and deployment generation. Record where work can wait, be transformed, be retried, fail, or be observed.
4. Identify missing mechanisms, theory, evidence, implementation choices, and first-use definitions.
5. Compare coverage against representative curricula, standards, seminal papers, current primary documentation, and research questions.
6. Expand the architecture-owned page rather than creating detached generic material.
7. Add or update worked derivations, failure modes, counters, simulator paths, implementation stages, and references.
8. Try to reconstruct a block diagram, interface/state specification, sizing worksheet, and verification plan from the text; log every decision that still requires guessing.
9. Validate links, anchors, fences, terminology, counts, and stale paths.
10. Record remaining gaps instead of hiding uncertainty.

This standard is a living review contract. It applies to architecture, RTL, verification, synthesis, backend, signoff, manufacturing, bring-up, and interview material.
