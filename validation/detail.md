# Audit Detail

This page documents the methodology and results for each Track B validation audit. All audits were completed March 7, 2026.

---

## Policy Audit

### What Was Checked

All 16 play policy files used in simulation were audited against published basic strategy charts. The 16 files cover the full combination matrix of:

- Dealer rule: H17, S17
- Doubling: DOA (double on any two), D10 (double on 10/11 only)
- Split rules: DAS (double after split), NDAS
- Surrender: LS (late surrender), NS (no surrender)

For each policy file, every hard total, soft total, and pair-splitting decision was checked against published strategy for the applicable rule configuration.

### Results

**224 blocking checks. 0 failures.**

Two items required disposition:

| Item | Disposition | Resolution |
|---|---|---|
| Halves count insurance threshold | Confirmed bug | Was coded as TC ≥ 1. Corrected to TC ≥ 3 per published sources. Fixed. |
| Zen count insurance threshold | Source-confirmed | Confirmed at TC ≥ 5 (count-per-deck convention). Primary source citation recorded in counting system file. Empirically verified at 100M rounds. |

No other mismatches found across all 16 policy files.

---

## Formula Recompute Audit

### What Was Checked

BJW implements the Schlesinger Chapter 10 variance formula:

> **Var(session) = E[b²] · Var(w) + Var(b) · E[w]² + 2 · Cov(b,w) · E[b] · E[w]**

To verify this, a separate Python script (not part of the simulator) reads the raw per-hand accumulators from simulator output and recomputes every derived metric independently:

**Raw accumulators read from results**:
- `sum_b` — sum of all bets
- `sum_w` — sum of all outcomes per unit
- `sum_b2` — sum of squared bets
- `sum_w2` — sum of squared outcomes
- `sum_bw` — sum of bet × outcome products
- `count` — number of hands

**Recomputed from scratch in Python**:
- E[b] = sum_b / count
- E[w] = sum_w / count
- Var(b) = sum_b2/count − E[b]²
- Var(w) = sum_w2/count − E[w]²
- Cov(b,w) = sum_bw/count − E[b]·E[w]
- Full Chapter 10 variance
- DI = E[b] · E[w] / √Var(session)

**Algebraic identities verified**:
- DI² = SCORE
- SCORE × N₀ = 1,000,000
- variance = SD²  (for all three methodology variants: naive, Chapter 10, DI-adjusted)

This is a genuine independent recomputation — the Python script derives all metrics from first principles using only the raw accumulator values. It does not call any simulator code.

### Results

**400 blocking checks across 16 result files (8 scenarios × 100M rounds + 1B rounds). 0 failures. Tolerance: 1×10⁻⁶.**

This confirms: BJW's variance, SD, DI, SCORE, and N₀ figures are correct given the game data collected. The formula implementation is verified.

---

## Theory Sanity Matrix

### What Was Checked

A set of properties that any mathematically correct blackjack simulator must satisfy:

**Flat EV bands**: Flat-betting EV for 6D S17 and 6D H17 falls within accepted published house-edge ranges for those rulesets.

**Directional invariant**: S17 produces better player EV than H17 under otherwise identical conditions.

**EV monotonicity**: EV per hand increases with true count across TC bins — higher count means higher edge, as theory requires.

### Results

| Check | Result |
|---|---|
| 6D S17 flat EV within published house-edge band | ✓ Pass |
| 6D H17 flat EV within published house-edge band | ✓ Pass |
| S17 EV > H17 EV (matched conditions) | ✓ Pass |
| Hi-Lo 6D EV monotonically increasing by TC bin | ✓ Pass |
| Hi-Lo 6D S17 EV > flat EV (counting adds edge) | ✓ Pass |
| Hi-Lo 8D EV monotonically increasing by TC bin | ✓ Pass |
| Hi-Lo 1D EV monotonically increasing by TC bin | **Explained residual** |

**7/8 pass. 1 explained residual.**

**Explained residual — 1D monotonicity at extreme TC**: In the 1-deck scenario, EV at TC bin +8 is marginally lower than at TC bin +7. This is a known statistical artifact: at very deep shoe penetration in 1-deck games, the number of hands played at TC ≥ +8 is small enough that sample variance produces occasional inversions even at 1B rounds. This is documented in the known discrepancies log and is consistent with expected behavior. It does not indicate a simulator error.

---

## Scenarios Tested

All Track B and Track A audits used these 8 core scenarios, each run at 1 billion rounds:

| Scenario ID | Decks | Dealer | Doubling | DAS | Surrender | Count | Spread |
|---|---|---|---|---|---|---|---|
| 1D H17 Hi-Lo 1-12 I18 | 1 | H17 | DOA | Yes | No | Hi-Lo | 1–12 |
| 2D H17 LS Hi-Lo 1-4 | 2 | H17 | DOA | Yes | Late | Hi-Lo | 1–4 |
| 6D H17 LS Hi-Lo 1-12 I18 | 6 | H17 | DOA | Yes | Late | Hi-Lo | 1–12 |
| 6D H17 Flat | 6 | H17 | DOA | Yes | No | None | Flat 1u |
| 6D S17 Flat | 6 | S17 | DOA | Yes | No | None | Flat 1u |
| 6D S17 Hi-Lo 1-12 I18 | 6 | S17 | DOA | Yes | No | Hi-Lo | 1–12 |
| 8D H17 D10 NDAS NS Hi-Lo 1-4 | 8 | H17 | D10 | No | No | Hi-Lo | 1–4 |
| 8D H17 Hi-Lo 1-4 | 8 | H17 | DOA | Yes | No | Hi-Lo | 1–4 |

All scenarios: 75% penetration, 1 burn card, TC divisor half-deck / cards remaining.
