# 05 · Backend (Physical Design) — Folder Index

*Gate netlist → layout.*

| # | Page | Coverage |
|---|------|----------|
| 01 | [Physical Design (PnR)](01_Physical_Design.md) | floorplan, placement, CTS, routing, ECO, advanced nodes; hand-off to PV/SI signoff |
| 02 | [Signal Integrity and Reliability](02_Signal_Integrity_Reliability.md) | crosstalk, EM, IR drop + grid models, power grid design, thermal, aging, antenna |
| 03 | [Floorplanning and Power Planning](03_Floorplanning_and_Power_Planning.md) | the floorplan as a data object (DEF, rows, sites, tracks); utilization vs routability; aspect ratio; IO/bump planning; macro placement rules and failures; blockages and halos; the power grid derived from a current budget; multi-voltage floorplanning; tap/endcap/spare/decap strategy; the floorplan review gate |
| 04 | [Placement, Legalization, and Optimization](04_Placement_Legalization_and_Optimization.md) | global/legalize/detailed and why it is three phases; HPWL and net weighting; legalization mechanics and displacement; in-place sizing, buffering, cloning, VT swap; buffer-insertion theory; congestion triage; scan reordering; placement with power intent; the review gate |
| 05 | [Clock Tree Synthesis](05_Clock_Tree_Synthesis.md) | ideal → propagated and what changes in the inequalities; the CTS spec and NDR; tree vs spine vs mesh vs multi-source; clustering and balancing; gating in the tree; skew groups; **useful skew derived**; CTS and OCV/CPPR; post-CTS hold fixing; multi-corner skew; clock power |
| 06 | [Routing and Parasitic Extraction](06_Routing_and_Parasitic_Extraction.md) | the metal stack as a tiered resource; global/track-assign/detail routing; the negotiated-congestion cost function; vias and redundancy; NDR and shielding; advanced-node rules and multi-patterning; antenna; extraction physics; rule-based vs field solver; SPEF; Elmore and reduction; RC corners and the MMMC explosion |

**Reading order.** 01 is the concept-level tour of the whole PnR flow — read it first. Then 03 → 04 → 05 → 06 walk the four stages in the order the tool runs them, each in depth. 02 is the analysis layer that judges the geometry those stages commit.

---

⬅ prev [04 · Synthesis](../04_Synthesis/00_Index.md) · [Root Index](../Index.md) · [Flow Overview](../Chip_Design_Flow_Overview.md) · next ➡ [06 · Signoff](../06_Signoff/00_Index.md)
