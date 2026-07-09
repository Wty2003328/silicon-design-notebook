# Verification Planning and Coverage Closure

> **Stage:** 03 · Verification. The *flow* that wraps the mechanics: how you decide what to verify (vplan), measure progress (coverage), and declare "done" (closure) — distinct from the testbench machinery in [UVM_Methodology](10_UVM_Methodology.md) and [Assertions_and_Coverage](09_Assertions_and_Coverage.md).
> **Prerequisites:** [UVM_Methodology](10_UVM_Methodology.md), [Assertions_and_Coverage](09_Assertions_and_Coverage.md). **Hands off to:** sign-off / tape-out readiness.

---

## 0. Why this page exists

Verification consumes ~60–70% of an ASIC project's effort, and the hardest question isn't "how do I write a test" — it's **"am I done?"** You can run a million random cycles and still have never exercised the FIFO-full-during-reset corner. Coverage-driven verification (CDV) answers "done" with data: a **verification plan** enumerates what must be checked, **coverage** measures what's actually been hit, and **closure** is the disciplined loop that drives coverage to the target. This page is that methodology — the management layer over [UVM](10_UVM_Methodology.md) and [SVA](09_Assertions_and_Coverage.md).

---

## 1. The verification plan (vplan)

The vplan is the contract for *what* gets verified, derived from the **design spec** and **µarch spec**, before tests are written.

| Element | Content |
|---|---|
| **Features** | every spec feature → one or more verification items |
| **Stimulus** | how each feature is exercised (directed, constrained-random, sequences) |
| **Checks** | how correctness is judged (scoreboard, reference model, assertions) |
| **Coverage** | the metric that proves the feature was hit (functional cover points, assertions) |
| **Owner / status** | who, and pass/fail/coverage% |

The vplan is **executable**: modern tools link each vplan item to the coverage objects that close it, so the plan auto-annotates with live coverage numbers. Unlinked vplan items = un-verified features = risk.

**Worked example — decomposing features into tests + coverage** (block-level test plan):

```ascii-graph
Test Plan for [Block/Feature]
|
+-- Feature 1: Basic Read/Write
|     +-- Test: single_write_read        [directed]
|     +-- Test: random_write_read        [constrained random]
|     +-- Coverage: addr ranges, data patterns
|
+-- Feature 2: Burst Transactions
|     +-- Test: burst_all_lengths        [directed sweep]
|     +-- Test: random_bursts            [constrained random]
|     +-- Coverage: burst_len x burst_type cross
|
+-- Feature 3: Error Handling
|     +-- Test: address_out_of_range     [error injection]
|     +-- Test: protocol_violation       [negative test]
|     +-- Coverage: error types, recovery paths
|
+-- Feature 4: Corner Cases
|     +-- Test: back_to_back_txns        [stress]
|     +-- Test: reset_during_txn         [interrupt]
|     +-- Test: fifo_full_empty          [boundary]
|     +-- Coverage: boundary conditions
```

And the executable feature→coverage mapping, AXI4 slave:

| Feature | Test Type | Coverage Item | Target |
|---------|-----------|---------------|--------|
| Single read | CR (constrained random) | `cg_axi_read.bv_addr` | 100% |
| Burst read (INCR4) | CR | `cg_axi_burst.bv_burst_len` with `len==4` | 100% |
| Out-of-order completion | CR | `cg_ooo.cp_ordering` | 100% |
| Error response (SLVERR) | Directed | `cp_slverr` | 100% |
| Back-to-back transactions | CR | `cg_perf.cp_back2back` | >80% |

The verification plan is the contract between design and verification teams. Each row maps a design feature to a measurable coverage goal -- without this mapping, coverage numbers are meaningless.

### Coverage Closure Methodology

---

## 2. The coverage taxonomy

"Coverage" is several different measurements; you need them all because each is blind to what the others catch.

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk", "nodeSpacing": 60, "rankSpacing": 60, "htmlLabels": false}}}%%
flowchart TD
    COV["Coverage"]:::r --> CODE["Code coverage<br/>(automatic)"]:::a
    COV --> FUNC["Functional coverage<br/>(you write it)"]:::b
    COV --> ASRT["Assertion coverage"]:::c
    CODE --> L["line / statement"]:::a
    CODE --> T["toggle"]:::a
    CODE --> BR["branch / FSM-state+transition"]:::a
    FUNC --> CP["covergroups / cover points"]:::b
    FUNC --> CR["cross coverage"]:::b
    classDef r fill:#fde68a,stroke:#b45309,color:#000
    classDef a fill:#bbf7d0,stroke:#15803d,color:#000
    classDef b fill:#bae6fd,stroke:#0369a1,color:#000
    classDef c fill:#c7d2fe,stroke:#4338ca,color:#000
