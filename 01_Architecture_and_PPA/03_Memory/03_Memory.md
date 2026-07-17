# Memory Circuits — the Bit-Storage Trade Surface

> **Prerequisites:** [CMOS_Fundamentals](../../00_Fundamentals/01_CMOS_Fundamentals.md) §2–§3 (the inverter VTC, gain, and noise-margin primitives) and §9 (Pelgrom $\sigma_{V_{th}}$ variation) — the device-level foundation the SRAM margin derivations build **on top of**. This page **owns** the SRAM-specific 6T butterfly/SNM, read-disturb, and write-margin derivations (§2); CMOS §12 carries a companion bitcell view.
> **Hands off to:** [Cache_Microarchitecture](01_Cache_Microarchitecture.md) (how these arrays are organised into caches), [Cache_Coherence](05_Cache_Coherence.md) (directory, sharer, and transaction-state storage), [DDR_Controller](04_DDR_Controller.md) (the DRAM interface/scheduling view), [TLB_and_Virtual_Memory](02_TLB_and_Virtual_Memory.md) (the CAM of §10 in its natural home).
> **Scope:** cell- and array-level *circuits* — what physical state stores a bit, what it costs to read/hold/write it, and why each memory technology lands where it does on that trade surface. Protocol/timing (DDR), cache organisation, memory BIST ([DFT_and_ATPG](../../06_Signoff/02_DFT_and_ATPG.md) §7), and the async FIFO ([Async_Design_and_CDC](../../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) §5) live on their own pages.

---

## 0. Why this page exists

Storing a bit is a **bet, not a given**. A bit needs a physical state that survives three abuses — being *held* against leakage and noise, being *read* without destroying itself, and being *written* on demand — and the transistors you spend to win one of those abuses are the same ones that lose you another. There is no cell that is simultaneously the densest, the fastest, the most stable, and the lowest-energy. Every memory on this page — 6T/8T SRAM (static random-access memory), 1T1C DRAM (dynamic random-access memory), CAM (content-addressable memory), MRAM (magnetoresistive RAM), ReRAM, compute-in-memory — is a **named compromise on one trade surface**, and the only interesting question about each is *which cost it traded away and how the rest of the system pays for it.*

This page derives the memory zoo from that surface instead of cataloguing schematics — and it owns the transistor-level SRAM margin theory (the 6T butterfly/SNM, read-disturb, and write-margin derivations of §2), built on the plain-inverter VTC, gain, and noise-margin primitives of [CMOS_Fundamentals](../../00_Fundamentals/01_CMOS_Fundamentals.md) §2–§3 and its Pelgrom variation model (§9). On that foundation we ask the architect's questions. Why does the 6T cell force *read stability* and *write-ability* to fight over the same access transistors, and why does that fight become unwinnable at low $V_{DD}$ — the reason 8T and 10T exist? Why does DRAM throw away the storage feedback loop entirely, and what does that one decision cost in destructive reads and refresh bandwidth? Why is ECC (error-correcting code) a *coding-theory* question about Hamming distance, not a bit-field? Why does a CAM burn 10–20× the power of an SRAM to answer one question, and why do we pay it anyway for a TLB? And what breaks — usefully — when compute-in-memory stops separating storage from arithmetic. By the end you should be able to place any cell on the surface and predict its failure mode, rather than recite its transistors.

A single quantitative motif recurs and is worth holding from the start: **memory signal is a ratio, and margin collapses as $V_{DD}$ scales.** SRAM read stability is a ratio of transistor strengths; DRAM read signal is a ratio of capacitances; both shrink faster than $V_{DD}$ does. Low-voltage operation, not raw area, is the modern memory wall.

### System view — cell physics becomes an architectural resource

Every architectural memory structure is a stack of abstractions. The cell fixes stability/density; the array adds wordlines, bitlines, sensing, and variation; the compiler wraps that array in a timed macro; the microarchitecture composes macros into caches, register files, CAMs, and buffers.

```mermaid
flowchart LR
    C["Bit cell<br/>6T / 8T / 1T1C / NVM"] --> A["Array<br/>WL + BL + sense + write"]
    A --> Y["Yield / ECC / redundancy"]
    Y --> M["Memory macro<br/>ports + timing + power"]
    M --> R["Register file / SRAM / CAM"]
    R --> U["Cache / TLB / queue / scratchpad"]
    U --> S["System PPA + reliability"]
```

---

## 1. The trade surface: store, read, hold

Fix the vocabulary first, because every later section is a move on these axes. A bitcell must support three primitives, and the *cost* of each is what differs across technologies:

- **Store** — commit a bit to some physical state (a latched voltage, charge on a capacitor, a resistance, a magnetisation). Cheaper state → denser cell.
- **Read** — recover the bit. The pivotal property is whether reading is **non-destructive** (the state survives being sensed) or **destructive** (sensing consumes the state, forcing a *restore*).
- **Hold** — retain the bit between accesses. **Volatile** cells forget without power (and some, like DRAM, forget even *with* power, on a leakage clock); **non-volatile** cells keep the bit with the supply off.

Four costs price those primitives: **area** (density), **latency** (access time), **stability/retention margin** (how much noise or leakage before the bit is wrong), and **energy per access**. The whole zoo is points on the {area, latency, margin, energy} surface:

| Cell | Stores bit as | Rel. cell area | Read | Volatile? | Where it wins |
|---|---|---|---|---|---|
| **SRAM 6T** | latched node (cross-coupled inverters) | 1× (~120 $F^2$) | non-destructive | yes | fast, self-restoring; L1/L2, register files |
| **SRAM 8T** | same, + isolated read port | ~1.3× | non-destructive | yes | low-$V_{DD}$ robustness, many ports |
| **DRAM 1T1C** | charge on a capacitor | ~0.05× (~6 $F^2$) | **destructive** | yes (+refresh) | density; main memory |
| **eDRAM 1T1C** | charge, on a logic process | ~0.3–0.5× | destructive | yes (+refresh) | dense on-die L3/L4 |
| **STT/SOT-MRAM** | magnetisation of an MTJ | ~1–1.5× | non-destructive | **no** | zero-leakage last-level cache |
| **ReRAM** | resistance (conductance) | ~0.5× | non-destructive | **no** | analog compute, embedded NVM |

Read the density column as the headline trade: **DRAM's 1T1C cell is ~20× smaller than 6T SRAM in feature-squared terms** precisely because it *deleted the feedback loop* that makes SRAM's read non-destructive and its hold self-restoring — and it pays that back, in full, as destructive reads and refresh (§5–§6). SRAM spends five-plus transistors to buy the opposite corner. Everything below is the detail of these bets.

---

## 2. The 6T SRAM cell: six transistors, three conflicting jobs

The 6T cell is two cross-coupled inverters (the bit, held by positive feedback) plus two access transistors that connect the internal nodes to a bitline pair when the wordline fires. That is the *whole* structure — but its sizing is subtle because **the same two access transistors are the read path and the write path, and read and write want them sized in opposite directions.** Derive the tension from the jobs rather than memorising a sizing table:

1. **Hold** wants the two inverters strong and balanced, so the feedback deeply pins both nodes. Access transistors off.
2. **Read** connects the precharged bitlines to the internal nodes. The "0"-storing node is now yanked *up* toward the bitline through the access transistor while its pull-down fights to hold it at ground. If the access transistor is too strong relative to the pull-down, the "0" node rises past the opposite inverter's switching threshold and **the read flips the cell** — a *read-disturb* failure. Read stability therefore wants a **strong pull-down, weak access**: cell ratio $CR = (W/L)_{pd}/(W/L)_{acc} > 1$.
3. **Write** must do the opposite of read: *deliberately* overpower the feedback and flip the node. The write driver drags the bitline to ground and the access transistor must beat the pull-up PMOS holding the node high. Write-ability therefore wants a **strong access, weak pull-up**: pull-up ratio $PR = (W/L)_{acc}/(W/L)_{pu} > 1$.

The conflict is now explicit and is *the* fact about the 6T cell: **access-transistor strength appears in $CR$ (want it small) and in $PR$ (want it large) with opposite sign.** Sizing is the search for a single access width that keeps *both* ratios comfortably above 1 — typically $CR, PR \approx 1.2$–$2.0$. There is no free lunch; you are dividing a fixed strength budget between two enemies.

### 2.1 Read-disturb, derived — why $CR$ must exceed 1

The read-stability condition is worth one derivation because it shows the margin is a *ratio* that scales badly. Read a "0" node $Q$: the precharged bitline (at $V_{DD}$) pushes current *into* $Q$ through the access transistor, while the pull-down NMOS sinks it to ground — a two-device fight whose balance sets how far $Q$ lifts. The access device sees gate $=V_{DD}$, drain $=$ bitline $=V_{DD}$, source $=Q\approx 0$, so $V_{DS}=V_{GS}=V_{DD}-V_Q$ and it is **saturated**:

$$
I_{acc} \;=\; \tfrac12\,\beta_{acc}\,(V_{DD}-V_Q-V_{th})^2 .
$$

The pull-down sees gate $=\bar Q=V_{DD}$ and a small drain voltage $V_{DS}=V_Q$, so it is in **triode**; for $V_Q\ll V_{DD}-V_{th}$ its current linearises to $I_{pd}\approx \beta_{pd}\,(V_{DD}-V_{th})\,V_Q$. Node $Q$ settles where $I_{acc}=I_{pd}$. Define the **cell ratio** $CR\equiv \beta_{pd}/\beta_{acc}=(W/L)_{pd}/(W/L)_{acc}$ (equal $\mu C_{ox}$), and drop the $V_Q$ inside the saturation term ($V_{DD}-V_Q-V_{th}\approx V_{DD}-V_{th}$):

$$
(V_{DD}-V_{th})^2 \;\approx\; 2\,CR\,(V_{DD}-V_{th})\,V_Q \quad\Longrightarrow\quad V_Q \;\approx\; \frac{1}{2\,CR}\,\big(V_{DD}-V_{th}\big)
$$

where $V_Q$ = voltage the "0" node rises to during read, $CR$ = cell ratio, $V_{th}$ = access-transistor threshold. The cell survives iff $V_Q$ stays below the feedback inverter's switching threshold $V_m \approx 0.4\,V_{DD}$. Plugging a textbook corner ($V_{DD}=1.0$ V, $V_{th}=0.3$ V, $CR=1.33$):

$$
V_Q = \frac{1}{2\times1.33}(1.0-0.3) = 0.263\text{ V} \;<\; V_m = 0.4\text{ V} \quad\checkmark \qquad (\text{margin } 0.137\text{ V})
$$

Drop $CR$ to 1.0 and $V_Q=0.35$ V (margin collapses to 50 mV); at $CR=0.8$, $V_Q=0.44$ V $> V_m$ and **the read flips the cell every time.** This is the whole reason $CR>1$ is non-negotiable. Write-ability has the mirror-image divider between access and pull-up; $PR>1$ is its non-negotiable counterpart.

### 2.2 SNM, and why low $V_{DD}$ is the real enemy

The combined DC-stability metric is the **static noise margin (SNM)**. Derive it from the cross-coupled pair rather than quoting it.

**The butterfly, constructed.** Each inverter has a voltage transfer characteristic (VTC) $v_{out}=f(v_{in})$. Cross-coupling ties $\bar Q=f(Q)$ and $Q=f(\bar Q)$, so the cell's DC operating points are the solutions of $Q=f(f(Q))$. Plot inverter-1's VTC as $\bar Q$ vs $Q$ and inverter-2's VTC with its axes *swapped* ($Q$ vs $\bar Q$) on the same $Q$–$\bar Q$ plane: the two curves cross **three times** — two stable corners near $(0,V_{DD})$ and $(V_{DD},0)$ and one metastable point at $(V_M,V_M)$ — enclosing two "eyes" or **lobes**. That double-eye is the *butterfly curve*.

**Why the margin is the largest inscribed *square*.** Model DC noise as two equal sources $V_n$ inserted in series at *both* storage nodes, in the worst-case (state-attacking) polarity. One source shifts inverter-1's VTC horizontally by $V_n$; the other shifts inverter-2's vertically by $V_n$ — *equal* shifts, because both nodes suffer the same worst-case disturbance. The cell stays bistable only while the two shifted curves still cross in three points; at the critical $V_n$ they become **tangent** (the stable corner and the metastable point are about to merge and annihilate). Sliding one curve right by $V_n$ and the other up by $V_n$ until they touch inscribes an axis-aligned **square of side $V_n$** in the lobe, so the maximum tolerable disturbance is the *side of the largest square that fits inside the smaller lobe*. That side **is** the SNM. The square (equal sides) follows directly from the equal H/V shift; asymmetric per-node noise would inscribe a rectangle, but the standard worst case is symmetric.

**The ideal ceiling, and why real cells fall short.** For an inverter of infinite gain the VTC is a vertical step at $V_M$; the lobe becomes a right-angled region whose largest inscribed square has side exactly $V_M$. With a balanced trip point $V_M=V_{DD}/2$ this is the **upper bound**

$$
\text{SNM}_{\text{hold}} \;\le\; \tfrac12\,V_{DD},
$$

reached only as the inverter voltage gain $g=\lvert dv_{out}/dv_{in}\rvert_{V_M}\to\infty$. Finite gain rounds the VTC corner, so the real inscribed square is strictly smaller; empirically $\text{SNM}_{\text{hold}}\approx 0.3$–$0.4\,V_{DD}$ (the ~280 mV at $V_{DD}=1.0$ V in the table below is $0.56\times$ the 500 mV ceiling). **Gain is stability:** every effect that erodes inverter gain — short-channel output conductance, and above all a supply approaching $V_{th}$ — shrinks the eye faster than $V_{DD}$ itself falls.

**Read degrades it because the pedestal lifts the lobe floor.** The §2.1 divider does more than risk a flip — even a *survived* read lifts the "0" node from $0$ to $V_Q$, which raises the driven inverter's low output level and pulls the bottom of the lower lobe up by $\approx V_Q$. The inscribed square shrinks by roughly that pedestal:

$$
\text{SNM}_{\text{read}} \;\approx\; \text{SNM}_{\text{hold}} - \kappa\,V_Q, \qquad \kappa = O(1),
$$

and it collapses to zero exactly as $V_Q\to V_M$ — the *same* condition as the §2.1 read-flip. So **read SNM < hold SNM** is structural, not incidental: it is the price of routing the read current *through* the storage node. Both margins shrink with $V_{DD}$ because the whole $I$–$V$ compresses as supply approaches $V_{th}$:

| Condition | $V_{DD}$ | Temp | SNM | Verdict |
|---|---|---|---|---|
| Hold (WL off) | 1.0 V | 25 °C | ~280 mV | best case, cell isolated |
| Read (WL on) | 1.0 V | 25 °C | ~180 mV | worst *operating* case |
| Read (WL on) | 0.8 V | 125 °C | ~90 mV | marginal |
| Read (WL on) | 0.6 V | 125 °C | ~20 mV | essentially zero — cell unusable |

**How SNM scales with $V_{DD}$ — and the $V_{DD,\min}$ floor it implies.** Two regimes. Well above threshold ($V_{DD}\gg 2V_{th}$) the whole $I$–$V$ scales with $V_{DD}$ and SNM tracks it *roughly linearly* ($\text{SNM}\approx c\,V_{DD}$, $c\approx0.2$ for read). But as $V_{DD}\to \sim\!2V_{th}$ the devices leave strong inversion, gain collapses, and SNM falls **super-linearly** — hence the table's $180\to90\to20$ mV as $V_{DD}$ goes $1.0\to0.8\to0.6$ V (a $1.7\times$ supply cut erasing $9\times$ the margin). The usable floor is set not by the *mean* SNM but by its spread. Random dopant fluctuation gives each cell an independent threshold offset $\sigma_{V_{th}}$ (Pelgrom, §12), inducing an SNM distribution of standard deviation $\sigma_{\text{SNM}}$; an $N$-cell array fails when its *worst* cell does, so the design constraint is

$$
\mu_{\text{SNM}}(V_{DD}) \;\ge\; k\,\sigma_{\text{SNM}}, \qquad k \approx Q^{-1}\!\big(1/N\big)\ \ (\sim\!5\text{ for }1\text{ Mb},\ \sim\!6\text{ for }1\text{ Gb}),
$$

and $V_{DD,\min}$ is the supply where equality bites. *Worked number:* with $\sigma_{\text{SNM}}\approx 18$ mV and $k=5$, a 1 Mb array needs $\mu_{\text{SNM}}\ge 90$ mV — which the table's read curve hits at $V_{DD}\approx 0.8$ V (its 90 mV/0.8 V row) *at this illustrative 125 °C corner*. Below that the 6T read cell is sub-$5\sigma$ and unusable; optimized cells with assists (§3) and the more favourable node projections of §12 push the real 6T floor to ~0.6–0.7 V, under which 8T takes over. Every 100 mV of $V_{DD}$ you *cannot* shed because of this floor is active energy you cannot save, since $P\propto V_{DD}^2$.

The last row of the table is the punchline: **at 0.6 V the 6T read SNM nearly vanishes.** You cannot size your way out — pushing $CR$ up to help read costs you $PR$ and write-ability, and process variation eats what remains. The 6T conflict is *structurally* unwinnable at low voltage — the motivation for §3's 8T and assists — and §2.3 shows the *write* side sets an equally hard floor from the opposite direction.

