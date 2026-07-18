# Benchmark to Results — The Exact End-to-End Architecture Simulation Workflow

> **First-time reader orientation:** A simulator does not read a C program and somehow “predict performance.” A toolchain first turns source code into a target-specific artifact: a CPU executable, GPU machine code or trace, or an NPU operator graph and mapping. The simulator consumes that artifact, creates a dynamic stream of work, applies timing and resource rules, accumulates raw counters, and only then reduces those counters into latency, throughput, energy, or speedup. This chapter follows every boundary in that chain.

> **Abbreviation key — skim now and return as needed:** central processing unit (CPU); application binary interface (ABI); executable and linkable format (ELF); instruction set architecture (ISA); intermediate representation (IR); region of interest (ROI); operating system (OS); operations per second (OPS); instructions per cycle (IPC); cycles per instruction (CPI); misses per thousand instructions (MPKI); translation lookaside buffer (TLB); last-level cache (LLC); miss-status holding register (MSHR, an entry that tracks an outstanding cache miss); reorder buffer (ROB, the CPU queue that preserves in-order retirement); physical address (PA); virtual address (VA); single instruction, multiple data (SIMD); graphics processing unit (GPU); CUDA programming platform; parallel thread execution (PTX); NVIDIA native GPU machine code/disassembly (commonly called SASS); streaming multiprocessor (SM); high-bandwidth memory (HBM); neural processing unit (NPU); Open Neural Network Exchange (ONNX); general matrix multiplication (GEMM); key-value (KV) cache; direct memory access (DMA); multiply-accumulate (MAC); static random-access memory (SRAM); dynamic random-access memory (DRAM); network-on-chip (NoC); comma-separated values (CSV); power, performance, and area (PPA).

> **Prerequisites:** [Simulation Methodology](01_Simulation_Methodology.md) for functional, timing, trace-driven, execution-driven, warm-up, and sampling concepts; [Architecture Primer](../01_Reader_Foundations/01_Architecture_Primer_and_Glossary.md) if ISA, cache, or virtual memory is unfamiliar.
> **Hands off to:** [gem5](../../01_CPU_Architecture/08_Simulation/01_gem5.md), [GPU Simulators](../../02_GPU_Architecture/04_Simulation/01_GPU_Simulators.md), [NPU Simulators](../../03_NPU_Architecture/04_Simulation/01_Accelerator_and_NPU_Simulators.md), and [DRAM Simulators](../../04_SoC_and_Chiplet_Architecture/06_Simulation/01_DRAM_Simulators.md).

---

## 0. The simulator begins at an abstraction boundary

Every simulation has an **input artifact**. The artifact fixes what the simulator knows and what earlier tools have already decided.

| Study | Artifact actually consumed | Decisions already embedded |
|---|---|---|
| CPU execution-driven | target ELF executable, arguments, environment; optionally kernel/disk image | compiler optimization, ISA, ABI, library code, instruction selection |
| CPU trace-driven | recorded instruction/address trace | dynamic path, addresses, often dependencies; sometimes timing of the capture machine |
| GPU PTX execution-driven | PTX in a CUDA fat binary plus launch/runtime calls | high-level CUDA lowering, but not final physical-register allocation or SASS scheduling |
| GPU SASS trace-driven | dynamic SASS trace with active masks and addresses | final machine instructions and the path observed on the capture GPU |
| NPU analytical/cycle model | operator graph or layer/GEMM descriptors plus mapping and hardware configuration | graph optimization, shape/layout/precision choices, operator decomposition |
| DRAM standalone | physical request trace or command trace | everything above the trace point: program path, translation, cache filtering, sometimes arrival timing |

The simulator cannot recover information discarded before that boundary. A DRAM trace containing only last-level-cache misses cannot tell you how many L1 hits occurred. A PTX simulation cannot recreate the exact SASS instruction schedule of an undocumented compiler backend. An ONNX layer list cannot reveal CPU preprocessing overhead unless that work is modeled separately.

