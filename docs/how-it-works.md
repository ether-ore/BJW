# How It Works

## Architecture Overview

BJW uses a two-component architecture:

```
Rules Files  →  Configurator  →  Manifest (JSON)  →  Simulator  →  Results (JSON)
(Python)                          (validated)        (TypeScript)
```

**Configurator (Python)**: Assembles simulation specifications from rule combinations. Validates all inputs against JSON schemas. Transpiles betting policies to Betting VM bytecode. Outputs a manifest file.

**Simulator (TypeScript)**: High-performance engine that executes manifests. Runs single-threaded or multi-worker parallel. Produces structured JSON results with full telemetry.

Nothing is hardcoded in the simulator. All behavior — rules, strategy, counting system, betting spread, deviations — comes from the manifest.

---

## The Manifest

A manifest is a complete, self-contained simulation specification. It includes:

- **Ruleset**: deck count, S17/H17, DOA/D10, DAS/NDAS, RSA, peek mode, penetration %, burn cards
- **Counting system**: tag values, balance type, TC divisor method, insurance threshold
- **Betting policy**: BVM bytecode encoding the spread or progression logic
- **Play policy**: full basic strategy decision map for the rule configuration
- **Deviations**: index plays to apply, with their TC thresholds
- **Seed**: PRNG seed for full reproducibility

Manifests are schema-validated before execution. A manifest hash serves as the experiment identifier — the same manifest + same seed produces the same results on any machine.

---

## The Betting VM

All betting strategies — flat bets, count-based spreads, progressions, state machines — execute through a single **Betting VM (BVM)** runtime. The configurator transpiles authored betting policies to BVM bytecode; the simulator executes that bytecode each round.

This means any betting strategy expressible in the policy language can be simulated without touching the simulator code.

**BVM strategy types**:

| Kind | Examples |
|---|---|
| `flat` | 1 unit every hand |
| `bucket_spread` | Hi-Lo 1-12: bet scales with true count |
| `progression` | 1-3-2-6, Paroli, Martingale |
| `state_machine` | Custom formulas with session state |

---

## Parallel Execution

For large simulations (100M–1B+ rounds), BJW spawns multiple worker processes that each run a subset of rounds and aggregate results. Aggregation uses a declarative rule engine — over 140 field-level rules specifying whether each metric is summed, histogrammed, or derived.

Parallel and single-threaded modes produce **identical results** for any given manifest and seed. This is verified via binary deck regression runs, where a pre-shuffled shoe sequence is fed identically to both modes and the outputs are required to match exactly — not approximately.

---

## Results

Simulator output is a structured JSON file containing:

- `round_summary`: rounds played, hands, wins, losses, pushes
- `summary.naive`: per-hand and per-round SD using simple Welford variance
- `summary.chapter10`: full Schlesinger Chapter 10 decomposition — E[b], E[w], Var(b), Var(w), Cov(b,w), DI, SCORE, N₀
- `summary.di_adjusted`: DI-adjusted variance using simulation-derived Var(w)
- `ev_by_tc`: EV breakdown by true count bucket
- `tc_frequencies`: time distribution across TC bins at bet-decision point
- `count_histogram`: running count distribution
- `action_summary`: doubles, splits, surrenders, insurance by scenario

All metrics feed the validation pipeline, which independently recomputes Chapter 10 figures from raw accumulators to verify formula correctness.
