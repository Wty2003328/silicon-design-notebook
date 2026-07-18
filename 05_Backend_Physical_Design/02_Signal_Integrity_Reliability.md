# Signal Integrity and Reliability — when wires stop being ideal and transistors stop being eternal

> **Prerequisites:** [CMOS_Fundamentals](../00_Fundamentals/01_CMOS_Fundamentals.md) (the $I_D$/$V_{th}$ model and delay-vs-$V_{DD}$ law §4, noise margins and the regenerative property §3, wire RC / Elmore delay §10, temperature inversion §9), [Physical_Design](01_Physical_Design.md) (the routing, power grid, and via structures this page stresses).
> **Hands off to:** [Power_Analysis_and_Signoff](../02_Power_and_Low_Power/06_Power_Analysis_and_Signoff.md) (owns the *power-integrity signoff criteria* — target impedance, IR/EM/decap pass–fail; this page owns the PnR-side *mechanisms and fixing levers*), [STA](../06_Signoff/01_STA.md) (folds crosstalk delta-delay and end-of-life aging margin into timing), [Physical_Verification_DRC_LVS](../06_Signoff/03_Physical_Verification_DRC_LVS.md) (runs the antenna DRC of §6).

---

## 0. Why this page exists

Digital design rests on two idealizations that let us reason about billions of transistors with Boolean algebra and a single clock:

1. **A wire is ideal.** It is an instantaneous equipotential, isolated from its neighbours, carrying a clean rail-to-rail signal driven only by its own gate — and it is powered from a fixed, uniform $V_{DD}$.
2. **A device is eternal.** A transistor's threshold voltage and a wire's geometry are constants: what you tape out is what runs, unchanged, for the life of the part.

RTL, synthesis, and most of STA are built on these lies, and at mature nodes they were close enough to the truth. At advanced nodes **both collapse**, and in five specific ways. This page derives each reliability effect *from the idealization it violates* — that is the through-line, not a catalogue of tool switches.

| Idealization the digital model assumes | How it breaks at an advanced node | Resulting effect (section) |
|---|---|---|
| A wire's voltage depends only on its own driver | Adjacent wires share coupling capacitance $C_c$; a neighbour's transition injects charge | **Crosstalk** — noise glitch and delay push/pull (§1) |
| Every cell sees the same fixed $V_{DD}$ | Current through the grid's resistance and inductance droops the rail | **IR drop / power integrity** (§2) |
| A wire is a permanent conductor | Current density physically transports metal atoms → voids and hillocks | **Electromigration** — a wear-out lifetime limit (§3) |
| A transistor's $V_{th}$ is fixed for life | Bias, temperature, and hot carriers shift $V_{th}$ over years | **Aging** — the chip slows as it ages (§4) |
| Fabrication is damage-free | Plasma-process charge accumulates on antennas and stresses thin gate oxide | **Antenna effect** — a manufacturing rule (§5) |

The unifying observation: these are **analog** (continuous charge and voltage) and **time-dependent** (wear-out over years) effects, and the discrete-time synchronous abstraction has no vocabulary for either. So real silicon needs a *separate signoff* — SI-aware STA, IR/EM grid analysis, aging-derated timing, and process-antenna DRC (§6) — to prove the idealizations hold *closely enough* that the Boolean machine on top of them still works, at speed, for its warranty.

**One root cause behind all five.** They worsen for the *same* reason that shrinking helps density: scaling. Wires get taller and packed closer (↑ $C_c$); current per unit metal cross-section rises (↑ EM, ↑ IR); gate oxides thin and internal fields climb (↑ aging, ↑ antenna damage). Density and signal-integrity/reliability are in direct, quantifiable tension — which is why every mitigation in this page is a *trade-off against area, routing resource, or performance*, never a free fix.

---

## 1. Crosstalk: when a wire stops being isolated

### 1.1 The coupling model — where the false assumption breaks