~~~mermaid
flowchart LR
    S["benchmark source + input"] --> C["compiler / graph exporter"]
    C --> A["ELF, PTX/SASS, ONNX, or trace"]
    A --> F["functional execution or trace reader"]
    F --> D["dynamic instructions / operators / requests"]
    D --> T["timing + resource model"]
    T --> E["timestamped events and state transitions"]
    E --> R["raw counters"]
    R --> M["metrics: cycles, IPC, bandwidth, energy"]
    M --> V["validation + uncertainty + final comparison"]
~~~

The most important audit question is therefore: **what exact artifact crossed each arrow?**

## 1. Freeze the experiment before compiling

A benchmark name is not a complete workload. “Standard Performance Evaluation Corporation (SPEC) CPU,” “ResNet,” or “Rodinia breadth-first search (BFS)” leaves many degrees of freedom that change the dynamic work.

Record at least:

- source revision and any patches;
- input dataset and command-line arguments;
- compiler name/version, optimization flags, target ISA extensions, ABI, and link mode;
- library versions and whether optimized libraries replace source kernels;
- thread count, affinity, environment variables, random seeds, and numerical precision;
- simulator revision, machine configuration, clock domains, memory configuration, and coherence protocol;
- ROI markers, fast-forward length, warm-up length, measurement length, and sample weights.

The reproducible experiment is the tuple

$$
X=(S,I,C,L,E,\mu,R),
$$

where $S$ is source, $I$ input, $C$ compiler/toolchain, $L$ linked libraries/runtime, $E$ execution environment, $\mu$ microarchitecture configuration, and $R$ measurement recipe. Changing any component creates a different experiment.

Two builds from the same source can have different instruction counts because `-O0` (no optimization) preserves debugging structure, `-O3` (aggressive optimization) inlines/vectorizes/unrolls, link-time optimization crosses source-file boundaries, and a different `-march` enables different instructions. This is why simulator validation requires the **same binary**, not merely the same algorithm.

## 2. CPU path: source code becomes a target ELF executable

Consider C/C++ compiled for a RISC-V or Arm target. The visible command may be one compiler invocation, but the logical stages are:

~~~text
source + headers
  -> preprocessing
  -> compiler intermediate representation
  -> optimization passes
  -> target assembly
  -> assembler
  -> relocatable object files
  -> linker + libraries + startup code
  -> ELF executable
~~~

### 2.1 Preprocessing and compiler IR

Preprocessing expands included headers and macros and selects conditional code. The compiler parses the result into an IR such as LLVM IR or the GNU Compiler Collection's (GCC's) internal forms. Optimization then transforms the *program*, not just individual statements:

- constant propagation/dead-code elimination remove work;
- inlining changes call/return frequency and instruction-cache footprint;
- loop unrolling trades fewer branches for more code bytes;
- vectorization replaces scalar iterations with SIMD/vector operations;
- alias analysis decides whether loads/stores may be reordered;
- instruction selection maps IR operations to the target ISA.

The source loop is therefore not a one-to-one list of instructions. One line may disappear, become several instructions, move outside a loop, or turn into one vector instruction.

### 2.2 Assembly, objects, and linking

The backend emits target assembly. The assembler encodes each assembly instruction into bytes and creates a relocatable object containing code, data, symbols, and relocation records. The linker:

1. resolves symbol references across objects and libraries;
2. lays out code/data sections at virtual addresses;
3. applies relocations;
4. adds startup code and metadata;
5. emits an ELF executable or shared object.

ELF is a container, not “the assembly.” Sections help link/debug; **program headers** tell the loader which byte ranges to map into memory and with which read/write/execute permissions. The executable also records its target ISA/ABI and entry point.

A disassembler such as `objdump -d` reverses instruction bytes into readable mnemonics. It is essential for confirming the actual ISA, but it still shows **static instructions**—each address once—not how often loops execute.

### 2.3 Static versus dynamic instruction count

Suppose a loop body contains eight static machine instructions and iterates one million times. Its dominant dynamic count is roughly eight million, plus setup, remainder, calls, and runtime code. Formally,

$$
N_{dyn}=\sum_{b\in\text{basic blocks}} N_b\,F_b,
$$

where $N_b$ is the number of static instructions in basic block $b$ and $F_b$ is its dynamic execution frequency. The simulator discovers $F_b$ by execution or reads it from a trace.

