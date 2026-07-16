# Memory Circuits — the Bit-Storage Trade Surface

> **Prerequisites:** [CMOS_Fundamentals](../00_Fundamentals/01_CMOS_Fundamentals.md) §12 (the 6T cell schematic, VTC, noise margins — the transistor-level foundation this page reasons *above*).
> **Hands off to:** [Cache_Microarchitecture](07_Cache_Microarchitecture.md) (how these arrays are organised into caches), [DDR_Controller](10_DDR_Controller.md) (the DRAM interface/scheduling view), [TLB_and_Virtual_Memory](08_TLB_and_Virtual_Memory.md) (the CAM of §10 in its natural home).
> **Scope:** cell- and array-level *circuits* — what physical state stores a bit, what it costs to read/hold/write it, and why each memory technology lands where it does on that trade surface. Protocol/timing (DDR), cache organisation, memory BIST ([DFT_and_ATPG](../06_Signoff/02_DFT_and_ATPG.md) §7), and the async FIFO ([Async_Design_and_CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) §5) live on their own pages.

---

## 0. Why this page exists

Storing a bit is a **bet, not a given**. A bit needs a physical state that survives three abuses — being *held* against leakage and noise, being *read* without destroying itself, and being *written* on demand — and the transistors you spend to win one of those abuses are the same ones that lose you another. There is no cell that is simultaneously the densest, the fastest, the most stable, and the lowest-energy. Every memory on this page — 6T/8T SRAM, 1T1C DRAM, CAM, MRAM, ReRAM, compute-in-memory — is a **named compromise on one trade surface**, and the only interesting question about each is *which cost it traded away and how the rest of the system pays for it.*

This page derives the memory zoo from that surface instead of cataloguing schematics. The transistor-level 6T cell, its butterfly curve, and its margin equations already live one level down in [CMOS_Fundamentals](../00_Fundamentals/01_CMOS_Fundamentals.md) §12; here we ask the architect's questions. Why does the 6T cell force *read stability* and *write-ability* to fight over the same access transistors, and why does that fight become unwinnable at low $V_{DD}$ — the reason 8T and 10T exist? Why does DRAM throw away the storage feedback loop entirely, and what does that one decision cost in destructive reads and refresh bandwidth? Why is ECC a *coding-theory* question about Hamming distance, not a bit-field? Why does a CAM burn 10–20× the power of an SRAM to answer one question, and why do we pay it anyway for a TLB? And what breaks — usefully — when compute-in-memory stops separating storage from arithmetic. By the end you should be able to place any cell on the surface and predict its failure mode, rather than recite its transistors.

A single quantitative motif recurs and is worth holding from the start: **memory signal is a ratio, and margin collapses as $V_{DD}$ scales.** SRAM read stability is a ratio of transistor strengths; DRAM read signal is a ratio of capacitances; both shrink faster than $V_{DD}$ does. Low-voltage operation, not raw area, is the modern memory wall.

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

The read-stability condition is worth one derivation because it shows the margin is a *ratio* that scales badly. During a read of a "0" node, the access transistor (in saturation) and the pull-down (in its linear region, $V_{DS}\approx 0$) form a voltage divider. Equating their currents gives the disturbed node voltage:

$$
V_Q \;\approx\; \frac{1}{2\,CR}\,\big(V_{DD}-V_{th}\big)
$$

where $V_Q$ = voltage the "0" node rises to during read, $CR$ = cell ratio, $V_{th}$ = access-transistor threshold. The cell survives iff $V_Q$ stays below the feedback inverter's switching threshold $V_m \approx 0.4\,V_{DD}$. Plugging a textbook corner ($V_{DD}=1.0$ V, $V_{th}=0.3$ V, $CR=1.33$):

$$
V_Q = \frac{1}{2\times1.33}(1.0-0.3) = 0.263\text{ V} \;<\; V_m = 0.4\text{ V} \quad\checkmark \qquad (\text{margin } 0.137\text{ V})
$$