### 2.3 Write margin — the mirror, and how the six transistors get sized

Writing is read's opposite: instead of *resisting* a node flip you must *force* one. To write a 0 into a node $Q$ holding 1, the column driver pulls its bitline to ground and the access transistor must drag $Q$ down through the opposite inverter's trip point $V_M$ **against the pull-up PMOS** fighting to hold $Q$ at $V_{DD}$. At the tipping point $Q=V_M$ the access NMOS (source at BL $=0$, gate $=V_{DD}$) must sink more than the PMOS can source:

$$
I_{acc}(V_M) \;>\; I_{pu}(V_M).
$$

Both devices sit in triode near $V_M$; carrying the square-law currents through and cancelling the common factors leaves a **ratio** condition — the write mirror of $CR$ — governed by the **pull-up ratio**

$$
PR \;\equiv\; \frac{(W/L)_{acc}}{(W/L)_{pu}} \;\gtrsim\; 1 .
$$

The larger the access relative to the pull-up, the easier the write; the standard metrics are the **bitline write margin** (how far below $V_{DD}$ the bitline must fall to trip the cell) and the **word-line write margin**, both monotone in $PR$. *Worked number:* a write is comfortable when the bitline trips the cell with $\gtrsim 150$–200 mV to spare at the slow corner; $PR\approx1.3$–1.8 delivers that at nominal $V_{DD}$ and collapses — like read — as $V_{DD}\to V_{th}$, which is why **write** assists (negative bitline, supply collapse, §3) turn mandatory at the same nodes read assists do.

**Sizing all six at once.** Access strength appears in $CR=(W/L)_{pd}/(W/L)_{acc}$ (want it *small* → strong pull-down) *and* in $PR=(W/L)_{acc}/(W/L)_{pu}$ (want it *large* → weak pull-up), so the only way to keep both above 1 is the strict ordering

$$
\big(W/L\big)_{pd} \;>\; \big(W/L\big)_{acc} \;>\; \big(W/L\big)_{pu}.
$$

A canonical high-density sizing $(W/L)_{pd}:(W/L)_{acc}:(W/L)_{pu} = 2:1.35:1$ gives $CR=1.48$ and $PR=1.35$ — both safely inside the $CR,PR\approx1.2$–2.0 window. On FinFETs the widths *quantise* to integer fins, so this becomes a discrete search (e.g. 2-fin pull-down / 1-fin access / 1-fin pull-up) with the asymmetry a continuous width would supply borrowed instead from threshold flavour. **The pull-down being the widest device is the single most important sizing fact about the 6T cell** — it is what buys read stability.

---

## 3. Breaking the 6T conflict: 8T, 10T, and assist circuits

The 6T conflict exists for one reason: read and write share the access transistors and both route through the *storage nodes*. Remove that sharing and the conflict dissolves. That is exactly what the **8T cell** does — it adds a dedicated 2-transistor read port (a read wordline gating a transistor whose gate is one storage node, discharging a separate read bitline) that **senses the bit without ever touching $Q$/$\bar Q$.** Writes still use the original 6T access transistors; reads no longer disturb anything.

**Why the read current no longer touches the storage node — derived.** The read port is two series NMOS: a *read* transistor whose gate is tied to storage node $\bar Q$, in series with a *read-access* transistor gated by the read wordline (RWL), together discharging a precharged read bitline (RBL). The storage node drives only the **gate** of the read transistor — and a MOS gate is a capacitor, drawing *zero DC current* in steady state. So when RWL fires, the discharge path is RBL → read-access → read transistor → ground, and every electron of it is sourced by $C_{RBL}$; **none is drawn from $Q$ or $\bar Q$.** The storage nodes do not move — there is no $V_Q$ pedestal, no divider, no lobe-floor lift. Contrast the 6T, where the read current is *forced through* the access transistor into the storage node (§2.1). With the pedestal gone, the read butterfly *is* the hold butterfly, so the $\kappa V_Q$ term of §2.2 vanishes and the consequence is decisive:

$$
\text{8T read SNM} \;=\; \text{hold SNM} \;\gg\; \text{6T read SNM}
$$

Concretely, at 0.6 V the 6T read SNM is ~20 mV (dead) while the 8T is ~250 mV (the full hold margin). **That is why 8T is the low-voltage cell of choice** — near-threshold caches, register files, and anything that must run at aggressively scaled supply. Quantitatively, applying the §2.2 floor $\mu_{\text{SNM}}\ge k\sigma_{\text{SNM}}$ to the *hold* curve (which 8T read now follows) drops $V_{DD,\min}$ by ~0.15–0.25 V versus the 6T read curve — an 8T cache runs stably to ~0.5–0.55 V where a 6T dies at ~0.75 V, and since $P_{\text{active}}\propto V_{DD}^2$ that ~0.2 V of head-room is a $1-(0.55/0.75)^2\approx 46\%$ active-energy cut wherever the workload tolerates the lower clock. The trade is paid in **area (~30 %)**: eight transistors vs six is $+33\%$ by count, but the two added read devices are minimum-size and the real adder is the RWL/RBL wiring track, netting ~30 % cell area plus one extra read bitline per column. You buy read robustness and, as a bonus, a naturally multi-portable read path (§7, §11). The **10T cell** goes further, adding a *differential* read port (RBL/RBLB): a differential sense amp resolves a *difference* that grows from both rails while rejecting common-mode bitline noise, so it trips at a smaller per-line swing — and therefore a shorter bitline-discharge time — than the single-ended 8T RBL that must fall a full threshold below precharge. That is the speed corner, favoured in register files where read latency is on the critical path; deep-subthreshold 10T variants isolate the read further still, pushing $V_{DD,\min}$ toward ~0.3 V.

**The cheaper alternative — assist circuits.** Before spending 30 % area on 8T, designers first try to *widen the 6T margins dynamically* with per-operation assists, exploiting the fact that read and write never happen in the same cycle:

- **Read assist — wordline under-drive.** Lower the read wordline below $V_{DD}$ (e.g. $0.55\to0.40$ V) so the access transistor is weaker *only during read* → effective $CR$ rises $1.1\to1.4$, worth ~30–40 mV of read SNM.
- **Write assist — negative bitline** (drive BL below ground, e.g. $-0.2$ V) or **supply collapse** (droop the cell $V_{DD}$ during write) → the access transistor wins the write fight more easily, worth ~+50 mV write margin.

Assists cost a small charge pump and ~5–10 % area but let a 6T cell keep operating a node or two lower than it otherwise could. The design decision is a ladder: **6T + assists** while the margins hold, **8T** when read SNM collapses regardless of assist (increasingly mandatory at 3 nm and below, §12), **10T** when read *speed* also matters.

---

## 4. From cell to array: yield, soft errors, retention

A working bitcell is necessary but not sufficient — an SRAM is millions of them, and at that scale three array-level realities dominate, each a trade in its own right.

**Yield and repair.** SRAM bitcells use the tightest design rules on the die, so they catch random defects disproportionately; large caches would yield near zero unrepaired. The fix is **redundancy** — spare rows (2–4/bank) and columns (2–8) that a fuse remaps to defective addresses at wafer test (laser fuse: permanent, robust; eFUSE: electrically programmable, even in-field, smaller but slightly less reliable). Yield with $R$ spares follows a Poisson model — the array survives if at most $R$ defects land in it:

$$
Y_{\text{repaired}} \;=\; e^{-DA}\sum_{k=0}^{R}\frac{(DA)^k}{k!}
$$

where $D$ = defect density, $A$ = array area, $R$ = number of spares. A 1 Mb array at $DA\approx0.015$ yields $e^{-0.015}\approx$ ~98.5 % unrepaired, but a 64 Mb L3 slice (64 such 1 Mb banks in series → $DA=0.96$) falls to $e^{-0.96}\approx$ **38 %** unrepaired; 4 spare rows/bank ($R=4$, per-bank $DA=0.015$ ⇒ $P(\le4)\approx1$) restores the product to **>90 %**. (At a full 512 Mb, unrepaired yield is $e^{-7.7}\approx0.05\%$ — essentially scrap — which is why big caches *cannot* ship without repair.) Repair is not optional at scale.

**Soft errors.** A neutron (cosmic secondary) or alpha particle (package contaminant) can deposit enough charge to flip a cell — a **soft error**, rate ~100–1000 FIT/Mb at sea level (1 FIT = 1 fail per $10^9$ device-hours). This is not a defect (the cell is fine next write); it is a reliability tax that scales with array size and *worsens as the stored charge shrinks*. Detection/correction is a coding problem handled in §9; the array-level knobs are parity (detect only, for latency-critical L1) vs full ECC (correct, for L2 and beyond).