An ISA instruction may also decode into several internal micro-operations (µops). Keep the denominators separate:

- architectural IPC uses retired ISA instructions;
- backend occupancy may count µops;
- a fused instruction may retire as two ISA instructions but occupy one backend entry, depending on the model.

Mixing `simInsts`, decoded µops, and issued operations produces plausible but meaningless IPC.

### 2.4 Worked lowering: one source loop becomes a dynamic instruction stream

Consider a scalar reduction:

```c
long sum(const int *a, long n) {
    long s = 0;
    for (long i = 0; i < n; ++i)
        s += a[i];
    return s;
}
```

An optimized 64-bit RISC-V (RV64) scalar build could produce assembly shaped like this (illustrative; exact registers/instructions depend on compiler, flags, alignment, alias knowledge, and enabled vector extensions):

```asm
sum:
    li      a5, 0          # accumulator s = 0
    beqz    a1, .done      # if n == 0, skip loop
.loop:
    lw      a4, 0(a0)      # load signed 32-bit a[i]
    addi    a0, a0, 4      # advance pointer
    add     a5, a5, a4     # s += a[i]
    addi    a1, a1, -1     # decrement remaining count
    bnez    a1, .loop      # choose next PC
.done:
    mv      a0, a5         # ABI return register
    ret
```

The loop contains five static machine instructions but creates about $5n$ dynamic instructions. For $n=10^6$, that is roughly five million loop instructions plus entry/exit. The `lw` produces a virtual address and may hit or miss caches; `add a5,a5,a4` creates a loop-carried dependence because iteration $i+1$ needs the prior accumulator; the pointer/counter updates can overlap; the branch creates prediction and redirect behavior. The timing simulator schedules these dependences and resources—it does not assign one source-line latency.

With vectorization enabled, the compiler may replace many scalar iterations with vector loads/reductions plus a scalar remainder. That changes dynamic instruction count, cache request widths, register pressure, and achievable IPC. Both runs still execute “the same C function,” which is precisely why the binary/disassembly belongs in simulation provenance.

### 2.5 Static and dynamic linking change the simulated workload

A statically linked executable contains most library code in the ELF and is easier for syscall-emulation environments. A dynamically linked executable invokes a loader and shared libraries at runtime. If the simulator lacks the correct target loader/library paths, it may fail or silently use a different environment.

Full-system simulation naturally runs the guest's dynamic loader and operating system. Syscall-emulation mode often intercepts system calls in the simulator; the application instructions execute, but kernel instructions do not. Thus the same user binary can have different total instruction counts in syscall-emulation and full-system modes even when its numerical output matches.

## 3. Loading: bytes become initial architectural state

Before the first benchmark instruction executes, a loader creates the initial machine state.

For a user-mode CPU simulation, the loader generally:

1. validates the ELF ISA and ABI;
2. maps loadable code/data segments into the simulated virtual address space;
3. allocates stack and heap regions;
4. places arguments, environment strings, and auxiliary vectors on the stack;
5. initializes the program counter to the ELF entry point and the stack pointer to the new stack;
6. loads or emulates required runtime support.

Startup code runs before `main`: it initializes the language/runtime, invokes constructors, and eventually calls the benchmark entry point. If statistics start at process entry, all of that work is included.

In full-system mode, the initial artifact is instead firmware/bootloader plus kernel and disk image. Firmware configures the virtual platform, the kernel initializes page tables and devices, the OS starts the process, and only then does the same ELF loading occur inside the guest. This is slower but includes kernel execution, scheduling, interrupts, page faults, and device behavior.

Address-space layout also matters. Randomized or different code/data placement can change instruction-cache indexing, TLB behavior, and physical DRAM-bank mapping. Reproducible studies either control it or report it.

### 3.1 Source becomes instructions; benchmark input remains data

The compiler translates the benchmark's **program source** into instructions. It normally does not translate the input dataset into assembly. Command-line strings are placed in the initial stack; compiled-in constants occupy ELF data sections; file or standard-input bytes are obtained later through system calls and copied into the program's simulated memory. In syscall-emulation mode, the simulator services those calls using its emulation layer; in full-system mode, the guest kernel and device/filesystem stack execute them.

