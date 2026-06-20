# Hardware Folder — Flow-Based Reorganization Plan (proposal)

> Goal: reorganize `hardware_design/` so the folder order *is* the chip-design flow — early PPA/architecture → frontend RTL → verification → synthesis → backend → signoff → manufacturing/bring-up — and fill in the missing flow steps.
> Current layout is by **topic** (Fundamentals, Architecture, Implementation, Clocking, Power, SystemVerilog). This proposal re-homes every page into a numbered **flow stage** and adds the missing stages.
>
> **Nothing has been moved yet.** This is for your review.

---

## Proposed flow taxonomy (numbered so Obsidian sorts them in flow order)

```
hardware_design/
  00_Fundamentals/                  device + logic + arithmetic prerequisites
  01_Architecture_and_PPA/          early power/perf analysis + architecture exploration
  02_Frontend_RTL_Design/           RTL coding, clocking/CDC, low-power intent
  03_Verification/                  functional + static frontend signoff
  04_Synthesis/                     RTL → gates
  05_Backend_Physical_Design/       gates → layout
  06_Signoff/                       timing, power, test, physical verification
  07_Manufacturing_and_Bringup/     fab, packaging, tapeout, post-silicon
  interview_prep/                   (unchanged)
```

---

## Where each existing page goes

| New stage | Pages moved in (from → ) |
|---|---|
| **00_Fundamentals** | CMOS_Fundamentals, Fabrication_Process, Basic_Knowledge→`Logic_Building_Blocks`, Adders→`Adders_and_Arithmetic`, Floating_Point |
| **01_Architecture_and_PPA** | Power_Fundamentals, Block_Activity_and_Power (early power), CPU_Architecture, OoO_Execution, Branch_Prediction_Deep_Dive, Cache_Microarchitecture, TLB_and_Virtual_Memory, RISC_V_ISA, Xiangshan_CPU_Design, Memory, DDR_Controller, Network_on_Chip, AHB_AXI_APB, ACE_and_CHI |
| **02_Frontend_RTL_Design** | Data_Types_and_Basics, Procedural_and_Processes, OOP_and_Randomization, Clock_Division, Async_Circuit_Design→`Async_and_CDC`, UPF_Power_Intent, Low_Power_Design, Power_Reduction_Techniques |
| **03_Verification** | Assertions_and_Coverage, IPC_and_Verification, UVM_Methodology, Formal_Verification |
| **04_Synthesis** | Synthesis_and_Optimization |
| **05_Backend_Physical_Design** | Physical_Design, Signal_Integrity_Reliability |
| **06_Signoff** | STA, Power_Analysis_and_Signoff, DFT_and_ATPG |
| **07_Manufacturing_and_Bringup** | IC_Packaging |

Note: the current **Power/** and **Clocking_and_Signals/** topic-folders get *distributed* across flow stages (power physics → 01, power intent → 02, power signoff → 06; CDC → 02, SI → 05). That is the whole point of a flow view — a cross-cutting concern appears at the stage where you actually do it, with cross-links to the others.

---

## Gap pages to add (the "missing steps")

Priority: **P1** = core flow step with no current home; **P2** = valuable but partially covered elsewhere.

| # | New page | Stage | Scope / why it's a gap | Pri |
|---|---|---|---|---|
| 1 | `Chip_Design_Flow_Overview` | root | The master narrative: the whole RTL-to-GDSII-to-silicon flow, hand-offs, what each stage consumes/produces, where iterations loop back. Ties the numbered folders together. | **P1** |
| 2 | `Performance_Modeling_and_DSE` | 01 | Early-stage **performance** analysis: analytical models, cycle-approximate vs cycle-accurate sim (gem5), SystemC/TLM, trace-driven, design-space exploration, PPA tradeoff methodology. Currently only scattered mentions. | **P1** |
| 3 | `RTL_Design_Methodology` | 02 | Frontend design *principles* (vs the existing coding *drills*): synchronous design rules, reset architecture (sync/async, reset domains), clock architecture, FSM styles, datapath/control split, parameterization & reuse, lint-clean RTL. | **P1** |
| 4 | `Lint_CDC_RDC_Signoff` | 03 | Static frontend signoff as a methodology: lint rulesets, CDC structural+functional checks, RDC, the "clean before sim" gate. (CDC *physics* stays in Async_and_CDC.) | **P1** |
| 5 | `Gate_Level_Sim_and_Emulation` | 03 | Post-synth dynamic verification: GLS (zero-delay vs SDF), X-prop, plus emulation & FPGA prototyping (Palladium/Veloce/ZeBu) for software-scale validation. | **P2** |
| 6 | `Verification_Planning_and_Coverage_Closure` | 03 | The verification *flow*: vplan, coverage model design, coverage closure loop, regression/triage, sign-off criteria. (Mechanics stay in Assertions/UVM.) | **P2** |
| 7 | `Constraints_SDC` | 04 | A dedicated SDC authoring page (clocks, I/O delays, exceptions, CDC constraints) — currently split between STA and Synthesis. | **P2** |
| 8 | `Physical_Verification_DRC_LVS` | 06 | DRC, LVS, antenna, ERC, DFM as a signoff step. (Currently a subsection inside Physical_Design.) | **P2** |
| 9 | `Tapeout_and_Post_Silicon_Bringup` | 07 | The end of the flow: GDSII/tapeout checklist, mask/reticle, first-silicon bring-up, post-silicon validation & debug, yield ramp, ECO/respin. Genuine gap. | **P1** |

Already-covered steps that just need cross-linking (no new page): CTS (in Physical_Design), DFT scan insertion (in DFT_and_ATPG), PDN/power-grid (in Signal_Integrity + Power_Analysis), reset basics (in Async).

---

## Execution approach

1. Create the numbered folders; **move** each page (preserve git history where possible).
2. Rewrite `hardware_design/Index.md` as a flow walkthrough (stage by stage).
3. **Fix all cross-links** — ~183 internal links, ~65 cross-folder links, and ~37 links from `ai_infra/` that point into hardware_design. I'll grep-and-replace every moved path and verify zero broken links at the end.
4. Add the gap pages (priority order), each at the notebook's existing depth bar.

**Risk:** link breakage is the main one; I'll do a final automated pass that checks every `.md` link resolves. Moves are reversible.

---

## Executed ✓ (final taxonomy)

Confirmed decisions and built:
- **Full physical move** — 38 pages relocated into the numbered flow folders; all 1,241 markdown links across the notebook re-pointed and verified (0 broken).
- **Verification folded into Frontend** → stage **03_Frontend_RTL_and_Verification** (one combined frontend stage).
- **Fabrication moved to Manufacturing** → **07_Manufacturing_and_Bringup**.
- **Power kept together** as one cross-cutting track → **02_Power_and_Low_Power** (all 6 power pages).
- **All 9 gap pages written** at depth: Chip_Design_Flow_Overview, Performance_Modeling_and_DSE, RTL_Design_Methodology, Lint_CDC_RDC_Signoff, Gate_Level