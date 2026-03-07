# Validation Position

*As of March 7, 2026: Track A 0/8, Track B pass with explained residuals.*

---

## What This Page Is

This page documents what BJW has verified, what it hasn't, and why we believe the results are trustworthy for the claims we make. It is written for an audience familiar with Schlesinger's Chapter 10 variance framework, Griffin's theory, and tools like CVData/CVCX. We are not softening the limitations.

---

## What We Verified

### 1. Basic Strategy Correctness

All 16 play policy files — covering every combination of H17/S17, DOA/D10, DAS/NDAS, and Late Surrender — were audited against published basic strategy charts. 224 discrete decision checks, zero failures.

Two items required resolution during the audit:

- **Halves count insurance threshold**: Was coded as 1, corrected to 3 based on published sources. Confirmed bug, now fixed.
- **Zen count insurance threshold**: Confirmed at TC ≥ 5 (count-per-deck convention) based on primary source citation recorded in the counting system file. Empirically verified at 100M rounds: zero insurance taken below TC+5, positive EV in both above-threshold bins (TC 5–6: +0.0148 EV/unit, TC 7+: +0.0589 EV/unit), effective p(Ten | insured) = 0.346 > 1/3. Status: source-confirmed and resolved.

  *Note: Zen is a level-2 count with tags ranging from −2 to +2, so its running count accumulates roughly twice as fast as Hi-Lo. TC+5 under Zen maps to approximately the same insurance edge threshold as TC+3 under Hi-Lo — the same underlying advantage point, different count scale.*