The loaded values then change the dynamic stream: a graph input can select a branch, a matrix size controls loop trip counts, a pointer index changes cache addresses, and compressed/sparse data changes how much work is skipped. Thus the same ELF with two datasets has the same static disassembly but can produce different dynamic instruction counts, memory requests, and cycle totals. Preserve the input files or their hashes, not just the executable.

## 4. Functional execution creates the dynamic instruction stream

At architectural level, one CPU step is:

1. use the program counter (PC) as the virtual fetch address;
2. translate/fetch instruction bytes;
3. decode bytes according to the selected ISA;
4. read source registers and compute the instruction's functional result;
5. read or write memory if required;
6. choose the next PC;
7. repeat until exit or an ROI event.

A **static instruction object** describes the instruction at one PC: opcode, source/destination classes, immediates, and semantic function. A **dynamic instruction object** is one particular execution of it, with sequence number, actual operands, predicted/actual branch result, effective address, and timing state. A loop reuses the static instruction but creates a new dynamic instance each iteration.

Functional execution answers values and control flow. The timing model answers when those effects may occur. The two must agree on architectural state, but they need not perform the same work internally. A trace-driven simulator skips most functional calculation because the trace already supplies the dynamic opcode, address, and control metadata.

## 5. The timing model turns dynamic work into cycles

An out-of-order CPU timing model does not simply add “four cycles per instruction.” It inserts each dynamic operation into modeled structures and lets constraints determine progress.

~~~text
for each simulated cycle or event time:
    retire oldest completed instructions, in order
    complete execution and memory responses due now
    wake dependent operations whose inputs became ready
    select ready operations subject to issue ports and unit availability
    dispatch/rename new decoded operations if ROB, queues, and registers have space
    fetch/decode more instructions unless redirected or frontend-stalled
    advance cache, coherence, network, and memory-controller events
    increment counters for every accepted, stalled, completed, and retired action
~~~

Order inside a particular simulator may differ, and event-driven implementations skip empty time, but the dependency rules are equivalent. Each operation carries latency $L$, initiation interval $II$, compatible ports, and dependencies. Queue limits create backpressure. Branch misprediction squashes younger dynamic instructions; cache misses occupy miss-status entries; store/load ordering may trigger replay.

The final cycle count is the first time at which the measured region's required work has retired or completed. It is an **emergent schedule**:

$$
C_{measured}=C_{base}+C_{frontend}+C_{dependencies}+C_{memory}+C_{contention}+C_{recovery}-C_{overlap}.
$$

These terms overlap and normally cannot be added from independent averages. The simulator's event ordering is what accounts for the overlap.

## 6. One load instruction becomes a system-wide event chain

A dynamic load illustrates how the component models connect:

~~~mermaid
flowchart LR
    L["load executes<br/>virtual address"] --> T["TLB lookup"]
    T -->|miss| W["page-table walk"]
    T --> P["physical address"]
    W --> P
    P --> C1["L1 cache tags/MSHR"]
    C1 -->|miss| COH["coherence request"]
    COH --> N["NoC packets/flits"]
    N --> LLC["shared cache/directory"]
    LLC -->|miss| MC["memory-controller queue"]
    MC --> D["DRAM commands/banks"]
    D --> RET["data response reverses path"]
    RET --> WB["register writeback + dependent wakeup"]
~~~

At every arrow, the simulator may allocate a queue entry, schedule an event, or stall because a bounded resource is full. A cache hit terminates the chain early. A last-level miss produces physical memory traffic. Coherence can source data from another cache instead of DRAM.

This chain explains trace placement:

- an instruction trace starts before address translation and caches;
- an L1-miss trace omits L1 hits but still exercises lower caches;
- an LLC-miss trace is appropriate for a standalone DRAM study;
- a DRAM command trace begins after request scheduling and is suitable for power/state analysis, not scheduler comparison.

Always document the trace point. “Memory trace” is otherwise ambiguous by several abstraction levels.

