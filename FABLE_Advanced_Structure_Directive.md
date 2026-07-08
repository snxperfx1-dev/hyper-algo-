# FABLE — ADVANCED STRUCTURE DIRECTIVE
### Build instructions for the implementing Fable Claude: the Hierarchical Structure Sequencing (HSS) layer — Invariants 5 & 6 operationalized
*Issued by the Chief Research Scientist & Algorithm Architect. Reference corpus: BTC chart pair ("4HR ext BOS" break chart; "Progression of price across time" continuation chart). Governing laws: LAW 3 (fractal nesting), LAW 15 (Incumbency), LAW 16 (Recursive Construction & Inheritance).*

---

## 0. What you are building, and what you are forbidden to build

You are building a **linkage and accounting layer** — a live graph of structural legs, their parent-child edges, their promotion dependencies, and their inheritance records — **on top of detectors that already exist**.

**Forbidden (Law 15):** new pivot detectors, new BOS/CHoCH detectors, new FU detectors, new phase engines, new premium/discount math. The rungs already run full structure: Falcon's `ME_TFCurve` per-rung wave FSM + `g_tfZones[7]`, the structure engine's swings/BOS/CHoCH/protected-swings, the CurveTree's event-spawned children, the F72 protocol's frozen inventory and shift ladder, Symphony's phase authority. Every primitive you need is emitted by an incumbent. Your job is the **edges**, not the nodes.