```

- **Code coverage** (automatic): line, statement, branch, toggle, FSM state+transition. Measures *"did my tests execute the RTL?"* It is **necessary but not sufficient** — 100% code coverage with the wrong checks proves nothing about correctness. Its real value is the inverse: **un-covered code = definitely un-tested**.
- **Functional coverage** (hand-written `covergroup`s): measures *"did my tests hit the scenarios I care about?"* — the FIFO going full, every opcode, every burst length. This is the one tied to the spec/vplan. **Cross coverage** captures *combinations* (full FIFO **AND** back-pressure **AND** error injection) — where the real bugs hide.
- **Assertion coverage**: did the [SVA](09_Assertions_and_Coverage.md) properties actually fire (not vacuously pass)?

---

## 3. Closure — the loop that ends verification

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk", "nodeSpacing": 60, "rankSpacing": 60, "htmlLabels": false}}}%%
flowchart TD
    R["Run regression<br/>(random + directed)"]:::a --> M["Merge coverage<br/>across all runs"]:::b
    M --> H["Find holes<br/>(un-hit cover points)"]:::c
    H -->|holes exist| W["Close the hole:<br/>new seed? new constraint?<br/>new directed test? RTL/cov bug?"]:::d
    W --> R
    H -->|no holes & all checks pass| DONE["Coverage closed →<br/>sign-off"]:::e
    classDef a fill:#fde68a,stroke:#b45309,color:#000
    classDef b fill:#bbf7d0,stroke:#15803d,color:#000
    classDef c fill:#bae6fd,stroke:#0369a1,color:#000
    classDef d fill:#fecaca,stroke:#991b1b,color:#000
    classDef e fill:#c7d2fe,stroke:#4338ca,color:#000
```

For each coverage hole, the triage is a decision tree:
1. **Reachable by more random?** → add seeds / loosen constraints (cheap).
2. **Random can't reach it economically?** → write a **directed test** or a targeted constraint.
3. **Truly unreachable?** → it's dead code or an impossible cross → **waive** it (justified) or fix the coverage model.
4. **Reachable but the test fails there?** → a real **DUT bug** → fix RTL.

Closure also means **all checks pass** (zero failing assertions/scoreboard mismatches) and **regression is green and stable** (no flakiness). Coverage at 100% with failing tests is not closed.

---

## 4. Constrained-random + coverage feedback (CDV)

The dominant methodology: **constrained-random** stimulus generates volume and reaches corners a human wouldn't script; **functional coverage** measures what it actually hit; the gap drives new constraints/seeds. Optionally **coverage-driven generation** automatically biases the random generator toward un-hit bins. This is why UVM is built around randomization + covergroups — they're the two ends of the same loop.

Directed tests still matter for: reset/init sequences, known-hard corners, and quick smoke tests. The real flow is random-for-breadth + directed-for-the-stubborn-corners.

**The canonical CDV cycle** — from plan to closure:

1. **Write verification plan** — decompose every design feature into testable requirements.
2. **Create coverage model** — instrument covergroups, coverpoints, and crosses that signal "this feature has been exercised."
3. **Develop constrained-random stimulus** — let the generator explore the legal space; use directed tests only for corner cases the randomizer cannot reach.
4. **Simulate and collect coverage** — run regressions with multiple seeds; merge coverage databases.
5. **Analyze coverage holes** — triage every uncovered bin (decision tree in §3).
6. **Add directed tests** — only after confirming a stimulus gap (not a model gap or dead code).
7. **Declare closure** — when functional and code coverage targets are met across all seeds.

Skipping step 5 is the most common mistake: chasing 100% without understanding *why* a bin is uncovered wastes effort on dead-code bins that can never fire.

**Regression management tiers:**

- **Nightly regression**: full test suite, all seeds, report coverage.
- **Smoke regression**: subset of critical tests, fast turnaround for CI.
- **Coverage-focused**: run only tests that target uncovered bins.
- **Seed management**: save seeds that hit interesting coverage, replay for debug.

Track coverage trend charts across nightly runs: a sudden coverage drop almost always indicates a regression bug in stimulus generation, not a design bug.

---

## 5. Sign-off criteria (the gate)

A block is "verification-signed-off" when:
- **Functional coverage = 100%** of the vplan-linked cover points (or every gap waived with justification).
- **Code coverage ≥ target** (commonly ~95–100% line/branch, ~90%+ toggle), gaps reviewed.
- **All assertions** pass and are **proven non-vacuous** (assertion coverage).
- **Regression green** across a clean, reproducible seed set.
- **Formal** ([Formal_Verification](12_Formal_Verification.md)) has discharged the properties it owns (control logic, CDC, connectivity).
- **GLS** ([Gate_Level_Sim_and_Emulation](13_Gate_Level_Sim_and_Emulation.md)) sanity + reset passes.

---

## 6. Numbers to memorize

| Quantity | Value | Why |
|---|---|---|
| Verification share of effort | ~60–70% of project | the dominant cost |
| Code coverage target | ~95–100% line/branch | un-covered = un-tested |
| Functional coverage target | 100% of vplan bins | the spec-linked metric |
| Code vs functional | necessary vs sufficient | need both |
| Where bugs hide | **cross** coverage bins | combinations, not singles |
| Vacuous assertion | passes without antecedent | assertion coverage catches it |
| Closure = | 100% cov **and** all checks pass **and** stable regression | not just coverage |

---

## Cross-references
- Mechanics: [Assertions_and_Coverage](09_Assertions_and_Coverage.md), [UVM_Methodology](10_UVM_Methodology.md), [OOP_and_Randomization](08_OOP_and_Randomization.md).
- Complementary engines: [Formal_Verification](12_Formal_Verification.md), [Lint_CDC_RDC_Signoff](07_Lint_CDC_RDC_Signoff.md), [Gate_Level_Sim_and_Emulation](13_Gate_Level_Sim_and_Emulation.md).
