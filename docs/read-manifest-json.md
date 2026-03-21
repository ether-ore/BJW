# How to Read a Raw Manifest JSON File

Example files:

- [Example manifest JSON](examples/manifest_6D_H17_DOA_DAS_NRSA_NS_peek=A_or_T_pen=75_hilo_hilo_1_to_12_common_burn1_tcdiv_half_deck_seed12345.json)
- [Matching results JSON](examples/result_6D_H17_DOA_DAS_NRSA_NS_peek=A_or_T_pen=75_hilo_hilo_1_to_12_common_burn1_tcdiv_half_deck_seed12345.json)

This page is for blackjack players, not programmers.

A raw manifest JSON file is the simulation's recipe card. It tells BJW exactly what game to run, what counting system to use, how to bet, and which playing decisions to follow.

If the manifest is wrong, the simulation is wrong.

## JSON in 60 Seconds

You do not need to know coding to read JSON.

- Curly braces `{ }` mean "this is a box of information."
- A line like `"decks": 6` means "the number of decks is 6."
- Square brackets `[ ]` mean "this is a list."
- `true` means yes.
- `false` means no.
- Commas just separate one line from the next.

That is enough to get started.

## What a Manifest Tells You

A manifest answers these blackjack questions:

- What game are we playing?
- What are the rules?
- Which counting system is being used?
- What bet spread is being used?
- Are deviations included?
- Which exact shuffle seed was used?

It does **not** tell you whether the run won or lost. That lives in the results file.

## Read the File in This Order

### 1. Start with `manifest_id`

This is the run's name tag.

Example:

```json
{
  "manifest_id": "6D_H17_DOA_DAS_NRSA_NS_peek=A_or_T_pen=75_Hi-Lo_hilo_1_to_12_common_burn1_tcdiv_half_deck_seed12345"
}
```

You do not need to decode every part of that long name. Just know that it usually packs in the key facts:

- `6D` = 6 decks
- `H17` = dealer hits soft 17
- `DOA` = double on any two
- `DAS` = double after split
- `NS` = no surrender
- `Hi-Lo` = counting system
- `hilo_1_to_12_common` = bet spread
- `seed12345` = random seed

### 2. Read `environment.ruleset`

This is the casino game itself.

Shortened real example:

```json
{
  "environment": {
    "ruleset": {
      "decks": 6,
      "dealer_hits_soft_17": true,
      "double_on_any_two": true,
      "double_after_split": true,
      "resplit_aces": false,
      "max_splits": 4,
      "surrender_allowed": false,
      "dealer_peek_mode": "A_or_T",
      "burn_cards": 1,
      "penetration_pct": 0.75
    }
  }
}
```

Plain-English translation:

| JSON line | What it means at the table |
|---|---|
| `decks: 6` | Six-deck shoe |
| `dealer_hits_soft_17: true` | Dealer hits soft 17, so this is H17 |
| `double_on_any_two: true` | You can double on any first two cards |
| `double_after_split: true` | DAS is allowed |
| `resplit_aces: false` | No resplitting aces |
| `surrender_allowed: false` | No surrender |
| `dealer_peek_mode: "A_or_T"` | Dealer peeks under ace or ten |
| `penetration_pct: 0.75` | About 75 percent of the shoe is dealt before shuffle |

If you only read one part of the file, read this part.

### 3. Read `environment.random_source`

This tells you how the shuffle was chosen.

```json
{
  "random_source": {
    "type": "prng",
    "seed": 12345
  }
}
```

What this means:

- `type: "prng"` = normal computer-generated randomness
- `seed: 12345` = the exact starting point for that randomness

If someone else uses the same manifest and same seed, they should be able to reproduce the same run setup.

### 4. Read `counting`

This is the counting system section.

Shortened real example:

```json
{
  "counting": {
    "system": "Hi-Lo Counting Strategy",
    "key": "hilo",
    "balanced": true,
    "uses_true_count": true,
    "starting_count": 0,
    "insurance": {
      "rule": "tc>=",
      "threshold": 3
    },
    "tc_divisor_mode": "half_deck"
  }
}
```

How to read it:

- `system` = the count name you already know as a player
- `balanced: true` = cards add up to zero across a full deck
- `uses_true_count: true` = this system converts running count to true count
- `starting_count: 0` = the shoe starts at zero
- `insurance threshold: 3` = insurance starts at TC +3
- `tc_divisor_mode: "half_deck"` = true count is measured in half-deck steps

There may also be a `card_values` block listing the tag for each rank. If you already know Hi-Lo tags, you can skim it.

### 5. Read `betting`

This section tells you how the bet spread works.

Shortened real example:

```json
{
  "betting": {
    "table_min_units": 1,
    "table_max_units": 100,
    "policy_source": {
      "key": "hilo_1_to_12_common"
    }
  }
}
```

That tells you:

- minimum bet = 1 unit
- maximum allowed bet = 100 units
- spread file = `hilo_1_to_12_common`

If you want the actual spread ladder, keep reading inside `policy_vm.bucket_spread.buckets`.

Example rows:

```json
[
  { "min": -20, "max": 0.9, "units": 1 },
  { "min": 2, "max": 2.9, "units": 2 },
  { "min": 3, "max": 3.9, "units": 4 },
  { "min": 7, "max": null, "units": 12 }
]
```

Plain English:

- 1 unit below TC +1
- 2 units at TC +2
- 4 units at TC +3
- 12 units at TC +7 and above

`null` here means "no upper limit."

### 6. Read `payouts`

This confirms the pay table.

```json
{
  "payouts": {
    "blackjack": 1.5,
    "normal": 1.0,
    "insurance": 2.0
  }
}
```

Translation:

- blackjack pays 3:2
- regular wins pay 1:1
- insurance pays 2:1

### 7. Skim `policies`

This is the giant decision section.

It usually contains:

- `play_policy`
- `insurance_policy`
- `bet_policy`
- sometimes `deviations`

This part looks intimidating because it can be huge. That is normal.

For a non-programmer, the main point is simple:

- `play_policy` = what the player does with each hand
- `insurance_policy` = when insurance is taken
- `bet_policy` = how bet size changes with the count

If you are not auditing exact hand decisions, you can mostly skim this section.

### 8. Treat `sim_hash`, `meta`, `bucketing`, and `presim` as support material

These sections matter, but they are not where a player should start.

- `sim_hash` = a fingerprint for the manifest
- `meta` = provenance notes, source files, and bookkeeping
- `bucketing` = the count range the sim tracks, like TC -10 through +10
- `presim` = machine-prepared lookup data used by the simulator

Useful for auditing. Not the first thing to read.

## A Good 30-Second Manifest Check

Before trusting a simulation, answer these six questions:

1. Is the game right? Decks, H17/S17, DAS, surrender, penetration.
2. Is the count right?
3. Is the spread right?
4. Are deviations included or not?
5. Is the blackjack payout right?
6. Is this the seed and file you meant to run?

If those six are right, the manifest is probably the one you wanted.

## Next Step

Once the manifest makes sense, read the matching [results file guide](read-results-json.md).