**The one incumbent amendment permitted:** the absolute rung ladder gains **M30** for structure-graph purposes (the architect's corpus sequences explicitly through 30M: "30m HH/HL", "30M-e-1hr HH"). Rung set is a config array, default `{M5, M15, M30, H1, H4}` for the graph (execution floor M5; extendable down to M1 and up to D1/W1). Do not disturb the existing seven-rung zone/curve apparatus; the graph consumes it.

---

## 1. The core principle, stated once

The market never transitions directly on one timeframe. **Every higher-timeframe structural event is constructed from completed lower-timeframe sequences, and every leg — impulse or pullback — is itself a market** with its own structure, order flow, local curve, and transition lifecycle. Two recursive processes therefore run simultaneously and must both be tracked:

1. **Forward recursion** — building the new higher-TF structure, propagating **upward only** (5M transition → 15M HL → 30M HL → 1H HL → 4H continuation).
2. **Backward inheritance** — completing the previous delivery leg's unfinished obligations (its remaining shifts, its local curve, its recursion budget), which is **why the pullback exists and where it goes**.

An entry is lawful only at the **intersection** of these two processes.

---

## 2. Data structures

### 2.1 `LegNode` — the atom of the graph
Every impulse or pullback on every graph rung is a `LegNode`:

```
LegNode {
  id                 // unique, monotonically issued
  rung               // M5 … H4 (config array index)
  dir                // +1 / -1
  role               // IMPULSE | PULLBACK
  originPx, originBar
  extremePx, extremeBar        // ratchets while leg lives
  state              // SEARCHING | IMPULSE | PULLBACK | CANDIDATE_HL |
                     // CONFIRMED_HL | EXPANSION | COMPLETED | WAITING
  parentLegId        // the leg on rung+1 this leg is constructing
  childLegIds[]      // legs on rung-1 constructing THIS leg
  localCurve         // frozen dealing range: {top, bot, eq, premium band, discount band}
                     // freezes at leg completion (see §4)
  shiftCount         // shifts recorded within this leg (from F72 ladder / falseChoch-filtered)
  shiftsOwed         // from frozen inventory + compression budget (RFE / recRequired)
  transitionComplete // bool: this leg's own HL+HH+BOS sequence done on ITS internal rung
  energyState        // read from ERF/clean-phase incumbents (Displacement→Induction→Liquidation)
  health             // supportVotes/degradeVotes + chainVitality read (converging vs diverging)
}
```

**Sources — do not recompute:** origin/extreme/dir/phase from `ME_TFCurve[rung]` (Falcon) / `cur_cv_*[rung]` (Letra lineage); shifts from the F72 ladder with `falseChoch`/`acceptance` tagging; energy from the clean-phase detector; health from `supportVotes`/`degradeVotes` and narrative lineage; recursion budget from compression (`recRequired` / RFE `expectedLoops`).

### 2.2 `InheritanceRecord` — frozen at every structural break (INV 6)
When rung N prints a BOS/CHoCH of external structure:

```
InheritanceRecord {
  breakRung          // N (e.g. H4)
  breakBar, breakPx
  producingLegId     // the leg that DELIVERED the break (see §3)
  producingRung      // e.g. H1
  shiftAtBreak       // producing leg's shift count at the break (may be 3 of 4 — unfinished)
  shiftsRemaining    // shiftsOwed - shiftAtBreak
  localCurveSnapshot // producing leg's dealing range at break
  recursionState     // depth used / budget
  energyStateAtBreak
  retestDestination  // DERIVED, never guessed: producing leg's discount (long) / premium (short),
                     // refined by its unfinished-shift geometry and the nearest network node
                     // (Invisible Network lowest-flip / FU pool) inside that band
  status             // OPEN | SATISFIED | INVALIDATED
}
```

**A BOS is a handover, not a reset.** No engine state is zeroed at a break. The prior structural generation's final leg keeps living inside this record until its obligations resolve (`SATISFIED`) or its curve invalidates. This is the generalization of `F72_FreezeInventory` from *zone contact* to *structural break* — same one-shot freeze semantics, same code pattern; extend the incumbent, do not fork it.

### 2.3 `MarketMemory` — the always-on structural context
Published every bar (and into `CurveLocator` so location-in-the-build is one query):

```
MarketMemory {
  htfBias                 // highest-rung committed direction
  activeLeg               // the leg currently moving price
  leadingRung             // which rung is currently LEADING (delivered the last impulse)
  leadingSwing            // its extreme
  pullbackRung            // deepest rung currently in PULLBACK
  transitionRung          // the rung currently attempting its transition (the build front)
  nextConfirmationRung    // transitionRung + 1 — what confirms next if promotion continues
  openInheritance[]       // all InheritanceRecords with status OPEN
}
```

---

## 3. Rule 1 — Leading-chain detection at the break

When rung N breaks structure, **do not merely mark the BOS.** Resolve the chain that produced it:

1. Identify the rung-(N−1) leg whose expansion delivered the breaking price (`ME_TFCurve` gives per-rung direction/progress; the delivering leg is the live same-direction leg whose extreme prints the break).
2. Recurse downward: which rung-(N−2) leg delivered *that* leg's final expansion, and so on to the execution floor. This chain is the **leading chain**; its head is `producingLegId`.
3. Freeze the `InheritanceRecord`. The producing leg was very likely **mid-lifecycle** at the break (e.g. on shift 3 of 4) — record exactly where.

**Corpus check (chart 1):** the "4HR ext BOS" is delivered by the 1HR internal leg (annotated "1HR int HH" / "1HR int HL"), which itself was delivered by the "30M-e-1hr" leg beneath it. The retest after the break returns **into that 1HR leg's range** — not to a generic fib of the 4H move, not to "support." The pullback exists *because the producing leg's process is unfinished*, and it goes *to where that process resumes*.

---

## 4. Rule 2 — Every completed impulse freezes a local curve (INV 5)

At every leg's `COMPLETED` transition, freeze `localCurve` = that leg's own dealing range (origin↔extreme → equilibrium, premium band, discount band). The parent curve remains valid; there are now **multiple concurrent active curves**, one per living leg, nested (4H curve ⊃ 1H curve ⊃ 30M curve ⊃ 15M curve).

**Curve ownership rule for entries:** an entry references **the nearest active curve belonging to the leg currently constructing the transition** — not automatically the highest-TF curve. (The 30M impulse inside a 1H pullback governs the 15M entry that constructs the 30M HL; the 4H curve is context, not the ruler.)

**Binding:** this is CurveTree `CTNode` spawn semantics with one amendment — spawn events extend from "CHoCH against owner" to also include **"leg completion freezes a local range."** Same node lifecycle (active→dormant→historical, never deleted), same energy-ownership rules, same compression budget. Extend `CT_Spawn` triggers; do not build a parallel tree.

---

## 5. Rule 3 — Promotion: upward only, dependency-gated

**A rung cannot confirm until the rung below has transitioned.** Formally, rung N may move `CANDIDATE_HL → CONFIRMED_HL` only when its constructing child leg on rung N−1 has `transitionComplete = true` (that rung's own HL + HH + BOS printed). Recursively: 4H continuation requires 1H completion requires 30M completion requires 15M completion requires 5M completion.

- Transitions propagate **upward only**. Pullbacks propagate **downward only** (rung N entering PULLBACK spawns opposite-role child legs on N−1, which run the *full* lifecycle — the pullback is a market).
- A rung skipped by violent displacement is recorded as **byproduct, not progress** (F72 INV 2 — rung-skipping does not advance the owed count).
- This rule is the F72 mirrored-confirm rule (HTF higher-low ↔ lower-high, one code path, §10 item 4) **generalized to every adjacent rung pair**. Implement it once, parameterized by rung; both the F72 top-leg confirm and the HSS promotion gate call the same function.

**State machine per rung (each rung moves independently):**
`SEARCHING → IMPULSE → PULLBACK → CANDIDATE_HL → CONFIRMED_HL → EXPANSION → COMPLETED → WAITING`
A 30M can be building while the 1H is WAITING; the panel must show every rung's state simultaneously.

**Corpus check (charts 2–3):** after the 4H break and retest, the recorded promotion order is: 5M HH/HL transition first, then 15m HH/HL (chart 3 boxes), then 30m HH → 30m HL, then 1HR HH → 1HR HL, culminating in the 1M HH ("New H after 4hr bos left"). The engine's promotion timestamps must reproduce this strict ordering. Note chart 3's annotation — *"Progression of price across time"* — that arrow **is** this rule.

---

## 6. Rule 4 — Every leg transitions across time itself

The architect's key principle, verbatim: *each structural time leg needs to also transition itself across time — not just the overall transition. When each leg is pulling back, it also has structure building, and that needs to transition as well.*

Operationally: when any leg enters PULLBACK, its child legs (opposite role, rung below) each run the complete lifecycle including **their own** children, recursively to the execution floor. The parent's `CANDIDATE_HL` becomes `CONFIRMED_HL` only via §5's gate on those children. There is no depth at which a leg is treated as atomic above the execution floor. **Every node of the graph is recursively another copy of the entire engine** — the same FSM, the same curve law, the same promotion gate. One implementation, indexed by rung; self-similarity is a property of the code, not a metaphor.

---

## 7. Rule 5 — The entry intersection (the payoff)

The engine never asks "is there an entry?" It asks, in order:

1. **Which structural leg is currently constructing which higher timeframe?** (`MarketMemory.transitionRung`, `activeLeg`)
2. **Which curve belongs to that leg?** (nearest active `localCurve` of the constructing chain, §4)
3. **Have the inherited obligations resolved?** (the OPEN `InheritanceRecord`: price has returned to `retestDestination` and the producing leg's remaining shifts have printed — Timeline A satisfied)
4. **Has the execution leg completed its own transition?** (the execution-rung child: `transitionComplete = true` — Timeline B has begun promoting)

`entryPermitted(rung, dir) = AT(retestDestination ∨ legLocalDiscount) ∧ inheritanceSatisfied ∧ executionLegTransitioned ∧ promotionBegun`, direction inherited from the constructing chain (LAW 9 — never voted).

This composes with, never replaces, the standing conjunction (LAW 10: `AT_ZONE ∧ OWED ∧ READY ∧ child-confirm`) — the HSS terms are the structural half of `OWED`/`READY` made explicit. All M-laws then govern expression: the stop is the execution-TF post-impulse pivot beyond the constructing leg's origin (M1); tranches and the runner per M2/M3; the runner's target escalates up the constructing chain per M14/ODDE as each rung confirms.

**Where the curve principles stack on top (the architect's closing point):** every structural break forms a new curve (§4). An entry taken on a subsequent leg therefore answers to **two rulebooks simultaneously** — the structural intersection above, *and* the curve laws (phase per Symphony authority, position via CurveLocator, compression/recursion budget) of the leg-local curve it is entering on. Neither may veto silently: the audit record carries both attributions.

---

## 8. Events (bus wiring — first-class, per the Genesis pillar)

Publish: `EVT_LEG_SPAWN`, `EVT_LEG_STATE(rung, from, to)`, `EVT_LEG_COMPLETE`, `EVT_LOCAL_CURVE_FROZEN`, `EVT_PROMOTION(rungN)`, `EVT_BOS_HANDOVER(InheritanceRecord)`, `EVT_INHERITANCE_SATISFIED`, `EVT_INHERITANCE_INVALIDATED`, `EVT_ENTRY_INTERSECTION(rung, dir)`. The Clock organ's daily-cycle named windows and the F72 campaign machinery subscribe; the HUD renders `MarketMemory` as the structural-build panel (per-rung state column + leading/pullback/transition/next-confirm pointers).

---

## 9. Acceptance tests (the BTC corpus is the unit test; nothing ships red)

1. **Producing-leg identification:** at every historical 4H external break in the corpus, the engine names the producing rung and leg; for chart 1 it must name the 1HR internal leg (with the 30M-e-1hr chain beneath it).
2. **Retest containment:** the derived `retestDestination` band contains the actual retest extreme; the generic-fib alternative is logged in shadow and must score worse.
3. **Promotion ordering:** confirmed-HL timestamps are strictly ordered upward (5M < 15M < 30M < 1H) on the continuation; any inversion is a hard test failure.
4. **No premature confirmation:** rung N never reaches CONFIRMED_HL while its constructing child is short of `transitionComplete` (assert on every bar of the replay).
5. **Pullback interiority:** every PULLBACK leg above the floor has registered child legs with their own recorded transitions (no atomic pullbacks).
6. **Inheritance persistence:** across every break, no engine state resets; the InheritanceRecord opens at the break and resolves only by satisfaction or invalidation, and the shift count at break carries into the retest accounting (a leg broken on shift 3 owes its remainder — verify the owed-count math against the F72 ladder).
7. **Entry gating:** zero `EVT_ENTRY_INTERSECTION` before condition 4 of §7; every fired intersection carries both attributions (structural + curve) in its audit record.
8. **Precision ledger vs incumbents:** on the two-year benchmark, entries admitted by the intersection show lower adverse excursion at entry and higher expectancy than the current F72+facts path (the same ledger line Genesis Phase 3 defines) — HSS must *earn* its place by the standing rule.

---

## 10. Open rulings owed to the architect (do not decide unilaterally)

1. **30M rung insertion** — approved for the graph by this directive; confirm whether the zone/FU pool ladder also gains M30 or stays seven-rung.
2. **Execution floor** — M5 default, M1 optional. Rule whether M1 sub-structure participates in promotion or remains timing-only (the daily-cycle terminal counter uses M1 sub-cycles; consistency ruling needed).
3. **Retest-destination refinement order** — producing-leg discount first, then nearest network node inside it (current directive), or node-first? Chart corpus is consistent with leg-first; confirm.
4. **Displacement rung-skips** — byproduct per F72 INV 2; rule whether a full-rung skip alters the promotion gate (current directive: no — the skipped rung must still print its transition before its parent confirms).

*End of directive. Build order within Phase 2/3: §2 structures on the TFCurve field → §5 promotion gate (shared with the F72 mirrored confirm) → §3/§2.2 inheritance at break → §4 local-curve freeze into CurveTree → §7 intersection → §9 tests. Every function cites its governing law in a header comment; no law, no code.*