A wire's voltage *should* be set only by its own driver. But every wire has capacitance not just to the ground/reference planes ($C_g$) but to each neighbour ($C_c$), and charge pushed through $C_c$ by a neighbour's transition lands on the victim. Split the victim's total capacitance:

$$
C_{total} = C_g + C_c, \qquad k_{couple} \equiv \frac{C_c}{C_c + C_g}
$$

where $C_g$ = capacitance to quiet nets and the ground planes (the "wanted" load), $C_c$ = coupling capacitance to switching neighbours, and $k_{couple}$ = the fraction of the victim's world that a neighbour controls. The whole of crosstalk is the consequence of $k_{couple}$ being non-negligible.

```text
              reference plane (above)
        ═══════════════════════════════
           │ C_top          │ C_top
        ┌──────┐    C_c    ┌──────┐
        │Victim│◄──┤├──────►│Aggr. │      C_c : wire-to-wire coupling
        └──────┘           └──────┘
           │ C_bot          │ C_bot
        ═══════════════════════════════
              reference plane (below)

   C_g = C_top + C_bot   (to reference)   →   k_couple = C_c / (C_c + C_g)
```

### 1.2 Why it worsens at advanced nodes

Take the parallel-plate intuition for the two capacitances a wire owns:

$$
C_c \propto \varepsilon\,\frac{H\,L}{S}, \qquad C_g \propto \varepsilon\,\frac{W\,L}{t_{ox}}
$$

where $H$ = wire height, $W$ = wire width, $L$ = the parallel run length, $S$ = edge-to-edge spacing, $t_{ox}$ = dielectric thickness to the plane. Scaling shrinks $S$ and $W$ aggressively, but $H$ is held tall — even *grown* — because a thinner wire would have crippling resistance ($R \propto 1/(W H)$). So the coupling ratio climbs:

$$
\frac{C_c}{C_g} \;\propto\; \frac{H/S}{W/t_{ox}}
$$

Taller, closer, thinner-spaced wires couple more, and the wire *aspect ratio* $H/W$ has risen past 2–3 to keep resistance in check. Concretely, M1 pitch fell from ~340 nm (130 nm node) to ~28 nm (7 nm node) while height barely dropped — so on lower metals $C_c$ reaches **50–80 % of total capacitance**. A second effect compounds it: higher wire resistance lengthens the victim's RC time constant, so an injected glitch *decays more slowly* and has more time to reach a receiver. Crosstalk is therefore not a legacy nuisance that scaling removes; it is a first-order effect that scaling *creates*.

### 1.3 The noise glitch — a functional failure

Hold the victim quiet and step the aggressor $0 \to V_{dd}$. In the fast-edge limit the two capacitors form a **capacitive charge divider**, and the victim's peak bump is

$$
V_{noise} \;\approx\; V_{dd}\,\frac{C_c}{C_c + C_g}
$$

with symbols as in §1.1. Two regimes decide whether this is fatal:

- **Floating / weakly-held victim** (a dynamic node, a tri-stated bus, a net whose driver is off): nothing fights the injected charge, so the bump reaches the full divider value and *persists*. This is the dangerous case and the reason dynamic logic and long buses are crosstalk-critical.
- **Driven victim:** the driver's on-resistance $R_v$ pulls the node back, racing the aggressor edge. The residual bump scales roughly with $\dfrac{R_v}{R_{agg}+R_v}$ and with how fast the aggressor slews — a *strong* victim driver and a *slow* aggressor edge both shrink it.

The failure test is regenerative (see [CMOS_Fundamentals](../00_Fundamentals/01_CMOS_Fundamentals.md) §3): if $V_{noise}$ stays below the receiving gate's noise margin, the gate's gain squashes it; if it exceeds the margin, the gate *amplifies* it into a full logic transition that can be latched — a silent functional failure. Designers therefore hand crosstalk a slice of a fixed noise budget: with total margin $NM \approx 0.35\,\text{V}$ at $V_{dd}=0.7\,\text{V}$, a common split is ~40 % crosstalk / ~30 % supply noise / ~30 % variation, giving a crosstalk allowance near **0.14 V** that no aggressor is allowed to breach.

