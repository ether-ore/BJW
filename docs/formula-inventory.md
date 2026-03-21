# BJW Blackjack/AP Formula Inventory

**Date**: 2026-03-16  
**Public site note**: This page mirrors the current BJW formula inventory maintained in the main project docs.

## Purpose

This document inventories the blackjack and advantage-play formulas implemented in BJW code.

For each formula family, it states:

- where the formula lives
- what the formula is trying to do
- where the inputs come from in the BJW pipeline
- what the variables mean
- what output the formula is expected to produce
- what terms to search for in books or online if you want to verify the underlying idea yourself

## Scope

Included:

- core simulator runtime math
- parallel aggregation math
- betting VM math
- result helper math
- validation/comparison math
- experiment/optimizer/reporting math that derives blackjack/AP metrics from results
- QA/statistical formulas that materially affect blackjack result interpretation

Excluded:

- static basic-strategy/deviation tables
- pure formatting with no arithmetic
- tests that only restate formulas already implemented elsewhere

## Important High-Level Findings

1. BJW’s core simulator does **not** compute blackjack EV from closed-form house-edge formulas. It simulates rounds and measures EV/variance from outcomes.
2. BJW’s current canonical counted-risk path is **observed per-round variance**, not legacy `chapter10` math.
3. Legacy `chapter10` / `di_adjusted` variance math still exists in audit/helpers/older scripts, but it is not the canonical runtime contract on this branch.
4. The repo currently contains more than one definition of some named metrics:
   - `DI` is sometimes `1000 * EV / SD` and sometimes plain `EV / SD`.
   - `N0` is sometimes `(SD / EV)^2` and sometimes `(1.96 * SD / EV)^2`.
   - Older counted experiment scripts still read `chapter10` fields.

Those semantic drifts are called out explicitly below.

## Formula Status Legend

- **Canonical runtime**: defines BJW’s primary simulator result contract
- **Diagnostic/legacy**: retained for research, audits, or backward compatibility
- **Reporting/helper**: derived outside the simulator for display or scripting
- **Statistical comparison**: used to compare runs or assess confidence

## How To Use This Document

If you are trying to verify a formula yourself:

1. Start from the operation index below.
2. Jump to the canonical runtime section first if one exists.
3. If the same name appears elsewhere, check the drift index before assuming the formulas are identical.
4. Use the `Search terms` block in that section to look for the external concept in books, BJ Attack, CVData discussions, or online references.

This document intentionally distinguishes:

- the formula BJW currently uses in active runtime
- helper/reporting formulas that only post-process results
- legacy or diagnostic formulas kept for audits

When a section gives a provenance note, the intended meanings are:

- `Standard external concept`: common blackjack/statistics/math idea, but this document does not yet record an exact page citation
- `BJW contract choice`: a runtime or reporting convention BJW chose and enforces
- `BJW-authored model/heuristic`: an explicit BJW approximation, policy model, or threshold that should not be mistaken for sourced literature
- `Legacy unsourced`: retained for compatibility or audit only and not currently backed by a verified external citation

When a section gives a citation-status note, the intended meanings are:

