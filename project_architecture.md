# Project Architecture

A technical breakdown of the LLM sentiment validation pipeline: its computational stages, the controls that prevent look-ahead bias and double-counting, the scoring mathematics, and the full statistical results with their uncertainty.

---

## 1. System overview

The pipeline is an **offline evaluation harness**. It consumes two inputs — a log of LLM sentiment emissions and a store of high-resolution price candles — and produces a scored, audited dataset plus a markdown report. It performs no inference and no trading; it grades inference that already happened.

```
                          OFFLINE VALIDATION HARNESS
 ┌──────────────────────────────────────────────────────────────────────┐
 │                                                                        │
 │  sentinel.log                                                          │
 │      │  (1) PARSE                                                      │
 │      ▼                                                                 │
 │  signals.csv ── real judgments vs. parse-failure fallbacks split      │
 │      │  (2) BASE BUILD                                                 │
 │      ▼                                                                 │
 │  base table  ── (ticker, t0, sentiment, conviction, status)           │
 │      │         + empty forward-return columns                          │
 │      │  (3) TEMPORAL JOIN  ◀──── 1-min candle store (per ticker-date)  │
 │      ▼                                                                 │
 │  priced table ── entry price + T+3/15/60m + 15:00 returns             │
 │      │  (4) DIRECTION-SIGNED SCORING                                   │
 │      ▼                                                                 │
 │  scored table ── 0–10 score, NaN for fallbacks                        │
 │      │  (5) STATISTICAL AUDIT + DEDUP                                  │
 │      ▼                                                                 │
 │  report.md  ── precision, decay curve, CIs, asymmetry                 │
 └──────────────────────────────────────────────────────────────────────┘
```

---

## 2. Stage detail

### Stage 1 — Parsing unstructured logs

The raw emission log mixes several line formats: rich JSON verdicts, a legacy simple format, and multiple infrastructure-failure types (rate-limit blocks, model-unavailable, unparseable-output fallbacks). A naive regex keyed to one format silently discards the rest.

The parser classifies every line and, critically, distinguishes a **real model judgment** from a **parse-failure fallback** that merely defaulted to NEUTRAL. This distinction is load-bearing: roughly half of all NEUTRAL-looking lines were fallbacks, not the model's opinion. Treating them as judgments would corrupt every downstream statistic.

| Emission class | Treatment |
|---|---|
| Rich JSON verdict with valid reasoning | `is_real_judgment = True` |
| `reasoning = parsing_failed` | `is_real_judgment = False` (excluded from scoring) |
| Legacy simple-format line | `is_real_judgment = False` |
| Rate-limit / model-unavailable / overloaded | logged as infrastructure failure, not a signal |

### Stage 2 — Base table

Each real or fallback emission becomes one row with identity columns populated and forward-return/score columns left null. A manifest of distinct `(ticker, trade_date)` pairs is emitted to drive candle retrieval — keyed by date so a signal's forward window is matched only against candles from its own session.

### Stage 3 — Temporal indexing (look-ahead control)

For each emission at `t0`:

- **Entry price** = the close of the candle at-or-before `t0`, accepted only if within a 2-minute tolerance; otherwise the nearest candle within tolerance. The lookup never reads a candle *after* `t0` to set the entry. This is the primary look-ahead guard.
- **Forward prices** at `t0 + 3m`, `t0 + 15m`, `t0 + 60m`, and the fixed `15:00:00` candle are read forward in time. These are *outcomes* and are permitted to post-date `t0`; they are never fed back into entry determination.
- Candles are loaded per `(ticker, date)` so a forward window can never bleed across trading days.

Because entry uses only `t ≤ t0` data and outcomes use only `t > t0` data, the join is causally clean: no future information informs the entry, and no entry information is contaminated by the outcome it is being scored against.

### Stage 4 — Direction-signed scoring (0–10)

Raw percentage change is meaningless without direction: a BEARISH call that falls 0.4% is *correct*. Each forward return is therefore signed by the model's stated direction:

```
directional_return = raw_pct_change × (+1 if BULLISH, −1 if BEARISH)
```

A positive directional return means the model was right. Per-horizon credit is weighted toward the windows that matter most for an intraday holding period, with partial credit scaling to a +1.0% cap:

| Horizon | Weight |
|---|---|
| T+3m | 1.5 |
| T+15m | 3.5 |
| T+60m | 2.0 |
| 15:00 square-off | 3.0 |

