# CPU Simulation Methodology and Evidence — From Source to Defensible Results

> **First-time reader orientation:** A CPU simulator does not turn C source directly into a performance number. A compiler creates a target executable; a loader creates architectural state; functional execution creates dynamic instructions and addresses; timing models schedule those operations through CPU and memory resources; counters are reduced into metrics; validation determines what the result can support.

> **Abbreviation key — skim now and return as needed:** executable and linkable format (ELF); instruction set architecture (ISA); application binary interface (ABI); intermediate representation (IR); program counter (PC); operating system (OS); syscall emulation (SE); full-system simulation (FS); region of interest (ROI); instructions per cycle (IPC); cycles per instruction (CPI); misses per thousand instructions (MPKI); translation lookaside buffer (TLB); miss-status holding register (MSHR); reorder buffer (ROB); network on chip (NoC); dynamic random-access memory (DRAM); register-transfer level (RTL); hardware performance counter (HPC).

> **Prerequisite:** [CPU Workloads, Performance Modeling, and DSE](01_CPU_Workloads_Performance_and_DSE.md). **Hands off to:** [gem5](../08_Simulation/01_gem5.md) and [NoC and Coherence Simulation](../08_Simulation/02_NoC_and_Coherence_Simulation.md).

---

## 0. Choose the CPU model by the claim

| Model | Consumes | Resolves | Appropriate claim |
|---|---|---|---|
| analytical/interval | instruction/cache summaries and parameters | bounds and dominant terms | screening widths, latency, bandwidth, or capacity |
| functional ISA | executable or instruction stream | correct architectural state and dynamic path | software bring-up, instruction count, checkpoint generation |
| trace-driven timing | recorded dynamic instructions/addresses/dependencies | timing for a frozen path | repeatable cache/predictor/front-end studies within trace limits |
| execution-driven timing | executable plus functional semantics | path and timing feedback | branch, dependence, memory, synchronization, and system performance |
| RTL/emulation | hardware description and software | implemented cycle/control behavior | design verification and late performance correlation |

Fidelity means “contains the mechanism needed by the question,” not “has more settings.” A cycle model with the wrong branch predictor or memory latency is less credible than a calibrated analytical model for that specific decision.

## 1. Freeze the CPU experiment

Record the tuple

$$
X=(S,I,C,L,E,\mu,R),
$$

where $S$ is source revision, $I$ input, $C$ compiler/toolchain, $L$ linked libraries/runtime, $E$ execution environment, $\mu$ CPU/memory configuration, and $R$ ROI/warm-up/sampling recipe.

Two builds of the same algorithm can differ because optimization changes inlining/vectorization/unrolling, `-march` changes legal instructions, and link choices change startup/library code. Validation requires the same binary and input, not just the same source name.

## 2. Source becomes a target executable

The logical compilation path is

~~~text
source + headers
  -> preprocessing
  -> compiler IR and optimization
  -> target assembly
  -> assembler
  -> relocatable object files
  -> linker + libraries + startup code
  -> target ELF executable
~~~

The compiler changes the program structure:

- dead-code elimination removes instructions;
- inlining removes calls but expands code;
- unrolling trades branches for footprint;
- vectorization replaces scalar iterations with vector instructions;
- alias analysis permits or prevents load/store motion;
- instruction selection uses the enabled ISA extensions.

The assembler encodes instructions and relocation records. The linker resolves symbols, lays out code/data, applies relocations, adds startup/runtime objects, and emits ELF program headers. Program headers tell the loader which segments to map and with what permissions.

A disassembly shows **static instructions**—one line per machine-code address. A loop creates a new **dynamic instruction** every iteration:

$$
N_{dyn}=\sum_b N_bF_b,
$$

where $N_b$ is static instructions in basic block $b$ and $F_b$ its runtime frequency.

### 2.1 The input dataset does not become assembly

Source becomes instructions; input remains data. Arguments and environment strings begin on the stack. File/input bytes enter through system calls and are copied into simulated memory. Values then choose branches, loop counts, indices, addresses, and synchronization behavior. The same ELF can therefore create different dynamic instruction and cache streams for different datasets.

## 3. Loading establishes the initial machine state

In user-mode/SE simulation, a loader generally:

1. validates ELF ISA and ABI;
2. maps code/data segments into a virtual address space;
3. creates stack/heap regions;
4. places arguments, environment, and auxiliary data;
5. initializes PC, stack pointer, and runtime state;
6. services system calls through simulator emulation.

