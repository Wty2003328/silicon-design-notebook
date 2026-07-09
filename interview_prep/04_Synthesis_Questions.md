# 04 — Synthesis: Interview Questions

Consolidated interview Q&A and worked problems from every page in `04_Synthesis/`, moved here to keep topic pages focused on concepts. Each section links back to its source page for full context.

---

## Constraints (SDC) — Telling the Tools What "Correct Timing" Means

*From [Constraints_SDC.md](../04_Synthesis/02_Constraints_SDC.md)*

**Q: What breaks if you forget to declare a divided clock?** STA treats the divider output as a normal data net on the source clock, so every flop clocked by the /2 is timed against the wrong period/edges — paths that are actually fine look failing, and real violations on the divided domain go unchecked. Always `create_generated_clock`.

**Q: You set a 3-cycle setup multicycle and the chip fails hold in silicon. Why?** You relaxed setup to 3 cycles but didn't set the matching hold multicycle (2). The default hold check still expects the data to be stable for the same-cycle relationship, which a genuinely multicycle path doesn't satisfy → hold violation. Setup MCP N needs hold MCP N−1.

**Q: When is a false path dangerous?** Always potentially — it tells STA to *stop checking* a path. If the path is in fact functionally active (you mis-judged the mode/CDC), STA never flags its violation and it fails in silicon. Treat every false_path as a reviewed waiver with explicit justification.

---

## Logic Synthesis and Optimization -- Senior Engineer Deep Dive

*From [Synthesis_and_Optimization.md](../04_Synthesis/01_Synthesis_and_Optimization.md)*

### Q1: Walk me through the synthesis flow from RTL to gates.

**A:** Synthesis has three phases. (1) **Translation**: The tool parses RTL
(Verilog/VHDL) into a technology-independent Boolean network using generic
(GTECH) cells. It infers flip-flops, latches, and memories from the coding
style. (2) **Optimization**: Boolean and structural optimization including
factoring, decomposition, retiming, resource sharing, and datapath
optimization. All done at the GTECH level. (3) **Technology Mapping**: The
optimized Boolean network is mapped to the target library cells using DAG
covering and Boolean matching algorithms. Cell selection considers timing
(pick faster cells on critical paths), area (smallest cells off critical),
and DRV constraints (max transition, capacitance, fanout). The output is a
gate-level netlist.

### Q2: What is the difference between set_false_path and set_clock_groups -asynchronous?

**A:** `set_false_path -from clk_a -to clk_b` removes timing analysis from
clk_a to clk_b only. You need a second command for clk_b to clk_a.
`set_clock_groups -asynchronous -group clk_a -group clk_b` is bidirectional --
it removes timing in BOTH directions. Additionally, set_clock_groups affects
SI/crosstalk analysis (the tool knows both clocks are physically present),
while set_false_path does not convey this information. For truly asynchronous
clock domains, set_clock_groups is the correct and preferred approach.

### Q3: Explain set_multicycle_path with a concrete example and the hold adjustment.

**A:** A multicycle path of N means data is valid for N clock cycles. For
example, a configuration register written every 4th cycle:
```tcl
set_multicycle_path 4 -setup -from config_reg -to core_reg
set_multicycle_path 3 -hold  -from config_reg -to core_reg
```
The setup check moves from launch+T to launch+4T, giving 4x timing budget.
The hold check (N-1=3) moves the hold reference to launch+(4-3)T = launch+T,
checking that old data doesn't corrupt capture one cycle after launch. Without
the hold adjustment, the hold check would be at launch+4T-0 = launch+4T, which
is overly pessimistic and would require huge insertion delays. The standard
practice is always to pair `-setup N` with `-hold (N-1)`.

### Q4: What is the difference between -physically_exclusive and -logically_exclusive in set_clock_groups?

**A:** Physically exclusive clocks CANNOT coexist on silicon (e.g., test_clk
and func_clk through the same MUX -- only one propagates). The tool can share
CTS resources between them since they never conflict. Logically exclusive
clocks DO coexist physically (both clock trees are built) but are functionally
mutually exclusive (controlled by a mode register). The tool must build
independent clock trees. Both remove timing analysis between the groups, but
the physical implications for CTS, power analysis, and SI are different.

### Q5: How does retiming work and what are its limitations?