- **BULLISH / BEARISH** rows are scored on signed directional return across the four horizons.
- **Genuine NEUTRAL** rows are scored for *flatness*: full credit if the asset stays within ±0.3% across horizons, scaled down as it breaches. Neutral is graded as a "no-edge" prediction, not forced into a directional hit/miss frame.
- **Fallback** rows (`is_real_judgment = False`) receive `score = NaN` and are excluded from all model-validation aggregates.

### Stage 5 — Deduplication and statistical audit (double-counting control)

The emission log re-states a held verdict on every refresh cycle. Scoring all emissions measures cache dwell time, not inference quality, and inflates whichever sentiment was held longest. The audit therefore reduces to **one row per `(ticker, date, sentiment)`, keeping the first emission**, before computing accuracy. Both views are reported so the artifact is visible rather than hidden.

---

## 3. Full statistical results

### 3.1 Volume reconciliation

| Quantity | Count |
|---|---|
| Raw emissions | 1,926 |
| Real model judgments | 990 |
| Parse-failure / legacy fallbacks | 936 |
| Infrastructure failures (rate-limit / unavailable / parse / overload) | 583 |
| Validated (real + price data) | 531 |
| Directional, un-deduped | 443 |
| Directional, deduped first-emission | 75 |

### 3.2 Un-deduped per-horizon (artifact-laden, shown for contrast)

| Horizon | Hit-rate | Mean directional return |
|---|---|---|
| T+3m | 43.8% | −0.037% |
| T+15m | 46.3% | −0.093% |
| T+60m | 35.7% | −0.338% |
| 15:00 | 35.2% | −0.518% |

The downward drift here is dominated by stale-verdict re-emissions accumulating against decaying positions.

### 3.3 Deduped per-horizon (the defensible view) with confidence intervals

| Horizon | Hit-rate | 95% CI | Sample |
|---|---|---|---|
| T+3m | 54.7% | [43.4%, 65.9%] | 75 |
| T+15m | 50.7% | [39.4%, 62.0%] | 75 |
| T+60m | 45.8% | [34.3%, 57.3%] | 72 |
| 15:00 | 40.0% | [28.9%, 51.1%] | 75 |

Every interval includes 50%. **No horizon shows a statistically significant directional edge** at this sample size. T+3m mean directional return is +0.011% (t ≈ 0.18 vs. zero), below realistic intraday round-trip cost (~0.05–0.12%).

### 3.4 Directional asymmetry (deduped, T+3m)

| Direction | Hit-rate | 95% CI | Sample |
|---|---|---|---|
| BEARISH | 60.7% | [42.6%, 78.8%] | 28 |
| BULLISH | 51.1% | [36.8%, 65.4%] | 47 |

The bearish lead is the most persistent pattern across all cuts, though still not significant at N = 28. Interpreted conservatively as support for a *risk-off veto* use, not a tradable short signal.

### 3.5 Time-of-day score distribution (all-emission scores)

These per-hour means are computed on all validated emissions (not the deduped set) and reflect score, not directional hit-rate:

| Hour (IST) | Mean score | Signals |
|---|---|---|
| 09:00 | 3.03 | 50 |
| 10:00 | 2.58 | 119 |
| 11:00 | 1.96 | 110 |
| 12:00 | 2.85 | 87 |
| 13:00 | 2.63 | 56 |
| 14:00 | 3.26 | 99 |
| 15:00 | 3.68 | 10 |

Scores trough during the 11:00 midday session and recover into the afternoon. The 15:00 bucket (N = 10) is too small to weight heavily.

---

## 4. Threats to validity

- **Small samples.** The deduped directional set is N = 75; per-direction and per-hour cuts are smaller still. All point estimates carry wide intervals.
- **Lagging input.** Signals were derived from picked-over real-time news rather than primary disclosures, which plausibly caps achievable edge regardless of model quality.
- **Single regime.** One multi-week run does not span varied market regimes; results may not generalize.
- **Survivorship in the universe.** Only assets the upstream scanner surfaced are present; assets never surfaced cannot be evaluated, so the study measures the model conditional on the scanner's coverage.

---

## 5. Transferable methodology

The defensible contributions, independent of the financial domain:

1. Separating real LLM judgments from parse-failure fallbacks before any statistic is computed.
2. Causally clean temporal joins (entry from `t ≤ t0`, outcome from `t > t0`).
3. First-emission deduplication to prevent cache-dwell double-counting.
4. Direction-signed scoring with horizon weighting.
5. Reporting confidence intervals and refusing to act on transaction-cost-negative point estimates.
