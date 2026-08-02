# 06 · Signoff — Folder Index

*Prove it's correct before tape-out.*

| # | Page | Coverage |
|---|------|----------|
| 01 | [Static Timing Analysis](01_STA.md) | cell delay, setup/hold, CRPR, OCV/AOCV/POCV, clock-gating checks, SDC walkthrough |
| 02 | [DFT and ATPG](02_DFT_and_ATPG.md) | scan, fault models, ATPG, at-speed, compression, memory BIST + March, AI-accelerator DFT |
| 03 | [Physical Verification (DRC/LVS)](03_Physical_Verification_DRC_LVS.md) | DRC, LVS graph-compare, ERC, antenna, density/fill, DFM |
| 04 | [Signoff Orchestration, ECOs, and Tape-out Readiness](04_Signoff_Orchestration_ECO_and_Tapeout_Readiness.md) | the full check inventory and its dependency graph; MMMC scenario explosion and pruning; signoff vs PnR timing correlation; the ECO taxonomy (timing / functional / metal-only / full-layer); spare-cell mechanics; the ECO flow; waivers, legitimate and not; convergence burn-down; a 40-item readiness checklist; the release package to the fab |

**Reading order.** 01–03 are the individual proofs. 04 is the orchestration: how dozens of checks are scheduled, how a failure becomes an ECO, and how a chip is declared ready. Read 04 last, and read it before you ever attend a tape-out review.

---

⬅ prev [05 · Backend (Physical Design)](../05_Backend_Physical_Design/00_Index.md) · [Root Index](../Index.md) · [Flow Overview](../Chip_Design_Flow_Overview.md) · next ➡ [07 · Manufacturing and Bring-up](../07_Manufacturing_and_Bringup/00_Index.md)