**Retention voltage.** The minimum supply at which an *idle* 6T cell still holds its bit is $V_{\text{retain}}\approx 0.4$–$0.5\,V_{DD,\text{nom}}$ (~0.3 V for an N5 cell nominal at 0.7 V). Drowsy/retention modes park unused SRAM at $V_{\text{retain}}$ to cut leakage while keeping data — the trade being that lower retention voltage saves more leakage but erodes SNM and soft-error immunity, so the entry/exit sequence must lower and raise $V_{DD}$ in a controlled window (~5–50 µs). See [Power_Reduction_Techniques](../../02_Power_and_Low_Power/03_Power_Reduction_Techniques.md) for how this fits state-retention power gating.

---

## 5. DRAM 1T1C: trading persistence for density

DRAM makes the opposite bet to SRAM at the deepest level: it **deletes the storage feedback loop.** A 1T1C cell is one access transistor and one capacitor — the bit is simply *charge present or absent* on $C_s$. No cross-coupled inverters means no self-restoring hold and no non-destructive read, but it also means ~6 $F^2$ instead of ~120 $F^2$: the density that makes gigabyte main memory economic.

**Where the $6F^2$ comes from.** With the feedback loop gone, the cell's *planar* footprint is just one access transistor plus the room to land a wordline and a bitline. In a folded-bitline layout the transistor sits at a wordline–bitline crossing on a pitch of $2F\times3F=6F^2$ ($F$ = minimum half-pitch feature; the $8F^2$ open-bitline layout trades this for better noise). The **storage capacitor is built in the third dimension** — a deep *trench* etched into the substrate, or a *stacked* cap towering above the transistor — so it adds the ~25 fF of $C_s$ without consuming planar area. That vertical-capacitor trick is the entire density argument: 6T spends ~120 $F^2$ on eight-plus coplanar devices and their cross-coupling wires; DRAM spends $6F^2$ and hides the hard part (the cap) in $z$ — the ~20× win of §1. Everything hard about DRAM is the bill for the missing feedback loop, and it comes in three parts.

**The read is destructive, and the signal is tiny.** Reading connects the storage cap $C_s$ (holding $V_{cell}\in\{0,V_{DD}\}$) to the bitline — **precharged to $V_{DD}/2$** and loaded by the large $C_{BL}$. Charge is conserved as the two caps share: before, $Q=C_{BL}\tfrac{V_{DD}}{2}+C_s V_{cell}$; after, both rest at the common $V_{BL}=Q/(C_{BL}+C_s)$, so the line moves by $\Delta V=V_{BL}-\tfrac{V_{DD}}{2}=\frac{C_s}{C_s+C_{BL}}\big(V_{cell}-\tfrac{V_{DD}}{2}\big)$ — magnitude (for a stored 1; the mirror for a 0):

$$
\Delta V \;=\; \frac{C_s}{C_s + C_{BL}}\cdot\frac{V_{DD}}{2}
$$

where $C_s$ = storage capacitance ($\approx$ 20–30 fF), $C_{BL}$ = bitline capacitance ($\approx$ 200–400 fF, set by how many cells share the line). The signal is a *capacitance ratio*, and because $C_{BL}\gg C_s$ it is minuscule: for $C_s=25$ fF, $C_{BL}=250$ fF, $V_{DD}=1.0$ V, $\Delta V\approx 45$ mV. Two consequences follow directly. First, sensing 45 mV in a noisy array demands a **differential sense amplifier** — a cross-coupled latch straddling the bitline pair that regeneratively amplifies the difference to full rail while rejecting common-mode noise (the two lines are matched, so supply/coupling noise cancels; CMRR ~30–40 dB). Second, that same amplification **restores** the cell: charge sharing left $C_s$ half-drained, so the sense amp must drive the full-rail value back into the cell every read. *Read = sense + rewrite*, always.

**The charge leaks — and the retention time is set by the worst cell.** With no feedback to replenish it, the stored charge bleeds off through sub-threshold, junction, gate, and dielectric paths ($I_{leak}\sim$ 2–20 fA/cell typical at 85 °C). Model the node as a capacitor discharging at roughly constant current, $V_{cell}(t)=V_{DD}-\frac{I_{leak}}{C_s}t$, so the time to lose a critical amount $\Delta V_{crit}$ of stored level is

$$
t_{ret} \;=\; \frac{C_s\,\Delta V_{crit}}{I_{leak}}
$$

where $\Delta V_{crit}$ = stored droop that pulls the read signal $\Delta V$ down to the sense-amp resolution — a far tighter bar than "charge halves," so only a couple hundred mV. *Worked number:* a *typical* cell ($C_s=25$ fF, $\Delta V_{crit}=0.2$ V, $I_{leak}=10$ fA) retains $t_{ret}=25\text{ fF}\times0.2\text{ V}/10\text{ fA}=0.5$ s. Yet JEDEC specs **~64 ms** (85 °C) — ~8× tighter — because a multi-gigabit die must survive its *leakiest tail cell* (~80 fA here ⇒ $t_{ret}=62$ ms), plus temperature (leakage roughly doubles per ~10 °C, hence ~2–4× longer retention per 20 °C cooler) and guardband. Refresh must therefore visit every cell inside the *worst-case* window, not the typical one — the subject of §6, and the reason variable-retention schemes (refresh weak rows often, strong rows rarely) pay off. Holding data at all requires periodically reading and rewriting every cell before it decays: **refresh**.

