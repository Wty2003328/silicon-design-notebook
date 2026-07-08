# Physical Verification — DRC, LVS, Antenna, and DFM Signoff

> **Stage:** 06 · Signoff. The checks that prove the *layout* is manufacturable and matches the netlist — the last gate before [GDSII hand-off](../07_Manufacturing_and_Bringup/Tapeout_and_Post_Silicon_Bringup.md).
> **Prerequisites:** [Physical_Design](../05_Backend_Physical_Design/Physical_Design.md), [Fabrication_Process](../07_Manufacturing_and_Bringup/Fabrication_Process.md) (the rules come from the process). **Hands off to:** tape-out.

---

## 0. Why this page exists

Timing signoff ([STA](STA.md)) proves the chip is *fast enough*; physical verification proves it can actually be *built and will be the circuit you designed*. The foundry will not accept a GDSII that violates its design rules, and a layout that passes DRC but doesn't match the schematic is a guaranteed dead chip. These checks — DRC (rules), LVS (layout = schematic), antenna, and DFM — are a hard, foundry-defined gate. Each catches a class of failure invisible to every earlier check. (The *physics* of why the rules exist is in [Fabrication_Process](../07_Manufacturing_and_Bringup/Fabrication_Process.md); this page is the *signoff*.)

---

## 1. DRC — Design Rule Check

DRC verifies the layout obeys the **foundry's geometric rules**, which encode what the [lithography + etch process](../07_Manufacturing_and_Bringup/Fabrication_Process.md) can actually print.

| Rule class | Example | Why the process needs it |
|---|---|---|
| **Width** | min metal width | thinner can't be patterned / opens |
| **Spacing** | min space between same-layer shapes | closer shorts / can't resolve |
| **Enclosure** | via must be enclosed by metal by X | mis-alignment margin |
| **Area / density** | min/max metal density per window | **CMP** planarity (dishing/erosion) |
| **Antenna** | max gate-area-to-metal ratio | plasma charge damage (see §3) |
| **Min-area / notch / corner** | no slivers | manufacturability |

At advanced nodes there are **thousands** of rules, plus **DPT/MPT coloring** rules (multi-patterning: shapes must be 2-colorable for double-patterning, or odd-cycle conflicts are flagged) and **DRC+ / equation-based** rules. DRC runs on the full GDSII and must reach **zero violations or justified waivers** — the foundry rejects anything else.

---

## 2. LVS — Layout Versus Schematic

LVS proves the **layout implements the intended netlist** — that the polygons, when extracted into devices and nets, are *electrically identical* to the schematic/netlist.

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk", "nodeSpacing": 60, "rankSpacing": 60, "htmlLabels": false}}}%%
flowchart TD
    LAY["Layout (GDSII)"]:::a --> EXT["Extract devices+nets<br/>from polygons"]:::b
    EXT --> NL1["Layout netlist"]:::c
    SCH["Source netlist<br/>(from synthesis)"]:::d --> NL2["Schematic netlist"]:::c
    NL1 --> CMP{"Graph compare<br/>(devices, nets,<br/>connectivity)"}:::e
    NL2 --> CMP
    CMP -->|match| OK["LVS clean"]:::ok
    CMP -->|mismatch| BAD["shorts / opens /<br/>wrong device / param"]:::bad
    classDef a fill:#fde68a,stroke:#b45309,color:#000
    classDef b fill:#bbf7d0,stroke:#15803d,color:#000
    classDef c fill:#bae6fd,stroke:#0369a1,color:#000
    classDef d fill:#c7d2fe,stroke:#4338ca,color:#000
    classDef e fill:#fde68a,stroke:#b45309,color:#000
    classDef ok fill:#bbf7d0,stroke:#15803d,color:#000
    classDef bad fill:#fecaca,stroke:#991b1b,color:#000