### 1.4 The Miller push/pull — a timing failure

Now let victim *and* aggressor switch together. The charge the victim's driver must move through $C_c$ depends on the *relative* swing across that capacitor, captured by an effective coupling capacitance:

$$
C_{c,\text{eff}} \;=\; C_c\left(1 - \frac{\Delta V_{agg}}{\Delta V_{vic}}\right)
$$

where $\Delta V_{agg}$ and $\Delta V_{vic}$ are the simultaneous, signed voltage changes over the transition window. Three cases fall out:

- **Same direction, matched rate** ($\Delta V_{agg}/\Delta V_{vic} = +1$): $C_{c,\text{eff}} = 0$ — no charge crosses $C_c$; the victim *speeds up*.
- **Aggressor quiet** ($\Delta V_{agg}/\Delta V_{vic} = 0$): $C_{c,\text{eff}} = C_c$ — the nominal, singly-coupled load.
- **Opposite direction** ($\Delta V_{agg}/\Delta V_{vic} = -1$): $C_{c,\text{eff}} = 2C_c$ — the **Miller doubling**; the victim sees up to $C_g + 2C_c$ and *slows down*.

With victim delay $\approx R_v(C_g + C_{c,\text{eff}})$, the worst-case opposite-switching delay increase relative to the quiescent $C_g + C_c$ load is

$$
\frac{\Delta d}{d} \;\approx\; \frac{C_c}{C_g + C_c} \quad\Big(\text{and the full best-to-worst swing is } \tfrac{2C_c}{C_g+C_c}\Big)
$$

When $C_c \approx C_g$ that is a **+50 % delay swing** on that stage — a first-order timing term, not a correction. This is why coupling cannot be waved away with a flat derate on high-$k_{couple}$ nets.

### 1.5 SI-aware STA, briefly

The timing tool assumes the *worst aligned* aggressor: for a setup (max-delay) check it slows the data path (opposite switching) and speeds the capturing clock (same switching) to maximize the required time; for hold it does the reverse. Because real aggressors rarely all align, **timing windows** filter the pessimism away — if an aggressor's switching window cannot overlap the victim's transition, no delta-delay is applied. How this delta-delay folds into slack is [STA](../06_Signoff/01_STA.md)'s job; here it is enough to know the mechanism the tool is modelling.

### 1.6 The mitigation trade-off: coupling vs routing resource

Every lever that reduces $C_c$ or its ratio *spends routing area*, so SI closure is a targeting problem, not a blanket fix:

| Lever | Mechanism | What it costs |
|---|---|---|
| Wider spacing (double-space NDR) | $C_c \propto 1/S$: 2× spacing ~halves $C_c$ on that run | 2–3× the track pitch per net → density loss |
| Shield wire (grounded neighbour) | Pins the neighbour at 0 (no injection) and gives a clean return path | A full extra track on each side — up to doubling the wiring |
| Layer / net-order assignment | Place quiet or same-direction nets adjacent | Free in area, but constrains the router |
| Slew / drive control | Stronger victim driver rejects noise; *slower aggressor* injects less | Double-edged: a fast aggressor edge injects *more*; slowing it spends its own slack |

You cannot double-space everything — density would collapse — so the design rule is quantitative: apply an NDR or shield only where the predicted delta-delay $\frac{2C_c}{C_g+C_c}\,d$ exceeds the net's slack, or the glitch $V_{dd}\frac{C_c}{C_c+C_g}$ exceeds its noise budget. Clocks and a few high-speed buses get shielded (they are latency- and integrity-critical and few in number); the mass of signal nets are handled by ordering and the router's own spacing heuristics. This is the routing-side mirror of the whole page's theme: buy integrity only where the physics says you must.