## 7. GPU path: CUDA source becomes PTX, cubin/SASS, and warps

A CUDA source file contains host C++ and device kernels. The CUDA compiler separates them:

1. host code is compiled for the CPU;
2. device code is lowered to PTX, NVIDIA's virtual ISA;
3. `ptxas` can translate PTX into a target-specific CUDA binary (**cubin**) containing SASS machine instructions;
4. PTX and one or more cubins are packed into a **fat binary** embedded in the host executable;
5. at runtime, the driver chooses a compatible cubin or just-in-time compiles PTX.

The kernel launch supplies grid dimensions, block dimensions, arguments, shared-memory allocation, and stream ordering. The simulator/runtime expands each block into warps, typically groups of 32 threads. One static SASS instruction can therefore generate many dynamic **warp instructions**, each carrying an active-lane mask; memory instructions additionally carry per-lane addresses that the coalescer combines into transactions.

Before launch, host code usually reads the dataset, allocates device memory, and copies tensors/arrays to it. A kernel-only simulation begins after some or all of that work, so its cycle count is not automatically application end-to-end latency. The run definition must say whether host preprocessing, transfers, launches, synchronization, and result copies are modeled or measured separately.

Two common simulator paths consume different artifacts:

- **PTX execution-driven:** functionally execute PTX threads, derive values/branches/addresses, then time the warp operations. It can run without a real target GPU but misses final SASS scheduling, register allocation, spills, and machine idioms.
- **SASS trace-driven:** instrument a real GPU run, record dynamic SASS opcodes, masks and addresses, then replay them through the timing model. It preserves the final instruction stream but freezes timing-dependent behavior from the capture run.

The cycle model schedules thread blocks onto SMs subject to registers, shared memory, block slots, and warp slots. Warp schedulers choose eligible warps; operand collectors, execution pipelines, caches, interconnect, and HBM apply backpressure. Kernel completion occurs when every block has completed and required stream dependencies are satisfied.

## 8. NPU path: a framework graph becomes operators, loop nests, and mappings

Most NPU architecture simulators do **not** consume assembly. Their boundary is higher.

~~~text
PyTorch / TensorFlow / JAX model
  -> exported graph (for example ONNX)
  -> shape inference + graph optimization
  -> fusion, layout, precision, quantization decisions
  -> operator list
  -> convolution/GEMM/attention loop dimensions
  -> tile + dataflow mapping
  -> analytical counts or cycle-level array schedule
~~~

An ONNX model stores a directed dataflow graph: operator nodes, tensor inputs/outputs, constant initializers such as weights, shapes/types, and operator-set versions. A frontend may fold constants, remove dead nodes, fuse patterns, choose layouts, or replace high-level attention with several primitive operators. Those passes change the simulated operator list.

Many NPU performance simulators use tensor **shapes and datatypes**, not every numerical input value. That is sufficient for dense, shape-determined work, but not for data-dependent sparsity, conditional execution, variable sequence lengths, mixture-of-experts routing, compression ratio, or early exit. Those studies need representative value-derived metadata or an execution/frontend model that generates the actual active operators and nonzero patterns.

The architecture simulator then lowers each supported operator to a workload description. Examples:

- matrix multiply → dimensions $M,N,K$;
- convolution → batch, channels, spatial dimensions, filter, stride and padding, or an equivalent GEMM;
- attention → projection GEMMs, score GEMM, softmax/reduction, value GEMM, and KV-cache traffic;
- elementwise/reduction → vector-engine operations and tensor bytes.

A mapping assigns loop tiles and order to the processing-element array and memory hierarchy. Analytical tools calculate MAC/access counts directly. Cycle models generate per-cycle operand demand and stalls. A production NPU compiler may go one step further and emit descriptors or micro-commands for DMA, matrix, vector, and synchronization engines; a layer-level simulator often bypasses that binary command encoding and models the same schedule from the mapping.

That distinction must be stated. “The NPU simulator ran ResNet” may mean it imported an optimized ONNX graph, or merely simulated a hand-written CSV of convolution shapes. Those are different coverage claims.

## 9. Measurement has five phases, not one Run button

A trustworthy detailed run separates:

1. **initialization:** boot, input loading, allocation, model construction;
2. **fast-forward:** execute cheaply to reach the desired program phase;
3. **warm-up:** establish representative cache, TLB, predictor, queues, and memory state;
4. **measurement ROI:** reset counters, run the selected work, then dump counters;
5. **drain/verification:** finish outstanding work as required and validate program output.

The usual sequence is

~~~text
load checkpoint or start program
fast-forward to pre-ROI marker
warm microarchitectural state
reset statistics
run fixed instruction count / work markers / kernel set
dump statistics
verify exit cause and output checksum
~~~

Resetting counters does not necessarily reset modeled state. That is desirable: the ROI begins with warm state but zero measured counts. Conversely, restoring a functional checkpoint may restore registers/memory without reconstructing every cache/predictor entry, which is why a warm-up interval remains necessary.

The ROI end condition must match the numerator. If cycles continue during post-ROI draining but retired instructions stop counting, IPC is biased low. If asynchronous GPU/NPU commands are launched but not synchronized before the marker, the host ROI may end before device work completes.

## 10. Raw counters become final metrics

Keep raw counts and derived metrics separate. The simulator should accumulate primitive events; a post-processing script or formula statistic reduces them with explicit denominators.

### 10.1 CPU metrics

$$
\text{IPC}=\frac{N_{retired}}{C},\qquad
\text{CPI}=\frac{C}{N_{retired}},\qquad
\text{MPKI}=1000\frac{N_{miss}}{N_{retired}},
$$

$$
T=\frac{C}{f},\qquad
\text{bandwidth}=\frac{B_{useful\ or\ transferred}}{T}.
$$

Specify whether bytes mean program-requested bytes, cache-line bytes, or DRAM-burst bytes. All three are legitimate and different.

Also specify reporting scope. Per-core IPC uses that core's retired instructions and active cycles; aggregate throughput may sum instructions across cores. A counter summed across cores divided by one core's cycle count is a chip throughput metric, not per-core IPC.

### 10.2 GPU metrics

$$
\text{warp IPC}=\frac{N_{warp\ issue\ or\ retire}}{C_{SM}},\qquad
\text{lane efficiency}=\frac{N_{active\ lane\ instructions}}{W\,N_{warp\ instructions}},
$$

$$
\text{occupancy}=\frac{\overline{N_{resident\ warps}}}{N_{warp\ slots}},\qquad
\text{HBM bandwidth}=\frac{B_{post-coalescing}}{C/f}.
$$

Warp issue count, thread/lane instruction count, and ISA instruction count are different. Name which one the tool reports.

Likewise, state whether warp IPC is per SM or summed across the GPU. For a per-SM value, use that SM's warp count and active cycles; for a chip aggregate, sum consistently across SMs and label it aggregate issue/retirement throughput.

### 10.3 NPU metrics

$$
\text{array utilization}=\frac{N_{useful\ MAC}}{P_{MAC}\,C},\qquad
\text{throughput}=\frac{N_{operations}}{C/f},
$$

$$
E=\sum_i N_i e_i,\qquad
P_{avg}=\frac{E}{C/f},
$$

where $N_i$ is an event count (MAC, SRAM read, NoC hop, DRAM access) and $e_i$ its energy. State whether one MAC counts as one operation or two arithmetic operations.

### 10.4 Memory metrics

For request $i$ with arrival $a_i$, completion $d_i$, and $B_i$ transferred bytes,

$$
L_i=d_i-a_i,\qquad
\bar L=\frac{1}{N}\sum_i L_i,\qquad
\text{bandwidth}=\frac{\sum_i B_i}{(t_{end}-t_{start})/f}.
$$

Report percentiles as well as average latency when queueing produces a long tail.

## 11. Worked reduction: raw CPU statistics to a result

Assume one measured interval reports:

| Raw item | Value |
|---|---:|
| retired instructions | $2.40\times10^8$ |
| simulated cycles | $1.20\times10^8$ |
| target clock | 2.5 GHz |
| LLC demand misses | $1.20\times10^6$ |
| DRAM bytes transferred | 9.6 GB |
| branch mispredictions | $7.2\times10^5$ |

