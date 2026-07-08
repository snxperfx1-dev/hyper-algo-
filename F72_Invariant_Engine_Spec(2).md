# F72 — The Invariant Engine
### Build spec for "trade like me" — for red-pen review, no code yet

---

## 0. What this replaces and why

Today the entry decision reduces to one line in `IE_EntryCycle` (~4790):

```
bool reclaim = (g_state.structure.choch==w.direction && w.direction!=DIR_NONE);
if(ec.terminal && reclaim) rd=ER_ENTRY_ACTIVE;
```

One CHoCH in the terminal zone arms the trade. This is a **catalogue** entry — "when I see X, do Y." It has no notion of *which shift number* it's on, *what structure delivered price*, *whether the count is finished*, or *whether the energy is cleaned*. It fires on the first CHoCH, which is why it takes the 15M-break fakeout and eats the reversal.

The design principle from your six charts: **the algorithm must not hold a library of setups. It holds a small set of laws that are always true, and derives every situation live from them.** It never knows in advance what will happen; it knows what is *always* true and reasons forward. This spec is organized around that — four invariants, deduced live, emerging into a decision — not a state ladder.

---

## 1. The four invariants (the whole engine)

Everything below is a live deduction over the *existing* shared state. Nothing here is a hardcoded setup.

**INVARIANT 1 — There is always structure.**
Price is always hitting *something* (demand / supply / flip zone). Because structure is visible, the timeframes that *delivered* price into the zone can be deduced. → **Delivery-leg inventory.**

**INVARIANT 2 — Price always sequences across time, low → high.**
It is illogical for price to hit a zone and immediately make 1H/4H structure. It must build 1M, then 5M, then 15M, upward. The number of rungs that express is deduced live from how price is actually sequencing against the frozen inventory. → **Shift ladder.**

