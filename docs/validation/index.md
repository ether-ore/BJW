# Validation

*As of March 13, 2026 for the published position, with repo organization updated on March 20, 2026.*

BJW validation is organized into two independent tracks:

**Track A - CVData parity**: Does BJW produce the same output as CVData (Wattenberger)?
**Track B - Internal math and rules**: Does BJW correctly implement published blackjack mathematics and basic strategy?

These are different questions and have different answers.

## Current Status

| Track | Status | Detail |
|---|---|---|
| Track A (CVData parity) | **0/8 scenarios pass** | March 13 rerun after two additional fixes still leaves residual gaps |
| Track B (Policy audit) | **224/224 checks pass** | All 16 play policy variants audited against published charts |
| Track B (Formula audit) | **400/400 checks pass** | Variance/risk outputs independently recomputed from raw accumulators |
| Track B (Theory matrix) | **7/8 pass, 1 explained residual** | Flat EV bands, directional invariants, monotonicity |

## What This Means

Track B passing means BJW correctly implements the math. Track A not passing means BJW and CVData still disagree on counted scenario outputs. We publish both results openly and treat Track B as the correctness claim.

## Canonical Repo Sources

- `docs/validation/README.md`
- `docs/validation/dossier/VALIDATION_STATUS.md`
- `docs/validation/cvdata/CVDATA_RECOMPARE_AFTER_FIXES_2026-03-13.md`
- `docs/validation/risk/OBSERVED_ROUND_VARIANCE_PRIMARY_REMEDIATION_2026-03-16.md`

## Navigation

-> [Full validation position](position.md) - complete disclosure including CVData delta table, root cause classification, and what we will and will not claim from BJW results

-> [Audit detail](detail.md) - methodology and results for each Track B audit (policy, formula, theory)
