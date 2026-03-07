# Blackjack Wonk

**Blackjack Wonk (BJW)** is a research-grade blackjack simulation framework for evaluating card counting strategies, betting systems, and rule set variations with statistical rigor.

BJW is a tool for people who already know the math — analysts, counters, and researchers who want to run controlled experiments rather than accept published tables at face value.

---

## What It Does

BJW simulates blackjack play under fully specified conditions and produces:

- **EV per hand** — expected value under the configured strategy and rules
- **SD, DI, SCORE, N₀** — via the Schlesinger Chapter 10 variance framework
- **EV by true count bucket** — per-TC breakdown of edge contribution
- **TC frequency distribution** — time spent at each count level
- **Action frequencies** — doubles, splits, surrenders, insurance decisions
- **Deviation tracking** — which index plays were taken and when

Every simulation is driven by a **manifest** — a JSON configuration capturing game rules, counting system, betting spread, play policy, deviations, and PRNG seed. Same manifest, same seed, same results. Always.

---

## What It Supports

**Deck counts**: 1, 2, 6, 8
**Dealer rules**: S17, H17
**Doubling**: DOA (any two cards), D10 (10/11 only)
**Split rules**: DAS, NDAS, RSA
**Surrender**: Late Surrender, No Surrender
**Penetration**: configurable by percentage cut card

**Counting systems**: Hi-Lo, KO, REKO, Zen, Halves, Hi-Opt I, Hi-Opt II, Omega II, Red Seven, Ace-Five, and others — see [Counting Systems](counting-systems.md)

**Betting**: flat, count-based spreads (1-2 through 1-16+), progressions (Martingale, Paroli, 1-3-2-6, and others), state machine policies

**Deviations**: Illustrious 18 (Hi-Lo), KO Preferred and Full sets, custom deviation policies

---

## Validation Status

BJW's math is independently verified against the Schlesinger Chapter 10 variance formula. Basic strategy across all 16 policy variants has been audited against published charts.

BJW does not currently match CVData output on counted scenarios. The gap is documented, classified by root cause, and published openly.

→ [Full validation position](validation/position.md)
→ [Audit detail](validation/detail.md)

---

## How It Works

→ [Architecture and design](how-it-works.md)
→ [Simulation methodology](methodology.md)

---

## Questions and Contact

Issues and questions: [github.com/ether-ore/BJW/issues](https://github.com/ether-ore/BJW/issues)