- `Exact reference recorded`: this document includes a specific external source in [15. Exact References Recorded](#15-exact-references-recorded)
- `No verified citation recorded`: this pass did not verify an exact external source for that formula or exact implementation

## Formula Map By Operation

- Counting and count conversion: [1.1 Running Count Tag Update](#11-running-count-tag-update), [1.3 Decks Remaining](#13-decks-remaining), [1.4 Deck Quantization](#14-deck-quantization), [1.5 True Count](#15-true-count), [1.6 TC Bucket Rounding and Clamping](#16-tc-bucket-rounding-and-clamping)
- Wonging and table-entry logic: [1.8 Wonging Entry/Exit Thresholds](#18-wonging-entryexit-thresholds), [1.9 Wonging Round Accounting](#19-wonging-round-accounting), [1.10 Penetration to Cut-Card Conversion](#110-penetration-to-cut-card-conversion)
- Kelly betting: [2.2 Kelly Edge Model](#22-kelly-edge-model), [2.3 Kelly Variance Model](#23-kelly-variance-model), [2.4 Kelly Fraction and Bet Size](#24-kelly-fraction-and-bet-size), [2.5 Kelly Edge-Table Discretization](#25-kelly-edge-table-discretization), [10.3 Kelly Cap Warning Threshold](#103-kelly-cap-warning-threshold)
- Progressions and non-count betting systems: [2.1 Count Spread Lookup](#21-count-spread-lookup), [2.6 Progression Bet Formulas](#26-progression-bet-formulas), [2.7 State-Machine Special Calculation](#27-state-machine-special-calculation-total_loss--040)
- Blackjack hand and round settlement: [3.1 Hand Total with Ace Adjustment](#31-hand-total-with-ace-adjustment), [3.2 Dealer Play Rule](#32-dealer-play-rule), [3.7 Insurance Settlement](#37-insurance-settlement), [3.8 Insurance Effective Probability and EV](#38-insurance-effective-probability-and-ev), [3.10 Natural, Surrender, Win/Loss/Push Settlement](#310-natural-surrender-winlosspush-settlement)
- Canonical performance and risk metrics: [4.1 Observed Sample Variance via Welford](#41-observed-sample-variance-via-welford), [4.2 EV per Hand, per Round, per Initial Bet, and Average Bet](#42-ev-per-hand-per-round-per-initial-bet-and-average-bet), [4.3 Primary Risk Basis Selection](#43-primary-risk-basis-selection), [4.4 SCORE](#44-score), [4.5 Desirability Index (DI)](#45-desirability-index-di), [4.6 Canonical Runtime N0](#46-canonical-runtime-n0), [4.7 Risk of Ruin (RoR) Approximation](#47-risk-of-ruin-ror-approximation)
- Parallel aggregation and contract sync: [5.1 Worker-Result Pooled Variance](#51-worker-result-pooled-variance), [4.3 Primary Risk Basis Selection](#43-primary-risk-basis-selection)
- Per-bucket and decision telemetry: [6.1 EV-by-Bucket Variance](#61-ev-by-bucket-variance), [6.2 Exact Decision Variable SD](#62-exact-decision-variable-sd)
- Legacy variance paths and audits: [7.1 Legacy `chapter10` Variance Decomposition](#71-legacy-chapter10-variance-decomposition), [7.2 Legacy `di_adjusted` Reconstruction](#72-legacy-di_adjusted-reconstruction), [7.3 RMS SD Validation Heuristic](#73-rms-sd-validation-heuristic)
- Helpers, optimizers, and reweighting: [8.1 Sharpe-Like Ratio](#81-sharpe-like-ratio), [8.2 Reweighting Formulas for Spread Optimization](#82-reweighting-formulas-for-spread-optimization), [8.3 Helper/API DI Formula](#83-helperapi-di-formula), [8.4 Helper/API SCORE Formula](#84-helperapi-score-formula), [8.5 Helper/API N0(Confidence) Formula](#85-helperapi-n0confidence-formula)
- Statistical comparison and QA gates: [9.1 Standard Error / Confidence Interval](#91-standard-error--confidence-interval-for-ev), [9.4 Z-Score for Run-to-Run Comparison](#94-z-score-for-run-to-run-comparison), [10.1 Initial Pair Baseline Tolerance](#101-initial-pair-baseline-tolerance)

## Same Name, Different Formula Index

- `DI`: canonical runtime [4.5 Desirability Index (DI)](#45-desirability-index-di), helper copy [8.3 Helper/API DI Formula](#83-helperapi-di-formula), drift summary [11.1 DI Scaling Drift](#111-di-scaling-drift)
- `N0`: canonical runtime [4.6 Canonical Runtime N0](#46-canonical-runtime-n0), confidence-style helper [8.5 Helper/API N0(Confidence) Formula](#85-helperapi-n0confidence-formula), drift summary [11.2 N0 Definition Drift](#112-n0-definition-drift)
- Counted risk basis: canonical contract [4.3 Primary Risk Basis Selection](#43-primary-risk-basis-selection), old counted scripts [11.3 Counted-Risk Basis Drift in Older Scripts](#113-counted-risk-basis-drift-in-older-scripts), top-level field mismatch [11.4 Top-Level Per-Hand Fields vs `risk_primary`](#114-top-level-per-hand-fields-vs-risk_primary)
- Legacy variance blocks: diagnostic only [7.1 Legacy `chapter10` Variance Decomposition](#71-legacy-chapter10-variance-decomposition), [7.2 Legacy `di_adjusted` Reconstruction](#72-legacy-di_adjusted-reconstruction)
- Kelly inputs vs Kelly output: edge model [2.2 Kelly Edge Model](#22-kelly-edge-model), variance model [2.3 Kelly Variance Model](#23-kelly-variance-model), final bet formula [2.4 Kelly Fraction and Bet Size](#24-kelly-fraction-and-bet-size)

## 1. Counting and Decision-Variable Formulas

**Section citation status**

No verified citation recorded section-wide. These are BJW implementations of standard counting conventions and manifest semantics, but this pass did not attach exact external references for each convention.

### 1.1 Running Count Tag Update

**Status**: Canonical runtime

**Formula**

```text
RC_new = RC_old + tag(exposed_card)
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts` `countTag(...)`, `revealCard(...)`
- `src/typescript/simulator_lite/src/utils.ts` `countTag(...)`, `toCountTag(...)`

**What it is for**

Updates the running count whenever a card becomes visible to the player.

**Variables**

- `RC`: current running count
- `tag(card)`: count-system tag value for the exposed card

**Where the variables come from**

- `RC` is simulator state carried through the shoe
- tag values come from `manifest.counting.card_values`
- Red Seven / suit-aware tags are resolved from the actual card rank+suit

**Expected output**

An integer or fractional running count, depending on the count system definition.

**Search terms**

- `running count update`
- `count tag values`
- `Hi-Lo tags`
- `KO tags`
- `Red Seven tag values`

### 1.2 Theoretical Running Count Bounds

**Status**: Reporting/helper

**Formula**

```text
RC_max = Σ max(0, tag_i)
RC_min = Σ min(0, tag_i)
RC_end = RC_max + RC_min
```

**Primary locations**

- `src/python/configurator/transpilers/betting_to_bvm.py` `_calculate_rc_bounds(...)`

**What it is for**

Computes the theoretical min/max/end-of-shoe running count for a count system and deck count.

**Variables**

- `tag_i`: tag value of each physical card in the shoe
- `RC_end`: total shoe unbalance

**Where the variables come from**

- `tag_i` comes from the count-system tag map under `data/rules/counting/...`
- the “physical cards in the shoe” come from the configured deck count and shoe composition used by the transpiler when it enumerates all cards
- `RC_end` is the sum of all shoe tags, so for balanced systems it should end near `0`, while for unbalanced systems it represents the system’s full-shoe unbalance

**Interpretation**

This section is not reading live simulator state. It is doing a static whole-shoe audit of the counting system definition.

**Expected output**

The minimum possible RC, maximum possible RC, and final end-of-shoe RC if the whole shoe were revealed.

**Search terms**

- `running count bounds`
- `end of shoe running count`
- `unbalanced count shoe end`
- `key count`

### 1.3 Decks Remaining

**Status**: Canonical runtime

**Formula**

```text
decks_remaining = cards_remaining / 52
```

or, when divisor basis is configured as cards dealt:

```text
decks_remaining = decks_total - quantize(cards_dealt / 52)
```

with a lower clamp:

```text
decks_remaining = max(0.0001, decks_remaining)
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts` `decksRemaining(...)`
- `src/typescript/simulator_lite/src/utils.ts` `decksRemaining(...)`

**What it is for**

Provides the divisor for true-count conversion and several diagnostic outputs.

**Variables**

- `cards_remaining`
- `cards_dealt`
- `decks_total`

**Where the variables come from**

- `cards_remaining = shoe.length - shoePtr`
- `cards_dealt = shoePtr`
- `decks_total = manifest.environment.ruleset.decks`

**Interpretation**

This is the raw shoe-depth input before the true-count formula in [1.5 True Count](#15-true-count).

**Expected output**

A positive estimate of decks remaining, optionally quantized.

**Search terms**

- `decks remaining`
- `true count divisor`
- `half deck estimation`
- `quarter deck estimation`

### 1.4 Deck Quantization

**Status**: Canonical runtime

**Formula**

```text
full_deck:    round(x)
half_deck:    round(2x) / 2
quarter_deck: round(4x) / 4
continuous:   x
```

**Primary locations**

- `src/typescript/simulator_lite/src/utils.ts` `quantizeDecksRemaining(...)`
- used by `src/typescript/simulator_lite/src/simulator.ts`

**What it is for**

Replicates common card-counter deck-estimation granularity before true-count division.

**Variables**

- `x`: continuous decks remaining or decks dealt

**Where the variables come from**

- `x` is produced immediately before quantization from either:
  - `cards_remaining / 52`, or
  - `cards_dealt / 52`
- which one BJW uses depends on the divisor-basis path described in [1.3 Decks Remaining](#13-decks-remaining)

**Interpretation**

This section is only about how BJW rounds deck estimates. It does not itself choose between cards-remaining and cards-dealt semantics.

**Expected output**

A quantized deck estimate consistent with the configured divisor mode.

**Search terms**

- `half deck true count`
- `quarter deck true count`
- `deck estimation granularity`

### 1.5 True Count

**Status**: Canonical runtime

**Formula**

```text
TC = RC / decks_remaining
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts` `trueCount(...)`
- `src/typescript/simulator_lite/src/utils.ts` `trueCount(...)`

**What it is for**

Converts running count to a depth-normalized count for balanced systems and TC-based policies.

**Variables**

- `RC`: running count
- `decks_remaining`: result of the current divisor policy

**Where the variables come from**

- `RC` comes from the running count state updated by [1.1 Running Count Tag Update](#11-running-count-tag-update)
- `decks_remaining` comes from [1.3 Decks Remaining](#13-decks-remaining), including any configured quantization from [1.4 Deck Quantization](#14-deck-quantization)

**Interpretation**

This is the continuous true count before bucket rounding and clamping in [1.6 TC Bucket Rounding and Clamping](#16-tc-bucket-rounding-and-clamping).

**Expected output**

A continuous true count before integer bucketing/clamping.

**Search terms**

- `true count formula`
- `running count divided by decks remaining`

### 1.6 TC Bucket Rounding and Clamping

**Status**: Canonical runtime

**Formula**

```text
nearest: round(TC)
away_from_zero:
  if TC >= 0: floor(TC + 0.5)
  else:       ceil(TC - 0.5)

bucket = clamp(raw_bucket, bucket_min, bucket_max)
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts`
- `src/typescript/simulator_lite/src/utils.ts` `calculateBucketWithMode(...)`

**What it is for**

Maps continuous TC into integer bins for bet spreads, EV-by-bucket, and deviation thresholds.

**Variables**

- `TC`
- `bucket_min`, `bucket_max`
- `rounding_mode`

**Where the variables come from**

- `TC` is the continuous value from [1.5 True Count](#15-true-count) or an insurance-specific TC path when insurance bucketing is active
- `bucket_min` and `bucket_max` come from `manifest.bucketing.range.min/max` (mirrored from `counting.bucketing.range`)
- `rounding_mode` comes from `counting.tc_bucket_rounding_mode`

**Interpretation**

This is the stage where BJW turns a continuous count into the integer decision variable actually used by spreads, deviations, and bucket telemetry.

**Expected output**

A final integer bucket used by policies and telemetry.

**Search terms**

- `true count rounding`
- `round away from zero`
- `count bucket clamp`

### 1.7 Depth-Adjusted Running Count Bucket

**Status**: Canonical runtime for systems that use depth adjustment

**Formula**

```text
adjustment = base + multiplier * decks_played
adjusted_rc_bucket = round(RC - adjustment)
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts` `calculateBucket(...)`

**What it is for**

Allows RC-based systems with a depth adjustment to bucket the same way their betting policy decides.

**Variables**

- `RC`
- `base`
- `multiplier`
- `decks_played = shoePtr / 52`

**Where the variables come from**

- `RC` is the live running count state
- `base` and `multiplier` come from `bucket_spread.depth_adjustment` in the BVM produced by the betting-policy transpiler
- `decks_played = shoePtr / 52`

**Interpretation**

This is a betting-policy alignment step. BJW uses it so EV bucketing and bet-spread matching are based on the same adjusted RC decision variable.

**Expected output**

An integer depth-adjusted RC bucket.

**Search terms**

- `running count depth adjustment`
- `warm count`
- `effective running count`

### 1.8 Wonging Entry/Exit Thresholds

**Status**: Canonical runtime

**Formula**

```text
enter if TC >= tc_ge or RC >= rc_ge
exit  if TC <= tc_le or RC <= rc_le
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts` `shouldEnterTable(...)`, `shouldExitTable(...)`

**What it is for**

Controls back-counting / wonging in and out of the game.

**Search terms**

- `wong in threshold`
- `wong out threshold`
- `back-counting entry count`

### 1.9 Wonging Round Accounting

**Status**: Canonical runtime

**Formula**

```text
total_rounds_observed += 1 each time entry/exit is rechecked

if wongingActive:
  rounds_played += 1
else:
  rounds_sat_out += 1

pct_time_at_table = 100 * rounds_played / total_rounds_observed
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts` wonging state update inside `playOneRound(...)`
- `src/typescript/simulator_lite/src/index.ts` derived recomputation of `pct_time_at_table`

**What it is for**

Separates rounds the player actually plays from rounds merely observed while back-counting.

**Important note**

- If `when_out = sit_out` or `when_out = reshuffle`, the observed round can increment wonging counters without incrementing `round_summary.rounds`.
- If `when_out = flat_min`, the round is still played and contributes to normal round/hand EV and variance.

**Search terms**

- `wonging rounds played vs observed`
- `back-counting sit-out accounting`
- `time at table percentage`

### 1.10 Penetration to Cut-Card Conversion

**Status**: Canonical runtime

**Formula**

```text
penetration_fraction = penetration_pct / 100
cut_card = max(1, floor(shoe_size_cards * penetration_fraction))

start new shoe before next round if shoePtr >= cut_card
```

with the consistency constraint:

```text
burn_cards < cut_card
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts` penetration parsing and `cutCard` setup
- `src/typescript/simulator_lite/src/simulator.ts` pre-round reshuffle check in `playOneRound(...)`

**What it is for**

Turns manifest penetration into the actual deepest card position BJW will deal before reshuffling.

**Important note**

Penetration does not enter EV through a closed-form blackjack formula. It changes which late-shoe count states are reachable, which then changes bucket occupancy, betting opportunities, and observed EV/variance.

**Search terms**

- `blackjack penetration cut card formula`
- `cut card position shoe game`
- `penetration effect on true count distribution`

## 2. Betting Formulas

**Section citation status**

Mixed. Some formulas below trace to general Kelly literature; others are BJW policy semantics or BJW-authored models with no verified external citation recorded for the exact implementation.

### 2.1 Count Spread Lookup

**Status**: Canonical runtime

**Formula**

No algebraic formula. BJW range-matches the current bucket against authored spread ranges:

```text
if lo <= bucket <= hi: use units_for_that_range
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts` `unitsFromCountBasedBetPolicy(...)`

**What it is for**

Implements ordinary RC- or TC-tiered bet ramps.

**Provenance status**

BJW contract choice. This is the simulator’s implementation of an authored spread schedule, not a standalone blackjack theorem.

**Citation status**

No verified citation recorded. This is a BJW range-matching execution rule for authored spread schedules.

**Search terms**

- `bet spread by true count`
- `tiered spread`
- `count-based bet ramp`

### 2.2 Kelly Edge Model

**Status**: Canonical runtime when a Kelly policy is used

**Formula**

Table-driven option:

```text
edge = edge_table[count_index]
```

Fallback slope/cap option:

```text
edge = min(max(0, count_value * edge_slope), edge_cap)
```

**Primary locations**

- `src/typescript/simulator_lite/src/betting_vm.ts` `calculateKellyUnits(...)`
- Kelly params produced by `src/python/configurator/transpilers/betting_to_bvm.py`

**What it is for**

Produces a modeled per-hand edge for Kelly bet sizing.

**Provenance status**

BJW-authored model/heuristic. Kelly bet sizing itself is an external concept, but this particular edge parameterization and its defaults are authored policy/model inputs inside BJW rather than a verified closed-form blackjack edge law.

**Citation status**

No verified citation recorded for this exact BJW edge model.

**Variables**

- `count_value`: TC for balanced systems, RC/effective count for unbalanced systems
- `edge_slope` / `edge_cap`
- `edge_table`

**Where the variables come from**

- `count_value` comes from `bucket_spread.input_var` in the active BVM context:
  - balanced systems usually feed TC
  - unbalanced systems usually feed RC or an effective RC-style value
- `edge_table` comes from `manifest.betting.policy_vm.bucket_spread.kelly_params.edge_table`
  - it is a lookup table keyed by integerized count values after the discretization rule in [2.5 Kelly Edge-Table Discretization](#25-kelly-edge-table-discretization)
- `edge_slope` comes from `kelly_params.edge_per_tc` if present, otherwise `kelly_params.edge_slope`, otherwise BJW falls back to `0.005`
- `edge_cap` comes from `kelly_params.edge_cap`, otherwise BJW falls back to `0.025`

**Interpretation**

- `edge_table` means “for this count bucket, assume this per-unit edge”
- `edge_slope` means “increase modeled edge linearly as count rises”
- `edge_cap` means “stop increasing the modeled edge beyond this ceiling”

These are authored model parameters, not values inferred from the current simulation at runtime.

**Expected output**

A modeled edge in units-per-unit-bet, used only for Kelly sizing.

**Related formulas**

- Kelly variance term: [2.3 Kelly Variance Model](#23-kelly-variance-model)
- Kelly bet conversion: [2.4 Kelly Fraction and Bet Size](#24-kelly-fraction-and-bet-size)
- Edge-table lookup rule: [2.5 Kelly Edge-Table Discretization](#25-kelly-edge-table-discretization)
- Kelly saturation warning: [10.3 Kelly Cap Warning Threshold](#103-kelly-cap-warning-threshold)

**Search terms**

- `Kelly blackjack edge model`
- `edge per true count`
- `edge cap`

### 2.3 Kelly Variance Model

**Status**: Canonical runtime when a Kelly policy is used

**Formula**

```text
variance = clamp(
  variance_base + max(0, count_value) * variance_tc_multiplier,
  1.0,
  3.0
)
```

**Primary locations**

- `src/typescript/simulator_lite/src/betting_vm.ts` `calculateKellyUnits(...)`

**What it is for**

Provides the variance term in the Kelly fraction model.

**Provenance status**

BJW-authored model/heuristic. The Kelly denominator concept is external, but BJW’s count-dependent variance parameterization and clamps are implementation/model choices.

**Citation status**

No verified citation recorded for this exact BJW variance model.

**Variables**

- `variance_base`
- `variance_tc_multiplier`
- `count_value`

**Where the variables come from**

- `variance_base` comes from `kelly_params.variance_base`, otherwise `kelly_params.base_variance`, otherwise BJW falls back to `1.3`
- `variance_tc_multiplier` comes from `kelly_params.variance_tc_multiplier`, otherwise BJW falls back to `0.02`
- `count_value` is the same betting-context input described in [2.2 Kelly Edge Model](#22-kelly-edge-model)

**Interpretation**

- `variance_base` is the modeled baseline variance at neutral or low counts
- `variance_tc_multiplier` says how much modeled variance increases as count rises
- the final `clamp(..., 1.0, 3.0)` is a BJW guardrail to keep the Kelly denominator in a bounded range

**Expected output**

A positive modeled variance estimate for the current count.

**Related formulas**

- Kelly edge model: [2.2 Kelly Edge Model](#22-kelly-edge-model)
- Kelly bet conversion: [2.4 Kelly Fraction and Bet Size](#24-kelly-fraction-and-bet-size)

**Search terms**

- `Kelly variance blackjack`
- `variance per true count`
- `fractional Kelly variance estimate`

### 2.4 Kelly Fraction and Bet Size

**Status**: Canonical runtime when a Kelly policy is used

**Formula**

```text
kelly_fraction = edge / variance
adjusted_kelly = kelly_fraction * fraction
raw_units = bankroll_scale * adjusted_kelly
units = clamp(round(raw_units), table_min, table_max)
```

**Primary locations**

- `src/typescript/simulator_lite/src/betting_vm.ts` `calculateKellyUnits(...)`

**What it is for**

Turns the modeled edge and variance into an actual bet size.

**Provenance status**

Mixed. The `f = edge / variance` Kelly structure is a standard external concept; the specific edge/variance inputs feeding it in BJW come from the BJW-authored models in [2.2 Kelly Edge Model](#22-kelly-edge-model) and [2.3 Kelly Variance Model](#23-kelly-variance-model).

**Citation status**

Exact reference recorded for the general Kelly concept: [R1](#r1-kelly-1956). No verified citation recorded for BJW’s exact blackjack notation and parameterization in this section.

**Variables**

- `edge`
- `variance`
- `fraction`: fractional Kelly multiplier
- `bankroll_scale`: Kelly bankroll sizing scale

**Where the variables come from**

- `edge` is the output of [2.2 Kelly Edge Model](#22-kelly-edge-model)
- `variance` is the output of [2.3 Kelly Variance Model](#23-kelly-variance-model)
- `fraction` comes from `kelly_params.fraction` or `kelly_params.fractional_kelly`
- `bankroll_scale` comes from `kelly_params.bankroll_scale`, with BJW defaulting if it is omitted

**Interpretation**

`bankroll_scale` here is a Kelly model parameter carried in the policy VM, not the live bankroll balance being tracked by bankroll-stop or bankroll-goal features.

**Related formulas**

- Kelly edge model: [2.2 Kelly Edge Model](#22-kelly-edge-model)
- Kelly variance model: [2.3 Kelly Variance Model](#23-kelly-variance-model)
- Kelly edge-table discretization: [2.5 Kelly Edge-Table Discretization](#25-kelly-edge-table-discretization)

**Expected output**

An integer bet size in BJW units.

**Search terms**

- `Kelly criterion`
- `fractional Kelly`
- `f = edge / variance`

### 2.5 Kelly Edge-Table Discretization

**Status**: Canonical runtime when a Kelly edge table is used

**Formula**

```text
count_index =
  floor(count) if count >= 0
  ceil(count)  if count < 0
```

then clamp to the nearest lower available table key.

**Primary locations**

- `src/typescript/simulator_lite/src/betting_vm.ts` `getEdgeFromTable(...)`

**What it is for**

Maps a continuous count to a discrete lookup key in an edge table.

**Citation status**

No verified citation recorded. This is a BJW lookup/discretization rule for an authored Kelly edge table.

**Search terms**

- `discretized true count lookup`
- `floor positive ceil negative count`

### 2.6 Progression Bet Formulas

**Status**: Canonical runtime for progression policies

**Formula**

Martingale fast path:

```text
units = base_units * 2^current_step
```

General BVM progression path:

```text
units = next_units
```

where `next_units` is produced by progression-specific pre-round actions such as:

```text
sequence progression: next_units = sequence[current_step]
Labouchere:           next_units = first(sequence) + last(sequence)
percentage progression: next_units = pct(prev_bet)
```

then clamp to table max/min.

**Primary locations**

- `src/typescript/simulator_lite/src/betting_vm.ts` `getProgressionUnits(...)`
- progression transpilation in `src/python/configurator/transpilers/betting_to_bvm.py`

**What it is for**

Implements Martingale, sequence progressions like `1-3-2-6`, Labouchere-style systems, and percentage progressions through the same VM.

**Provenance status**

BJW contract choice. These are implementations of authored betting-policy semantics carried in the BVM, not a single externally sourced blackjack formula.

**Citation status**

No verified citation recorded. This section documents BJW betting-policy execution semantics rather than a single literature formula.

**Important note**

The `2^current_step` expression is only the Martingale-specific fast path. It is not the generic formula for every BJW progression policy.

**Search terms**

- `Martingale`
- `doubling progression`
- `1-3-2-6 betting progression`
- `Labouchere`

### 2.7 State-Machine Special Calculation (`total_loss * 0.40`)

**Status**: Canonical runtime for authored state-machine policies, but repo-authored rather than literature-backed

**Formula**

```text
units = max(1, floor(total_loss * 0.40))
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts` `unitsFromAuthoredBetPolicy(...)`

**What it is for**

Implements a custom “bet 40% of accumulated loss” rule for certain authored policies.

**Variables**

- `total_loss`: state-machine-tracked cumulative loss

**Where the variables come from**

- `total_loss` is the authored state-machine loss accumulator maintained in BJW bet-policy state
- in the BVM-based path the parallel state variable is `total_loss_units`
- this rule is only activated when an authored state-machine policy names the matching special calculation

**Interpretation**

This is repo-authored betting logic, not a generic blackjack theorem or a simulator-wide bankroll formula.

**Citation status**

No verified citation recorded. This is a repo-authored state-machine rule rather than a verified external formula.

**Expected output**

An integer bet size that scales with accumulated loss.

**Search terms**

- `loss-recovery betting`
- `40 percent of total loss betting`

## 3. Blackjack Engine and Settlement Formulas

**Section provenance note**

Most formulas in this section are standard blackjack rule/accounting mechanics or straightforward arithmetic identities over settled outcomes. This document does not yet carry exact book/page citations for each such rule.

**Section citation status**

No verified citation recorded section-wide. Most formulas here are direct consequences of blackjack rules or direct arithmetic over settled outcomes.

### 3.1 Hand Total with Ace Adjustment

**Status**: Canonical runtime

**Formula**

```text
sum = Σ card_values with Ace initially counted as 11
while sum > 21 and aces > 0:
  sum -= 10
  aces -= 1
soft = (remaining_aces_counted_as_11 > 0)
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts` `handTotal(...)`

**What it is for**

Computes hard/soft hand totals for play, dealer rules, and settlement.

**Search terms**

- `blackjack hand total aces`
- `soft hand calculation`

### 3.2 Dealer Play Rule

**Status**: Canonical runtime

**Formula**

```text
if total > 17: stand
if total < 17: hit
if total == 17:
  hit on soft 17 only when H17 is enabled
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts` `dealerPlays(...)`

**What it is for**

Executes dealer drawing rules under H17/S17.

**Search terms**

- `dealer hits soft 17`
- `S17 H17 rule`

### 3.3 Blackjack Detection

**Status**: Canonical runtime

**Formula**

```text
is_blackjack = (hand_length == 2) and (hand_total == 21)
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts` `isBlackjack(...)`

**What it is for**

Identifies naturals for peek/natural settlement.

### 3.4 Double Eligibility

**Status**: Canonical runtime

**Formula**

```text
must have exactly 2 cards
if after split:
  DAS must be allowed
  if split aces: double_after_split_aces must be allowed
if DOA: allowed
else: only total 10 or 11
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts` `canDouble(...)`

**What it is for**

Applies table rules to doubling decisions.

**Search terms**

- `double on any two`
- `double 10 or 11 only`
- `DAS`

### 3.5 Surrender Eligibility

**Status**: Canonical runtime

**Formula**

```text
surrender_allowed and (not after_split or surrender_after_split_allowed)
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts` `canSurrender(...)`

**What it is for**

Applies late-surrender rules.

### 3.6 Insurance Decision Threshold

**Status**: Canonical runtime

**Formula**

Policy-threshold form:

```text
take insurance if bucket >= threshold
```

with RC- or TC-based authored strings like `dealer_upcard:A:TC>=3`.

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts` `shouldTakeInsurance(...)`

**What it is for**

Applies authored insurance thresholds.

**Search terms**

- `insurance index`
- `insurance true count threshold`
- `insurance running count threshold`

### 3.7 Insurance Settlement

**Status**: Canonical runtime

**Formula**

```text
insurance_bet = main_bet / 2
if dealer_has_blackjack:
  payout = 2 * insurance_bet
  net = +2 * insurance_bet   (profit-only tracking)
else:
  payout = 0
  net = -insurance_bet
```

summary-level corrected net:

```text
insurance_net_units = 1.5 * payout_units - bet_units
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts` `settleInsuranceBet(...)`
- `src/typescript/simulator_lite/src/simulator.ts` summary finalization

**What it is for**

Tracks insurance separately from the main hand and then centralizes net reporting.

**Variables**

- `payout_units`: profit-only 2:1 payout accumulation
- `bet_units`: total insurance stakes

**Where the variables come from**

- `payout_units` comes from `counters.insurance_payout_units`, which is incremented in `settleInsuranceBet(...)` only on winning insurance events
- `bet_units` comes from `counters.insurance_bet_units`, which is incremented on every insurance bet regardless of outcome
- both counters are then combined during summary finalization to produce `insurance_net_units`

**Why the summary uses `1.5 * payout_units - bet_units`**

At runtime, winning insurance payouts are tracked as profit-only `2 * bet`, not returned stake plus profit.

If `win_bet_units` is the total insurance stake on winning events:

```text
payout_units = 2 * win_bet_units
win_bet_units = payout_units / 2

true_net
  = (+2 * win_bet_units) - (loss_bet_units)
  = (+2 * win_bet_units) - (bet_units - win_bet_units)
  = 3 * win_bet_units - bet_units
  = 1.5 * payout_units - bet_units
```

So the `1.5` coefficient is not a blackjack payoff rule by itself; it is the algebra needed to reconstruct total insurance net from profit-only payout tracking plus total stake.

**Expected output**

Insurance wins/losses, net units, and derived insurance QA metrics.

**Search terms**

- `insurance 2:1 payout`
- `insurance side bet blackjack`

### 3.8 Insurance Effective Probability and EV

**Status**: Canonical runtime for insurance diagnostics

**Formula**

```text
effective_p = (payout_units / 2) / bet_units
ev_per_unit = 3 * effective_p - 1
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts` insurance bucket outputs
- `src/typescript/simulator_lite/src/simulator.ts` `debug.insurance_qa`

**What it is for**

Reconstructs the implied win probability and EV of insurance from observed bucket results.

**Variables**

- `effective_p`: estimated probability dealer hole card is ten, inferred from payouts
- `ev_per_unit`: expected profit per unit of insurance stake

**Where the variables come from**

- `effective_p` is reconstructed from observed insurance bucket telemetry using the bucket’s `payout_units` and `bet_units`
- those bucket-level inputs come from the same insurance settlement counters described in [3.7 Insurance Settlement](#37-insurance-settlement)
- `ev_per_unit` is then derived from that reconstructed `effective_p`

**Why `3p - 1`**

Insurance pays +2 on a win and -1 on a loss:

```text
EV = 2p + (-1)(1 - p) = 3p - 1
```

**Search terms**

- `insurance EV 3p - 1`
- `insurance break-even 1/3`

### 3.9 Insurance Break-Even Calibration

**Status**: Canonical runtime for diagnostics

**Formula**

Simple calibration:

```text
threshold = 1/3 + margin
```
with current simple-calibration default:

- `margin = 0.00`

Monotone-safe calibration:

```text
threshold = 1/3 + margin
peff[i] >= peff[i-1] - epsilon
eligible if insured_events >= min_events or bet_units >= min_bet_units
```

with current monotone-safe defaults:

- `margin = 0.01`
- `epsilon = 0.005`
- `min_events = 500`
- `min_bet_units = 1000`

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts` `computeInsuranceCalibration(...)`
- `src/typescript/simulator_lite/src/simulator.ts` `computeMonotoneSafeCalibration(...)`
- `src/typescript/simulator_lite/src/simulator.ts` `checkMonotonicity(...)`

**What it is for**

Builds recommended insurance thresholds from observed bucket profitability.

**Provenance status**

BJW-authored model/heuristic. The `1/3` break-even point is a standard insurance concept, but the margin, monotonicity rule, event thresholds, and monotone-safe construction are BJW diagnostic heuristics.

**Citation status**

No verified citation recorded for BJW’s exact calibration heuristic.

**Search terms**

- `insurance break-even probability one third`
- `insurance threshold calibration`
- `monotonic insurance index`

### 3.10 Natural, Surrender, Win/Loss/Push Settlement

**Status**: Canonical runtime

**Formula**

```text
player natural result = blackjack_payout * bet
surrender result = -0.5 * base_bet
regular win = +wager
regular loss = -wager
push = 0
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts` natural handling and hand settlement
- shadow-action settlement inside decision telemetry

**What it is for**

Produces the actual round/hand PnL the rest of BJW summarizes.

**Important note**

`blackjack_payout` is read from `manifest.payouts.blackjack`.

Common values are:

- `1.5` for `3:2`
- `1.2` for `6:5`

BJW does not hardcode only those two values; it uses the numeric payout carried in the manifest.

**Search terms**

- `blackjack 3:2 payout`
- `blackjack 6:5 payout`
- `late surrender half bet`
- `push zero outcome`

### 3.11 Initial Pair Probability

**Status**: Canonical runtime QA

**Formula**

```text
Pr(pair in d decks) = 13 * C(4d, 2) / C(52d, 2)
```

implemented as:

```text
pairs_per_rank = (4d * (4d - 1)) / 2
total_pairs = (52d * (52d - 1)) / 2
pair_rate = 13 * pairs_per_rank / total_pairs
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts` `theoreticalPairRate(...)`

**What it is for**

Checks that shoe randomness is producing plausible initial pair frequency.

**Search terms**

- `pair probability blackjack d deck`
- `combinatorics same rank two cards`

### 3.12 Pair/Split/Win-Loss-Push/Volume Rates

**Status**: Canonical runtime

**Formula**

```text
initial_pair_rate = initial_pairs_total / rounds
rounds_with_split_rate = rounds_with_split / rounds
winRateRound = roundWins / rounds
lossRateRound = roundLosses / rounds
pushRateRound = roundPushes / rounds
winRateHand = handWins / hands
lossRateHand = handLosses / hands
pushRateHand = handPushes / hands
avgHandsPerRound = hands / rounds
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts`

**What it is for**

Creates result summary rates and QA checks.

## 4. Canonical Runtime Performance and Risk Formulas

### 4.1 Observed Sample Variance via Welford

**Status**: Canonical runtime

**Formula**

Online update:

```text
delta  = x - mean
count += 1
mean  += delta / count
delta2 = x - mean
m2    += delta * delta2
```

final sample variance:

```text
variance = m2 / (n - 1)
sd = sqrt(max(0, variance))
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts` `recordHandOutcome(...)`, `recordRoundOutcome(...)`

**What it is for**

Tracks observed per-hand and per-round variance directly from simulated outcomes.

**Provenance status**

Standard external concept. Welford’s online variance update is standard statistics, applied here to blackjack outcomes.

**Citation status**

Exact reference recorded: [R2](#r2-welford-1962).

**Variables**

- `x`: realized hand or round net result in units
- `mean`, `m2`, `count`: Welford accumulators

**Where the variables come from**

- hand-level `x` values come from realized hand results recorded through `recordHandOutcome(...)`
- round-level `x` values come from realized round net results recorded through `recordRoundOutcome(...)`
- those hand and round net values are produced by settlement logic in Section 3 and passed into the Welford accumulators during play
- `mean`, `m2`, and `count` are the mutable online accumulators maintained for hands and rounds separately

**Expected output**

Observed sample variance/SD for hand outcomes and round outcomes.

**Search terms**

- `Welford algorithm`
- `online sample variance`

### 4.2 EV per Hand, per Round, per Initial Bet, and Average Bet

**Status**: Canonical runtime

**Formula**

```text
EV_per_hand = sumNetUnits / handsTotal
EV_per_round = sumNetUnits / actualRoundsPlayed
initialBetUnitsTotal = Σ (bet_units * frequency)
EV_per_initial_bet_units = sumNetUnits / initialBetUnitsTotal
EV_per_initial_bet_pct = 100 * sumNetUnits / initialBetUnitsTotal
avgBetUnits = totalBetUnits / totalRoundsPlayed
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts`
- extracted again in `scripts/validation/compare_to_reference.py`

**What it is for**

Produces BJW’s core profitability measures.

**Provenance status**

Standard bookkeeping identities over BJW runtime accumulators, with BJW-specific choices about which accumulators are exposed and named.

**Citation status**

No verified citation recorded. These are bookkeeping identities over BJW runtime accumulators.

**Variables**

- `sumNetUnits`: total realized profit/loss
- `handsTotal`: total hands played
- `actualRoundsPlayed`: total rounds played
- `betUnitsHistogram`: initial-bet histogram
- `totalBetUnits`: total wager actually placed across played rounds

**Where the variables come from**

- `sumNetUnits` is the top-level net accumulator updated from settled round outcomes
- `handsTotal` comes from hand-level settlement counting
- `actualRoundsPlayed` comes from accepted rounds in the main loop
- `betUnitsHistogram` is built from initial round bets only
- `totalBetUnits` is built from actual round wager exposure and can include doubles/splits on played rounds
- `initialBetUnitsTotal` is reconstructed from `betUnitsHistogram`

**Important note**

`avgBetUnits` is the average total wager per played round from `betting.totalBetUnits / betting.totalRoundsPlayed`.

That is not always the same as average initial bet, because `totalBetUnits` can include additional wager exposure from doubles and splits on rounds that were actually played.

BJW’s canonical runtime `risk_primary` formulas do not use `avgBetUnits` directly, but helper scripts, optimizers, and reporting code often use it for normalization and spread comparison.

**Search terms**

- `EV per hand`
- `IBA`
- `initial bet advantage`
- `total win divided by initial bet`

### 4.3 Primary Risk Basis Selection

**Status**: Canonical runtime

**Formula**

```text
if counting and count-based betting:
  primary basis = per_round
else:
  primary basis = per_hand
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts`
- `src/typescript/simulator_lite/src/index.ts` `syncObservedRiskContract(...)`

**What it is for**

Decides whether `risk_primary` uses observed round variance or observed hand variance.

**Provenance status**

BJW contract choice. This is not an external blackjack theorem; it is the current canonical result-contract rule BJW applies after simulation.

**Citation status**

No verified citation recorded. This is a BJW result-contract rule.

**Important note**

`syncObservedRiskContract(...)` is the step that writes:

- `summary.observed`
- top-level per-hand aliases like `summary.sdPerHandUnits`
- canonical `summary.risk_primary`

after worker aggregation and pooled-variance recomputation.

### 4.4 SCORE

**Status**: Canonical runtime

**Formula**

```text
SCORE = 1,000,000 * EV^2 / Variance
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts`
- `src/typescript/simulator_lite/src/index.ts`
- `src/python/wonk_tools/results.py`
- `src/python/bjw/catalog/results.py`
- `scripts/validation/compare_to_reference.py`
- `scripts/validation/intake_reference.py`
- display/reporting duplicates in `scripts/sim_ui.py`, `scripts/gui/widgets/simulation_manager.py`

**What it is for**

Produces the standardized risk-adjusted comparison metric BJW reports.

**Provenance status**

Standard blackjack/AP concept, but the exact citation is not recorded here. The formula is well known in blackjack literature; BJW’s additional choice is which EV/variance basis feeds it.

**Citation status**

No verified citation recorded for an exact source in this pass.

**Variables**

- `EV`: canonical EV on the current risk basis
- `Variance`: canonical variance on the current risk basis

**Where the variables come from**

- for active runtime results, both values come from `summary.risk_primary`
- that object is populated by `syncObservedRiskContract(...)` after pooled aggregation and basis selection
- for non-counting runs this usually means per-hand observed EV/variance
- for counted runs on this branch this usually means per-round observed EV/variance

**Search terms**

- `SCORE blackjack`
- `1,000,000 * EV^2 / variance`
- `Blackjack Attack SCORE`

### 4.5 Desirability Index (DI)

**Status**: Canonical runtime

**Formula**

```text
DI = 1000 * EV / SD
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts`
- `src/typescript/simulator_lite/src/index.ts`
- `src/python/wonk_tools/results.py`
- `src/python/bjw/catalog/results.py`
- `scripts/validation/compare_to_reference.py`
- display/reporting duplicates in `scripts/sim_ui.py`, `scripts/gui/widgets/simulation_manager.py`

**What it is for**

Produces BJW’s standard DI metric.

**Provenance status**

Standard blackjack/AP concept, but the exact citation is not recorded here. BJW’s runtime contract defines which EV/SD basis feeds the DI.

**Citation status**

No verified citation recorded for an exact source in this pass.

**Search terms**

- `Desirability Index blackjack`
- `DI = 1000 EV / SD`
- `Blackjack Attack DI`

### 4.6 Canonical Runtime N0

**Status**: Canonical runtime

**Formula**

```text
N0 = (SD / EV)^2
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts`
- `src/typescript/simulator_lite/src/index.ts`
- `src/python/bjw/catalog/results.py` (via `1_000_000 / SCORE`)
- `scripts/validation/compare_to_reference.py`

**What it is for**

Produces BJW’s canonical `risk_primary.n0` and preserves the algebraic identity:

**Provenance status**

Standard blackjack/AP concept, but the exact citation is not recorded here. BJW’s runtime contract defines the basis and therefore the unit semantics.

**Citation status**

No verified citation recorded for an exact source in this pass.

```text
DI^2 = SCORE
SCORE * N0 = 1,000,000
```

only when all three metrics are derived from the same EV and variance basis.

**Search terms**

- `N0 blackjack`
- `N-zero`
- `hands to one standard deviation`

**Important note**

This is **not** the same formula used in some helper/experiment scripts, which compute a 95%-confidence hands-to-profit quantity using `1.96`; see [8.5 Helper/API N0(Confidence) Formula](#85-helperapi-n0confidence-formula) and [11.2 N0 Definition Drift](#112-n0-definition-drift).

### 4.7 Risk of Ruin (RoR) Approximation

**Status**: Canonical runtime

**Formula**

```text
RoR ≈ exp(-2 * bankroll * edge / variance)
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts`
- duplicate reporting/optimizer copies in `scripts/compare_spreads.py`, `scripts/optimize_spread.py`

**What it is for**

Approximates the probability of eventual ruin for a positive-edge game.

**Provenance status**

Standard external approximation, not an exact blackjack-specific theorem. BJW uses it as an approximation layer over simulated EV/variance.

**Citation status**

No verified citation recorded for the exact approximation source used here.

**Variables**

- `bankroll`: bankroll in units
- `edge`: EV on the primary risk basis
- `variance`: variance on the same basis

**Where the variables come from**

- `bankroll` comes from bankroll settings or bankroll-analysis inputs expressed in betting units
- `edge` is the canonical EV chosen on the same basis as `summary.risk_primary`
- `variance` is the canonical variance on that same basis

**Interpretation**

The approximation only makes sense if `bankroll`, `edge`, and `variance` all share the same unit basis.

**Important note**

This is a gambler’s-ruin-style exponential approximation, not an exact blackjack theorem for BJW’s variable-bet counted games.

It is most defensible as a large-bankroll, positive-drift approximation. Reliability degrades when bankroll is small, edge is small, or the process departs materially from an i.i.d. random walk.

**Search terms**

- `risk of ruin exponential approximation`
- `exp(-2BR/variance)`
- `blackjack bankroll approximation`

### 4.8 Bankroll Sizing from Target RoR

**Status**: Canonical runtime

**Formula**

```text
bankroll = -ln(target_ror) * variance / (2 * edge)
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts`

**What it is for**

Inverts the RoR approximation to recommend bankroll sizes for target ruin probabilities.

**Provenance status**

Derived from the same standard approximation described in [4.7 Risk of Ruin (RoR) Approximation](#47-risk-of-ruin-ror-approximation), so its limits are the same.

**Citation status**

No verified citation recorded beyond the approximation note in [4.7 Risk of Ruin (RoR) Approximation](#47-risk-of-ruin-ror-approximation).

**Search terms**

- `bankroll from risk of ruin`
- `-ln(RoR) variance / 2 edge`

## 5. Parallel Aggregation and Pooled Variance Formulas

**Section provenance note**

This section is standard statistics, not blackjack-specific literature.

**Section citation status**

Exact reference recorded for the combined-variance algorithms discussed here: [R3](#r3-chan-golub-leveque-1983).

### 5.1 Pooled Mean and Variance for Merged Worker Results

**Status**: Canonical runtime aggregation

**Formula**

Two-sample pooled mean:

```text
pooled_mean = (n1 * mean1 + n2 * mean2) / (n1 + n2)
```

Two-sample pooled variance:

```text
pooled_var =
  ((n1 - 1) * var1
   + (n2 - 1) * var2
   + n1 * n2 / (n1 + n2) * (mean1 - mean2)^2)
  / (n1 + n2 - 1)
```

Multi-worker pooled variance:

```text
var_pooled = Σ[(n_i - 1) * var_i + n_i * (mean_i - mean_pooled)^2] / (N - 1)
```

**Primary locations**

- `src/typescript/simulator_lite/src/index.ts`

**What it is for**

Combines worker-local variances without losing between-worker mean differences.

**Important note**

This is a standard statistics combine formula, not a blackjack-specific result.

**Search terms**

- `pooled variance formula`
- `parallel variance combine`
- `between group variance term`
- `combining variance from batches`
- `Chan Golub LeVeque variance`

## 6. Per-Bucket and Decision-Telemetry Formulas

**Section provenance note**

These are BJW runtime bookkeeping and sample-moment formulas built from simulator telemetry. They are mostly arithmetic identities over recorded outcomes rather than externally named blackjack formulas.

**Section citation status**

No verified citation recorded section-wide. These are BJW telemetry and bookkeeping formulas.

### 6.1 EV-by-Bucket Variance

**Status**: Canonical runtime telemetry

**Formula**

```text
EV_per_hand = sumNetUnits / hands
variance =
  (sumNetUnitsSq - sumNetUnits^2 / hands) / (hands - 1)
SD = sqrt(max(0, variance))
avgBetUnits = totalBetUnits / rounds
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts` `evByBucket`

**What it is for**

Computes EV and SD per count bucket from raw aggregates.

**Search terms**

- `sample variance from sum of squares`
- `EV by true count`

### 6.2 Exact Decision Variable SD

**Status**: Canonical runtime telemetry

**Formula**

Same sample-variance identity as above, applied to the exact decision variable map.

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts` `exactDecisionVarStats`

**What it is for**

Checks monotonicity and non-bucketed decision-value behavior.

### 6.3 Decision Telemetry Action EV/SD

**Status**: Canonical runtime telemetry

**Formula**

```text
avgEV = sumEV / count
variance = (sumEVSq - sumEV^2 / count) / (count - 1)
sdEV = sqrt(max(0, variance))
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts` `decision_telemetry`
- pooled again in `src/typescript/simulator_lite/src/index.ts`

**What it is for**

Measures counterfactual action EV/dispersion for deviation discovery.

## 7. Legacy and Diagnostic Variance Formulas

### 7.1 Legacy `chapter10` Variance Decomposition

**Status**: Diagnostic/legacy only

**Formula**

Per-hand and per-round versions both use:

```text
E_b   = sum_b / count
E_w   = sum_w / count
E_b2  = sum_b2 / count
E_w2  = sum_w2 / count
E_bw  = sum_bw / count
Var_b = E_b2 - E_b^2
Var_w = E_w2 - E_w^2
Cov_bw = E_bw - E_b * E_w

variance = E_b2 * Var_w + Var_b * E_w^2 + 2 * Cov_bw * E_b * E_w
sd = sqrt(max(0, variance))
```

**Primary locations**

- raw telemetry capture in `src/typescript/simulator_lite/src/simulator.ts` `recordBetOutcome(...)`, `recordRoundBetOutcome(...)`
- recompute in `scripts/validation/run_formula_recompute_audit.py`

**What it is for**

Legacy BJW decomposition of variable-betting variance from bet/outcome accumulators.

**Variables**

- `b`: bet size
- `w`: normalized per-unit outcome

**Where the variables come from**

- per-hand `b` and `w` are captured by `recordBetOutcome(...)`
  - `b` is the realized wager on that hand
  - `w = hand_outcome / b`
- per-round `b` and `w` are captured by `recordRoundBetOutcome(...)`
  - `b` is the initial round bet
  - `w = round_outcome / initial_bet`
- the raw sums `sum_b`, `sum_w`, `sum_b2`, `sum_w2`, `sum_bw`, and `count` are stored in legacy variance telemetry and recomputed by `run_formula_recompute_audit.py`

**Important note**

This is **not** BJW’s canonical counted-risk formula on this branch.

Its provenance is currently repo-local and unresolved as an external citation. It should not be treated as a verified Schlesinger, Griffin, or Epstein formula unless an exact source is found.

**Citation status**

No verified citation recorded.

**Search terms**

- `variance of product bet outcome covariance`
- `variable betting variance decomposition`

### 7.2 Legacy `di_adjusted` Reconstruction

**Status**: Diagnostic/legacy only

**Formula**

```text
E[bet^2] = Σ(bet^2 * frequency) / rounds
base_variance = E[bet^2] * Var_w
var_bet_component = Var_b * E_w^2
covariance_component = 2 * Cov_bw * E_b * E_w
variance = base_variance + var_bet_component + covariance_component
sd = sqrt(variance)
count_correlation_factor = variance / base_variance
```

**Primary locations**

- `scripts/validation/run_formula_recompute_audit.py`

**What it is for**

Audits older result files that still contain `di_adjusted`.

**Citation status**

No verified citation recorded.

**Provenance status**

Legacy unsourced. This block exists to recompute and audit older BJW outputs, not as a currently validated canonical variance formula.

### 7.3 RMS SD Validation Heuristic

**Status**: Diagnostic/validation only

**Formula**

```text
E_b = Σ(bet * count) / N
E_b2 = Σ(bet^2 * count) / N
rms_bet = sqrt(E_b2)

non-counting expected SD scaling:
  SD_expected = baseline_SD * rms_bet

counting heuristic:
  SD_expected = baseline_SD * rms_bet * sqrt(0.71)
```

**Primary locations**

- `src/python/simulator_support/sd_validation.py`

**What it is for**

Sanity-checks whether observed SD is in a plausible range for a bet histogram.

**Important note**

This is a simulator-side heuristic, not BJW’s canonical runtime variance model.

The repo does not currently carry a verified external citation for the `0.71` factor. Treat it as a legacy BJW validation heuristic, not a sourced blackjack formula.

**Citation status**

No verified citation recorded.

**Search terms**

- `RMS bet scaling`
- `sqrt(E[bet^2])`
- `variance reduction factor 0.71`

## 8. Reporting, Helper, Optimizer, and Experiment Formulas

This section covers formulas outside the simulator that still produce blackjack/AP claims from result files.

### 8.1 Sharpe-Like Ratio

**Status**: Reporting/helper/optimizer

**Formula**

```text
Sharpe_like = EV / SD
```

**Primary locations**

- `scripts/compare_spreads.py`
- `scripts/optimize_spread.py`
- `scripts/ml/fitness_evaluator.py`
- `scripts/ml/analyze_results.py`
- `scripts/analysis/results_to_markdown.py`
- `scripts/sim_ui.py`
- `scripts/gui/widgets/simulation_manager.py`

**What it is for**

Used as a generic risk-adjusted comparison metric in optimizers and UI.

**Provenance status**

Standard generic ratio concept, but not BJW’s canonical blackjack DI unless the `1000` scaling and correct runtime basis are applied.

**Citation status**

No verified citation recorded for an exact source in this pass.

**Important note**

This is **not** BJW’s standard DI unless the script explicitly multiplies by `1000`.

**Search terms**

- `Sharpe ratio`
- `EV over SD`

### 8.2 Reweighting Formulas for Spread Optimization

**Status**: Optimizer/ML only

**Formula**

```text
EV_new = Σ(freq_i * EV_i * bet_i)
Variance_new = Σ(freq_i * SD_i^2 * bet_i^2)
SD_new = sqrt(Variance_new)
AvgBet = Σ(freq_i * bet_i)
Sharpe = EV_new / SD_new
```

**Primary locations**

- `scripts/optimize_spread.py`
- `scripts/ml/fitness_evaluator.py`

**What it is for**

Approximates the performance of alternative bet ramps from one baseline simulation.

**Provenance status**

BJW-authored model/heuristic. This is an internal approximation method for optimization, not a verified exact law for recomputing counted-strategy EV/variance under arbitrary new spreads.

**Citation status**

No verified citation recorded.

**Variables**

- `freq_i`: bucket frequency from `evByBucket`
- `EV_i`: per-hand EV in bucket `i` from `evByBucket[*].evPerHandUnits`
- `SD_i`: per-hand SD in bucket `i` from `evByBucket[*].sdPerHandUnits`
- `bet_i`: proposed bet size in bucket `i`

**Where the variables come from**

- `freq_i` is built from baseline `evByBucket` telemetry by dividing bucket hands by total baseline hands
- `EV_i` and `SD_i` are read directly from the baseline result’s per-bucket telemetry
- `bet_i` comes from the candidate spread allocation being optimized or scored
- `AvgBet` is then reconstructed from `freq_i * bet_i`, not read directly from runtime `risk_primary`

**Important note**

These optimizers currently reweight hand-level bucket telemetry, not canonical counted `risk_primary` round-basis risk. That makes them a useful approximation tool, but not a direct implementation of the new counted-risk contract.

**Search terms**

- `reweighting bet spread optimization`
- `bucket frequency weighted EV`

### 8.3 Helper/API DI Formula

**Status**: Reporting/helper

**Formula**

```text
DI = 1000 * EV / SD
```

**Primary locations**

- `src/python/wonk_tools/results.py` `get_di(...)`

**What it is for**

Convenience API for scripts and reports.

**Provenance status**

Helper wrapper around the DI concept in [4.5 Desirability Index (DI)](#45-desirability-index-di). The main risk is semantic drift, not different mathematics.

**Citation status**

No verified citation recorded beyond the base DI concept note in [4.5 Desirability Index (DI)](#45-desirability-index-di).

### 8.4 Helper/API SCORE Formula

**Status**: Reporting/helper

**Formula**

```text
SCORE = 1,000,000 * EV^2 / variance
```

**Primary locations**

- `src/python/wonk_tools/results.py` `get_score(...)`

**Provenance status**

Helper wrapper around the SCORE concept in [4.4 SCORE](#44-score). The main risk is basis drift if callers assume the wrong variance source.

**Citation status**

No verified citation recorded beyond the base SCORE concept note in [4.4 SCORE](#44-score).

### 8.5 Helper/API N0(Confidence) Formula

**Status**: Reporting/helper

**Formula**

```text
N0_confidence = (z * SD / EV)^2
```

with:

- `z = 1.96` for 95%
- `z = 2.576` for 99%

**Primary locations**

- `src/python/wonk_tools/results.py` `get_n0(...)`

**What it is for**

Returns a “hands needed for chosen confidence of being ahead” quantity.

**Provenance status**

Helper/reporting convention. This is not BJW’s canonical runtime N0 and should be treated as a confidence-style derivative quantity.

**Citation status**

No verified citation recorded for an exact source in this pass.

**Important note**

This is a **different definition** from runtime `risk_primary.n0`; see [4.6 Canonical Runtime N0](#46-canonical-runtime-n0) and [11.2 N0 Definition Drift](#112-n0-definition-drift).

Units depend on the basis of `EV` and `SD`:

- with per-hand inputs, the output is in hands
- with per-round inputs, the output is in rounds

On current results, `methodology='primary'` returns `summary.risk_primary.n0`, whose unit follows the canonical runtime basis rather than always meaning hands.

**Search terms**

- `hands to 95% confidence of profit`
- `1.96 SD over EV squared`

### 8.6 Catalog Fallback Formulas

**Status**: Reporting/helper

**Formula**

```text
DI = 1000 * EV / SD
SCORE = 1,000,000 * EV^2 / variance
N0 = 1,000,000 / SCORE
```

**Primary locations**

- `src/python/bjw/catalog/results.py`

**What it is for**

Lets the results browser backfill DI/SCORE/N0 when result files do not explicitly provide them.

**Provenance status**

Fallback reporting formulas derived from the canonical metric relationships, used when richer runtime fields are unavailable.

**Citation status**

No verified citation recorded beyond the base metric sections they derive from.

### 8.7 Result-Comparison Confidence Heuristics

**Status**: Reporting/UI

**Formulas**

Simple change percentages:

```text
pct_change = |new - old| / |old| * 100
```

and heuristic thresholds such as:

- `< 1%`: likely noise
- `< 3%`: may be within single-run variance

**Primary locations**

- `scripts/sim_ui.py` `assess_comparison_confidence(...)`

**What it is for**

Provides operator-facing warnings when optimization improvements are probably too small to trust.

**Provenance status**

BJW-authored heuristic. The percentage cutoffs are operator-facing warning thresholds, not sourced statistical decision rules.

**Citation status**

No verified citation recorded.

### 8.8 Win per Hand / Win per Round / EV% of Average Bet / Per-100-Hand Conversions

**Status**: Reporting/helper

**Formulas**

```text
win_per_hand = net_units / hands
win_per_round = net_units / rounds
ev_pct_of_avg_bet = 100 * EV_per_hand / avg_bet
EV_per_100_hands = EV_per_hand * 100
SD_per_100_hands = SD_per_hand * sqrt(100) = SD_per_hand * 10
EV_basis_points = EV_fraction * 100 * 100
rate_pct = 100 * rate
```

**Primary locations**

- `scripts/sim_ui.py`
- `scripts/gui/widgets/simulation_manager.py`
- `scripts/analysis/analyze_kelly.py`
- `scripts/experiments/experiment1_rule_variation_matrix.py`
- `src/python/wonk_tools/results.py`

**What it is for**

Turns simulator outputs into more operator-friendly units.

**Provenance status**

Standard unit-conversion arithmetic over BJW result fields.

**Citation status**

No verified citation recorded. These are direct unit conversions.

## 9. Statistical Confidence and Comparison Formulas

**Section provenance note**

This section is standard statistics and hypothesis-testing math, not blackjack-specific literature.

**Section citation status**

Exact reference recorded for confidence-interval and mean-comparison background: [R4](#r4-nistsematech-e-handbook-confidence-and-comparison-pages). Other formulas in this section are standard statistics, but no additional exact citation is recorded subsection-by-subsection in this pass.

### 9.1 Standard Error / 95% CI on EV

**Status**: Statistical comparison

**Formula**

Single-run EV CI:

```text
SEM = SD / sqrt(n)
CI_95_half_width = 1.96 * SEM
```

EV delta between two independent runs:

```text
SEM_delta = sqrt((SD_a / sqrt(n_a))^2 + (SD_b / sqrt(n_b))^2)
CI_95_half_width = 1.96 * SEM_delta
```

BJW also expresses some of these in percent or basis points:

```text
CI_95_EV_pct = 1.96 * SD / sqrt(n) * 100
CI_95_EV_bp = EV_to_bp(1.96 * SD / sqrt(n))
```

**Primary locations**

- `scripts/experiments/experiment1_rule_variation_matrix.py`
- `scripts/experiments/common.py`
- `scripts/experiments/experiment2_score_di_n0_matrix.py`
- `scripts/experiments/experiment3_ko_warmline_penetration.py`

**What it is for**

Quantifies uncertainty around EV or EV deltas.

**Search terms**

- `standard error mean`
- `95% confidence interval 1.96`
- `difference of two means standard error`

### 9.2 Standard Error of EV-by-Bucket Difference

**Status**: Statistical comparison

**Formula**

```text
variance = (sum_sq - sum^2 / n) / (n - 1)
ours_sd = sqrt(variance)
ours_se = ours_sd / sqrt(n)
se_delta = sqrt(ours_se^2 + ref_se^2)
pass if |delta| <= 2 * se_delta
```

**Primary locations**

- `scripts/validation/compare_to_reference.py`

**What it is for**

Assesses whether BJW EV-by-TC differs materially from the reference tool’s EV-by-TC.

### 9.3 t-Critical / Mean ± CI for Repeated Runs

**Status**: Statistical comparison

**Formula**

```text
CI = t_(0.95, df) * std / sqrt(n)
```

or via SciPy:

```text
stats.t.interval(0.95, n-1, loc=mean, scale=stats.sem(samples))
```

**Primary locations**

- `scripts/validation/compare_to_reference.py` `mean_ci95(...)`
- `scripts/ml/analyze_results.py`

**What it is for**

Builds confidence intervals when multiple independent simulation runs exist.

### 9.4 Z-Score for Run-to-Run Comparison

**Status**: Statistical comparison

**Formula**

```text
SE_diff = sqrt(SD1^2 / N1 + SD2^2 / N2)
z = EV_diff / SE_diff
```

**Primary locations**

- `scripts/gui/widgets/comparison_tool.py`
- `scripts/analyze_penetration_effects.py`
- `scripts/validate_exact_decision_var_monotonicity.py`

**What it is for**

Scores how statistically meaningful an observed EV difference is.

**Search terms**

- `z score difference of means`
- `SE(diff) = sqrt(sd1^2/n1 + sd2^2/n2)`

### 9.5 Two-Sample t-Test for Sharpe Comparison

**Status**: Statistical comparison

**Formula**

Implemented through SciPy:

```text
stats.ttest_ind(candidate_sharpes, baseline_sharpes)
```

**Primary locations**

- `scripts/ml/analyze_results.py`

**What it is for**

Tests whether candidate spread Sharpe means differ significantly from baseline Sharpe means.

### 9.6 Minimum Sample Heuristic for EV Precision

**Status**: Statistical comparison / experiment design

**Formula**

Used in some experiment scripts as a back-of-the-envelope minimum rounds-per-bucket rule:

```text
min_rounds ≈ (1.96 * SD / desired_precision)^2
```

Current hard-coded example:

```text
MIN_BUCKET_ROUNDS_FOR_SIG = (1.96 * 1.1 / 0.01)^2
```

**Primary locations**

- `scripts/experiments/experiment3_ko_warmline_penetration.py`
- `scripts/experiments/experiment7_rc_penetration_surface.py`

**What it is for**

Defines a rough minimum bucket sample size before treating a bucket EV as statistically useful.

**Search terms**

- `sample size for confidence interval width`
- `required n for desired standard error`

## 10. QA Threshold and Sanity-Check Formulas

**Section provenance note**

Unless otherwise stated, thresholds in this section are BJW-authored QA heuristics rather than externally sourced blackjack constants.

**Section citation status**

No verified citation recorded section-wide unless a subsection says otherwise.

### 10.1 Initial Pair Baseline Tolerance

**Status**: Runtime QA

**Formula**

```text
initial_pair_rate = initial_pairs_total / rounds
deviation = |initial_pair_rate - theoretical_pair_rate|
PASS if deviation <= 0.6 percentage points
WARN if deviation <= 1.2 percentage points
FAIL otherwise
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts` `performInstrumentationChecks(...)`

**What it is for**

Checks that pair frequency looks plausible for a random shoe source.

### 10.2 Rounds-With-Split Bound

**Status**: Runtime QA

**Formula**

```text
rounds_with_split_rate = rounds_with_split / rounds
expected bound: rounds_with_split_rate <= 0.08
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts` `performInstrumentationChecks(...)`

**What it is for**

Warns when split frequency looks suspiciously high.

### 10.3 Kelly Cap Warning Threshold

**Status**: Runtime QA

**Formula**

```text
pct_bets_at_cap = 100 * betsAtCap / totalRoundsPlayed
warning if pct_bets_at_cap > 10
```

**Primary locations**

- `src/typescript/simulator_lite/src/simulator.ts`

**What it is for**

Warns that the Kelly model is frequently slamming into table limits, which can make the spread unrealistic.

## 11. Known Semantic Drifts You Should Audit Carefully

### 11.1 DI Scaling Drift

There are two DI-style formulas in the repo:

- standard DI:

```text
DI = 1000 * EV / SD
```

- dimensionless EV/SD ratio, sometimes still labeled “DI” in old experiment/report code:

```text
ratio = EV / SD
```

**Files to inspect**

- standard DI: runtime, helpers, comparator, UI
- dimensionless ratio: `scripts/experiments/experiment2_score_di_n0_matrix.py`, `scripts/experiments/experiment3_ko_warmline_penetration.py`, `scripts/experiments/common.py`

**Primary definitions**

- Canonical runtime DI: [4.5 Desirability Index (DI)](#45-desirability-index-di)
- Helper/API DI: [8.3 Helper/API DI Formula](#83-helperapi-di-formula)
- Dimensionless ratio used in some optimizers: [8.1 Sharpe-Like Ratio](#81-sharpe-like-ratio)

### 11.2 N0 Definition Drift

There are two N0-style formulas in the repo:

- runtime/canonical:

```text
N0 = (SD / EV)^2
```

- confidence-style “hands to 95% confidence”:

```text
N0_95 = (1.96 * SD / EV)^2
```

**Files to inspect**

- canonical runtime N0: simulator, aggregator, catalog, comparator
- confidence-style N0: `src/python/wonk_tools/results.py`, `scripts/experiments/common.py`, `scripts/experiments/experiment2_score_di_n0_matrix.py`

**Primary definitions**

- Canonical runtime N0: [4.6 Canonical Runtime N0](#46-canonical-runtime-n0)
- Confidence-style helper N0: [8.5 Helper/API N0(Confidence) Formula](#85-helperapi-n0confidence-formula)

### 11.3 Counted-Risk Basis Drift in Older Scripts

Some older experiment/report code still assumes counted runs should read `summary.chapter10` and/or per-hand counted SD.

**Files to inspect**

- `scripts/experiments/experiment2_score_di_n0_matrix.py`
- `scripts/experiments/experiment3_ko_warmline_penetration.py`
- `scripts/experiments/common.py` when `prefer_ch10=True`
- `src/python/wonk_tools/results.py` old methodology options

**Primary definitions**

- Canonical counted-risk basis: [4.3 Primary Risk Basis Selection](#43-primary-risk-basis-selection)
- Legacy diagnostic variance: [7.1 Legacy `chapter10` Variance Decomposition](#71-legacy-chapter10-variance-decomposition)

### 11.4 Top-Level Per-Hand Fields vs `risk_primary`

Some UI/reporting scripts still compute DI/SCORE/Sharpe from:

```text
summary.evPerHandUnits
summary.sdPerHandUnits
summary.varPerHandUnits
```

instead of reading:

```text
summary.risk_primary
```

That means counted-run displays in those scripts can still show per-hand risk math even though canonical counted risk is now per-round.

**Files to inspect**

- `scripts/sim_ui.py`
- `scripts/gui/widgets/simulation_manager.py`
- `scripts/compare_spreads.py`
- `scripts/analysis/results_to_markdown.py`
- `scripts/ml/analyze_results.py`

**Primary definitions**

- Contract-sync step: [4.3 Primary Risk Basis Selection](#43-primary-risk-basis-selection)
- Canonical SCORE/DI/N0: [4.4 SCORE](#44-score), [4.5 Desirability Index (DI)](#45-desirability-index-di), [4.6 Canonical Runtime N0](#46-canonical-runtime-n0)

## 12. What You Will Not Find in BJW

You will **not** find a closed-form “blackjack edge formula” for the simulator’s main EV result.

BJW’s core engine:

- deals cards
- applies rules and policy lookups
- settles outcomes
- measures EV/variance from observed results

So for the most important outputs, the core “formula” is often:

```text
simulate outcomes, then compute sample moments from the realized data
```

That is why many of the most important audit targets are:

- count conversion
- betting logic
- settlement logic
- variance basis selection
- post-simulation summaries

not a single house-edge equation.

## 13. Fast Search Guide

If you want to verify BJW formulas against books or external material, these are the terms most likely to get you to the right place:

- `true count = running count / decks remaining`
- `true count rounding away from zero`
- `fractional Kelly blackjack`
- `edge / variance Kelly`
- `SCORE blackjack`
- `Desirability Index blackjack`
- `N0 blackjack`
- `risk of ruin exp(-2BR/V)`
- `insurance EV = 3p - 1`
- `insurance break-even 1/3`
- `pair probability 13*C(4d,2)/C(52d,2)`
- `sample variance from sum of squares`
- `Welford online variance`
- `standard error of difference of means`
- `95% CI 1.96`

## 14. Minimum File Set to Read First

If you want the shortest possible manual audit path, start here:

1. `src/typescript/simulator_lite/src/simulator.ts`
2. `src/typescript/simulator_lite/src/betting_vm.ts`
3. `src/typescript/simulator_lite/src/index.ts`
4. `src/typescript/simulator_lite/src/utils.ts`
5. `src/python/wonk_tools/results.py`
6. `scripts/validation/compare_to_reference.py`
7. `scripts/validation/run_formula_recompute_audit.py`
8. `src/python/simulator_support/sd_validation.py`

Then read the experiment/reporting duplicates only if you want to audit naming drift and non-canonical analysis math.

## 15. Exact References Recorded

### R1. Kelly (1956)

J. L. Kelly Jr., "A New Interpretation of Information Rate," *Bell System Technical Journal*, 35(4), 917-926, 1956.

- Official publication page: [Nokia Bell Labs](https://www.nokia.com/bell-labs/publications-and-media/publications/a-new-interpretation-of-information-rate/)
- DOI: [10.1002/j.1538-7305.1956.tb03809.x](https://doi.org/10.1002/j.1538-7305.1956.tb03809.x)

**Used in this document**

- 2.4 Kelly Fraction and Bet Size

### R2. Welford (1962)

B. P. Welford, "Note on a Method for Calculating Corrected Sums of Squares and Products," *Technometrics*, 4(3), 419-420, 1962.

- DOI: [10.1080/00401706.1962.10490022](https://doi.org/10.1080/00401706.1962.10490022)

**Used in this document**

- 4.1 Observed Sample Variance via Welford

### R3. Chan, Golub, and LeVeque (1983)

T. F. Chan, G. H. Golub, and R. J. LeVeque, "Algorithms for Computing the Sample Variance: Analysis and Recommendations," *The American Statistician*, 37(3), 242-247, 1983.

- DOI: [10.1080/00031305.1983.10483115](https://doi.org/10.1080/00031305.1983.10483115)
- Public PDF mirror used in this pass: [PKU mirror](https://www.math.pku.edu.cn/teachers/litj/notes/numer_anal/AmerStat_37_242_Chan_Variance.pdf)

**Used in this document**

- 5.1 Pooled Mean and Variance for Merged Worker Results

### R4. NIST/SEMATECH e-Handbook Confidence and Comparison Pages

NIST/SEMATECH e-Handbook of Statistical Methods, especially the confidence-interval and process-comparison pages used in this pass.

- Handbook home: [NIST/SEMATECH e-Handbook](https://www.itl.nist.gov/div898/handbook/)
- Confidence intervals overview: [7.1.4 What are confidence intervals?](https://www.itl.nist.gov/div898/handbook/prc/section1/prc14.htm)
- Confidence-interval approach with unknown standard deviation: [7.2.2.1 Confidence interval approach](https://www.itl.nist.gov/div898/handbook/prc/section2/prc221.htm)
- Two-process mean comparison background: [7.3.1 Do two processes have the same mean?](https://www.itl.nist.gov/div898/handbook/prc/section3/prc31.htm)

**Used in this document**

- 9.1 Standard Error / 95% CI on EV
- 9.2 Standard Error of EV-by-Bucket Difference
- 9.3 t-Critical / Mean +- CI for Repeated Runs
- 9.4 Z-Score for Run-to-Run Comparison
- 9.5 Two-Sample t-Test for Sharpe Comparison
