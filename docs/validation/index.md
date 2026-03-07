# Validation

*As of March 7, 2026.*

---

BJW validation is organized into two independent tracks:

**Track A — CVData parity**: Does BJW produce the same output as CVData (Wattenberger)?
**Track B — Internal math and rules**: Does BJW correctly implement published blackjack mathematics and basic strategy?

These are different questions and have different answers.

---

## Current Status

| Track | Status | Detail |
|---|---|---|
| Track A (CVData parity) | **0/8 scenarios pass** | Gaps documented and classified by root cause |
| Track B (Policy audit) | **224/224 checks pass** | All 16 play policy variants audited against published charts |
| Track B (Formula audit) | **400/400 checks pass** | Chapter 10 independently recomputed from raw accumulators |
| Track B (Theory matrix) | **7/8 pass, 1 explained residual** | Flat EV bands, directional invariants, monotonicity |

---

## What This Means

Track B passing means BJW correctly implements the math. The Schlesinger Chapter 10 variance formula is not approximated — it is computed exactly, and verified independently by recomputing every figure from raw per-hand accumulators in a separate Python script.

Track A not passing means BJW and CVData disagree on counted scenario outputs. CVData is a closed-source commercial tool. Without access to its internals, the exact source of the disagreement cannot be fully resolved. The leading mechanism — TC bucket occupancy drift — is documented and classified. It is not a formula error.

**Our position**: Track B passing is the correctness claim. Track A is a parity check against one specific external tool, not a correctness gate. We publish both results openly.

---

## Navigation

→ [Full validation position](position.md) — complete disclosure including CVData delta table, root cause classification, and what we will and won't claim from BJW results

→ [Audit detail](detail.md) — methodology and results for each Track B audit (policy, formula, theory)