Drop $CR$ to 1.0 and $V_Q=0.35$ V (margin collapses to 50 mV); at $CR=0.8$, $V_Q=0.44$ V $> V_m$ and **the read flips the cell every time.** This is the whole reason $CR>1$ is non-negotiable. Write-ability has the mirror-image divider between access and pull-up; $PR>1$ is its non-negotiable counterpart.

### 2.2 SNM, and why low $V_{DD}$ is the real enemy

The combined stability metric is the **static noise margin (SNM)**: plot both inverters' transfer curves on the same axes (the "butterfly curve"), and the SNM is the side of the largest square that fits inside either eye — the DC noise a node can absorb before the two stable states merge. During read the access transistors distort the curves and shrink the eye, so **read SNM < hold SNM**, and both shrink with $V_{DD}$ because the whole $I$–$V$ compresses as supply approaches $V_{th}$:

| Condition | $V_{DD}$ | Temp | SNM | Verdict |
|---|---|---|---|---|
| Hold (WL off) | 1.0 V | 25 °C | ~280 mV | best case, cell isolated |
| Read (WL on) | 1.0 V | 25 °C | ~180 mV | worst *operating* case |
| Read (WL on) | 0.8 V | 125 °C | ~90 mV | marginal |
| Read (WL on) | 0.6 V | 125 °C | ~20 mV | essentially zero — cell unusable |

The last row is the punchline: **at 0.6 V the 6T read SNM nearly vanishes.** You cannot size your way out — pushing $CR$ up to help read costs you $PR$ and write-ability, and process variation ($\sigma_{V_{th}}$, §12) eats what margin remains. The 6T conflict is *structurally* unwinnable at low voltage, which is the entire motivation for the next section.

---

## 3. Breaking the 6T conflict: 8T, 10T, and assist circuits

The 6T conflict exists for one reason: read and write share the access transistors and both route through the *storage nodes*. Remove that sharing and the conflict dissolves. That is exactly what the **8T cell** does — it adds a dedicated 2-transistor read port (a read wordline gating a transistor whose gate is one storage node, discharging a separate read bitline) that **senses the bit without ever touching $Q$/$\bar Q$.** Writes still use the original 6T access transistors; reads no longer disturb anything. The consequence is decisive:

$$
\text{8T read SNM} \;=\; \text{hold SNM} \;\gg\; \text{6T read SNM}
$$

Concretely, at 0.6 V the 6T read SNM is ~20 mV (dead) while the 8T is ~250 mV (the full hold margin). **That is why 8T is the low-voltage cell of choice** — near-threshold caches, register files, and anything that must run at aggressively scaled supply. The trade is paid in **area (~30 %)** and one extra read bitline per column; you buy read robustness and, as a bonus, a naturally multi-portable read path (§7, §11). The **10T cell** goes further, adding a *differential* read port for faster sensing — the speed corner, favoured in register files where read latency is on the critical path.

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

where $D$ = defect density, $A$ = array area, $R$ = number of spares. A 1 Mb array at $DA\approx0.015$ yields ~98.5 % unrepaired, but a 64 MB L3 (64 such banks in series) can fall **below 50 %** unrepaired; 4 spare rows/bank restores it to **>90 %**. Repair is not optional at scale — it is the difference between a shippable die and scrap.

**Soft errors.** A neutron (cosmic secondary) or alpha particle (package contaminant) can deposit enough charge to flip a cell — a **soft error**, rate ~100–1000 FIT/Mb at sea level (1 FIT = 1 fail per $10^9$ device-hours). This is not a defect (the cell is fine next write); it is a reliability tax that scales with array size and *worsens as the stored charge shrinks*. Detection/correction is a coding problem handled in §9; the array-level knobs are parity (detect only, for latency-critical L1) vs full ECC (correct, for L2 and beyond).