The upshot on the trade surface: DRAM buys ~20× density and pays with a destructive, restore-coupled read (~2–4× SRAM latency: a DDR4-3200 row-miss is $t_{RP}+t_{RCD}+t_{CL}\approx 41$ ns vs a row-hit's ~14 ns) and a standing refresh tax. **eDRAM** ports the 1T1C cell onto a logic process — denser than 6T (~2–3×), slower than SRAM (~5–10 ns), still refreshed — and has shipped as dense last-level cache (IBM POWER L3/L4, Intel Crystalwell L4).

---

## 6. The refresh tax: the price of volatility

Refresh is the recurring cost of DRAM's density bet, and it is a *bandwidth and power* tax, not merely a background chore. The model is simple and worth internalising. Every row must be visited within the retention window $t_{REFW}$ (64 ms); a refresh command takes $t_{RFC}$ during which the rank (all-bank refresh) is unavailable; commands are spaced $t_{REFI}$ apart. The fraction of time DRAM cannot serve real traffic is

$$
\text{Overhead} \;=\; \frac{t_{RFC}}{t_{REFI}}, \qquad t_{REFI} = \frac{t_{REFW}}{N_{cmd}} = \frac{64\text{ ms}}{8192} = 7.8125\ \mu\text{s}
$$

where $t_{RFC}$ = refresh cycle time (grows with density — more rows per command), $t_{REFI}$ = average refresh interval, $N_{cmd}$ = refresh commands per window (8192). The window $t_{REFW}=64$ ms is *exactly* the worst-cell $t_{ret}$ of §5, and $N_{cmd}=8192$ is a JEDEC convention, so all $R_{tot}$ rows are covered each window and one command refreshes $R_{tot}/8192$ of them: $t_{RFC}\approx (R_{tot}/8192)\,t_{RC}$ grows **linearly with die row count**. The tax **rises with capacity**, because a bigger die refreshes more rows per command so $t_{RFC}$ climbs while $t_{REFI}$ is fixed by the retention window:

| Density | $t_{RFC}$ | Overhead | Bandwidth lost (per 25.6 GB/s channel) |
|---|---|---|---|
| 4 Gb | 260 ns | 3.3 % | ~0.85 GB/s |
| 8 Gb | 350 ns | 4.5 % | ~1.15 GB/s |
| 16 Gb | 550 ns | 7.0 % | ~1.80 GB/s |
| 24 Gb | 650 ns | 8.3 % | ~2.13 GB/s |

At 16 Gb, **7 % of channel bandwidth evaporates into refresh** — on a DDR5-5600 channel that is ~3 GB/s, tens of millions of cache-line fetches per second that can never be served. It is also a **power** cost: in mobile LPDDR, self-refresh can be 30–40 % of DRAM power during idle. The mitigations all attack one term:

- **Per-bank / same-bank refresh** (DDR4/DDR5): refresh one bank (or one bank per group) while the rest stay live, so $t_{RFC}$ overlaps real traffic — DDR5 same-bank refresh cuts effective overhead to ~2.5–3 %.
- **Fine-granularity refresh**: fewer rows per command, more commands — shorter latency spikes, used hot.
- **Temperature-compensated refresh**: retention lengthens when cool, so drop the rate and reclaim bandwidth.
- **Targeted refresh**: refresh rows adjacent to a hammered row (a *RowHammer* defence, not a retention one).
- **Self-refresh**: an on-die counter runs refresh with the controller and I/O quiescent — the low-power sleep state (~15–30 mW vs ~65 mW auto-refresh for a x8 DDR4), at the cost of a ~100–200 ns exit latency.

**Off-chip DRAM comes in two flavours of the same bandwidth trade.** **HBM** (high-bandwidth memory) stacks DRAM dies over a wide, slow interface (1024-bit, ~6–9 Gbps/pin, ~1 TB/s/stack; HBM3E ~960 GB/s, 36 GB) via a TSV (through-silicon via) interposer — used where bandwidth dominates cost (AI accelerators, H100/B200). **GDDR** takes the narrow-and-fast corner (384-bit, ~24 Gbps/pin, ~1.15 TB/s; RTX 4090) — cheaper, no interposer, lower total bandwidth. Wide-and-slow vs narrow-and-fast is the same knob, set by whether pins or per-pin rate is the scarcer resource. Protocol and scheduling: [DDR_Controller](04_DDR_Controller.md).

---

## 7. Multi-port memories and register files: why ports cost quadratically

A single-port SRAM serves one access per cycle; a superscalar core's register file must serve *many* — a $W$-wide machine reads ~$2W$ operands and writes ~$W/2$ results per cycle (a 4-wide integer core needs roughly **8 read + 4 write ports**). The question is why you cannot simply keep adding ports, and the answer is a hard area law. **Each port adds a wordline (a horizontal wire across every cell in the row) and one or two bitlines (vertical wires down every cell in the column).** Count the wires: with $R$ read and $W$ write ports the cell must host $\sim(R+W)$ wordlines stacked vertically and $\sim(R+2W)$ bitlines side by side (writes are usually differential, reads single-ended). If the cell is **wire-limited**, its *height* is set by the wordline count and its *width* by the bitline count, both $\propto P$ with $P=R+W$, so area is the product of two quantities each linear in $P$:

$$
A_{\text{cell}} \;\approx\; \underbrace{(\text{height}\propto P)}_{\text{wordlines}} \times \underbrace{(\text{width}\propto P)}_{\text{bitlines}} \;\propto\; P^2 .
$$

Interpolating from the transistor-limited floor $A_{6T}$ (which dominates at small $P$, where the six devices — not the wires — set the size) to this wire-limited asymptote:

$$
A_{\text{cell}}(P) \;\approx\; A_{6T}\,\big(1 + \alpha P^2\big)
$$

where $P=R+W$ = total port count, $\alpha \approx 0.05$–0.10 (process/layout dependent). *Worked number:* a **2R1W** cell ($P=3$) gives $1+\alpha\cdot9\approx1.7$ (at $\alpha=0.075$), an **8R4W** cell ($P=12$) gives $1+\alpha\cdot144\approx11.8$ — a **~7× larger cell** for 4× the ports, and the pure wire-limited law $(R{+}W)^2$ alone predicts $(12/3)^2=16\times$. Access time degrades in parallel: the cell's linear dimension grows $\propto P$, so the bitline a read must discharge lengthens $\propto P$ and read delay climbs at least linearly (up to $\propto P^2$ for distributed RC) — every extra bitline loads the sense node, every extra wordline loads the decoder — so a monolithic 12-port cell is both huge and slow (>1.5 ns, often multi-cycle). Beyond ~8–12 ports a monolithic array is untenable.

**The escape is banking, and it is a pure conflict-vs-cost trade.** Split the file into $B$ banks with few ports each; total port bandwidth is the sum, but two accesses to the *same* bank in one cycle **conflict** and one must stall. More banks → less area and faster access → *more* conflicts (fewer banks to spread across):

| Register file (128×64b, 8R4W-equivalent) | Area (7nm) | Access | Bank-conflict rate |
|---|---|---|---|
| Monolithic 12-port | ~0.015 mm² | ~1.5 ns | 0 % |
| 2-bank, 6-port each | ~0.010 mm² | ~0.9 ns | ~15 % |
| 4-bank, 3-port each | ~0.008 mm² | ~0.6 ns | ~35 % |

Real cores sit at 2–4 banks (integer) and 2 (FP/SIMD), the knee where conflict cost balances area/timing win: ARM Cortex-A78 128–160 entries at 4R2W (banked); Apple M1 ~350 at ~6R3W; Zen 4 ~180 at 4R2W; Golden Cove ~280 at ~6R4W. GPUs, needing dozens of operand reads per cycle across a warp, bank *heavily* (e.g. 16 banks of 1R1W) and lean on the compiler to lay out registers conflict-free. Two further levers matter: a **bypass network** (compare each read address against pending writes and forward the write data directly) hides the write-then-read latency that would otherwise cost a cycle on back-to-back dependents; and the **SRAM-vs-flip-flop** choice — flip-flop arrays give ~50 ps reads and trivial multi-porting but ~10× the area/bit, so they win only for small (16–32-entry), very-high-port files, while anything 64+ entries goes SRAM. **Column multiplexing** is the dual knob for single-port arrays: sharing one sense amp across $M$ columns trades sense-amp area for a taller, narrower array (better floorplan fit, longer bitlines).

The through-line: **port count is the most expensive axis in memory design.** It is why the physical register file is a first-order area/timing constraint in an OoO core ([OoO_Execution](../02_CPU/03_OoO_Execution.md) §2.3), and why width rarely exceeds 6–8.

---

## 8. The memory compiler: SRAM as generated IP

You do not draw SRAM by hand — a **memory compiler** (ARM Artisan, Synopsys, foundry) synthesises a custom array from a specification and hands back the views the flow needs. The reason this matters conceptually is that the compiler *exposes the trade surface as knobs*, and picking them is a PPA (power, performance, area) decision:

| Knob | Trade it sets |
|---|---|
| Depth × width | capacity vs aspect ratio |
| Ports (1P / 2P / 1R1W …) | bandwidth vs the quadratic area of §7 |
| Mux factor (4/8/16) | sense-amp area vs array height, bitline length (speed) |
| Power mode (high-speed / high-density / low-leakage) | which corner of the trade surface the bitcell targets |
| Redundancy (spare rows/cols) | die area vs yield (§4) |

What comes back is a set of *abstractions*, not transistors: a **`.lib`** (Liberty timing/power for STA (static timing analysis) — the load-bearing number is **access time = clock-to-Q**, ~0.8–2.0 ns for embedded SRAM at 28 nm, which sets your pipeline's memory latency; plus setup/hold, which are tighter on address than data because address decode is on the critical path), a **`.lef`** (physical abstract for place-and-route), a behavioural **`.v`** for simulation, and the **`.gds`** for manufacturing. The practical discipline is to close timing on the slow corner (SS, low-$V$, hot) and hold on the fast corner (FF, high-$V$, cold), and to remember that the compiler's "high-density" bitcell is a *different, less stable* cell than its "high-speed" one — the §2 trade, exposed as a menu item.

---

## 9. ECC: coding theory for memory

Soft errors (§4) and DRAM bit-flips demand error correction, and the right way to understand ECC is as **coding theory, not a check-bit recipe.** The single governing quantity is the **minimum Hamming distance** $d_{\min}$ of the code — the fewest bit-flips that turn one valid codeword into another. From it, everything follows:

$$
\text{correct } t \text{ errors and detect } s\ge t : \quad d_{\min} \;\ge\; t + s + 1
$$

*Why that bound (sphere-packing).* Put a Hamming ball of radius $t$ around every codeword — all words within $t$ flips of it. Correcting $t$ errors means these balls are **disjoint**, so a received word lands in exactly one and you decode to its centre. *Additionally* detecting $s\ge t$ more means no *other* codeword lies within distance $t+s$ of a codeword, else an $(t{+}s)$-reachable corruption could look either correctable-toward-the-wrong-centre or clean. The nearest two codewords are $d_{\min}$ apart, and jointly needing radius-$t$ balls disjoint *and* $s$ further steps still short of the next centre gives $t+s+1\le d_{\min}$. Set $t=s=1$ → SEC ($d_{\min}\ge3$); $t=1,s=2$ → SECDED ($d_{\min}\ge4$).

To *correct* a single-bit error you need $d_{\min}=3$ (each valid word sits in its own radius-1 ball, so a 1-flip word has a unique nearest neighbour). To *correct one and detect two* (SECDED — the workhorse for caches and register files) you need $d_{\min}=4$, which a **Hamming code plus one overall parity bit** achieves. The parity bit is what distinguishes a correctable single flip (odd parity change) from an uncorrectable double flip (even parity change, syndrome non-zero) — without it the code cannot tell them apart and would "correct" a double error into a *third* wrong value.

**The (72,64) overhead argument.** How many check bits $r$ does 64 data bits need? Each of the $2^r$ syndromes must name one of the $n$ correctable single-bit positions plus the no-error case: $2^r \ge n + 1$ with $n = 64 + r$. For $r=7$: $2^7 = 128 \ge 72$ ✓. Add the one SECDED parity bit → **8 check bits on 64 data bits = Hamming(72,64), a 12.5 % overhead.**

This is the **Hamming (sphere-packing) bound** made concrete: the general SEC form $2^{k}\,(1+n)\le 2^{n}$ — each of $2^k$ codewords owning a disjoint radius-1 ball of $1+n$ words, all packed into $2^n$ — rearranges to $2^{r}\ge n+1$ with $r=n-k$, the very inequality above (equivalently $2^r\ge k+r+1$ counting data bits $k$). Because $r$ grows only *logarithmically* ($r\approx\log_2 k$) while data grows linearly, **SECDED overhead falls with word width**:

| Data bits $k$ | Hamming $r$ | +parity | Total check | Overhead |
|---|---|---|---|---|
| 8 | 4 | 1 | 5 | 62.5 % |
| 16 | 5 | 1 | 6 | 37.5 % |
| 32 | 6 | 1 | 7 | 21.9 % |
| **64** | **7** | **1** | **8** | **12.5 %** |
| 128 | 8 | 1 | 9 | 7.0 % |
| 256 | 9 | 1 | 10 | 3.9 % |

64-bit is the sweet spot: wider codewords amortise the check bits further (128+9, 256+10) but lose a whole line to any *uncorrectable* double error and grow the decode tree. On read, XOR the recomputed check bits against the stored ones to get the syndrome: zero = clean; non-zero with odd parity = single error *at the position the syndrome encodes*, flip it; non-zero with even parity = double error, flag uncorrectable — the syndrome names the *position*, the parity bit names the *kind* (single vs double), which is precisely how $d_{\min}=4$ buys correct-1-detect-2.

**When to escalate.** SECDED corrects one bit per word; escalate when the error model outgrows that:

- **BCH / multi-bit codes** ($t>1$): generalise Hamming, adding up to $m$ check bits per corrected bit ($m=\lceil\log_2(n{+}1)\rceil$, so $r\le m\,t$). Over GF($2^9$) ($n=511$): correcting $t=2$ needs $r=18$ → **BCH(511,493)**, ~3.7 % overhead; $t=3$ needs $r=27$ → **BCH(511,484)**, ~5.6 %; each extra corrected bit costs another $m=9$ check bits and a longer Galois-field decode (several cycles). Overhead climbs steeply for *short* words (a 64-bit 2-error BCH pays ~22 %), so multi-bit BCH earns its keep only on wide blocks — NAND flash pages, large server SRAM — where flips *accumulate* faster than scrubbing clears them.
- **Chipkill** (DRAM): a whole DRAM *chip* can fail, flipping many bits of one word at once — far beyond SECDED. Chipkill **spreads each codeword across many chips** so any single-chip failure corrupts at most one (or a few) symbols per word, which a symbol-oriented code (Reed–Solomon-like) then corrects. This is the standard for server memory: it converts a catastrophic multi-bit burst into a correctable single-symbol error.

**Scrubbing** is the temporal complement: single-bit errors accumulate, and two in one word defeats SECDED, so a background scrubber walks the array during idle, reads-checks-corrects, and resets the count. The rate need only outpace accumulation — a 32 MB ($=256$ Mb) L3 at ~100 FIT/Mb accrues $256\times100=25{,}600$ FIT, i.e. one soft error every $10^9/25{,}600\approx 39{,}000$ h (~4.5 yr — recall 1 FIT $=1$ fail/$10^9$ h, §4). A 24 h scrub therefore clears each error long before a second can land in the *same* 64-bit word: the array sees only $\sim\!6\times10^{-4}$ errors per 24 h window across its ~4M words, so the odds of two colliding in one word before a scrub sit below $\sim\!10^{-13}$. That vast margin is why scrub intervals are set by the far higher *avionics-altitude / large-array* rates, not this nominal one. The goal of the whole stack is to push **silent data corruption** (an undetected wrong bit reaching software) below the server target of ~1 per $10^{17}$ device-hours; parity detects but cannot correct, SECDED corrects one, BCH/chipkill extend the reach, and end-to-end CRC guards the paths *between* protected structures.

---

## 10. CAM and TCAM: memory that compares itself

Ordinary memory answers *"what is at this address?"*. A **content-addressable memory** answers the inverse — *"which address holds this value?"* — by comparing the search key against **every stored entry in parallel, in one cycle.** That parallelism is the entire value proposition and the entire cost. Mechanically, a CAM cell is a storage cell (a 6T core) plus a small comparison circuit that ties into a per-entry **match line**: the match line is precharged high, and *any* mismatching bit in the entry pulls it low (a wired-NOR across the row). An entry matches only if every bit matches; a priority encoder then reduces the match lines to an index.

**The cost is power and area, and it is structural.** Where an SRAM read activates *one* row, a CAM search activates *all* $N$ entries every cycle — every cell drives a comparison, every match line precharges and (mostly) discharges.

**The energy law, derived.** An indexed SRAM read decodes the address and asserts exactly *one* wordline, so its dynamic energy is $\sim$(one row's bitline swing) — $O(1)$ in table size. A content search has *no* address (the queried value could sit in any entry), so it must present the key to **all $N$ rows at once** and let every match line evaluate: each of the $N$ match lines $C_{ML}$ precharges to $V_{DD}$ and (for the all-but-one mismatching rows) discharges, while every search-line pair toggles down all $N$ rows —

$$
E_{\text{search}} \;\approx\; N\,\big(C_{ML}+C_{SL}\big)\,V_{DD}^2 \;=\; O(N), \qquad\text{versus}\qquad E_{\text{SRAM}} = O(1).
$$

That $O(N)$-vs-$O(1)$ gap **is** the 5–20× per-bit penalty below — you pay, every cycle, to interrogate the whole table instead of one indexed row. Per bit, that is:

| Operation | Relative energy |
|---|---|
| SRAM read (1 row active) | 1× |
| Binary CAM search (all rows active) | 5–8× |
| TCAM search (all rows, 2× storage) | 10–20× |

The area is structural too: a binary-CAM cell is a 6T core plus a ~4-transistor XOR-to-matchline comparator = 9–10T (~1.6×); a **TCAM** cell stores three states (0/1/X ⇒ two storage bits) plus compare = ~16T (~2.7×) and doubles the search energy again. So you use a CAM **only where parallel exact-match against a whole table is the operation**, and its speed is worth the burn: a **TLB** compares a virtual page number against all resident translations in one cycle ([TLB_and_Virtual_Memory](02_TLB_and_Virtual_Memory.md)); a fully-associative cache tag array does the same; network switches match addresses. The design rule is a latency/associativity test: **pay CAM energy iff (a)** the lookup is on a per-access critical path that cannot absorb the multi-cycle latency of the alternative (a TLB must resolve *every* load/store address in ~1 cycle) **and (b)** the table is fully-associative by requirement, not by choice. Fail either and an **SRAM hash table** ($O(1)$ energy, one row active, 2–5 cycles) or a set-associative array dominates — which is exactly why a switch ASIC splits ~70 % SRAM hash for exact match, ~30 % TCAM for what genuinely needs it.

**TCAM** adds a third stored state — *don't-care* (X) — at ~2× the binary-CAM cell (two storage bits encoding 0/1/X). Don't-care is what makes **longest-prefix match** a single-cycle operation: an IP route `10.1.0.0/16` stores 16 real bits and 16 X's, so a lookup matches all prefixes it falls under at once, and the priority encoder (entries ordered longest-prefix-first) returns the most specific. Nothing else does variable-length prefix match in $O(1)$ — which is why core routers spend the power (a datacenter switch may carry ~18 Mb of TCAM burning 5–15 W). The power mitigations are all ways to *not* activate every entry: **partition** by prefix length and search only the relevant block, **selectively precharge** a subset per cycle (throughput for power), or **gate** search lines that are don't-care across a block.

---

## 11. Compute-in-memory: moving the ALU onto the bitline

The premise of compute-in-memory (CIM / PIM) is a measured fact about modern workloads: **moving the data costs far more than computing on it.** In a von Neumann machine a MAC is ~100–1000 pJ once you count reading the operand out of the array, across a bus, into an ALU, and writing the result back — and the *movement* is 10–100× the arithmetic. For memory-bound kernels (AI inference, graph analytics) that ratio is the whole energy budget. CIM attacks it by **doing the arithmetic where the bits already sit**, so operands never leave the array.

**The core trick is to read many cells at once and let the bitline do the sum.** In SRAM CIM, assert *multiple* read wordlines simultaneously; each activated cell either discharges the shared read bitline or not, so the bitline settles to a voltage encoding *how many* cells discharged:

$$
V_{RBL} \;=\; V_{DD} - n\cdot\Delta V, \qquad n = \sum_i X_i \wedge W_i
$$

where $X_i$ = input (wordline drive) on row $i$, $W_i$ = stored weight (bit) in cell $i$, $n$ = popcount of their AND. That single analog settle *is* a dot product; an ADC on the bitline digitises it, and 128 columns compute 128 dot products in parallel — a full matrix-vector row per access, at ~5–10 fJ/MAC versus ~100 pJ for the read-ALU-writeback path. **ReRAM crossbars** push the idea to its physical limit: a cell's conductance $G$ *is* the weight, Ohm's law does the multiply ($I=V\!\cdot\!G$) and Kirchhoff's law sums the currents on each column, so $Y=GX$ falls out in one $O(1)$ step at ~1 fJ/MAC.

**The trade-off is that you have converted a digital operation into an analog one, and inherited analog's problems.** The energy win is real (10–100×, sometimes quoted higher against full-system baselines), but: the **ADC** dominates area/energy and caps precision at ~3–4 effective bits; **process variation** offsets each column, needing calibration; **analog noise** accumulates with array height, capping usable columns at ~128–256; and it is **integer/fixed-point only** — floating-point must be decomposed in software. ReRAM adds device-level costs: limited endurance ($10^6$–$10^9$ writes vs SRAM's $>10^{15}$), conductance drift, non-linear programming, and extra mask layers. CIM therefore wins decisively for **low-precision, weight-stationary inference** and not much else.

A gentler, digital cousin is **near-memory processing** — keep computation exact but move the compute *unit* next to the array rather than into the cell. **HBM-PIM** (Samsung Aquabolt-XL) drops a small MAC/SIMD engine beside each DRAM bank, feeding it from the bank's *internal* bandwidth (~4× the host interface) instead of over the pin-limited HBM link. For a large GEMM the weights never cross the host boundary — ~4× speedup and ~10× energy on the streaming-bound case — at the cost of a custom command path, weight layout constraints, and host↔PIM synchronisation. Both CIM and near-memory are the same idea as §0's trade surface pushed to its edge: *stop treating storage and compute as separate, and the data-movement cost that dominates everything else disappears* — if you can pay the precision and programmability price. See [NPU_Accelerators](../06_NPU/01_NPU_Accelerators.md) for where these land in accelerator design.

---

## 12. Scaling limits and emerging non-volatile memories

Every trade on this page tightens as processes scale, and the memory cell — not logic — is increasingly the limiter.

**The 6T bitcell is running out of margin.** As transistors shrink, the access device does not scale as favourably as the pull-down, so $CR$ drifts toward 1 exactly as the §2 conflict predicts, while random dopant fluctuation blows up threshold spread. The Pelgrom law $\sigma_{V_{th}}\propto 1/\sqrt{WL}$ means the *smallest* transistors have the *worst* variation, precisely where margin is thinnest:

| | 7 nm | 5 nm | 3 nm | 2 nm (proj.) |
|---|---|---|---|---|
| 6T cell area (µm²) | 0.027 | 0.021 | 0.015 | 0.010 |
| $V_{DD,\min}$ (read-stable) | 0.70 V | 0.65 V | 0.55 V | 0.45 V |
| Read SNM at $V_{DD,\min}$ | 120 mV | 90 mV | 60 mV | 40 mV |
| Cell ratio $CR$ | 1.5 | 1.3 | 1.1 | 1.0 (marginal) |
| $\sigma_{V_{th}}$ | 30 mV | 35 mV | 45 mV | 55 mV |

By 2 nm, $CR\to1.0$ leaves almost no gap between "read-stable" and "writable," and $\sigma_{V_{th}}$ is ~40 % of the nominal margin — forcing 5–6σ guardbands, mandatory read/write assists (§3), bitline segmentation to fight leakage from hundreds of off-cells on a line, and increasingly the **8T cell** (its ~30 % area is paid back by running ~0.1 V lower, cutting active power ~33 % since $P\propto V_{DD}^2$). **DRAM scaling** hits a different wall: the capacitor must keep ~25–30 fF (below ~10 fF the $\Delta V$ of §5 drops under the sense floor and retention violates spec) while its footprint shrinks, so the capacitor grows *taller* — aspect ratios past 30:1 (1z) toward 50:1 (1a nm), near the etch limit even with high-k dielectrics, and the access transistor must hold $I_{\text{off}}<1$ fA yet pass $>30$ µA, driving DRAM-specific 3D transistors (RCAT, saddle-fin).

**These walls are why non-volatile cells are entering the cache hierarchy.** **STT-MRAM** stores the bit as the magnetisation of a magnetic tunnel junction (parallel = low resistance = 0, anti-parallel = high = 1); it is comparable to 6T in area, non-volatile (zero standby leakage, >10 yr retention), but writes are slow (10–35 ns, current-switched) — a good trade for a large, mostly-idle LLC where leakage, not write latency, is the enemy. **SOT-MRAM** routes the write current through a separate channel instead of the tunnel barrier, cutting write latency to SRAM-like 1–5 ns and giving effectively unlimited endurance, at ~2× area — the leading candidate to replace SRAM in advanced-node L3/L4 where 100+ MB of SRAM would burn tens of watts of leakage. **eMRAM** uses the same MTJ as an embedded-NVM replacement for eFlash (8–64 MB in the BEOL metal stack, addable to any logic node without touching the transistors), for firmware, secure boot, and persistent config. Each is another point on the §1 surface — trading write speed or area for the non-volatility that SRAM and DRAM cannot offer.

---

## Numbers to memorize

| Parameter | Typical | Range | Why this value (section) |
|---|---|---|---|
| 6T cell ratio $CR$ | 1.3 | 1.2–2.0 | read stability, $V_Q=(V_{DD}-V_{th})/2CR$ (§2) |
| 6T pull-up ratio $PR$ | 1.5 | 1.2–2.0 | write-ability, mirror of $CR$ (§2) |
| 6T read SNM @ nominal | ~180 mV | 20–280 | hold > read; collapses at low $V_{DD}$ (§2) |
| 6T cell area | ~0.02 µm² (N5) | 0.010–0.027 | ~120 $F^2$; densest logic-rule cell (§1, §12) |
| 8T area penalty | ~30 % | — | buys read-decoupling, low-$V_{DD}$ (§3) |
| DRAM cell | ~6 $F^2$ | — | ~20× denser than 6T; no feedback (§1, §5) |
| DRAM read signal $\Delta V$ | ~45 mV | 30–70 | $C_s/(C_s{+}C_{BL})\cdot V_{DD}/2$ (§5) |
| DRAM retention | 64 ms | 32–64 | leakage vs sense floor (§5, §6) |
| Refresh interval $t_{REFI}$ | 7.8 µs | — | 64 ms / 8192 (§6) |
| Refresh overhead (16 Gb) | 7 % | 3–10 % | $t_{RFC}/t_{REFI}$, grows with density (§6) |
| SECDED overhead (64b) | 12.5 % | — | Hamming(72,64), $d_{\min}=4$ (§9) |
| SRAM soft-error rate | ~100 FIT/Mb | 100–1000 | neutron/alpha flux (§4) |
| Port-area exponent | quadratic | $\alpha P^2$ | wordline+bitline per port (§7) |
| CAM/TCAM search power | 10–20× SRAM | 5–20× | all $N$ entries active/cycle (§10) |
| SRAM CIM energy/MAC | ~5–10 fJ | 1–10 | bitline compute vs ~100 pJ read+ALU (§11) |
| Embedded SRAM access | ~1 ns | 0.8–2.0 | compiler clock-to-Q @ 28 nm (§8) |
| SNM ideal ceiling | $\tfrac12 V_{DD}$ | — | infinite-gain butterfly limit; real ~0.3–0.4 $V_{DD}$ (§2) |
| $V_{DD,\min}$ floor rule | $\mu_{\text{SNM}}\!\ge\!k\sigma_{\text{SNM}}$ | $k\!\approx\!5$–6 | worst cell of an $N$-cell array (§2–§3) |
| DRAM retention model | $t_{ret}=C_s\Delta V_{crit}/I_{leak}$ | — | worst tail cell sets the 64 ms spec (§5) |
| SEC (Hamming) bound | $2^r\!\ge\!k{+}r{+}1$ | $r\!\approx\!\log_2 k$ | sphere-packing; (72,64) for 64b (§9) |
| Port area (worked) | 2R1W→8R4W ≈ 7× | — | $A_{6T}(1{+}\alpha P^2)$, $P{=}R{+}W$ (§7) |

**Memory-technology one-liners:** SRAM 6T = fast, non-destructive, self-restoring, poor density, dies at low $V_{DD}$. 8T = 6T + isolated read port = low-$V_{DD}$ robustness for 30 % area. DRAM 1T1C = 20× density, destructive read, needs refresh. MRAM = non-volatile, zero leakage, slow writes. CAM = parallel match at 10–20× the power.

---

## Worked problems

**1 — Read-disturb margin and the $V_{DD}$ wall (§2).** A 6T cell has $CR=1.33$, $V_{th}=0.3$ V, $V_m=0.4\,V_{DD}$. At $V_{DD}=1.0$ V the disturbed node reaches $V_Q=(1.0-0.3)/(2{\times}1.33)=0.263$ V, safely below $V_m=0.40$ V (margin 137 mV). Scale to $V_{DD}=0.6$ V: $V_m=0.24$ V and $V_Q=(0.6-0.3)/2.66=0.113$ V — margin only 127 mV *nominally*, but process spread ($\sigma_{V_{th}}$) plus the WL-on SNM collapse push read SNM to ~20 mV at hot corner. This is why the cell is unusable at 0.6 V and why 8T (read SNM = hold SNM ≈ 250 mV) is the answer.

**2 — Refresh bandwidth tax (§6).** DDR4-3200, 25.6 GB/s, 16 Gb chips: $t_{RFC}=550$ ns, $t_{REFI}=7.8125$ µs → overhead $=550/7812.5=7.0\%$. Lost bandwidth $=25.6\times0.070=1.80$ GB/s $=$ ~28 M cache lines/s that can never be served. DDR5 same-bank refresh overlaps other banks, dropping effective overhead to ~2.5–3 % — recovering most of it.

**3 — SECDED overhead, derived (§9).** Protect a 64-byte line (512 data bits). Check bits $r$ satisfy $2^r \ge (512+r)+1$; $r=10$ gives $1024\ge523$ ✓. But practice splits the line into eight 64-bit sub-words, each with its own Hamming(72,64): $8\times8=64$ check bits = 8 bytes = **12.5 %** overhead — *and* up to 8 independent single-bit errors per line are correctable (one per sub-word), which a single 522-bit code could not do. The split trades a hair of overhead for far better multi-error coverage.

**4 — Why 8 read ports for a 4-wide core (§7).** Four instructions × 2 source operands = 8 reads/cycle minimum (3-source ops like FMA push it higher), so the PRF needs ~8 read ports even though only 4 instructions issue. The quadratic port-area law ($A\propto\alpha P^2$) then forces banking rather than a monolithic 8R4W array — the trade taken by every core in §7's table.

**5 — The butterfly ceiling and the $V_{DD,\min}$ floor (§2.2).** An infinite-gain inverter with balanced $V_M$ gives the SNM ceiling $\tfrac12 V_{DD}=500$ mV at $V_{DD}=1.0$ V; a real cell reaches only ~280 mV (hold) / ~180 mV (read), the gap being finite gain. A 1 Mb array ($k\approx5$) with $\sigma_{\text{SNM}}=18$ mV needs read $\mu_{\text{SNM}}\ge5\times18=90$ mV; the read curve (180→90→20 mV over $V_{DD}=1.0\to0.8\to0.6$ V) reaches 90 mV right at **$V_{DD}\approx0.8$ V** at this 125 °C corner (optimized cells + assists reach ~0.6–0.7 V, §3/§12). An 8T cell follows the *hold* curve (~250 mV even at 0.6 V), clearing 90 mV well below 0.5 V and unlocking the ~46 % active-energy cut of §3.

**6 — DRAM retention, typical vs worst cell (§5).** $C_s=25$ fF, $\Delta V_{crit}=0.2$ V. A typical cell ($I_{leak}=10$ fA) retains $t_{ret}=25\text{ fF}\times0.2\text{ V}/10\text{ fA}=0.5$ s; the leakiest tail cell (~80 fA) retains only $25\text{ fF}\times0.2\text{ V}/80\text{ fA}=62$ ms — which is why JEDEC pins $t_{REFW}=64$ ms to the *worst* cell, not the average. The whole array pays an ~8× refresh penalty for a few outliers, the motive for row-granular variable-retention refresh (§5–§6).

**7 — CAM vs SRAM-hash, and why latency decides (§10).** A 512-entry TLB search drives all $N=512$ match lines, $E_{\text{search}}\approx N(C_{ML}+C_{SL})V_{DD}^2$ — the $O(N)$ cost behind §10's 10–20×/bit. An SRAM hash touches one row ($O(1)$) but resolves in 2–5 cycles. The TLB pays CAM because translation sits on *every* load/store's critical path and must finish in ~1 cycle; a routing table with a 100-cycle budget hashes instead. **The crossover is latency and required associativity, not table size.**

---

## Cross-references

- **Down the stack (what these cells are built from):** [CMOS_Fundamentals](../../00_Fundamentals/01_CMOS_Fundamentals.md) §2 (the inverter VTC and gain), §3 (noise margins and the regenerative property), §9 (Pelgrom $\sigma_{V_{th}}$ variation) — the device-level primitives this page's §2 butterfly/SNM, read-disturb, and write-margin derivations build on (CMOS §12 carries a companion bitcell view).
- **Up the stack (what builds on these arrays):** [Cache_Microarchitecture](01_Cache_Microarchitecture.md) (SRAM/eDRAM organised into set-associative caches; tag CAMs), [Cache_Coherence](05_Cache_Coherence.md) (SRAM/CAM structures for directory sharers and transient transaction state), [TLB_and_Virtual_Memory](02_TLB_and_Virtual_Memory.md) (the fully-associative CAM of §10), [DDR_Controller](04_DDR_Controller.md) (the DRAM protocol/scheduling view of §5–§6), [OoO_Execution](../02_CPU/03_OoO_Execution.md) §2 (the physical register file whose port cost is §7), [NPU_Accelerators](../06_NPU/01_NPU_Accelerators.md) & [GPU_Architecture](../05_GPU/01_GPU_Architecture.md) (consumers of the CIM/near-memory of §11 and the banked register files of §7), [Performance_Modeling_and_DSE](../01_Modeling/01_Performance_Modeling_and_DSE.md) (consumes this page's DRAM latency/bandwidth and SRAM access time as the $p_{\text{mem}}$ and roofline $\beta$ of its CPI stack).
- **Adjacent / sibling structures:** [Async_Design_and_CDC](../../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) §5 (the async FIFO — a CDC structure that happens to be a memory), [DFT_and_ATPG](../../06_Signoff/02_DFT_and_ATPG.md) §7 (memory BIST and March tests that verify these arrays), [Power_Reduction_Techniques](../../02_Power_and_Low_Power/03_Power_Reduction_Techniques.md) (SRAM retention/drowsy modes of §4), [Floating_Point](../../00_Fundamentals/04_Floating_Point.md) (ECC context).

---

## References

1. Rabaey, J., Chandrakasan, A., and Nikolić, B., *Digital Integrated Circuits: A Design Perspective*, 2nd ed., Prentice Hall, 2003. Ch. 12, SRAM/DRAM cells, sensing, and SNM.
2. Seevinck, E., List, F., and Lohstroh, J., "Static-Noise Margin Analysis of MOS SRAM Cells," *IEEE JSSC*, 22(5), 1987. The SNM/butterfly-curve definition of §2.
3. Chang, L. et al., "Stable SRAM Cell Design for the 32 nm Node and Beyond," *VLSI Symp.*, 2005. The 8T read-decoupled cell of §3.
4. JEDEC, *DDR4 (JESD79-4)* and *DDR5 (JESD79-5)* SDRAM Standards. Refresh timing, $t_{RFC}$/$t_{REFI}$, same-bank refresh of §6.
5. Hamming, R.W., "Error Detecting and Error Correcting Codes," *Bell System Technical Journal*, 29(2), 1950. The Hamming-distance foundation of §9.
6. Lin, S. and Costello, D.J., *Error Control Coding*, 2nd ed., Pearson, 2004. SECDED, BCH, and the distance bound of §9.
7. Pagiamtzis, K. and Sheikholeslami, A., "Content-Addressable Memory (CAM) Circuits and Architectures: A Tutorial and Survey," *IEEE JSSC*, 41(3), 2006. The CAM/TCAM power/area analysis of §10.
8. Verma, N. et al., "In-Memory Computing: Advances and Prospects," *IEEE Solid-State Circuits Magazine*, 11(3), 2019. The bitline-compute energy model of §11.
9. Kwon, Y.-C. et al., "A 20 nm 6 GB Function-In-Memory DRAM (HBM-PIM)," *ISSCC*, 2021. The near-memory processing of §11.
