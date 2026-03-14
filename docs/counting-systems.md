# Counting Systems

BJW supports the following counting systems. All are implemented as tag-value files with explicit balance type, insurance threshold, and TC divisor configuration.

---

## Supported Systems

| System | Balance | Level | TC Method | Insurance Threshold | Deviations Available |
|---|---|---|---|---|---|
| Hi-Lo | Balanced | 1 | True count | TC ≥ 3 | Illustrious 18, Full set |
| KO (Knock-Out) | Unbalanced | 1 | Running count | RC ≥ 3 | KO Preferred, KO Full |
| REKO | Unbalanced | 1 | Running count | RC ≥ 2 | — |
| Zen Count | Balanced | 2 | True count | TC ≥ 5 † | — |
| Halves | Balanced | 3 | True count | TC ≥ 3 | — |
| Hi-Opt I | Balanced | 1 | True count | TC ≥ 3 | — |
| Hi-Opt II | Balanced | 2 | True count | TC ≥ 3 | — |
| Omega II | Balanced | 2 | True count | TC ≥ 3 | — |
| Red Seven | Unbalanced | 1 | Running count | RC ≥ 2 | — |
| Ace-Five | Balanced ‡ | 1 | Running count | Never | — |

**† Zen Count insurance note**: Zen uses ±2 tags; its running count accumulates roughly twice as fast as Hi-Lo. TC ≥ 5 under Zen corresponds to approximately the same insurance edge threshold as TC ≥ 3 under Hi-Lo. Threshold confirmed against primary source and empirically verified at 100M rounds (zero insurance taken below TC+5; positive EV in both above-threshold bins).

**‡ Ace-Five balance note**: Ace-Five's tag values (Ace = −1, Five = +1, all others = 0) sum to zero across a complete deck, so it satisfies the mathematical definition of a balanced count. However, it does not use a true count divisor — the betting trigger is a raw running count threshold. It is listed separately from the unbalanced systems (KO, REKO, Red Seven) which require an initial running count offset to compensate for deck imbalance.

---

## Specialty / Research Systems

These systems are included for research purposes. They are not standard advantage play systems.

| System | Description |
|---|---|
| High-Only | Tags only high cards (−1); research use |
| Non-Tens-Aces | Tags low/mid cards only; research use |
| Tens Only | Tags tens only; research use |
| Ace/Ten Front Count (H17) | Front-loaded Ace/Ten tracking; H17 variant |
| Ace/Ten Front Count (S17) | Front-loaded Ace/Ten tracking; S17 variant |
| Not Counting | Flat/progression betting with no count; disables TC logic |

---

## Tag Values

For reference, the card tag values for the primary systems:

| Card | Hi-Lo | KO | Zen | Halves | Hi-Opt I | Hi-Opt II | Omega II |
|---|---|---|---|---|---|---|---|
| 2 | +1 | +1 | +1 | +0.5 | 0 | +1 | +1 |
| 3 | +1 | +1 | +1 | +1 | +1 | +1 | +1 |
| 4 | +1 | +1 | +2 | +1 | +1 | +2 | +2 |
| 5 | +1 | +1 | +2 | +1.5 | +1 | +2 | +2 |
| 6 | +1 | +1 | +2 | +1 | +1 | +1 | +2 |
| 7 | 0 | +1 | +1 | +0.5 | 0 | +1 | +1 |
| 8 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| 9 | 0 | 0 | 0 | −0.5 | 0 | 0 | −1 |
| T/J/Q/K | −1 | −1 | −2 | −1 | −1 | −2 | −2 |
| A | −1 | −1 | −1 | −1 | 0 | 0 | 0 |

---

## True Count Divisor

For all balanced systems, BJW calculates true count using:

- **Divisor**: decks remaining (half-deck quantization)
- **Basis**: cards remaining (not cards dealt)
- **Burn card**: 1 card burned per shoe (configurable)

This configuration is used in all validation suite runs. See [Methodology](methodology.md) for detail on why this matters.