**Retention voltage.** The minimum supply at which an *idle* 6T cell still holds its bit is $V_{\text{retain}}\approx 0.4$–$0.5\,V_{DD,\text{nom}}$ (~0.3 V for an N5 cell nominal at 0.7 V). Drowsy/retention modes park unused SRAM at $V_{\text{retain}}$ to cut leakage while keeping data — the trade being that lower retention voltage saves more leakage but erodes SNM and soft-error immunity, so the entry/exit sequence must lower and raise $V_{DD}$ in a controlled window (~5–50 µs). See [Power_Reduction_Techniques](../02_Power_and_Low_Power/03_Power_Reduction_Techniques.md) for how this fits state-retention power gating.

---

## 5. DRAM 1T1C: trading persistence for density

DRAM makes the opposite bet to SRAM at the deepest level: it **deletes the storage feedback loop.** A 1T1C cell is one access transistor and one capacitor — the bit is simply *charge present or absent* on $C_s$. No cross-coupled inverters means no self-restoring hold and no non-destructive read, but it also means ~6 $F^2$ instead of ~120 $F^2$: the density that makes gigabyte main memory economic. Everything hard about DRAM is the bill for the missing feedback loop, and it comes in three parts.

**The read is destructive, and the signal is tiny.** Reading connects $C_s$ (precharged bitline sits at $V_{DD}/2$) to the large bitline capacitance $C_{BL}$; charge redistributes, nudging the bitline by

$$
\Delta V \;=\; \frac{C_s}{C_s + C_{BL}}\cdot\frac{V_{DD}}{2}
$$

where $C_s$ = storage capacitance ($\approx$ 20–30 fF), $C_{BL}$ = bitline capacitance ($\approx$ 200–400 fF, set by how many cells share the line). The signal is a *capacitance ratio*, and because $C_{BL}\gg C_s$ it is minuscule: for $C_s=25$ fF, $C_{BL}=250$ fF, $V_{DD}=1.0$ V, $\Delta V\approx 45$ mV. Two consequences follow directly. First, sensing 45 mV in a noisy array demands a **differential sense amplifier** — a cross-coupled latch straddling the bitline pair that regeneratively amplifies the difference to full rail while rejecting common-mode noise (the two lines are matched, so supply/coupling noise cancels; CMRR ~30–40 dB). Second, that same amplification **restores** the cell: charge sharing left $C_s$ half-drained, so the sense amp must drive the full-rail value back into the cell every read. *Read = sense + rewrite*, always.

**The charge leaks.** With no feedback to replenish it, the stored charge bleeds off through sub-threshold, junction, gate, and dielectric paths (~2–20 fA/cell at 85 °C). The bit is "lost" not when charge halves but when $\Delta V$ falls below the sense threshold — a far tighter bar — giving an effective **retention time of ~64 ms** (JEDEC, 85 °C; ~2–4× longer per 20 °C cooler). Holding data therefore requires periodically reading and rewriting every cell before it decays: **refresh**, the subject of §6.