Application instructions execute, but guest-kernel instructions do not. In FS simulation, firmware and kernel boot, the guest OS creates page tables, handles faults/interrupts, loads the process, maps libraries, schedules threads, and executes kernel paths. Use SE for controlled user-code studies; use FS for OS, devices, virtualization, scheduling, page faults, and kernel-heavy workloads.

Address layout affects I-cache/TLB indexing and physical bank mapping. Control randomization or report it.

## 4. Functional execution creates dynamic instructions

For each PC, the ISA decoder creates an immutable static description of opcode, operands, immediates, and semantics. Every execution creates a dynamic record with sequence number, predicted/actual control flow, renamed operands, effective address, fault state, and stage timing.

An out-of-order dynamic instruction typically:

1. is fetched from predicted PC through instruction translation/cache;
2. decodes, possibly into micro-operations;
3. renames destinations and allocates ROB/IQ/LSQ/PRF resources;
4. waits for operands and an execution port;
5. executes semantics or sends a timing memory request;
6. writes back, wakes dependents, and commits in order;
7. is squashed if an older fault, branch misprediction, or ordering violation redirects execution.

Functional semantics determine values and correct control flow. Timing resources determine when each action may occur. Architectural instructions, micro-operations, issued operations, and committed operations are distinct counter domains.

## 5. One CPU load becomes a hierarchy of events

~~~text
dynamic load
 -> effective virtual address
 -> TLB lookup or page walk
 -> physical address
 -> L1 data-cache lookup
 -> lower cache / coherence transaction on miss
 -> NoC messages and directory/home lookup
 -> LLC or DRAM controller request
 -> DRAM commands / possible owner-cache response
 -> refill and load completion
 -> dependent wakeup
 -> eventual in-order retirement
~~~

At each arrow, a bounded queue or port may reject progress. A load can overlap with independent instructions; an MSHR may fill; coherence may add dependent messages; DRAM scheduling may reorder service. Cycle count emerges from event ordering and dependencies, not from attaching one fixed latency to the assembly instruction.

## 6. Event-driven timing

A simplified cycle/event loop is

~~~text
retire oldest completed instructions
process execution and memory responses due now
wake dependents
select ready operations subject to ports/units
rename and dispatch if queues/registers have capacity
fetch and decode unless redirected or starved
advance cache, coherence, NoC, and DRAM events
update primitive counters
~~~

Event-driven implementations can skip empty time, but causality must be identical. Resource capacity creates backpressure; latency schedules completion; initiation interval controls how often a pipelined unit accepts new work.

### 6.1 The $(L,II,P)$ cost record for a CPU resource

For an operation class, record latency $L$, initiation interval $II$, and parallel unit/port count $P$:

- $L$: cycles from issue until a result can wake a dependent operation;
- $II$: cycles before the same pipeline can accept another independent operation;
- $P$: equivalent pipelines or ports that can operate concurrently.

A four-cycle-latency fully pipelined adder has $(L,II)=(4,1)$: dependent adds are four cycles apart, but independent adds can start every cycle. With two identical pipelines, peak acceptance is $P/II=2$ operations/cycle, subject to issue width, operand ports, dependencies, and writeback. A nonpipelined divider with $(20,20,1)$ both delays its consumer and blocks later divides. The simulator schedules readiness at `issue_time + L`, reserves capacity for $II$, and lets dependencies/resources determine actual timing.

Memory has the same form but dynamic latency: an L1 hit may schedule a response after a configured $L$, while a miss allocates an MSHR and creates dependent cache/NoC/DRAM events whose queueing determines completion. Modeling a miss as one constant erases overlap, backpressure, and contention.

### 6.2 Event ordering, queueing, and host cost

A discrete-event implementation stores timestamped completions in a priority queue and repeatedly processes the earliest. Same-time events need a deterministic phase/priority order for writeback, wakeup, squash, and issue. No consequence may be scheduled before its cause. Cycle-stepped and event-skipping implementations are equivalent only if they preserve those ordering rules.

If a finite resource receives requests at rate $\lambda$ and serves at rate $\mu$, its utilization is $\rho=\lambda/\mu$. A simple M/M/1 approximation gives residence time $1/(\mu-\lambda)$; a detailed simulator produces the analogous wait by serializing grants rather than adding a “contention penalty.” Removing the queue or giving it infinite service structurally removes the saturation knee.