---

## 2. IR drop / power integrity: when the supply stops being ideal

> **Scope:** the PnR mechanism and fixing levers. The impedance-domain model (target impedance, package–die anti-resonance), the pass–fail signoff criteria, and full IR/decap budgeting live in [Power_Analysis_and_Signoff](../02_Power_and_Low_Power/06_Power_Analysis_and_Signoff.md) — cross-linked, not re-derived here.

The idealization: every cell sees a fixed $V_{DD}$. The reality: the supply reaches each cell through a resistive-and-inductive grid, and the current the logic draws makes the local rail droop.

$$
\Delta V_{static} = I_{avg}\,R_{grid}, \qquad
\Delta V_{dynamic} = \underbrace{L\,\frac{dI}{dt}}_{\text{simultaneous switching}} + I_{peak}\,R_{grid}
$$

where $R_{grid}$ = effective resistance from bump to cell, $L$ = supply-path inductance, and $I_{avg}/I_{peak}$ are the average/peak currents. The static term is a resistive floor (budget **3–5 % of $V_{DD}$**, ~21–35 mV at 0.7 V); the $L\,di/dt$ term — driven by thousands of flops switching on the same clock edge — is the transient that actually fails silicon (budget **8–10 %**).

**Why droop belongs on this page (the link to timing).** A drooped rail slows the cells under it, because drive current falls with headroom. From the delay law $d \propto \dfrac{V_{dd}}{(V_{dd}-V_{th})^{\alpha}}$ ([CMOS_Fundamentals](../00_Fundamentals/01_CMOS_Fundamentals.md) §4), a few-percent droop costs a few-percent delay — and because droop is *local* (worst at grid-far, high-activity regions), it creates timing failures that nominal fixed-$V_{DD}$ STA never sees. That is precisely why signoff runs **voltage-aware STA** on an IR-derated voltage map.

**The PnR levers, and their costs:** widen straps (↓ $R_{grid}$, ↑ routing blockage); add power vias, multi-cut (↓ $R$ *and* ↑ EM robustness, §3); add decoupling capacitance to source the transient locally; spread high-activity cells to break hotspots; add bumps/pads. Decap sizing follows charge conservation,

$$
C_{decap} \;\gtrsim\; \frac{I_{peak}\,\Delta t}{\Delta V_{budget}}
$$

($I_{peak}$ = transient current, $\Delta t$ = its duration, $\Delta V_{budget}$ = allowed droop) — but decap is thin-oxide gate area that *leaks* and *displaces logic*, so you provision the minimum that holds the transient (fill roughly 5–10 % of area), not the maximum. Decap-vs-leakage/area is the same targeted trade-off as everywhere on this page.

---

## 3. Electromigration: when a wire stops being a permanent conductor

### 3.1 The mechanism and why it is a *lifetime* limit

At high current density the conducting electrons transfer momentum to the metal lattice — the "electron wind" — and metal atoms drift in the direction of electron flow. Where that atom flux *diverges* (at vias, grain-boundary junctions, width steps) atoms pile up or deplete: **voids** open the wire (an eventual open circuit) and **hillocks** extrude sideways (an eventual short to a neighbour). The wire is fully functional at $t=0$ and fails after a predictable operating time, so EM is not a pass/fail *function* check — it is a **wear-out constraint on lifetime**, signed off against a target like 10 years.

### 3.2 Black's equation and its two lessons

The rate of that wear-out is Black's law:

$$
\boxed{\,MTTF \;=\; A\,J^{-n}\,\exp\!\left(\frac{E_a}{kT}\right)\,}
$$

where $MTTF$ = mean time to failure, $A$ = a geometry/material constant, $J$ = current density, $n$ = the current-density exponent (1 when void *growth*-limited, 2 when void *nucleation*-limited; typically 1–2), $E_a$ = activation energy (Cu grain-boundary 0.7–0.9 eV, Cu interface 0.8–1.0 eV, Al 0.5–0.7 eV), $k$ = Boltzmann constant ($8.617\times10^{-5}$ eV/K), $T$ = absolute temperature. Two consequences drive every EM decision:

