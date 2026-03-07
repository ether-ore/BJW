# Methodology

## Simulation Scale

**Minimum for conclusions: 1 billion rounds.**

Results from smaller runs are used only for development testing. All published results and validation work use 1B rounds per configuration. At 1B rounds, standard error on EV is on the order of 0.001pp for a 6-deck 1-unit flat betting scenario — well below any meaningful decision threshold.

---

## Reproducibility

Every BJW simulation is fully specified by a **manifest file**. The manifest captures:

- All game rules (deck count, S17/H17, DOA/D10, DAS, RSA, surrender, penetration, burn cards)
- Counting system (tag values, balance type, TC divisor configuration, insurance threshold)
- Betting policy (BVM bytecode encoding the spread or progression)
- Play policy (complete basic strategy decision map)
- Deviation set (index plays with TC thresholds)
- PRNG seed

Same manifest + same seed = same results on any machine, any run. The manifest hash serves as the experiment identifier.

Manifests are schema-validated before execution. If a manifest references a counting system, betting spread, or deviation set that is incompatible with the declared rules, the configurator rejects it before any simulation runs.

---

## Parallel Execution

Large simulations use multi-worker parallel execution. The current validation suite runs use 8 workers.

**Addition principle**: When running in parallel, each worker processes a distinct subset of rounds. The aggregated result must be exactly equal to a single-threaded run over the same rounds — not "close," exactly equal. This is verified via binary deck regression runs: a pre-shuffled shoe sequence is fed identically to single-threaded and multi-threaded modes, and all output metrics are required to match exactly. Any discrepancy, no matter how small, indicates a bug.

---

## True Count Divisor

For all balanced counting systems, true count is calculated as:

```
TC = running_count / decks_remaining
```

Where `decks_remaining` is quantized to the nearest half-deck using cards remaining (not cards dealt).

This configuration — half-deck quantization, cards-remaining basis — is used in all validation suite runs and all published results. The distinction between cards-remaining and cards-dealt matters: using cards-dealt as the divisor basis inverts the TC direction at deep penetration and produces dramatically wrong results. BJW supports both configurations but only cards-remaining is used in any published work.

---

## TC Calculation at Shoe Depth Extremes

At very deep penetration in single-deck games (few cards remaining), the decks-remaining divisor approaches zero. BJW handles this via half-deck quantization with a floor — the divisor is clamped to prevent division by near-zero values. This is the correct behavior for half-deck TC conventions and is consistent with how human counters apply the divisor in practice.

---

## Burn Cards

All validation suite runs use 1 burn card per shoe. This is configurable per manifest.

---

## Penetration

All validation suite runs use 75% penetration (cut card at 75% of shoe depth). This is specified as a percentage in the manifest and translated to a cut card position at shoe generation time.

---

## Betting VM

Betting policies execute through the Betting VM (BVM) — a register-based virtual machine that runs the same bytecode regardless of strategy type. This ensures flat betting, spread betting, and progressive betting all go through the same execution path, with no special-casing in the simulator for any particular strategy type.

State in the BVM persists across rounds within a session. This is important for progressions — the BVM remembers where a 1-3-2-6 sequence is between hands.

---

## What We Do Not Publish as Absolute Claims

Given the CVData parity gap on counted scenarios:

- We do not publish absolute EV claims for counted scenarios without noting the CVData delta
- We do not publish absolute SCORE or N₀ claims for counted scenarios as standalone figures
- We do publish relative comparisons, marginal contribution analyses, and directional claims freely — systematic offsets cancel in relative comparisons within the same engine

See [Validation Position](validation/position.md) for full detail.

---

## Milestones

| Date | Milestone |
|---|---|
| October 2025 | Initial simulator build; basic strategy, Hi-Lo, and flat/spread betting operational |
| March 2026 | Full validation campaign: policy audit (224 checks), formula recompute audit (400 checks), theory sanity matrix, CVData comparison across 8 core scenarios at 1B rounds each |
| March 7, 2026 | Option 1 lock: Track B pass accepted as correctness baseline; Track A residuals documented and classified |
