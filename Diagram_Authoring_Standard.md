# Diagram authoring standard — choose the notation that matches the mechanism

Mermaid is useful for ownership, sequence, and architectural dataflow, but its boxes and arrows are not electrical symbols. A transistor stack, feedback latch, carry cell, clock aperture, or bus waveform loses essential information when redrawn as generic rectangles. This notebook therefore uses a **multi-notation diagram stack**.

## 1. Required notation by question

| Question being answered | Default notation | Why | Do not use as the primary figure |
|---|---|---|---|
| Which transistor, gate, storage element, or arithmetic cell is connected to which? | **TikZJax + CircuiTikZ** (`tikz` fenced block) | standard electrical and logic symbols; text source is reviewable in Git; labels and equations share one typesetting system | Mermaid rectangles pretending to be gates |
| What changes on each cycle, phase, or handshake edge? | **WaveDrom** | compact clocks, buses, unknowns, gaps, and named phases | a prose-only timing list |
| How does a request, token, or transaction move through several components? | **Mermaid** flowchart or sequence diagram | automatic layout and readable causal ownership | a transistor schematic |
| How does voltage/current evolve when the reader changes a component? | **CircuitJS**, optional interactive lab | executable circuit plus probes and live parameter changes | a static figure claiming to be a simulation |
| Does a manually placed overview need physical grouping or a custom visual grammar? | **draw.io `.drawio.svg`**, exceptional | editable while remaining a normal SVG outside Obsidian | opaque screenshots or non-portable binary exports |

One subject often needs more than one figure. For example, an SR latch needs a CircuiTikZ gate schematic, a WaveDrom command trace, and a state diagram; none substitutes for the others.

## 2. Adopted plugins and the decision

### 2.1 TikZJax is the default circuit renderer

The vault tracks [Obsidian TikZJax](https://github.com/artisticat1/obsidian-tikzjax), whose documented package set includes `circuitikz`. It renders inline `tikz` blocks and keeps the schematic source beside the explanation. The currently vendored release is `0.5.2`; this is intentionally conservative because its supported package set and fenced-block contract are explicit.

The newer [TeXcore](https://community.obsidian.md/plugins/texcore) is worth re-evaluating after it matures. It adds asynchronous WebAssembly compilation, package fetching, equation references, and a basic editor, but at the time of this decision it is a new `0.0.x` plugin whose first render downloads runtime/package assets. It is not yet the notebook's reproducibility baseline.

### 2.2 CircuitJS is an optional executable companion

[Obsidian CircuitJS](https://github.com/StevenGann/obsidian-circuitjs) can embed interactive offline simulations from `circuitjs` blocks. Use it when interaction changes the lesson—for example, sweeping an RC time constant, observing a slow regenerative resolution, or demonstrating charge sharing. Do not duplicate every static schematic as a simulator blob: CircuitJS's exported netlist is less readable in review than CircuiTikZ source.

### 2.3 draw.io and Kroki are not the default

The current [draw.io Obsidian plugin](https://community.obsidian.md/plugins/drawio) works offline and stores editable `.drawio.svg` files. It is appropriate for a carefully placed poster or package/floorplan view, but manual XML/SVG changes are harder to review and keep stylistically uniform than text-authored circuits.

[Obsidian Kroki](https://github.com/gregzuro/obsidian-kroki) supports many renderers, including TikZ, WaveDrom, Symbolator, and WireViz. Its default path sends source to a rendering server; a self-host removes that dependency but adds infrastructure. The notebook already renders its selected languages locally, so Kroki would add a second rendering path without improving the core circuit lessons.

## 3. CircuiTikZ block contract

Every circuit block must:

1. load `circuitikz`, then include `\begin{document}` and `\end{document}` because TikZJax uses the standalone document class;
2. label inputs, outputs, power rails, clocks/enables, and any internal node discussed in prose;
3. use a conventional left-to-right signal direction or top-to-bottom supply direction;
4. show complementary bubbles or overbars explicitly rather than relying on color;
5. be followed by a causal trace: what turns on first, which node moves, what regenerates, and what state remains;
6. state what the schematic abstracts away—parasitics, device sizing, body connections, analog slopes, or library-cell internals;
7. have a text explanation sufficient to retain meaning if a renderer is temporarily unavailable.

### 3.1 Obsidian-safe layout contract

Use one visual grammar across the vault:

- Keep the drawn bounding box at or below **12.5 cm wide** and about **6.5 cm high**. A long pipeline must fold into two rows with a clearly marked continuation instead of shrinking labels or overflowing the note.
- Start circuit environments with `american, thick, scale=0.9, transform shape` unless a device-level vertical stack needs its natural size.
- Place ordinary data inputs on the left and outputs on the right. Bring selects, clocks, enables, resets, and wordlines from below; place supplies above and grounds below.
- Leave at least **1.2 cm** between neighboring gate bodies and at least **0.8 cm** between parallel wires. Feedback may wrap above or below the forward path, but must not cross a label or masquerade as an unconnected wire.
- Put only the terminal name (`Y`, `Q`, `sum`, `grant`) at a schematic output. Put long Boolean equations in the paragraph or display equation beside the figure, not on a wire that expands the canvas.
- Use `node[circ]{}` at every functional fan-out or feedback junction. A crossing without a dot is not a connection.
- Prefer a small repeated-cell drawing plus an ellipsis/bracket over showing dozens of identical cells. The prose must state the replication count and index direction.
- Use rectangular TikZ blocks only for owned hardware structures such as a decoder, mux stage, register bank, or priority cell. Use CircuiTikZ gate/device symbols wherever the figure claims gate- or transistor-level connectivity.

After changing a figure, inspect it in Obsidian reading view at the normal note width. Reject it if any label overlaps a wire, a complementary mark is ambiguous, the reader must scroll horizontally, or the signal direction reverses without an explicit continuation arrow.

Minimal plugin/render test:

```tikz
\usepackage{circuitikz}
\begin{document}
\begin{circuitikz}[american,thick,scale=0.9,transform shape]
  \node[and port] (g) at (0,0) {};
  \draw (g.in 1) -- ++(-0.8,0) node[left] {$A$};
  \draw (g.in 2) -- ++(-0.8,0) node[left] {$B$};
  \draw (g.out) -- ++(0.8,0) node[right] {$Y$};
\end{circuitikz}
\end{document}
```

If that block does not render after pulling the vault, reload Obsidian so the newly tracked community plugin is activated.

## 4. The explanatory contract around every figure

A diagram is evidence only when the text makes its logic reconstructable. Introduce figures in this order:

1. **Contract:** the observable behavior the structure must guarantee.
2. **Minimum circuit/model:** the smallest structure meeting the contract.
3. **Concrete trace:** one input vector, transition, or transaction through named internal nodes.
4. **Failure:** a vector, timing relationship, workload, or physical corner that breaks or limits it.
5. **Derived repair:** the exact added gate, state, queue, path, precision bit, or protocol phase.
6. **Replay:** run the same trace through the repaired structure.
7. **Cost:** delay, area, energy, wiring, numerical error, simulator speed, and verification state.
8. **Selection boundary:** when the older/simpler structure still wins.

This contract applies to all Fundamentals and architecture pages, independent of diagram language.