**INVARIANT 3 — There are always four entry shifts** (Wyckoff's spring/test/LPS1/LPS2 — the same imprint at highs and at S/D).
You never know *how* they'll be expressed (big loops / failure-swing / skipped rungs / nested). The engine tracks them *building*, never predicts them. The count is not literally 4 — it is: *shifts to climb the frozen leg inventory + the top leg's break-and-confirm.* Four is the common expression, not a constant. → **Shift count = emergent, not fixed.**

**INVARIANT 4 — Energy is always cleaned: Displacement → Induction → Liquidation.**
The market is energy (money = stored work = energy), so propagated energy *must* be cleaned before price can turn. A break is not a turn. Only after Displacement→Induction→Liquidation completes at that break/high is price *ready* to turn. This is a conservation law, apparent at every high and every transition, fractally. → **Turn-readiness gate.**

The entry decision is the **conjunction of the four deductions**, not the survival of a veto stack. This is the structural cure for the combinatorial brittleness in `SymphonyFactsConfirm`: instead of AND-ing ~10 independent vetoes (any one kills the trade), the engine asks whether four always-true laws currently point the same way.

---

## 2. The campaign object — laws held per zone, on the existing tree

The invariants are held **per terminal region** in a campaign object. Critically, from Charts 2–4: a campaign is **keyed to the zone / node identity**, is **persistent across excursions** (survives a full 250-pip round-trip away and back), and there can be **multiple concurrent** campaigns of opposite orientation (demand *and* the high), which **hand off** to each other.

This must **not** be a flat array bolted alongside the engine. It is **semantics layered onto the existing `CTNode` tree** (`ct_tree[]`, 3663). The tree already models curves-inside-curves with ownership transfer:

- `ownerNodeId` (403) — *flips on ownership transfer*. Today that flip **orphans** the demand campaign when price runs to the high. Under this model the flip is **correct and wanted**: it *spawns the high's campaign* while the demand campaign sits dormant-but-alive at 3-of-4. Two live nodes, neither deleted.
- `CTNode.depth` — recursion nesting = TF-rung distance (the fourth shift's nested sub-sequence).
- `supportVotes` / `degradeVotes` (414–415) — converging vs diverging pullback votes = the **FU-respect / campaign-merge** discriminator (Principle 9). Child respects parent FU → support-dominant → *do not reset the count, do not flag transition-complete*.
- `budgetDepth` (400) — compression-derived; governs how shifts *express* (big loops vs failure-swing), never how many are *owed*.

**New per-campaign fields** (proposed — a struct keyed to the owning node id, or fields on an extended node record):

```
struct F72Campaign
{
   int    nodeId;            // owning CTNode id (survives excursions; 0 = dormant slot free)
   int    zoneOrient;        // DIR_LONG at demand / DIR_SHORT at a high — symmetric
   double zoneTop, zoneBot;  // the anchoring zone band (or lowest FU flip)
   int    status;            // BUILDING / PARTIAL / SOS_LOCKED / READY / COMPLETE / DORMANT / DEAD

   // INVARIANT 1 — delivery-leg inventory, FROZEN at zone contact
   bool   legExists[7];      // per rung: did a discrete HH/HL of that TF's scale deliver price in?
   int    topLegRung;        // highest rung with a delivering leg (the SOS leg)

   // INVARIANT 2/3 — shift ladder against the frozen inventory
   bool   rungBroken[7];     // which frozen legs the shifts have broken (low->high)
   bool   rungConfirmed[7];  // break-AND-confirm state per rung (top leg needs both)
   int    shiftCount;        // shifts banked (pullback-confirmed breaks only)
   int    expectedShifts;    // = deduced from inventory: climb legs + top-leg confirm
   bool   sosFired;          // top delivery leg broke -> Sign of Strength -> return OWED

   // INVARIANT 4 — turn-readiness (energy cleaning) scoped to THIS node's turn
   int    cleanPhase;        // NONE / DISPLACEMENT / INDUCTION / LIQUIDATION / CLEANED
   bool   turnReady;         // cleanPhase == CLEANED at the break/high

   int    childCampaign;     // nodeId of the nested 4th-shift sub-campaign (recursion)
};
```

A small registry (`F72Campaign g_camp[N]`, N≈4–6 capped by flip-zone count, **not** curve count — see §7) holds the live campaigns. Spawn / dorm / kill is driven by `ownerNodeId` transfer events, not by a scan.

---

## 3. INVARIANT 1 — the delivery-leg inventory (frozen at contact)

**Tell (from your description, Charts 5–6):** you *shift to each timeframe and look for a clear HH+HL of that TF's scale in the approach.* If 5M shows no clean higher-high/higher-low, that rung has no leg — not enough space — and price skips it.

**Deduction rule:** at zone contact, for each configured rung `k` (1M→5M→15M→30M→1H→4H→…), `legExists[k] = true` iff a **resolved swing pair at rung k's pivot scale** (a pivot high + a confirming pivot low, or the mirror) exists in the impulse **between the pre-zone origin and the zone**. Present = leg; absent = skipped rung.

- Raw material already computed: `waveMatrix.dir[7]`/`phase[7]`/`progress[7]` (per-TF wave grid) and `htf.dir[7]`. Nothing today reads them as "the structural legs standing between this zone and price, ordered by TF." This deduction is the missing read.
- **FROZEN at contact.** The delivery legs are what *brought* price in; they do not change while price works the zone. This is what lets SOS be pinned to a specific leg (§5). If the inventory re-derived live, "the delivery leg" would drift and SOS could never attach. → one-shot snapshot on the spawn event.
- `topLegRung` = highest `k` with `legExists[k]`. This is the SOS leg.

**Chart-6 worked example:** only 15M and 1H legs were visible in the approach → `legExists = {…,15M:1,1H:1}`, `topLegRung = 1H`. 5M/1M had no discrete leg → skipped; the in-zone 1M shifts *manufacture* 1M structure as byproduct.

---

## 4. INVARIANT 2/3 — the shift ladder (low→high, rung-skipping)

**Deduction:** shifts build from the lowest rung up. When a shift prints (a **pullback-confirmed** structural break — not a raw level touch, Chart 6: 15M broke and price *could* have pulled back = a shift, but ran straight on = folded), map it to the **lowest unbroken `legExists` rung** and mark `rungBroken[k]`. A break at a rung with `legExists[k]=false` is tagged **byproduct** and does **not** advance the ladder (this is 5M being skipped).

- `shiftCount` increments only on pullback-confirmed breaks of *inventory* legs (or the in-zone failure-swing + micro shifts that climb to the first real leg).
- `expectedShifts` is **deduced, not hardcoded**: number of `legExists` rungs to climb, **plus one** for the top leg's confirm (break-and-confirm counts as two events on the top rung → this is how 3-climb + 1-confirm reconstructs "four" from a 1H-only delivery). Lower legs cost one each; **top leg costs break + confirm.** *(This accounting is the open item flagged in §8.)*
- Compression/`budgetDepth` governs *expression* only: high compression → failure-swing + tiny micro-shifts; low → big loops. Same count, different shape.

---

## 5. Sign of Strength + expansion lock (the top-leg break)

From Chart 7: the break of the top delivery leg (15M/1H) is normally **shift 2 or 3**, and that break **is the Sign of Strength** — the market announcing opposing strong-hand interest (Wyckoff SOS).

**Law:** SOS and true expansion **cannot be the same event.** You cannot have SOS-breaks-1H and immediately run the entry expansion — illogical. So:

- On the top-leg break: `sosFired = true`, `status = SOS_LOCKED`. A return is now **OWED**.
- **Expansion permission is denied** until the top leg is **broken AND confirmed** (`rungBroken[topLegRung] && rungConfirmed[topLegRung]`). This replaces the `wave.completion < maxEntryComplete` / `geometryCapacity > minEntryRoomPct` heuristics in both `EE_HandleEntries` (5431–5433) and `SymphonyFactsConfirm` (8144–8146). Those heuristics *proxy* "is there room"; the real question is "has price earned structural permission on the timeframe above." SOS-locked = not earned yet.
- The confirm (the 1H higher-low / lower-high) **is** the fourth shift. It is not a point — it is a **nested campaign** (§6).

---

## 6. INVARIANT 4 — turn-readiness (energy cleaning), and recursion

**Two separate gates must both be true to arm the counter (Chart 8):**

1. **OWED** (count/structure): `sosFired && shiftCount < expectedShifts` → a return is owed.
2. **READY** (energy): `cleanPhase == CLEANED` → Displacement→Induction→Liquidation has completed at the break/high.

This is the discriminator that was missing — *why sell the 1H break but not the 15M break.* When 15M broke, price had only **displaced** (vented straight-line force) and kept going → not ready. By the 1H HH, the full clean had run at the high → ready. The counter arms only on **OWED AND READY**.

**Reuse, don't rebuild:** the engine already detects this as `IE_LiquidationWave` / `liqSubPhase` (Push/Displacement/Induction/Terminal Liquidation/Objective Arrival, 4699–4704) and the phases `PH_INDUCTION`/`PH_LIQUIDATION`/`PH_EXP_INDUCTION`/`PH_EXP_LIQUIDITY`. Today these run as **labels on the master wave**. The fix: **scope them to each campaign node's turn** — `cleanPhase` per campaign, driven by the existing displacement/induction/liquidation detection applied to that node's break/high. `IE_LiquidationWave` stops being a global label and becomes the per-campaign readiness gate. This is a re-wiring of machinery you already built.

**Recursion (Chart 6, closing the loop):** the fourth-shift confirm leg descending into the zone is **itself a curve** → it spawns a **child campaign** (`childCampaign`) that runs its own frozen inventory + ladder + clean gate, floored at 1M, rung-skipping allowed. Because 1H is already broken above, the *lower* rungs (1M/5M/15M) are the relevant ones. "Price needs to build as it's completing the fourth shift." Same code path on a deeper `CTNode.depth`. **Terminator:** nesting bottoms out at the lowest configured rung (1M); rungs with no visible structure are simply skipped, exactly like 5M on the way up.

---

## 7. Orientation symmetry and hand-off (highs run it too)

From Charts 3–4: the high is a curve, so it runs its **own** four-shift transition on the **sell** side. There are **N concurrent campaigns**, opposite orientation, that point at each other:

- Demand campaign (long): banks 3 shifts, SOS-locks, expands prematurely → the high.
- The `ownerNodeId` flip at the high **spawns the high's campaign** (short) — this is the flip that today orphans demand; now it's the spawn event.
- High campaign runs its inventory/ladder/clean identically (mirrored). When OWED+READY, it arms the **counter-sell**, **targeting the demand zone that still holds the partial count** — which is itself the highest-value long setup on arrival.
- The counter-sell's liquidation *is* the excursion that carries the demand campaign back for its 4th shift. **Hand-off.**

**Direction is inherited from zone orientation, never voted.** Buy at demand, sell at the high — same laws read in opposite orientation. This is the fluidity: one reasoning loop, both sides, no per-case code.

**Concurrency ceiling:** capped by **flip-zone count** (owner-TF-or-higher flips — the "lowest active flip" discipline), not curve count. *(Confirm ceiling — §8.)*

---

## 8. Emergence — how entry-active and direction fall out

`entryCycleActive` becomes an **emergent conjunction**, replacing the `reclaim` line:

```
For the campaign owning price at the current zone:
   OWED    = sosFired && (shiftCount < expectedShifts)      // INV 3 — count unfinished
   READY   = (cleanPhase == CLEANED)                         // INV 4 — energy cleaned
   AT_ZONE = price within zoneTop..zoneBot                   // INV 1 — location
   entryDir = zoneOrient                                      // inherited, never voted

   entryCycleActive = AT_ZONE && OWED && READY && (child confirm building or done)

Expansion permission (separate, for the post-4th-shift run):
   canExpand = rungBroken[topLegRung] && rungConfirmed[topLegRung]  // INV 2 — top leg broken+confirmed
```

The counter (sell at the high, back to demand) is the *same* emergence on the high's campaign with `zoneOrient = DIR_SHORT`.

---

## 9. Wiring into the existing code

**`FalconEntryCycle` (667) — add:**
`shiftCount`, `expectedShifts`, `sosFired`, `topLegRung`, `cleanPhase`, `turnReady`, `canExpand`, `activeCampaignId`, `legExists[7]`, `rungBroken[7]`, `rungConfirmed[7]`. Keep existing fields (nothing cut).

**`IE_EntryCycle` (4726) — replace the `reclaim` block (4785–4799)** with:
1. resolve/spawn the campaign owning price (from `ownerNodeId`);
2. on spawn: freeze `legExists[]` inventory (INV 1);
3. score shifts against frozen inventory, set `sosFired` on top-leg break (INV 2/3);
4. drive `cleanPhase` from the scoped `IE_LiquidationWave` (INV 4);
5. emit `entryCycleActive` / `entryDir` / `canExpand` from §8.

**`CurveTreeRun` (3633) — `ownerNodeId` transfer:** on flip, **spawn** the new-orientation campaign and set the prior to DORMANT (not deleted). This is the demand↔high hand-off. Uses `supportVotes`/`degradeVotes` for the FU-merge check (don't reset a count when a child respects the parent FU).

**Replaces / retires:** the `reclaim` trigger; the `wave.completion`/`geometryCapacity` room heuristics (→ `canExpand`); collapses much of the flat veto stack in `SymphonyFactsConfirm` into the four-law emergence.

**Orphaned fields finally consumed:**
- `falseChoch` (2476) / `acceptance` (2477) → flip-vs-fib-interference tag on a shift (only terminal-class breaks count).
- `clusterDensity` (2458) / `vacuum` (2460) / `sweepProbability` (2468) → terminal-zone compression + the "cleaned/ready" read.
- `waveMatrix.*` / `htf.dir[7]` per-rung → the delivery-leg inventory (INV 1).
- `supportVotes`/`degradeVotes` → campaign-merge (Principle 9).

---

## 10. Open items for your red pen

1. **Top-leg shift accounting.** Does the highest leg always cost **two** shifts (break the high, then the pullback that confirms the higher-low), while lower legs cost **one** each? That's how §4 reconstructs "four" from a 1H-only delivery (3 climb + 1 confirm). Right, or does every leg carry its own break+confirm?

2. **Concurrency ceiling.** Cap the registry by **flip-zone count** (owner-TF-or-higher flips)? Or a different bound?

3. **`cleanPhase` source of truth.** Reuse `liqSubPhase` scoped per-node as the Displacement→Induction→Liquidation detector, or does the cleaning read want its own detector off `clusterDensity`/`vacuum`/velocity-collapse?

4. **"Confirm" definition, mirrored.** For a demand (long) campaign the top-leg confirm is the HTF **higher-low**; for a high (short) campaign it's the HTF **lower-high**. Is that the single mirrored rule to code?

5. **Spawn granularity.** Does *every* new high that breaks its internal trend spawn a short campaign, or only a high **at an FU/flip the engine already mapped**? (Bounded vs runaway — I've assumed flip-mapped only.)

---

*Six charts → one closed, self-similar mechanism: zone-anchored shift sequencer · orientation symmetry · cross-TF delivery-leg ladder · SOS + expansion permission · energy-cleaning turn-readiness · recursive nesting — all as live deductions from four always-true laws, layered on the existing CurveNode tree. No code until this is marked up.*