**A:** Retiming moves registers forward or backward across combinational logic
to balance pipeline stage delays. Forward retiming moves a register from before
a logic block to after it; backward retiming does the reverse. The tool
verifies functional equivalence and adjusts register count as needed (at fanout
points, forward retiming may duplicate registers; at reconvergent points,
backward retiming may merge them). Limitations: cannot retime across I/O
boundaries (changes interface timing), clock domain boundaries (changes CDC
behavior), or registers with reset/initial values (would change reset state).
Also cannot retime if register has dont_touch or scan attributes.

### Q6: Explain the impact of clock gating on power and area.

**A:** Clock gating eliminates switching power on the clock tree and flip-flops
when the enable is inactive. In a typical SoC, the clock tree consumes 30-50%
of total dynamic power, so gating can save 20-40% dynamic power. Area overhead
is one ICG cell per group of gated FFs (typically 4-32 FFs per ICG). The ICG
cell contains a latch (to prevent glitches) and an AND gate. It adds ~1 gate
equivalent per group. The break-even point is typically 3-4 FFs: gating fewer
FFs wastes area; gating more FFs saves net area by potentially reducing clock
tree buffering. Set minimum bitwidth with `set_clock_gating_style
-minimum_bitwidth 4`.

### Q7: What is boundary optimization and when would you disable it?

**A:** Boundary optimization allows the synthesis tool to propagate constants,
remove unused ports, and merge duplicate logic across hierarchical boundaries.
For example, if a port is tied to constant 0, the tool can propagate this into
the sub-module and simplify the logic. Disable it when: (1) You need to
preserve the exact port interface for ECO or late integration. (2) The
sub-module is shared (instantiated multiple times) and boundary optimization
would specialize it differently for each instance. (3) You need to match a
reference netlist for formal verification against a specific hierarchy.

### Q8: How do you constrain a DDR interface in SDC?

**A:** DDR interfaces transfer data on both rising and falling clock edges.
You need input/output delays referenced to both edges:
```tcl
set_input_delay -clock ddr_clk -max 0.8 [get_ports data]
set_input_delay -clock ddr_clk -max 0.8 -clock_fall -add_delay [get_ports data]
```
The `-clock_fall` references the falling edge, and `-add_delay` prevents the
second constraint from overwriting the first. Both max and min delays should be
specified for setup and hold analysis. The generated clock for DDR typically has
non-50% duty cycle or phase shift that must be accurately modeled.

### Q9: What is the difference between compile_ultra and compile_ultra -incremental?

**A:** `compile_ultra` performs full optimization: ungrouping, Boolean
restructuring, datapath optimization, technology mapping, and timing/area/power
optimization from scratch. `compile_ultra -incremental` performs only local,
non-destructive optimizations: cell resizing, buffer insertion/removal, pin
swapping, and gate cloning. It preserves the overall structure. Use incremental
after: adjusting constraints and re-optimizing, adding path groups, or minor
ECO changes. Do NOT use incremental when the design has major timing violations
requiring structural changes.

### Q10: How do you handle max_delay constraints for CDC paths?