- **Current *density* is the enemy, not current.** $MTTF \propto J^{-n}$, so with $n=2$, halving $J$ — e.g. doubling a wire's width at the same current — **quadruples** lifetime. This single fact is why power straps are wide and why signal EM sizes a wire to a *current budget*.
- **Temperature is exponential.** The $\exp(E_a/kT)$ term means a 10–15 °C rise can *halve* MTTF, which couples EM directly to thermal (§3.4).

From the first lesson comes the design abstraction EM analysis actually uses: for a target lifetime the process gives a maximum current density $J_{max}$, so a wire carrying current $I$ needs cross-section

$$
A \;\ge\; \frac{I}{J_{max}} \quad\Longrightarrow\quad W_{min} = \frac{I}{J_{max}\,t_{metal}}
$$

Direct (unidirectional) current on power/ground is the worst case; bidirectional AC/RMS current on signals and clocks **self-heals** on each reversal, so its $J_{max}$ is ~5–15× higher. Typical 7 nm Cu limits are $J_{max}^{DC}\approx 1\text{–}3\ \text{MA/cm}^2$ and $J_{max}^{RMS}\approx 5\text{–}15\ \text{MA/cm}^2$. **Vias are the choke point** — a single via cut has a tiny cross-section (~0.1–0.2 mA per cut), so multi-cut vias are mandatory on any current-carrying connection, and this is exactly why the IR-drop fix "add power vias" (§2) doubles as an EM fix.

### 3.3 The trade-off: widening vs congestion

Widening a wire to meet $J_{max}$ consumes tracks and creates routing congestion. The alternatives — more parallel straps (same area cost but with redundancy against a single void), splitting current across layers, or lowering the net's activity/frequency — all trade area or effort for margin. Real flows widen the *top* power metals aggressively (they carry bulk current and cost few signal tracks) and handle signal EM by upsizing only the handful of high-activity, high-fanout nets (clock spines, reset, always-on enables) rather than paying the area everywhere. EM-driven widening is thus another instance of spending resource only where the current density demands it.

### 3.4 Self-heating folds thermal back in

At advanced nodes a wire heats *itself*: $\Delta T_{sh} = I^2 R\,R_{th}$, and the low-$k$ dielectrics that reduce coupling capacitance conduct heat poorly (~0.15 W/m·K vs ~1.4 for SiO₂), so Joule heat generated in the wire cannot escape. $\Delta T_{sh}$ of 10–30 °C is common on heavily loaded lines, and because Black's law is exponential in $T$, EM signoff must use the **self-heated** temperature, not ambient. (Chip-level thermal budgeting, throttling, and runaway math live in [Power_Analysis_and_Signoff](../02_Power_and_Low_Power/06_Power_Analysis_and_Signoff.md).)

---

## 4. Aging: when a transistor stops being eternal

### 4.1 The idealization and why it forces end-of-life signoff

A transistor's $V_{th}$ is treated as a constant, but bias, temperature, and hot carriers shift it upward over years, so **the chip slows down as it ages**. A part that passes timing *fresh* can miss its frequency after 3–5 years in the field. The consequence is unavoidable: timing must be signed off at **end-of-life**, on an aged model, not at $t=0$. Three mechanisms drive the drift:

- **NBTI (negative-bias temperature instability)** — a PMOS held on ($V_{gs}<0$) traps holes and breaks Si–H bonds at the oxide interface, raising $|V_{tp}|$. The dominant mode, and it partially *recovers* when the stress is removed (so it is duty-cycle dependent).
- **PBTI** — the NMOS dual, significant with high-$k$ gate stacks (electron trapping); raises $V_{tn}$.
- **HCI (hot-carrier injection)** — carriers accelerated by the drain-end field are injected into the oxide; worst *during switching* (simultaneous high $V_{ds}$ and current), so it scales with activity and punishes always-on, high-toggle paths like clock trees.