Simulator host work per target cycle grows roughly as

$$
w\propto n_{event}i_{event},
$$

where $n_{event}$ is modeled events per target cycle and $i_{event}$ is host work per event. O3 scheduling, wrong-path execution, cache probes, coherence transients, and DRAM commands raise the first term; complex dynamic data structures raise the second. This explains why fast-forward, checkpoints, sampling, traces, and selective detail attack different costs.

### 6.3 Trace-driven versus execution-driven CPU studies

A dynamic trace may record PC, opcode, branches, addresses, and dependencies. It is fast and repeatable, but freezes some behavior. A different cache timing can change thread interleaving, spin counts, prefetch timeliness, system calls, and sometimes addresses. Use traces when the path is stable and the studied component does not meaningfully feed back into it; use execution-driven simulation when timing changes future work.

Always name the trace point. A CPU load/store trace, L1-miss trace, LLC-miss trace, and DRAM-command trace describe different work.

## 7. Measurement phases and checkpoints

A defensible run separates:

1. initialization/boot/input loading;
2. functional fast-forward;
3. detailed warm-up of caches, TLBs, predictor, coherence, and queues;
4. statistics reset and measured ROI;
5. drain, exit-cause check, and output verification.

Resetting counters should not reset warm state. A functional checkpoint may preserve registers/memory without reconstructing every detailed microarchitectural structure, so warm-up remains necessary. The ROI end condition must align numerator and denominator: if cycles include drain but retired instructions do not, IPC is biased.

## 8. Raw counters become CPU metrics

Keep primitive counts and derived metrics separate:

$$
\text{IPC}=\frac{N_{retired}}{C},\qquad
\text{CPI}=\frac{C}{N_{retired}},\qquad
\text{MPKI}=1000\frac{N_{miss}}{N_{retired}},
$$

$$
T=\frac{C}{f},\qquad
\text{bandwidth}=\frac{\sum_i B_i}{T}.
$$

Define scope. Per-core IPC uses one core's instructions and active cycles; total instructions divided by one global cycle count is chip throughput. Define bytes as requested, cache-line, coherence, or DRAM bytes.

For repeated dumps, select the intended snapshot or subtract start counters from end counters. Do not silently parse the first matching name.

## 9. Sampling, aggregation, and uncertainty

For fixed-instruction samples representing instruction fractions,

$$
\widehat{\text{CPI}}=\sum_iw_i\text{CPI}_i,\qquad \widehat{\text{IPC}}=1/\widehat{\text{CPI}}.
$$

Across programs, calculate equivalent-work speedup per program, then use geometric mean for ratios. Report per-program results and confidence intervals. Separate random run-to-run variability from systematic warm-up, configuration, and model-form errors.

For an approximately independent interval metric with mean $\mu$, standard deviation $\sigma$, coefficient of variation $c_v=\sigma/\mu$, desired relative half-width $\varepsilon$, and normal critical value $z$ (about 1.96 for 95%), a first sample-count estimate is

$$
n\gtrsim\left(\frac{z c_v}{\varepsilon}\right)^2.
$$

Autocorrelated intervals reduce effective sample size, so block, phase-select, or space samples farther apart and verify convergence. A statistical confidence interval covers random sampling variation under its assumptions; it does not include a wrong binary, unrepresentative phase selection, or cold-state bias.

Warm-up is a separate timescale. Checkpoints may preserve architectural state while predictor/cache/TLB/queue state is cold or stale. Measure each structure's convergence and warm long-lived state functionally where possible before a shorter detailed pipeline/queue warm-up. More measured samples do not average away the same cold-start bias repeated at every sample.

## 10. Correctness, conservation, and validation

Before trusting performance:

- verify exit cause, output/checksum, ROI marker counts, and no early instruction limit;
- check retired instructions against functional/reference execution;
- reconcile cache accesses = hits + misses with documented exceptions;
- reconcile request/response and injected/delivered bytes plus in-flight work;
- ensure IPC cannot exceed modeled width indefinitely;
- ensure cycles/frequency agrees with simulated seconds;
- compare key counters against hardware or RTL using the same binary/input.

Validation is hierarchical:

1. instruction count and output;
2. branch counts/mispredictions;
3. cache/TLB accesses/misses;
4. memory traffic/latency;
5. cycles/IPC;
6. power/activity if used for PPA.

Matching total runtime for the wrong reasons is not validation. Compare intermediate mechanisms and reserve absolute claims for calibrated configurations; relative trends may be credible over a wider neighborhood.

### 10.1 CPU error budget and decision resolution

Separate four error sources because they require different fixes:

- **workload error:** the binary, input, phase weights, or software stack does not represent deployment;
- **sampling error:** finite sampled intervals vary statistically or omit rare phases;
- **parameter error:** cache latency, predictor table timing, DRAM timing, or clock is misconfigured;
- **model-form error:** the simulator omits a CPU mechanism such as wrong-path cache pollution, speculative memory replay, prefetch throttling, voltage droop, or OS interference.

Repeated seeds reduce random sampling error; they do not remove a cold-cache bias or an omitted replay path. Calibration can tune parameters inside a model form; it cannot make a frozen committed-instruction trace reproduce wrong-path fetches. Report systematic and random components separately.

When independent relative errors from multiple stages feed a final quantity, a first-order root-sum-square estimate is

$$
\sigma_{total}\approx\sqrt{\sum_i\sigma_i^2}.
$$

Shared or correlated bias does not cancel this way and can approach linear addition. For example, a traffic-count error can inflate both simulated delay and the activity passed to a power model. Co-validate the composed performance→power result rather than treating independently validated components as automatically correct together.

For design-space exploration, absolute error and ranking error are different. A model that predicts every configuration 10% slow can preserve ranking, while a lower mean error with configuration-dependent scatter can reverse close choices. If per-configuration random error has standard deviation $s$, two independent design means generally need a true separation on the order of $2.8s$ for a rough 95% pairwise distinction. Treat closer points as unresolved, increase samples/fidelity, or make the decision using PPA and risk rather than false precision.

## 11. CPU and RISC-V tool selection

| Tool family | Best use | Important boundary |
|---|---|---|
| Spike/QEMU/fast ISA models | functional execution, boot, checkpoints | little or no detailed timing |
| gem5 | configurable core/cache/full-system timing | configuration and validation determine credibility |
| ChampSim | trace-driven cache/prefetch/branch studies | dynamic trace path is fixed |
| Sniper/zSim | faster many-core exploration | simplified timing/communication mechanisms |
| SST | component-scale/many-node simulation | model integration and traffic definitions |
| Verilator/FPGA/emulation | RTL behavior and software | much slower or later, but implementation-specific |

For an open RISC-V core such as XiangShan, combine ISA reference/differential testing for correctness, RTL simulation/emulation for implementation behavior, and an architecture model for broad DSE. Correlate instruction mix, branch, cache, and cycle counters at shared ROIs.

## 12. Result provenance package

Preserve:

1. source/input hashes and compiler command;
2. ELF hash, headers, and disassembly;
3. simulator revision and full resolved configuration;
4. command line, environment, checkpoint, ROI, warm-up, and sample weights;
5. raw unedited logs/counter snapshots;
6. post-processing version and formulas;
7. derived table with units/denominators;
8. validation evidence, error bars, and known omissions.

Another reader should be able to walk backward from a plotted point to raw counters, run configuration, executable, compiler, input, and source.

## Cross-references

- [gem5](../08_Simulation/01_gem5.md) gives concrete CPU objects, output files, switching, checkpoints, and calibration.
- [NoC and Coherence Simulation](../08_Simulation/02_NoC_and_Coherence_Simulation.md) follows misses into protocol messages and flits.
- [CPU PPA and Physical Implementation](02_CPU_PPA_and_Physical_Implementation.md) checks whether assumed latencies/frequency are realizable.

## References

1. gem5 Project documentation and J. Lowe-Power et al., “The gem5 Simulator: Version 20.0+.”
2. N. Binkert et al., “The gem5 Simulator,” ACM SIGARCH Computer Architecture News, 2011.
3. T. Sherwood et al., “Automatically Characterizing Large Scale Program Behavior,” ASPLOS 2002.
4. T. E. Carlson et al., “Sniper,” SC 2011.

---

← [CPU PPA and Physical Implementation](02_CPU_PPA_and_Physical_Implementation.md) · [CPU book index](../00_Index.md) · next → [Core Foundations](../01_Core_Foundations/00_Index.md)