Then

$$
\text{IPC}=2.40/1.20=2.0,\qquad T=1.20\times10^8/(2.5\times10^9)=0.048\text{ s},
$$

$$
\text{LLC MPKI}=1000(1.20\times10^6)/(2.40\times10^8)=5.0,
$$

$$
\text{DRAM bandwidth}=9.6\text{ GB}/0.048\text{ s}=200\text{ GB/s},\qquad
\text{branch MPKI}=3.0.
$$

If a baseline takes $1.44\times10^8$ cycles for the **same dynamic task and clock**, speedup is

$$
S=\frac{T_{base}}{T_{new}}=\frac{1.44}{1.20}=1.20\times.
$$

Do not compute speedup from IPC if the instruction count changes between binaries; use execution time or cycles for equivalent work.

## 12. Samples and benchmark suites are reduced carefully

For fixed-instruction phase samples whose weights $w_i$ represent fractions of the workload's dynamic instructions and sum to one, combine CPI before inverting:

$$
\widehat{\text{CPI}}=\sum_i w_i\,\text{CPI}_i,\qquad
\widehat{\text{IPC}}=1/\widehat{\text{CPI}}.
$$

The weighted arithmetic mean of sample IPCs is generally wrong because IPC is the reciprocal of CPI. If sample lengths vary, use total instructions divided by total cycles or explicitly normalize the weights.

Across benchmark programs, calculate each program's speedup first. For $K$ ratios,

$$
S_{geo}=\left(\prod_{k=1}^{K}S_k\right)^{1/K}.
$$

Also report per-benchmark results; a geometric mean can hide regressions. For throughput workloads with different job counts or service objectives, weighted speedup, harmonic speedup, fairness, or tail latency may be more appropriate than one mean.

Repeated stochastic runs should report a confidence interval. Compiler/configuration variation is not measurement noise and should not be averaged away; it is a separate sensitivity experiment.

## 13. Correctness and conservation checks before trusting performance

Performance is meaningless if the wrong work ran. Check:

- benchmark exit status and output/checksum against a native or functional reference;
- retired instruction count and ROI markers against expectation;
- no fatal simulator event, deadlock timeout, or early instruction limit;
- requests generated = hits + misses at each cache boundary, with documented exceptions;
- bytes injected = bytes delivered plus bytes still in flight at the measurement boundary;
- NPU useful MAC count matches the operator shapes;
- GPU grid/block counts and completed kernels match the launch log;
- energy totals equal the sum of component event counts times per-event costs;
- runtime derived from cycles/frequency matches any simulator time statistic.

Sanity bounds catch unit mistakes:

- IPC cannot exceed the modeled retirement/issue width indefinitely;
- utilization must lie in $[0,1]$;
- transferred bytes cannot be below architecturally required useful bytes unless compression is modeled;
- a fixed amount of work at a higher clock has fewer seconds for the same cycles, but not fewer cycles merely because the reporting frequency changed.

## 14. What the final result package should contain

A defensible result is not one spreadsheet cell. Preserve:

1. build manifest and compiler command;
2. executable/trace/model hashes;
3. disassembly or graph/operator summary;
4. simulator configuration and revision;
5. full command line and environment;
6. raw unedited statistics/logs;
7. post-processing formula/code version;
8. derived table with units and denominator definitions;
9. validation comparison and known model gaps;
10. sampling weights, repetitions, and uncertainty.

The provenance chain should let another reader start at a plotted point and walk backward to raw counters, simulation run, input artifact, compiler, source, and benchmark input. If any arrow is missing, the number may be impossible to reproduce or interpret.

## 15. Common failures and their signatures