### 4.2 The power-law model

BTI aging follows a sublinear power law in time with strong temperature and field acceleration:

$$
\Delta V_{th} \;\propto\; t^{\,n}\,\exp\!\left(-\frac{E_a}{kT}\right)\,f(V_{gs})
$$

where $t$ = cumulative stress time, $n$ = the time exponent (≈ 0.16–0.25 for NBTI, set by the reaction-diffusion / trap-generation kinetics), $E_a$ = activation energy, and $f(V_{gs})$ = the roughly exponential dependence on gate overdrive. The **sublinear $t^n$** is the load-bearing shape: aging is *fast early and then saturates*, so most of the lifetime shift accrues in the first weeks-to-months — which is *why* a bounded guard-band can cover a 10-year target instead of an ever-growing one. Numbers to anchor it: NBTI gives $\Delta V_{th}\approx 30\text{–}50\ \text{mV}$ (PMOS) over 10 years at 125 °C, worth **~5–10 % delay** degradation; PBTI adds 10–30 mV (NMOS).

### 4.3 Aging-aware STA and the guard-band trade-off

There are two ways to cover the drift, and the choice is a real performance trade:

- **Aged libraries** — characterize cells fresh *and* at 10-year stress, and sign off timing on the aged models. Accurate, but it needs the aged libs and multiplies the corner count.
- **Aging guard-band** — apply a flat extra margin (usually folded into OCV/derate in [STA](../06_Signoff/01_STA.md)). Cheap, but pessimistic.

The trade cuts to the core of the business: **every millivolt of aging guard-band is frequency you surrender for the entire fleet, for the whole life of the part, to protect against a worst-case corner most units never hit.** Over-margin and you ship silicon slower than it can actually run; under-margin and a fraction fails in warranty. Modern flows use aged libraries *plus* a small residual guard-band, and lean on NBTI **recovery** — real workloads are not 100 % duty cycle — to avoid signing off at the pessimistic DC-stress bound. Note the thermal coupling again: aging is exponential in $T$, so the self-heated hotspots that age fastest are the same ones EM fails first — thermal, EM, and aging all key off one local temperature. The device physics behind $V_{th}$, $E_a$, and temperature inversion lives in [CMOS_Fundamentals](../00_Fundamentals/01_CMOS_Fundamentals.md).

---

## 5. Antenna effect: a manufacturing-reliability rule

The idealization here is subtler: that fabrication is damage-free, so the finished layout is what you get. But the chip is *built up* layer by layer, and during plasma etch and ion-implant steps, charge collects on the metal that already exists. A long metal segment — an "antenna" — connected to a transistor gate but **not yet** connected to a diffusion (which would bleed charge to the substrate) accumulates that charge, and the resulting voltage appears across the thin gate oxide. It can rupture or degrade the oxide **before the chip is ever powered on**. This is therefore a *manufacturing*-reliability rule, a function of the layout's construction order and geometry, not of any runtime signal.

The metric is the **antenna ratio**:

$$
AR \;=\; \frac{A_{metal}\ (\text{charge-collecting conductor connected to the gate})}{A_{gate}\ (\text{gate-oxide area it must discharge into})}
$$

Above a process limit (tens-to-hundreds:1, tighter on the thinner lower layers) the accumulated charge is deemed damaging, and it is checked as a DRC rule. The fixes are geometric, each with a small cost:

- **Layer jumping ("bridge up")** — break a long lower-layer run by hopping to a higher layer and back, so no *as-fabricated* segment exceeds $AR$ (higher layers are added after the gate already has a discharge path). Costs vias and a little delay.
- **Antenna / jumper diode** — add a reverse diode from the net to substrate that harmlessly bleeds process charge. Costs a little area and junction leakage.
- **Reorder the connection** so the protective diffusion attaches early.