```

The tool **extracts** devices and connectivity from the layout, then does a **graph isomorphism** comparison against the source netlist. Mismatches it catches:
- **Shorts** — two nets that should be separate are connected (the classic killer).
- **Opens** — one net that should be connected is split.
- **Wrong/missing device**, wrong transistor W/L, wrong connectivity.
- **Property mismatches** — device parameters off.

LVS clean is non-negotiable: a short between power and ground, or a swapped connection, makes the chip dead regardless of perfect timing. Paired with LVS, **ERC** (Electrical Rule Check) flags floating gates, missing well/substrate ties, etc.

---

## 3. Antenna check — a process-damage rule worth its own section

During fabrication, a long metal wire connected to a thin transistor gate but **not yet** connected to a diffusion can collect charge from the plasma etch/deposition. If the accumulated charge-to-gate-area ratio (**antenna ratio**) exceeds the process limit, the voltage damages the thin gate oxide (Fowler-Nordheim tunneling → Vt shift / [TDDB](../05_Backend_Physical_Design/Signal_Integrity_Reliability.md)). Fixes the tool/router applies:
- **Antenna (jumper) diodes** to bleed charge to substrate, or
- **Layer jumping** — break the long wire and route a piece on a higher layer so no single-layer antenna is too large.

It's a DRC-class rule but distinct because the failure is **reliability/yield**, not a geometric short, and it depends on the *order* metals are deposited.

---

## 4. DFM — Design for Manufacturability (beyond pass/fail)

DFM goes past binary rule-checking to *improve yield*:
- **CMP / density fill** — add dummy metal/poly to equalize density so [CMP](../07_Manufacturing_and_Bringup/Fabrication_Process.md) doesn't dish or erode.
- **Litho hotspot / OPC-awareness** — flag patterns that print marginally even if DRC-legal; recommend wire spreading, via doubling.
- **Redundant vias** — replace single vias with double vias where space allows (a single via is a yield/EM risk).
- **Critical-area analysis** — estimate random-defect-limited yield from the layout's susceptibility to particle defects.

DFM is "DRC-clean but better" — the gap between *manufacturable* and *high-yielding*.

---

## 5. Where PV sits — the signoff gate

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk", "nodeSpacing": 60, "rankSpacing": 60, "htmlLabels": false}}}%%
flowchart TD
    RL["Routed layout"] --> DRC["DRC"] --> LVS["LVS"] --> ANT["Antenna"] --> DFM["DFM / fill"] --> GDS["clean →<br/>GDSII tape-out"]
    GDS -.->|"any unwaived violation blocks tape-out"| RL
    classDef s fill:#dbeafe,stroke:#1d4ed8,color:#000
    classDef g fill:#dcfce7,stroke:#15803d,color:#000
    class RL,DRC,LVS,ANT,DFM s
    class GDS g
```

Physical verification is run on the **full-chip merged GDSII** (including IP, memories, analog) and must be **clean or fully waived** before the [tape-out](../07_Manufacturing_and_Bringup/Tapeout_and_Post_Silicon_Bringup.md) hand-off. It's typically the last thing standing between "design done" and "send to fab."

---

## 6. Numbers / facts to memorize

| Fact | Value/why |
|---|---|
| DRC rule count (advanced node) | thousands (+ multi-patterning coloring) |
| DRC gate | zero unwaived violations — foundry rejects otherwise |
| LVS method | extract → **graph isomorphism** vs source netlist |
| LVS top killers | power/ground short, swapped net, wrong device |
| Antenna failure | gate-oxide damage from plasma charge (Fowler-Nordheim) |
| Antenna fixes | jumper diode, layer jumping |
| DFM levers | density fill, double vias, hotspot fixes |
| Runs on | full merged GDSII (incl. IP/memory/analog) |

---

## 7. Interview Q&A

**Q: DRC passes but the chip is dead — what check did you skip?** Almost certainly **LVS**: DRC only proves the geometry is legal, not that it implements the right circuit. A poly-to-metal mis-connection or a power/ground short can be perfectly DRC-clean and electrically fatal. LVS (extract + compare to netlist) is what catches it.

**Q: What is the antenna effect and how is it fixed?** A long single-layer metal connected to a thin gate collects plasma charge during fab; if the charge/gate-area ratio exceeds the limit, the gate oxide is damaged. Fix by adding antenna diodes (bleed charge) or jumping the route to another layer so no single segment is too large relative to the gate.

**Q: Why add dummy metal fill if the design already passes DRC?** For **CMP planarity** and yield: chemical-mechanical polishing dishes/erodes low-density regions, changing thickness and hurting yield and timing. Density-balancing fill (and double vias, hotspot fixes) is DFM — making a DRC-clean layout actually high-yielding.

---

## Cross-references
- Upstream layout: [Physical_Design](../05_Backend_Physical_Design/Physical_Design.md). SI/reliability: [Signal_Integrity_Reliability](../05_Backend_Physical_Design/Signal_Integrity_Reliability.md).
- Rule physics: [Fabrication_Process](../07_Manufacturing_and_Bringup/Fabrication_Process.md). Hand-off: [Tapeout_and_Post_Silicon_Bringup](../07_Manufacturing_and_Bringup/Tapeout_and_Post_Silicon_Bringup.md).
- Other signoffs: [STA](STA.md), [Power_Analysis_and_Signoff](../02_Power_and_Low_Power/Power_Analysis_and_Signoff.md), [DFT_and_ATPG](DFT_and_ATPG.md).
