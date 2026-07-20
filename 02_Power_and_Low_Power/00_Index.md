# 02 · Power and Low-Power — Folder Index

*Cross-cutting track: power is budgeted from workloads, partitioned at architecture, captured as power intent, implemented in synthesis/backend, and verified through signoff.*

| # | Page | Coverage |
|---|------|----------|
| 01 | [Power Fundamentals](01_Power_Fundamentals.md) | switching/short-circuit/leakage physics, leakage-by-node breakdown, scaling/Dennard, sub-threshold swing |
| 02 | [Block Activity and Power](02_Block_Activity_and_Power.md) | per-block/per-mode modeling, RTL power, glitch, emulation power, on-die telemetry |
| 03 | [Low-Power Architecture and Domain Partitioning](03_Low_Power_Architecture_and_Domain_Partitioning.md) | power/voltage/clock/reset domains, partition strategy, AON design, boundary matrix, worked SoC |
| 04 | [Power Reduction Techniques](04_Power_Reduction_Techniques.md) | clock gating, DVFS, power gating + retention, multi-$V_t$, body biasing, operand isolation |
| 05 | [UPF/CPF Power-Intent Flow](05_UPF_and_CPF_Power_Intent.md) | IEEE 1801 and CPF, domains/supplies, isolation, level shifting, retention, PST, RTL-to-signoff flow |
| 06 | [Power Analysis and Signoff](06_Power_Analysis_and_Signoff.md) | PrimeTime PX/Voltus flows, activity annotation, IR/EM, glitch, peak/di-dt, thermal, backside power |

---

⬅ prev [01 · Architecture and PPA](../01_Architecture_and_PPA/00_Index.md) · [Root Index](../Index.md) · [Flow Overview](../Chip_Design_Flow_Overview.md) · next ➡ [03 · Frontend RTL and Verification](../03_Frontend_RTL_and_Verification/00_Index.md)