The rule itself and its checking are owned by [Physical_Verification_DRC_LVS](../06_Signoff/03_Physical_Verification_DRC_LVS.md); the plasma steps that cause it are in [Fabrication_Process](../07_Manufacturing_and_Bringup/01_Fabrication_Process.md). It earns its place *here* because it is the one reliability failure that is written into the die at birth.

---

## 6. Reliability signoff: the analog / time-dependent gate

Everything above is what stands between a *functionally correct netlist* and a *part that works, at speed, for its warranty*. Because the synchronous digital model cannot express any of it, signoff adds a distinct set of checks — this is the deliverable the whole page builds toward. The essential wear-out and integrity mechanisms, each tied to the idealization it violates:

| Mechanism | Idealization broken | What it degrades | Where signed off |
|---|---|---|---|
| Crosstalk (noise + delta-delay) | Wire is isolated | Glitch → logic flip; delay push/pull | SI-aware STA (§1) |
| IR drop (static + dynamic) | Supply is a fixed $V_{DD}$ | Localized slow-down | Voltage-aware STA (§2) |
| Electromigration | Wire is permanent | Open/short after years | EM current-density check (§3) |
| Aging (NBTI/PBTI/HCI) | $V_{th}$ is fixed | End-of-life slow-down | Aged-library STA (§4) |
| Antenna | Fabrication is damage-free | Gate-oxide rupture at build time | Antenna DRC (§5) |
| TDDB / stress-migration / ESD | Oxide & metal are ideal forever | Oxide breakdown, void, discharge damage | Device rules — [CMOS_Fundamentals](../00_Fundamentals/01_CMOS_Fundamentals.md) §7, DRC |

The minimal signoff checklist that follows from the table:

- **Crosstalk** — noise (glitch vs margin) and delta-delay, on coupling-dominated nets, with timing-window filtering.
- **EM** — DC on power/ground, AC/RMS on signals and clocks, peak limits, and multi-cut vias verified against per-cut current.
- **IR drop** — static and dynamic, feeding a voltage-aware timing run; grid EM checked on the same current map.
- **Aging** — NBTI/PBTI margin in timing via aged libraries; HCI checked on always-on, high-toggle paths.
- **Antenna** — every gate's antenna ratio within process limits, fixed by jumpers or diodes.

The power-integrity half of this (IR, dynamic droop, decap, grid EM) shares its criteria with [Power_Analysis_and_Signoff](../02_Power_and_Low_Power/06_Power_Analysis_and_Signoff.md); the device-level wear-out (TDDB, ESD, latch-up) is owned by [CMOS_Fundamentals](../00_Fundamentals/01_CMOS_Fundamentals.md). This page owns the *interconnect and physical-design* view: coupling, the grid as a resistive network, metal wear-out, and the process-antenna rule.

---

## Numbers to memorize