| Failure | Symptom | Repair |
|---|---|---|
| wrong ISA/ABI binary | loader error or illegal instruction | inspect ELF header and disassembly; cross-compile explicitly |
| debug/unoptimized build | huge instruction count, low vector use | record flags and compare disassembly/dynamic counts |
| simulator measures initialization | IPC dominated by loader/input parsing | bracket the intended ROI and reset stats |
| cold detailed state after checkpoint | inflated early miss rates | add validated warm-up |
| PTX result compared with SASS instruction count | contradictory IPC/cycle trends | compare cycles/time and name instruction domain |
| NPU CSV called “full model simulation” | unsupported vector/control operators disappear | preserve graph-to-operator coverage report |
| CPU load trace fed directly to DRAM | impossible traffic and bandwidth | filter through translation/caches or label synthetic study |
| average sample IPC | biased phase aggregation | weight CPI or aggregate instructions/cycles |
| bytes denominator undocumented | two “bandwidth” values disagree | state useful/cache-line/DRAM bytes and boundary |
| output not verified | fast simulation of wrong work | compare checksum/reference before performance |

## Numbers to remember

| Item | Rule |
|---|---|
| source line to instruction | never one-to-one after optimization |
| static vs dynamic | dynamic count = static block size × execution frequency |
| CPU result | retired instructions and cycles must cover the same ROI |
| GPU artifact | PTX is virtual ISA; cubin contains target SASS |
| NPU artifact | graph/operator/mapping often replaces assembly |
| trace meaning | defined by the exact point where it was captured |
| elapsed time | cycles divided by the modeled clock frequency |
| energy | event count × per-event energy, summed over components |
| sample aggregation | weight CPI, then invert; do not average IPC blindly |
| suite speedup | per-benchmark ratios, then geometric mean |

## Cross-references

- [Workload Characterization and Sampling](../02_Performance_Analysis/02_Workload_Characterization_and_Sampling.md) explains benchmark selection and phase representativeness.
- [CPU gem5](../../01_CPU_Architecture/08_Simulation/01_gem5.md) instantiates the ELF loader, dynamic instruction, event, and statistics path.
- [GPU Simulators](../../02_GPU_Architecture/04_Simulation/01_GPU_Simulators.md) instantiates PTX/SASS and trace replay.
- [NPU Simulators](../../03_NPU_Architecture/04_Simulation/01_Accelerator_and_NPU_Simulators.md) instantiates graph, mapping, access-count, and cycle models.
- [NoC and Coherence Simulation](../../01_CPU_Architecture/08_Simulation/02_NoC_and_Coherence_Simulation.md) follows cache requests into protocol messages and flits.
- [DRAM Simulators](../../04_SoC_and_Chiplet_Architecture/06_Simulation/01_DRAM_Simulators.md) follows physical requests into legal commands and power.

## References

1. gem5 Project, “Execution Basics,” “SE Binary Workloads,” and “Understanding gem5 Statistics and Output” — [execution](https://www.gem5.org/documentation/general_docs/cpu_models/execution_basics), [workload API](https://www.gem5.org/documentation/general_docs/stdlib_api/gem5.components.boards.se_binary_workload.html), [statistics](https://www.gem5.org/documentation/learning_gem5/part1/gem5_stats/).
2. Tool Interface Standards Committee, *Executable and Linking Format (ELF) Specification*; System V ABI processor supplements define target relocations and calling conventions.
3. NVIDIA, “CUDA Platform” and *CUDA Compiler Driver NVCC* — [fatbins, PTX, and cubins](https://docs.nvidia.com/cuda/cuda-programming-guide/01-introduction/cuda-platform.html), [compiler workflow](https://docs.nvidia.com/cuda/cuda-compiler-driver-nvcc/).
4. Accel-Sim Project, “Accel-Sim Tracer, SASS Frontend, Simulation Engine, and Statistics” — [official repository](https://github.com/accel-sim/accel-sim-framework).
5. ONNX, *Intermediate Representation Specification* — [official specification](https://onnx.ai/onnx/repo-docs/IR.html).
6. NVLabs, “Timeloop” — [official repository](https://github.com/NVlabs/timeloop).
7. SCALE-Sim Project, “SCALE-Sim v3 Inputs and Outputs” — [official repository](https://github.com/scalesim-project/SCALE-Sim).
8. CMU SAFARI, “Ramulator 2.1 User Guide” — [official repository](https://github.com/CMU-SAFARI/ramulator2).

---

← [Analytical Models](02_Analytical_Models.md) · [Simulation Methodology index](00_Index.md)
