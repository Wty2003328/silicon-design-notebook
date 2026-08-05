# Silicon Design Notebook

**Digital IC design & verification — from transistor physics to tape-out and post-silicon bring-up.**

A comprehensive, bottom-up technical reference covering digital IC design end to end: CMOS (complementary metal-oxide-semiconductor) device physics, RISC-V CPU microarchitecture, memory/interconnect, RTL (register-transfer level) and verification methodology, synthesis, physical design, timing/power/DFT (design for test) signoff, fabrication, and packaging.

Built for senior-level and research-position preparation. Target roles include CPU/GPU/NPU and AI-system architecture, RTL and verification, physical design/static timing analysis (STA), design for test (DFT), performance modeling, and accelerator research.

**It is also written to be entered from scratch.** If your background is one undergraduate digital-design course — Boolean algebra, K-maps, FSMs, some Verilog — start at **[Start Here — the Learning Path](Start_Here_Learning_Path.md)**. It lays out five capability stages, each with an entry check, an exact reading order, a build-it project, and exit questions, and it names the constraint each stage removes. The premise of the whole notebook is that chip design *is* the systematic removal of the simplifications a first course makes: gates are free, wires are instant, clocks are perfect, area and power do not exist, and passing a testbench means being correct. Every one of those is false in silicon, and each falsehood generates a discipline.