| Quantity | Typical value | Why it matters (section) |
|---|---|---|
| $C_c$ fraction of total cap (7 nm lower metals) | 50–80 % | coupling is first-order, not a correction (§1.1–1.2) |
| M1 pitch, 130 nm → 7 nm | ~340 nm → ~28 nm | spacing shrinks while height holds → ↑ $C_c$ (§1.2) |
| Crosstalk noise budget | ~0.14 V of a ~0.35 V $NM$ at 0.7 V | glitch vs margin (§1.3) |
| Miller effective coupling cap | $0 \to C_c \to 2C_c$ | speed-up / nominal / slow-down (§1.4) |
| Worst crosstalk delay swing | up to $\frac{2C_c}{C_g+C_c}$ (≈ 50 % if $C_c\approx C_g$) | SI delta-delay (§1.4) |
| Static IR-drop budget | 3–5 % $V_{DD}$ (~21–35 mV at 0.7 V) | resistive floor (§2) |
| Dynamic IR-drop budget | 8–10 % $V_{DD}$ | $L\,di/dt$ transient (§2) |
| IR droop → delay | few-% droop ≈ few-% slow-down | why voltage-aware STA (§2) |
| EM $J_{max}$, 7 nm Cu | DC 1–3, RMS 5–15 MA/cm² | current-density budget (§3.2) |
| Black's exponent effect | halve $J$ → 4× MTTF ($n=2$) | why straps are wide (§3.2) |
| Via EM per cut | ~0.1–0.2 mA | why multi-cut vias (§3.2) |
| Wire self-heating, 7 nm | $\Delta T \approx 10\text{–}30\,°\text{C}$ (low-$k$ ~0.15 W/m·K) | feeds Black's $T$ (§3.4) |
| NBTI aging, 10 yr @ 125 °C | $\Delta V_{th} \approx 30\text{–}50$ mV → 5–10 % delay | end-of-life signoff (§4) |
| NBTI time exponent $n$ | ≈ 0.16–0.25 | sublinear → bounded guard-band (§4.2) |
| Antenna ratio limit | tens-to-hundreds : 1 | gate-oxide protection (§5) |
| ESD protection targets | HBM ≥ 2 kV, CDM ≥ 500 V | pad/clamp rules (§6) |

**Constants:** Boltzmann $k = 8.617\times10^{-5}$ eV/K. Activation energies: Cu grain-boundary 0.7–0.9 eV, Cu interface 0.8–1.0 eV, Al 0.5–0.7 eV.

---

## Cross-references

- **Down the stack (the physics this rests on):** [CMOS_Fundamentals](../00_Fundamentals/01_CMOS_Fundamentals.md) (the $I_D$/$V_{th}$ device model behind aging, the noise margins §1.3 tests against, wire RC/Elmore behind glitch decay, temperature inversion, and the device wear-out — TDDB, ESD, latch-up — this page cross-links rather than re-derives).
- **Up the stack (what consumes these results):** [STA](../06_Signoff/01_STA.md) (folds crosstalk delta-delay and end-of-life aging margin into slack via SI-aware and OCV/derated timing), [Power_Analysis_and_Signoff](../02_Power_and_Low_Power/06_Power_Analysis_and_Signoff.md) (owns the power-integrity *criteria* — target impedance, IR/EM/decap pass–fail — that §2 feeds), [Physical_Verification_DRC_LVS](../06_Signoff/03_Physical_Verification_DRC_LVS.md) (runs the antenna and metal-density DRC of §5).
- **Adjacent / prerequisite:** [Physical_Design](01_Physical_Design.md) (the routing, power grid, via stacks, and NDR/shield resources every mitigation on this page spends), [Fabrication_Process](../07_Manufacturing_and_Bringup/01_Fabrication_Process.md) (the plasma steps that create the antenna effect and the BEOL stack that sets $C_c$).

---

## References

1. Black, J.R., "Electromigration — A brief survey and some recent results," *IEEE Trans. Electron Devices*, 16(4), 1969. The origin of the $MTTF = A J^{-n}e^{E_a/kT}$ model of §3.2.
2. JEDEC, *JEP122H: Failure Mechanisms and Models for Semiconductor Devices*, 2016. Acceleration models for EM, BTI, HCI, TDDB used across §3–§4.
3. Ho, R., Mai, K.W., and Horowitz, M.A., "The Future of Wires," *Proc. IEEE*, 89(4), 2001. The interconnect-scaling and coupling argument of §1.2.
4. Alam, M.A. and Mahapatra, S., "A comprehensive model for PMOS NBTI degradation," *Microelectronics Reliability*, 45(1), 2005. The reaction-diffusion power-law $\Delta V_{th}\propto t^{n}$ of §4.2.
5. Sakurai, T. and Newton, A.R., "Alpha-power law MOSFET model and its applications to CMOS inverter delay," *IEEE JSSC*, 25(2), 1990. The delay-vs-$V_{DD}$ relation behind the IR-droop-to-delay link of §2.