The upshot on the trade surface: DRAM buys ~20× density and pays with a destructive, restore-coupled read (~2–4× SRAM latency: a DDR4-3200 row-miss is $t_{RP}+t_{RCD}+t_{CL}\approx 41$ ns vs a row-hit's ~14 ns) and a standing refresh tax. **eDRAM** ports the 1T1C cell onto a logic process — denser than 6T (~2–3×), slower than SRAM (~5–10 ns), still refreshed — and has shipped as dense last-level cache (IBM POWER L3/L4, Intel Crystalwell L4).

---

## 6. The refresh tax: the price of volatility

Refresh is the recurring cost of DRAM's density bet, and it is a *bandwidth and power* tax, not merely a background chore. The model is simple and worth internalising. Every row must be visited within the retention window $t_{REFW}$ (64 ms); a refresh command takes $t_{RFC}$ during which the rank (all-bank refresh) is unavailable; commands are spaced $t_{REFI}$ apart. The fraction of time DRAM cannot serve real traffic is

$$
\text{Overhead} \;=\; \frac{t_{RFC}}{t_{REFI}}, \qquad t_{REFI} = \frac{t_{REFW}}{N_{cmd}} = \frac{64\text{ ms}}{8192} = 7.8125\ \mu\text{s}
$$

where $t_{RFC}$ = refresh cycle time (grows with density — more rows per command), $t_{REFI}$ = average refresh interval, $N_{cmd}$ = refresh commands per window (8192). The tax **rises with capacity**, because a bigger die refreshes more rows per command so $t_{RFC}$ climbs while $t_{REFI}$ is fixed by the retention window:

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

**Off-chip DRAM comes in two flavours of the same bandwidth trade.** **HBM** stacks DRAM dies over a wide, slow interface (1024-bit, ~6–9 Gbps/pin, ~1 TB/s/stack; HBM3E ~960 GB/s, 36 GB) via a TSV interposer — used where bandwidth dominates cost (AI accelerators, H100/B200). **GDDR** takes the narrow-and-fast corner (384-bit, ~24 Gbps/pin, ~1.15 TB/s; RTX 4090) — cheaper, no interposer, lower total bandwidth. Wide-and-slow vs narrow-and-fast is the same knob, set by whether pins or per-pin rate is the scarcer resource. Protocol and scheduling: [DDR_Controller](10_DDR_Controller.md).

---

## 7. Multi-port memories and register files: why ports cost quadratically

A single-port SRAM serves one access per cycle; a superscalar core's register file must serve *many* — a $W$-wide machine reads ~$2W$ operands and writes ~$W/2$ results per cycle (a 4-wide integer core needs roughly **8 read + 4 write ports**). The question is why you cannot simply keep adding ports, and the answer is a hard area law. **Each port adds a wordline (a horizontal wire across every cell in the row) and one or two bitlines (vertical wires down every cell in the column); the cell must grow in *both* dimensions to route them, so its area grows quadratically:**

$$
A_{\text{cell}}(P) \;\approx\; A_{6T}\,\big(1 + \alpha P^2\big)
$$

where $P$ = total port count, $\alpha \approx 0.05$–0.10 (process/layout dependent). Access time degrades in parallel — every extra bitline loads the sense node, every extra wordline loads the decoder — so a monolithic 12-port cell is both huge and slow (>1.5 ns, often multi-cycle). Beyond ~8–12 ports a monolithic array is untenable.

**The escape is banking, and it is a pure conflict-vs-cost trade.** Split the file into $B$ banks with few ports each; total port bandwidth is the sum, but two accesses to the *same* bank in one cycle **conflict** and one must stall. More banks → less area and faster access → *more* conflicts (fewer banks to spread across):

| Register file (128×64b, 8R4W-equivalent) | Area (7nm) | Access | Bank-conflict rate |
|---|---|---|---|
| Monolithic 12-port | ~0.015 mm² | ~1.5 ns | 0 % |
| 2-bank, 6-port each | ~0.010 mm² | ~0.9 ns | ~15 % |
| 4-bank, 3-port each | ~0.008 mm² | ~0.6 ns | ~35 % |

Real cores sit at 2–4 banks (integer) and 2 (FP/SIMD), the knee where conflict cost balances area/timing win: ARM Cortex-A78 128–160 entries at 4R2W (banked); Apple M1 ~350 at ~6R3W; Zen 4 ~180 at 4R2W; Golden Cove ~280 at ~6R4W. GPUs, needing dozens of operand reads per cycle across a warp, bank *heavily* (e.g. 16 banks of 1R1W) and lean on the compiler to lay out registers conflict-free. Two further levers matter: a **bypass network** (compare each read address against pending writes and forward the write data directly) hides the write-then-read latency that would otherwise cost a cycle on back-to-back dependents; and the **SRAM-vs-flip-flop** choice — flip-flop arrays give ~50 ps reads and trivial multi-porting but ~10× the area/bit, so they win only for small (16–32-entry), very-high-port files, while anything 64+ entries goes SRAM. **Column multiplexing** is the dual knob for single-port arrays: sharing one sense amp across $M$ columns trades sense-amp area for a taller, narrower array (better floorplan fit, longer bitlines).

The through-line: **port count is the most expensive axis in memory design.** It is why the physical register file is a first-order area/timing constraint in an OoO core ([OoO_Execution](05_OoO_Execution.md) §2.3), and why width rarely exceeds 6–8.

---

## 8. The memory compiler: SRAM as generated IP

You do not draw SRAM by hand — a **memory compiler** (ARM Artisan, Synopsys, foundry) synthesises a custom array from a specification and hands back the views the flow needs. The reason this matters conceptually is that the compiler *exposes the trade surface as knobs*, and picking them is a PPA decision:

| Knob | Trade it sets |
|---|---|
| Depth × width | capacity vs aspect ratio |
| Ports (1P / 2P / 1R1W …) | bandwidth vs the quadratic area of §7 |
| Mux factor (4/8/16) | sense-amp area vs array height, bitline length (speed) |
| Power mode (high-speed / high-density / low-leakage) | which corner of the trade surface the bitcell targets |
| Redundancy (spare rows/cols) | die area vs yield (§4) |

What comes back is a set of *abstractions*, not transistors: a **`.lib`** (Liberty timing/power for STA — the load-bearing number is **access time = clock-to-Q**, ~0.8–2.0 ns for embedded SRAM at 28 nm, which sets your pipeline's memory latency; plus setup/hold, which are tighter on address than data because address decode is on the critical path), a **`.lef`** (physical abstract for place-and-route), a behavioural **`.v`** for simulation, and the **`.gds`** for manufacturing. The practical discipline is to close timing on the slow corner (SS, low-$V$, hot) and hold on the fast corner (FF, high-$V$, cold), and to remember that the compiler's "high-density" bitcell is a *different, less stable* cell than its "high-speed" one — the §2 trade, exposed as a menu item.

---

## 9. ECC: coding theory for memory

Soft errors (§4) and DRAM bit-flips demand error correction, and the right way to understand ECC is as **coding theory, not a check-bit recipe.** The single governing quantity is the **minimum Hamming distance** $d_{\min}$ of the code — the fewest bit-flips that turn one valid codeword into another. From it, everything follows:

$$
\text{correct } t \text{ errors and detect } s\ge t : \quad d_{\min} \;\ge\; t + s + 1
$$

To *correct* a single-bit error you need $d_{\min}=3$ (each valid word sits in its own radius-1 ball, so a 1-flip word has a unique nearest neighbour). To *correct one and detect two* (SECDED — the workhorse for caches and register files) you need $d_{\min}=4$, which a **Hamming code plus one overall parity bit** achieves. The parity bit is what distinguishes a correctable single flip (odd parity change) from an uncorrectable double flip (even parity change, syndrome non-zero) — without it the code cannot tell them apart and would "correct" a double error into a *third* wrong value.

**The (72,64) overhead argument.** How many check bits $r$ does 64 data bits need? Each of the $2^r$ syndromes must name one of the $n$ correctable single-bit positions plus the no-error case: $2^r \ge n + 1$ with $n = 64 + r$. For $r=7$: $2^7 = 128 \ge 72$ ✓. Add the one SECDED parity bit → **8 check bits on 64 data bits = Hamming(72,64), a 12.5 % overhead.** On read, XOR the recomputed check bits against the stored ones to get the syndrome: zero = clean; non-zero with odd parity = single error *at the position the syndrome encodes*, flip it; non-zero with even parity = double error, flag uncorrectable. The overhead is a *fixed cost per word*, so wider codewords amortise it — 128+9 or 256+10 cut the percentage — but wider words also mean a whole line is lost to a single uncorrectable error and decode logic grows, so 64-bit SECDED is the common sweet spot.

**When to escalate.** SECDED corrects one bit per word; escalate when the error model outgrows that:

- **BCH / multi-bit codes** ($t>1$): generalise Hamming with ~$m\cdot t$ check bits ($m=\lceil\log_2 n\rceil$). BCH(511,484) corrects 2 bits at 5.3 % overhead; correcting 3 costs ~10.6 % and a Galois-field decode of several cycles. Used where flips *accumulate* faster than they are scrubbed — NAND flash, large server SRAM.
- **Chipkill** (DRAM): a whole DRAM *chip* can fail, flipping many bits of one word at once — far beyond SECDED. Chipkill **spreads each codeword across many chips** so any single-chip failure corrupts at most one (or a few) symbols per word, which a symbol-oriented code (Reed–Solomon-like) then corrects. This is the standard for server memory: it converts a catastrophic multi-bit burst into a correctable single-symbol error.

**Scrubbing** is the temporal complement: single-bit errors accumulate, and two in one word defeats SECDED, so a background scrubber walks the array during idle, reads-checks-corrects, and resets the count. The rate need only outpace accumulation — for a 32 MB L3 at ~100 FIT/Mb the array sees ~1 error/35 h, so a 24 h scrub keeps the probability of two-in-a-word at ~$10^{-7}$/cycle. The goal of the whole stack is to push **silent data corruption** (an undetected wrong bit reaching software) below the server target of ~1 per $10^{17}$ device-hours; parity detects but cannot correct, SECDED corrects one, BCH/chipkill extend the reach, and end-to-end CRC guards the paths *between* protected structures.

---

## 10. CAM and TCAM: memory that compares itself

Ordinary memory answers *"what is at this address?"*. A **content-addressable memory** answers the inverse — *"which address holds this value?"* — by comparing the search key against **every stored entry in parallel, in one cycle.** That parallelism is the entire value proposition and the entire cost. Mechanically, a CAM cell is a storage cell (a 6T core) plus a small comparison circuit that ties into a per-entry **match line**: the match line is precharged high, and *any* mismatching bit in the entry pulls it low (a wired-NOR across the row). An entry matches only if every bit matches; a priority encoder then reduces the match lines to an index.

**The cost is power and area, and it is structural.** Where an SRAM read activates *one* row, a CAM search activates *all* $N$ entries every cycle — every cell drives a comparison, every match line precharges and (mostly) discharges. Per bit, that is:

| Operation | Relative energy |
|---|---|
| SRAM read (1 row active) | 1× |
| Binary CAM search (all rows active) | 5–8× |
| TCAM search (all rows, 2× storage) | 10–20× |

Add ~2–3× the area (10–16T/cell). So you use a CAM **only where parallel exact-match against a whole table is the operation**, and its speed is worth the burn: a **TLB** compares a virtual page number against all resident translations in one cycle ([TLB_and_Virtual_Memory](08_TLB_and_Virtual_Memory.md)); a fully-associative cache tag array does the same; network switches match addresses. When exact match is enough and you can tolerate variable latency, an **SRAM hash table** (2–5 cycles, only one row active, standard density) is far cheaper — the design split in a switch ASIC is typically ~70 % SRAM hash for exact match, ~30 % TCAM for what genuinely needs it.

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

A gentler, digital cousin is **near-memory processing** — keep computation exact but move the compute *unit* next to the array rather than into the cell. **HBM-PIM** (Samsung Aquabolt-XL) drops a small MAC/SIMD engine beside each DRAM bank, feeding it from the bank's *internal* bandwidth (~4× the host interface) instead of over the pin-limited HBM link. For a large GEMM the weights never cross the host boundary — ~4× speedup and ~10× energy on the streaming-bound case — at the cost of a custom command path, weight layout constraints, and host↔PIM synchronisation. Both CIM and near-memory are the same idea as §0's trade surface pushed to its edge: *stop treating storage and compute as separate, and the data-movement cost that dominates everything else disappears* — if you can pay the precision and programmability price. See [NPU_Accelerators](16_NPU_Accelerators.md) for where these land in accelerator design.

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

**Memory-technology one-liners:** SRAM 6T = fast, non-destructive, self-restoring, poor density, dies at low $V_{DD}$. 8T = 6T + isolated read port = low-$V_{DD}$ robustness for 30 % area. DRAM 1T1C = 20× density, destructive read, needs refresh. MRAM = non-volatile, zero leakage, slow writes. CAM = parallel match at 10–20× the power.

---

## Worked problems

**1 — Read-disturb margin and the $V_{DD}$ wall (§2).** A 6T cell has $CR=1.33$, $V_{th}=0.3$ V, $V_m=0.4\,V_{DD}$. At $V_{DD}=1.0$ V the disturbed node reaches $V_Q=(1.0-0.3)/(2{\times}1.33)=0.263$ V, safely below $V_m=0.40$ V (margin 137 mV). Scale to $V_{DD}=0.6$ V: $V_m=0.24$ V and $V_Q=(0.6-0.3)/2.66=0.113$ V — margin only 127 mV *nominally*, but process spread ($\sigma_{V_{th}}$) plus the WL-on SNM collapse push read SNM to ~20 mV at hot corner. This is why the cell is unusable at 0.6 V and why 8T (read SNM = hold SNM ≈ 250 mV) is the answer.

**2 — Refresh bandwidth tax (§6).** DDR4-3200, 25.6 GB/s, 16 Gb chips: $t_{RFC}=550$ ns, $t_{REFI}=7.8125$ µs → overhead $=550/7812.5=7.0\%$. Lost bandwidth $=25.6\times0.070=1.80$ GB/s $=$ ~28 M cache lines/s that can never be served. DDR5 same-bank refresh overlaps other banks, dropping effective overhead to ~2.5–3 % — recovering most of it.

**3 — SECDED overhead, derived (§9).** Protect a 64-byte line (512 data bits). Check bits $r$ satisfy $2^r \ge (512+r)+1$; $r=10$ gives $1024\ge523$ ✓. But practice splits the line into eight 64-bit sub-words, each with its own Hamming(72,64): $8\times8=64$ check bits = 8 bytes = **12.5 %** overhead — *and* up to 8 independent single-bit errors per line are correctable (one per sub-word), which a single 522-bit code could not do. The split trades a hair of overhead for far better multi-error coverage.

**4 — Why 8 read ports for a 4-wide core (§7).** Four instructions × 2 source operands = 8 reads/cycle minimum (3-source ops like FMA push it higher), so the PRF needs ~8 read ports even though only 4 instructions issue. The quadratic port-area law ($A\propto\alpha P^2$) then forces banking rather than a monolithic 8R4W array — the trade taken by every core in §7's table.

---

## Cross-references

- **Down the stack (what these cells are built from):** [CMOS_Fundamentals](../00_Fundamentals/01_CMOS_Fundamentals.md) §12 (6T schematic, VTC, the SNM/butterfly and read/write-margin derivations this page reasons above), §3 (noise margins), §9 (process variation and the $\sigma_{V_{th}}$ scaling behind this page's §12).
- **Up the stack (what builds on these arrays):** [Cache_Microarchitecture](07_Cache_Microarchitecture.md) (SRAM/eDRAM organised into set-associative caches; tag CAMs), [TLB_and_Virtual_Memory](08_TLB_and_Virtual_Memory.md) (the fully-associative CAM of §10), [DDR_Controller](10_DDR_Controller.md) (the DRAM protocol/scheduling view of §5–§6), [OoO_Execution](05_OoO_Execution.md) §2 (the physical register file whose port cost is §7), [NPU_Accelerators](16_NPU_Accelerators.md) & [GPU_Architecture](15_GPU_Architecture.md) (consumers of the CIM/near-memory of §11 and the banked register files of §7).
- **Adjacent / sibling structures:** [Async_Design_and_CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) §5 (the async FIFO — a CDC structure that happens to be a memory), [DFT_and_ATPG](../06_Signoff/02_DFT_and_ATPG.md) §7 (memory BIST and March tests that verify these arrays), [Power_Reduction_Techniques](../02_Power_and_Low_Power/03_Power_Reduction_Techniques.md) (SRAM retention/drowsy modes of §4), [Floating_Point](../00_Fundamentals/04_Floating_Point.md) (ECC context).

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