This notebook was split out of the combined "Silicon to Serving" project — see the sibling [AI-infrastructure notebook](https://github.com/Wty2003328/silicon-to-serving) for the other half (GPU/TPU architecture, CUDA kernels, transformers, distributed training, LLM serving).

## Getting Started

- **Obsidian (recommended):** open this folder as a vault — cross-references, zoomable Mermaid diagrams, and WaveDrom timing diagrams render with the included plugin configuration.
- **GitHub:** every page renders directly in the GitHub preview (Mermaid supported).
- **Any markdown viewer:** standard markdown with relative links throughout.
- **No EDA licenses?** [08 · Design Methodology and EDA Infrastructure](08_Cross_Cutting_Engineering/03_Design_Methodology_and_EDA_Infrastructure.md) §13 lists a complete open-source toolchain — Verilator, cocotb, SymbiYosys, Yosys, OpenSTA, OpenROAD/OpenLane, KLayout, and an open PDK — that runs a design from RTL to GDSII on a laptop. Every build-it project in the learning path is doable with it.

**Four entry points, depending on what you need:**

| Start here | If you want |
|---|---|
| [Start_Here_Learning_Path.md](Start_Here_Learning_Path.md) | The ordered curriculum from a digital-design course to expert — stages, projects, and exit checks |
| [Chip_Design_Flow_Overview.md](Chip_Design_Flow_Overview.md) | The spine: every stage, hand-off artifact, and iteration loop from spec to silicon |
| [Index.md](Index.md) | The full page-by-page index with coverage summaries |
| [Glossary.md](Glossary.md) | Any acronym, defined once, with a pointer to the page that derives it |

Use [Concept_Dependency_Map.md](Concept_Dependency_Map.md) whenever a page assumes something you have not met — it lists what each concept requires, where it is derived, and what specific confusion results from skipping the prerequisite. Authors and reviewers should also use the [Research-Depth and Evidence Standard](Research_Depth_and_Evidence_Standard.md), which defines the required chain from workload to mechanism, theory, measurable evidence, validation, research questions, and an implementation-reconstruction contract.

## Dependencies & Setup

To view this notebook exactly as intended, the following Obsidian configuration is recommended:

1. **TikZJax + CircuiTikZ (Circuit Schematics)** — the tracked `obsidian-tikzjax` plugin renders text-authored transistor, logic-gate, storage, and arithmetic circuits.
2. **Mermaid Zoom Plugin (Architecture Diagrams)** — install the [mermaid-zoom](https://github.com/xiaozhuang0433/mermaid-zoom) community plugin so dense architecture diagrams open in a full-screen, scrollable modal.
3. **WaveDrom Plugin (Timing Diagrams)** — install the `obsidian-wavedrom` community plugin so fenced `wavedrom` blocks render clock, handshake, reset, protocol, and memory-command timing.
4. **Custom CSS snippet** for diagram scaling and code/LaTeX wrapping — included in this vault's `.obsidian/snippets/`.

See the [Diagram Authoring Standard](Diagram_Authoring_Standard.md) for when to use CircuiTikZ, WaveDrom, Mermaid, optional CircuitJS, or an editable draw.io SVG.

## Structure

Organized by the chip-design flow — the folders are numbered `00 → 07` in flow order.

```ascii-graph
hardware_design_notebook/
├── Start_Here_Learning_Path.md    the on-ramp: five stages, projects, exit checks
├── Concept_Dependency_Map.md      what every concept requires, and where it is derived
├── Glossary.md                    ~600 terms and acronyms, each pointing at its owning page
├── Chip_Design_Flow_Overview.md   the master narrative: spec -> silicon
├── Index.md                       full page index and reading paths
├── Research_Depth_and_Evidence_Standard.md   notebook-wide technical quality rubric
├── 00_Fundamentals/               device physics, logic, arithmetic, memory arrays,
│                                  fixed-point/DSP datapaths, SystemC/TLM (7 pages)
├── 01_Architecture_and_PPA/       4 chip-architecture books, 34 subdomains, 102 chapters
├── 02_Power_and_Low_Power/        power physics, domain architecture, reduction, UPF/CPF, signoff (6 pages)
├── 03_Frontend_RTL_and_Verification/   RTL coding, clocks/CDC, UVM, formal, coverage,
│                                  design patterns, flow control, arithmetic/memory RTL (16 pages)
├── 04_Synthesis/                  RTL -> gates: theory, SDC, standard-cell libraries,
│                                  the actual flow and QoR closure, physical synthesis (5 pages)
├── 05_Backend_Physical_Design/    gates -> layout: floorplan/power plan, placement, CTS,
│                                  routing and extraction, signal integrity (6 pages)
├── 06_Signoff/                    STA, DFT/ATPG, physical verification,
│                                  signoff orchestration and ECOs (4 pages)
├── 07_Manufacturing_and_Bringup/  fab, packaging, tapeout, post-silicon (3 pages)
├── 08_Cross_Cutting_Engineering/  security, functional safety, design methodology and
│                                  EDA infrastructure, IP reuse and register automation (4 pages)
└── interview_prep/                per-folder interview Q&A (00-07) + RTL coding canon +
                                   numeric bank + frontend bank + cram sheet (12 pages)
```

Two folders are **tracks rather than stages**: `02` (power) and `08` (security, safety, methodology, IP reuse). Nothing in them happens at a point in the flow — each is a constraint or a practice that applies at every stage at once.

Pages are numbered in reading order within each domain and subdomain, and every level has a `00_Index.md` defining ownership, reading order, and handoffs. The architecture section is organized as four self-contained CPU, GPU, NPU, and SoC/chiplet books. Each book owns its workload definition, AI-workload mapping and serving analysis, AI compiler/runtime/scheduler/state/operations implementation, performance modeling, design-space exploration, physical implementation, PPA estimation, simulation workflow, evidence standards, and hardware implementation blueprints. Total: **155 flow pages + 12 interview-prep banks**.

See [Index.md](Index.md) for the full page-by-page index with coverage summaries.

## Content Style

Substantive pages follow the [Research-Depth and Evidence Standard](Research_Depth_and_Evidence_Standard.md). The common structure is:

- **Section 0: Why this page exists** — one-paragraph framing of what problem this page solves
- **Numbered sections** — deep technical content with derivations, not summaries
- **Procedural feature evolution** — minimum baseline → concrete trace → observed failure → derived feature and enabling state/control → replay → costs, losing cases, and evidence
- **CircuiTikZ diagrams** — transistor, gate, storage, and arithmetic schematics with real electrical symbols
- **Mermaid diagrams** — ownership, architecture, dataflow, state, cause-effect, and experiment loops; never use generic rectangles as a substitute for a circuit schematic
- **WaveDrom diagrams** — cycle-level clocks, latches/flip-flops, reset, clock-domain crossing, handshakes, protocols, and memory timing
- **Numbers to memorize** — tables of constants that come up in interviews
- **Worked problems** — interview-style problems with full solutions
- **Cross-references** — links to prerequisite and downstream pages
- **Evidence and research boundary** — counters/traces, validation, assumptions, failure modes, and open problems
- **Implementation reconstruction** — contracts, owned state, interfaces, sizing, policies, invariants, physical effects, verification, and staged bring-up sufficient to derive an original design specification
- **AI-stack reconstruction** — model and IR artifacts, compiler legality, executable/runtime contracts, request and persistent-state machines, distributed execution, numerical/quality validation, observability, security, rollout, rollback, and incident recovery

## History

This notebook's git history was split out of the combined `silicon-to-serving` repository — commit history for this notebook's content is preserved back to the original project.

## License

[MIT](LICENSE) — use freely for study, interview prep, or teaching.
