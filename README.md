# Silicon Design Notebook

**Digital IC design & verification — from transistor physics to tape-out and post-silicon bring-up.**

A comprehensive, bottom-up technical reference covering digital IC design end to end: CMOS (complementary metal-oxide-semiconductor) device physics, RISC-V CPU microarchitecture, memory/interconnect, RTL (register-transfer level) and verification methodology, synthesis, physical design, timing/power/DFT (design for test) signoff, fabrication, and packaging.

Built for senior-level interview preparation and professional reference. Target roles: RTL design engineer, CPU/microarchitecture engineer, physical design/STA engineer, DFT engineer, verification engineer.

This notebook was split out of the combined "Silicon to Serving" project — see the sibling [AI-infrastructure notebook](https://github.com/Wty2003328/silicon-to-serving) for the other half (GPU/TPU architecture, CUDA kernels, transformers, distributed training, LLM serving).

## Getting Started

- **Obsidian (recommended):** open this folder as a vault — cross-references and Mermaid diagrams render natively.
- **GitHub:** every page renders directly in the GitHub preview (Mermaid supported).
- **Any markdown viewer:** standard markdown with relative links throughout.

Start at [Index.md](Index.md) or [Chip_Design_Flow_Overview.md](Chip_Design_Flow_Overview.md).

## Dependencies & Setup

To view this notebook exactly as intended, the following Obsidian configuration is recommended:

1. **Mermaid Zoom Plugin (Architecture Diagrams)** — install the [mermaid-zoom](https://github.com/xiaozhuang0433/mermaid-zoom) community plugin so dense architecture diagrams open in a full-screen, scrollable modal.
2. **Custom CSS snippet** for code/LaTeX wrapping — included in this vault's `.obsidian/snippets/`.

## Structure

Organized by the chip-design flow — the folders are numbered `00 → 07` in flow order.

```ascii-graph
hardware_design_notebook/
├── Chip_Design_Flow_Overview.md   the master narrative: spec -> silicon
├── Index.md                       full page index and reading paths
├── 00_Fundamentals/               device physics, logic, arithmetic, SystemC/TLM (5 pages)
├── 01_Architecture_and_PPA/       25 nested subdomains across CPU/GPU/NPU, memory, fabrics, modeling, simulators (45 pages)
├── 02_Power_and_Low_Power/        power physics, intent (UPF), reduction, signoff (5 pages)
├── 03_Frontend_RTL_and_Verification/   RTL coding, clocks/CDC, UVM, formal, coverage (13 pages)
├── 04_Synthesis/                  RTL -> gates + SDC constraints (2 pages)
├── 05_Backend_Physical_Design/    gates -> layout, signal integrity (2 pages)
├── 06_Signoff/                    STA, DFT/ATPG, physical verification (3 pages)
├── 07_Manufacturing_and_Bringup/  fab, packaging, tapeout, post-silicon (3 pages)
└── interview_prep/                per-folder interview Q&A (00-07) + RTL coding canon + numeric bank
```

Pages are numbered in reading order within each domain and subdomain, and every level has a `00_Index.md` defining ownership, reading order, and handoffs. Total: **78 flow pages + 10 interview-prep banks**.

See [Index.md](Index.md) for the full page-by-page index with coverage summaries.

## Content Style

Every page follows the same structure:

- **Section 0: Why this page exists** — one-paragraph framing of what problem this page solves
- **Numbered sections** — deep technical content with derivations, not summaries
- **Mermaid diagrams** — architecture, dataflow, cause-effect chains
- **Numbers to memorize** — tables of constants that come up in interviews
- **Worked problems** — interview-style problems with full solutions
- **Cross-references** — links to prerequisite and downstream pages

## History

This notebook's git history was split out of the combined `silicon-to-serving` repository — commit history for this notebook's content is preserved back to the original project.

## License

[MIT](LICENSE) — use freely for study, interview prep, or teaching.
