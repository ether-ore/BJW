# How to Read a Results JSON File

Canonical source: [docs/guides/how-to-read-a-results-json.md](https://github.com/ether-ore/BJW/blob/main/docs/guides/how-to-read-a-results-json.md)

Example files:

- [Example results JSON](examples/result_6D_H17_DOA_DAS_NRSA_NS_peek=A_or_T_pen=75_hilo_hilo_1_to_12_common_burn1_tcdiv_half_deck_seed12345.json)
- [Matching manifest JSON](examples/manifest_6D_H17_DOA_DAS_NRSA_NS_peek=A_or_T_pen=75_hilo_hilo_1_to_12_common_burn1_tcdiv_half_deck_seed12345.json)

This page is for blackjack players who want to understand BJW output without needing to think like a programmer.

A results JSON file is the simulation's scorecard. It tells you what happened after the run was finished.

If the manifest is the recipe, the results file is the box score.

## JSON in 30 Seconds

You only need four ideas:

- `{ }` = a box of information
- `"name": value` = the value attached to that label
- `[ ]` = a list
- `true` and `false` mean yes and no

Read the labels on the left first. That is where the meaning lives.

## What a Results File Can Tell You

A results file answers questions like:

- Did this game and strategy win in the long run?
- What was the player's edge?
- How volatile was it?
- What was the average bet?
- Which counts made the money?
- Did the run actually match the intended manifest?

## Read the File in This Order

### 1. Start with `manifest`, `rules`, and `counting`

Before reading profit numbers, make sure you opened the right file.

Shortened real example:

```json
{
  "manifest": {
    "manifest_id": "6D_H17_DOA_DAS_NRSA_NS_peek=A_or_T_pen=75_Hi-Lo_hilo_1_to_12_common_burn1_tcdiv_half_deck_seed12345"
  },
  "rules": {
    "decks": 6,
    "h17": true,
    "ls": false,
    "doa": true,
    "das": true
  },
  "counting": {
    "system": "Hi-Lo Counting Strategy",
    "bucket_name": "TC"
  }
}
```

Quick translation:

- `h17: true` means this is H17
- `ls: false` means no surrender
- `doa: true` means double on any two
- `das: true` means double after split
- `bucket_name: "TC"` means count-based reports are grouped by true count

This is the "did I open the right run?" section.

### 2. Read `run` and `effective`

This tells you how big the run was and what random source it actually used.

```json
{
  "run": {
    "rounds": 1000000000
  },
  "effective": {
    "random_source": {
      "type": "prng",
      "seed": 12345
    }
  }
}
```

What matters here:

- `rounds` = total rounds actually completed
- bigger sample = more trustworthy result
- `seed` = exact shuffle seed used
- there may also be a `requestedRounds` field, but `rounds` is the first number to trust

For BJW, one billion rounds is serious evidence. A tiny sample is not.

### 3. Read `summary` next

This is the money section. Most players should spend the most time here.

Shortened real example:

```json
{
  "summary": {
    "sumNetUnits": 4662554.5,
    "evPerHandUnits": 0.004535779856689924,
    "evPerInitialBetPct": 0.27457923681015795,
    "score": 2.579142776778825,
    "sdPerHandUnits": 2.826339517799622
  }
}
```

How to read each line:

| JSON line | Plain-English meaning |
|---|---|
| `sumNetUnits` | Total units won in this full simulation |
| `evPerHandUnits` | Average profit per hand, in units |
| `evPerInitialBetPct` | Edge as a percent of the original bet |
| `score` | A blackjack comparison rating. Higher is better. |
| `sdPerHandUnits` | Volatility per hand. Higher means a wilder ride. |

The two most useful fields for most readers are:

- `evPerHandUnits`
- `evPerInitialBetPct`

Example:

- `evPerHandUnits: 0.0045` means about **0.45 units won per 100 hands**
- `evPerInitialBetPct: 0.2746` means about a **0.275 percent player edge over the initial bet**

If your unit is $25, then 0.45 units per 100 hands is about $11.25 per 100 hands.

### 4. Read `betting`

This tells you how much action the player actually got down.

Shortened real example:

```json
{
  "betting": {
    "avgBetUnits": 1.914674491,
    "totalBetUnits": 1914674491,
    "totalRoundsPlayed": 1000000000
  }
}
```

What it means:

- `avgBetUnits` = average bet size during the whole run
- `totalBetUnits` = total action wagered
- if there is a `bet_units_histogram`, it shows how often each bet size came up

This matters because two winning strategies can have very different average bets and risk levels.

### 5. Read `round_summary` and `hand_summary`

These tell you how often the player won, lost, or pushed.

Shortened real example:

```json
{
  "round_summary": {
    "rounds": 1000000000,
    "roundWins": 433148393,
    "roundLosses": 481405977,
    "roundPushes": 85445630
  },
  "hand_summary": {
    "handsTotal": 1027949911,
    "handWins": 447190358,
    "handLosses": 494955861,
    "handPushes": 85803692
  }
}
```

Why there are two sections:

- a **round** is one starting hand
- a **hand** includes split hands

So if you split once, one round can become two hands.

That is why `handsTotal` is usually bigger than `rounds`.

### 6. Read `evByBucket` and `count_histogram`

This is where a counter learns the most.

`count_histogram` tells you how often each count happened.

`evByBucket` tells you how profitable each count was.

Example bucket:

```json
{
  "bucket": 0,
  "rounds": 250979402,
  "handsTotal": 257963556,
  "evPerHandUnits": -0.006151,
  "avgBetUnits": 1
}
```

Plain English:

- at TC 0, the sim played about 251 million rounds
- the average bet there was 1 unit
- EV at that bucket was slightly negative

That is normal. Counters usually lose at neutral or bad counts and make their money at the better counts.

This section answers the practical question:

"Where did the edge actually come from?"

### 7. Check `checks`, `warnings`, `downgrades`, and `ignored_settings`

These are the "anything weird happened?" sections.

What to hope for:

- checks say `PASS`
- warnings are empty or harmless
- downgrades are empty
- ignored settings are empty

If something here looks off, slow down before trusting the result.

### 8. Treat the rest as deeper audit material

A BJW results file may also include:

- `streak_statistics`
- `round_level`
- `decision_count_diagnostics`
- `insurance_buckets`
- `actions_by_state_upcard`
- `meta`
- `parallelExecution`
- `result`

These are useful, but they are not the first place a blackjack player should look.

## A Good 30-Second Results Check

When you open a results file, answer these in order:

1. Is this the right game and counting system?
2. How many rounds were run?
3. Is `evPerHandUnits` positive or negative?
4. What is `evPerInitialBetPct`?
5. What is the average bet?
6. Did the checks pass?
7. Do the count buckets make sense?

If you can answer those seven questions, you understand the file well enough for most practical purposes.

## Next Step

If you want to understand why the run was configured that way in the first place, read the matching [manifest file guide](read-manifest-json.md).