**A:** Use `set_max_delay -datapath_only` for CDC paths. The `-datapath_only`
flag is critical -- it constrains only the combinational data path delay between
the source and destination registers, excluding clock tree delays. Without this
flag, the tool would include clock path delays in the check, which is
meaningless for asynchronous crossings (there's no defined phase relationship).
The max_delay value should be set to ensure the CDC data arrives within the
reconvergence window expected by the synchronizer. Typical value: 1-2 clock
periods of the destination domain.

### Q11: What is wire load model and why is it obsolete?

**A:** Wire load models (WLM) are statistical lookup tables that estimate wire
capacitance and resistance based on net fanout. They were used in synthesis when
physical information wasn't available. They're largely obsolete at advanced
nodes (< 28nm) because: (1) Wire delay is a dominant portion of total delay
and statistical estimation is too inaccurate. (2) Topographical synthesis
(DCT/DC-Topo or Genus physical mode) uses actual placement data for wire
estimation. (3) Post-P&R timing can differ by 30-50% from WLM-based synthesis.
Modern flows use physical-aware synthesis or directly feed floorplan data.

### Q12: Explain the concept of critical range and its importance.

**A:** Critical range defines how deep below the worst negative slack the tool
should optimize. If WNS = -0.5ns and critical range = 0.3ns, the tool
optimizes all paths with slack between -0.5ns and -0.2ns. Without critical
range, the tool focuses only on the single worst path, which may fix WNS but
create new violations as resources are pulled from near-critical paths. Setting
appropriate critical range (typically 10-20% of clock period) distributes
optimization effort across all nearly-critical paths, resulting in better
overall TNS (Total Negative Slack) and faster convergence.

### Q13: What are Design Rule Violations (DRV) and why do they matter?

**A:** DRVs include max_transition (signal slew too slow), max_capacitance
(output overloaded), and max_fanout (too many loads). They matter because:
(1) Excessive transition time causes increased short-circuit power and can
violate library characterization ranges (making timing analysis inaccurate).
(2) Excessive capacitance stresses drivers and degrades reliability (EM).
(3) High fanout creates long-wire routing problems in P&R. The tool fixes
DRVs by inserting buffers, upsizing drivers, or restructuring logic. DRV
fixing can compete with timing optimization -- sometimes fixing a DRV worsens
timing on a critical path.

### Q14: How does multi-Vt optimization work?

**A:** The library provides cells in multiple threshold voltage variants:
HVT (slow, low leakage), SVT (moderate), LVT (fast, high leakage), and
sometimes ULVT. The tool starts with all-HVT (minimum leakage) and selectively
swaps cells on timing-critical paths to lower Vt for speed. The result is
typically 70-80% HVT, 15-25% SVT, and 5% LVT. This achieves 3-5x leakage
reduction compared to all-LVT while meeting the same timing. The
`set_multi_vth_constraint` command can limit the percentage of LVT cells,
preventing the optimizer from overusing fast-but-leaky cells.

### Q15: What is SAIF and how does it improve power optimization?

**A:** SAIF (Switching Activity Interchange Format) captures signal toggle rates
and static probabilities from RTL or gate-level simulation. Without SAIF,
synthesis assumes a default activity factor (typically 10%) for all signals,
leading to inaccurate power estimation. With SAIF back-annotation, the tool
knows actual signal activities and can: (1) Focus clock gating on
highest-activity registers. (2) Apply operand isolation to frequently-inactive
but power-hungry units. (3) Size cells appropriately for their actual switching
frequency. This typically improves power estimation accuracy from +/-50% to
+/-15% and enables 10-20% additional power optimization.

### Q16: When should you ungroup a module and when should you not?

**A:** Ungroup when: the module is on a critical timing path and hierarchy
boundaries prevent cross-module optimization; the module is small glue logic;
the module is a wrapper with no meaningful hierarchy. Do NOT ungroup when:
the module is large (flattening makes optimization intractable -- DC may run
out of memory); you need the hierarchy for ECO insertion; the module is
instantiated multiple times and needs to be preserved as a single optimized
entity; the module is a hard macro or IP. A good practice: ungroup small
modules (<5K gates) on critical paths, keep large modules (>50K gates) grouped.

### Q17: Explain the timing impact of source latency vs network latency.

**A:** Source latency is the delay from the actual clock source (e.g.,
oscillator) to the clock definition point (e.g., PLL input or chip clock
port). Network latency is the delay from the clock definition point through
the clock tree to flip-flop clock pins. Pre-CTS, network latency is estimated
(set by the user); post-CTS, it's replaced by actual propagated delay. Source
latency persists in both pre- and post-CTS analysis because it's external to
the clock tree. The distinction matters for IO timing: set_input_delay and
set_output_delay reference the clock definition point, so source latency
affects when the external clock edge "actually" arrives.

### Q18: What is operand isolation and when is it beneficial?

**A:** Operand isolation gates the inputs of arithmetic blocks (multipliers,
adders, ALUs) to zero when their output is not needed. This prevents
unnecessary toggling inside the block, saving dynamic power. It's beneficial
when: (1) The block is large (multiplier, divider) with significant internal
switching. (2) The block is inactive a significant fraction of cycles (>30%).
(3) The gating logic (AND gates on inputs) overhead is small relative to the
block. It's NOT beneficial for small blocks or blocks that are active nearly
every cycle. The area overhead is one AND gate per input bit.

### Q19: How do you handle timing closure when synthesis and P&R don't correlate?

**A:** Poor correlation means synthesis timing is optimistic relative to P&R.
Solutions: (1) Use topographical synthesis (DCT -spg or Genus physical mode)
with the actual floorplan -- this models wire delays based on placement,
improving correlation to within 5-10%. (2) Back-annotate parasitics from an
initial P&R run and re-synthesize with realistic wire delays. (3) Over-constrain
synthesis (tighten clock period by 10-15%, increase uncertainty) so that
synthesis timing has margin for P&R degradation. (4) Use consistent libraries
and operating conditions between synthesis and P&R. (5) Ensure DRV constraints
are tight enough to prevent P&R from needing excessive buffering.

### Q20: What is the purpose of group_path and how does it affect optimization?

**A:** `group_path` creates named path groups that the optimizer treats as
independent optimization targets. Without path groups, the tool's global
cost function may sacrifice one path group for another. With explicit groups:
(1) Each group gets dedicated optimization effort. (2) You can assign weights
(-weight 2.0) to prioritize certain groups. (3) You can see per-group WNS/TNS
in reports. Common groups: reg2reg, input-to-reg, reg-to-output, memory paths.
This is especially important when different path groups have very different
timing characteristics -- the tool won't "steal" cells from one group to help
another.

### Q21: Explain the three steps in Genus synthesis (syn_generic, syn_map, syn_opt).

**A:** (1) **syn_generic**: Technology-independent optimization. Performs
Boolean optimization, resource sharing, operator merging, clock gating
inference, and structural transformations. Output is a GTECH netlist.
(2) **syn_map**: Maps the generic netlist to target library cells. Performs
DAG covering, Boolean matching, cell selection for timing/area/power, and
initial DRV fixing. Output is a technology-mapped netlist. (3) **syn_opt**:
Post-mapping optimization. Performs local timing closure (cell sizing, buffer
insertion, pin swapping), hold fixing, leakage optimization, and DRV cleanup.
This three-step approach gives users control -- you can iterate on syn_opt
without re-running syn_generic and syn_map.

### Q22: How do you write RTL that synthesizes efficiently?

**A:** Key guidelines: (1) Use synchronous resets (async resets add logic to
every FF). (2) Avoid latches (use edge-triggered FFs). (3) Code enable
patterns so clock gating is inferred (`if (en) q <= d;` without explicit
else). (4) Avoid very wide case statements with many overlapping conditions.
(5) Keep arithmetic expressions simple and let the tool optimize (don't
manually instantiate adder structures unless absolutely necessary). (6) Use
named `generate` blocks for parameterized code. (7) Avoid combinational loops.
(8) Minimize the use of `casex`/`casez` (can infer X-propagation issues).
(9) Register all module outputs for better boundary optimization. (10) Use
consistent coding styles (always_ff, always_comb in SystemVerilog) for
reliable inference.

### Q23: What happens when you run compile_ultra with -retime?

**A:** The tool performs adaptive retiming: it identifies pipeline stages with
unbalanced combinational delays and moves registers to equalize them. For
example, if stage 1 has 1.5ns combo delay and stage 2 has 0.5ns, retiming
moves logic from stage 1 into stage 2, potentially achieving 1.0ns each. The
tool must verify: (1) Functional equivalence is preserved. (2) I/O timing
is not affected (registers at I/O boundaries are not moved). (3) Clock domain
crossings are not disrupted. (4) Reset behavior is preserved. Retiming can
improve maximum frequency by 10-30% without RTL changes, but it changes the
register locations, which can complicate debug and ECO.

### Q24: What is the difference between set_max_delay and set_multicycle_path?

**A:** `set_multicycle_path N` moves the capture edge to N cycles after launch,
and the hold edge accordingly. It maintains the relationship with the clock
structure. `set_max_delay` specifies an absolute maximum delay in nanoseconds,
independent of clock period. Use multicycle for synchronous paths where data
is valid for multiple cycles. Use max_delay for asynchronous CDC paths (with
`-datapath_only`) where there's no meaningful clock relationship. Do NOT use
max_delay as a substitute for multicycle on synchronous paths -- it bypasses
the clock-based timing framework and can lead to incorrect hold analysis.