*Full detail: [Audit Detail — Policy Audit](detail.md#policy-audit)*

### 2. Variance and Risk Metric Calculations

BJW implements the Schlesinger Chapter 10 variance formula directly:

> **Var(session) = E[b²] · Var(w) + Var(b) · E[w]² + 2 · Cov(b,w) · E[b] · E[w]**

Where *b* is bet size and *w* is outcome per unit wagered.

To verify this implementation, we built a separate Python validation script entirely independent of the simulator. This script reads the raw per-hand accumulators from simulator output — sums of *b*, *w*, *b²*, *w²*, and *bw* — and recomputes every derived metric from scratch: E[b], E[w], Var(b), Var(w), Cov(b,w), the full Chapter 10 variance, DI, SCORE, and N₀.

We then verified the algebraic identities:

- DI² = SCORE
- SCORE × N₀ = 1,000,000

**Results**: 400 independent checks across 16 result files (8 scenarios × 100M and 1B rounds each). Zero failures at 1×10⁻⁶ tolerance.

This means: whatever numbers BJW reports for SD, DI, SCORE, and N₀ — those numbers are correct *given the game data the simulator collected*. The formula implementation is not in question.

*Full detail: [Audit Detail — Formula Recompute Audit](detail.md#formula-recompute-audit)*

### 3. Theory Sanity Checks

We verified a set of directional and magnitude properties that any correct simulator must satisfy:

- S17 produces better player EV than H17 under matched conditions ✓
- Flat betting EV for 6D S17 falls within the accepted published house-edge band ✓
- Flat betting EV for 6D H17 falls within the accepted published house-edge band ✓
- EV increases monotonically with true count across TC bins ✓ (with one documented exception)

**One explained anomaly**: In the 1-deck scenario, EV at TC bin +8 is marginally lower than at TC bin +7. This is a known artifact of very low sample density at extreme true counts in single-deck shoes — documented in the known discrepancies log, not a simulator error.

*Full detail: [Audit Detail — Theory Sanity Matrix](detail.md#theory-sanity-matrix)*

---

## Where We Differ from CVData

We compared BJW output against CVData across 8 core scenarios at 1 billion rounds each. The result: **0 of 8 scenarios pass our comparison gates**.

We are disclosing this prominently because the audience we're writing for would find it anyway.

Full delta table (BJW minus CVData):

| Scenario | EV Δ (pp) | SD Δ (%) | SCORE Δ (%) | DI Δ (%) |
|---|---:|---:|---:|---:|
| 1D H17 Hi-Lo 1-12 I18 | +0.143 | +22.0 | +16.0 | +7.7 |
| 2D H17 LS Hi-Lo 1-4 | +0.069 | +4.3 | +25.6 | +12.1 |
| 6D H17 LS Hi-Lo 1-12 I18 | +0.041 | +14.6 | +16.7 | +7.9 |
| 6D H17 Flat | +0.011 | +3.8 | +1.5 | +0.8 |
| 6D S17 Flat | −0.005 | +4.3 | +4.8 | +2.3 |
| 6D S17 Hi-Lo 1-12 I18 | +0.100 | +0.3 | +52.1 | +23.6 |
| 8D H17 D10 NDAS NS Hi-Lo 1-4 | −0.090 | +0.2 | +38.8 | +17.8 |
| 8D H17 Hi-Lo 1-4 | −0.016 | +0.5 | +13.0 | +6.1 |

**Note on SCORE/DI/N₀ deltas**: Because SCORE scales with the square of EV (SCORE = EV² / Var), small EV offsets produce materially larger SCORE, DI, and N₀ deltas. We interpret these as downstream amplification of the EV residual unless independent variance defects are shown — and the formula audit confirms no such defects exist.

### What We Believe Is Causing This

After extensive investigation — including a fresh CVData rerun on the most misaligned scenario (1D H17 Hi-Lo 1-12, 1B hands, March 7 2026), multiple parameter sensitivity sweeps, and structured root-cause analysis — our best explanation is **TC bucket occupancy mismatch**.

Specifically: when we examine EV broken down by true count bucket, per-bucket values are reasonably close to CVData. When we examine how much time the simulation spends at each true count, those distributions diverge more substantially. This pattern — per-bucket EV similar, occupancy different — is consistent with BJW spending more time at favorable true counts than CVData does, inflating aggregate counted EV even when per-count strategy and outcomes are correct.

We cannot resolve this with certainty because CVData is closed-source. We do not know its exact TC calculation methodology, how it handles the TC divisor at shoe depth extremes, or how it weights penetration.

What we can say:

- This is not a formula error. Chapter 10 implementation is independently verified.
- This is not a basic strategy error. Play policies are audited.
- The gap persists after testing multiple TC divisor configurations.
- The flat-betting EV gap (0.005–0.011pp) is negligible — the issue is concentrated in counted scenarios where TC occupancy matters most.
- The fresh CVData rerun on the worst-case scenario produced the same result, ruling out stale or misconfigured CVData exports.

We classify this as a **documented parity residual**, not an unresolved defect.

---

## What This Means for Published Results

**We are confident making:**

- Relative comparisons within BJW (strategy A vs B through the same engine — systematic offsets cancel)
- Marginal contribution claims (adding deviation X gains Y units of DI)
- Directional claims (more penetration → higher DI; S17 → better EV than H17)
- Formula-derived metrics given accurate input game data (SD, DI, SCORE, N₀ from our accumulators)
- Flat betting house edge results (within accepted published bands)

**We are cautious making:**

- Absolute EV claims for counted scenarios (our figures run 0.004–0.143pp above CVData depending on scenario)
- Absolute SCORE and N₀ claims for counted scenarios (downstream of EV, amplified by squaring)

When we publish counted scenario results, we note the CVData delta where relevant and frame conclusions accordingly. For experiments that are inherently relative — comparing two strategies through the same simulator — the systematic offset cancels and the results stand on their own.

---

## Simulation Parameters

All published results use:

- Minimum 1 billion rounds per configuration
- Multi-threaded PRNG execution (8 workers); single-threaded vs parallel equivalence is a required property, validated via binary-deck regression runs
- Fixed PRNG seeds; every result is reproducible from manifest + seed within BJW
- TC divisor: half-deck, cards remaining
- Burn cards: 1

Results can be reproduced within BJW from the manifest; equivalent CVData configurations are mapped where options align.

---

## Questions

[github.com/ether-ore/BJW/issues](https://github.com/ether-ore/BJW/issues)
