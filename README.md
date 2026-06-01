# Event-Driven LLM Sentiment Validation Pipeline

**An empirical study of real-time large-language-model semantic inference against high-resolution time-series ground truth.**

> This repository is a **documentation and research portfolio**. It contains no proprietary execution logic, no API credentials, and no live trading code. It documents the architecture, data-engineering schema, and statistical findings of a validation pipeline built to answer one question: *can an LLM reading unstructured real-time text emit signals that predict immediate, objectively-measurable forward outcomes?*

---

## 1. Motivation

Large language models are increasingly asked to interpret unstructured event streams — news, filings, alerts — and emit a structured judgment. The hard part is not getting a judgment; it is knowing whether that judgment has any predictive content once measured against objective reality, rather than sounding plausible.

This project treats public equity markets purely as a **high-velocity, high-volatility test harness**: a domain that produces a relentless stream of unstructured text *and* an unforgiving, timestamped ground truth (price) against which any claim can be falsified within minutes. The markets are the laboratory, not the point. The point is LLM evaluation methodology.

The model under test was Google Gemini (Flash class), emitting one of `BULLISH | BEARISH | NEUTRAL` per processed text event, with a self-reported conviction field.

---

## 2. What this pipeline does

```
 ┌─────────────────┐     ┌──────────────────┐     ┌─────────────────────┐
 │  Unstructured    │     │   LLM Inference   │     │  Structured signal   │
 │  text event      ├────▶│   (sentiment +    ├────▶│  + emission          │
 │  stream          │     │    conviction)    │     │  timestamp (t0)      │
 └─────────────────┘     └──────────────────┘     └──────────┬──────────┘
                                                              │
                          ┌───────────────────────────────────┘
                          ▼
 ┌──────────────────────────────┐     ┌──────────────────────────────┐
 │  Temporal indexing:           │     │  Forward-horizon evaluation:   │
 │  match t0 to 1-min candle      ├────▶│  signed returns at             │
 │  close (entry price)           │     │  T+3m / T+15m / T+60m / 15:00  │
 └──────────────────────────────┘     └──────────────┬───────────────┘
                                                       ▼
                                       ┌──────────────────────────────┐
                                       │  Direction-signed scoring      │
                                       │  (0–10) + statistical audit    │
                                       └──────────────────────────────┘
```

Three layers:

1. **Timestamp synchronization** — capture the exact moment a text event is processed and a sentiment vector is emitted.
2. **Dynamic time-series mapping** — cross-reference each emission against 1-minute historical candles to establish the entry price at the moment of emission, using *only* data available at or before `t0` (no look-ahead).
3. **Forward-horizon evaluation** — measure direction-signed returns at an immediate horizon (T+3m), a standard window (T+15m), an extended-lag window (T+60m), and the intraday square-off (15:00 IST).

---

## 3. Headline findings

A full statistical treatment is in [`project_architecture.md`](./project_architecture.md). The condensed result:

| Metric | Value | Note |
|---|---|---|
| Raw text emissions processed | 1,926 | includes parse-failure fallbacks |
| Validated events (real judgment + price) | 531 | |
| Directional signals evaluated | 443 | BULLISH/BEARISH only |
| **Un-deduped 15m directional accuracy** | **46.3%** | *artifact-laden — see below* |
| Events in lowest score bucket (0–2 / 10) | 299 / 531 | |

### The central methodological lesson: an artifact masquerading as a verdict

The un-deduped 46.3% is **not a measurement of the model**. The system re-emitted an identical verdict every refresh cycle while a signal sat in cache, so scoring all 443 rows measured *cache dwell time*, not inference quality. Counting the same judgment many times is double-counting.

Isolating to **first-emissions only** (N = 75 directional signals) changes the picture:

| Horizon | Deduped directional accuracy | Sample |
|---|---|---|
| T+3m | 54.7% | N = 75 |
| T+15m | 50.7% | N = 75 |
| T+60m | 45.8% | N = 72 |
| 15:00 square-off | 40.0% | N = 75 |

### The honest caveat: the edge is not statistically established

The 54.7% T+3m figure carries a 95% confidence interval of **[43.4%, 65.9%]**. That interval straddles 50%. With N = 75 the apparent short-term edge is **statistically indistinguishable from a coin flip**. The mean directional return at T+3m (+0.011%) is below realistic round-trip transaction costs. This repository deliberately reports the negative-leaning result rather than the flattering point estimate — that is the discipline the project exists to demonstrate.

### Two patterns that survived every cut

- **Monotonic alpha decay.** Accuracy falls with hold time (54.7% → 50.7% → 45.8% → 40.0%). Any signal present has a very short shelf life.
- **Asymmetric edge.** The model identifies downside more reliably than upside: BEARISH first-emissions hit 60.7% at T+3m versus 51.1% for BULLISH. This supports using the model as a *risk-off veto*, not a long trigger.

---

## 4. Repository contents

| File | Purpose |
|---|---|
| `README.md` | This overview |
| `project_architecture.md` | Computational logic, look-ahead/double-counting controls, full statistics |
| `.gitignore` | Sanitizes the public tree of credentials, keys, venvs, data, logs |

No `.csv` data, `.pem` keys, `.env` files, or execution scripts are tracked. See `.gitignore`.

---

## 5. Data schema (reference)

Each validated event is represented as one row:

| Column | Type | Description |
|---|---|---|
| `timestamp` | ISO 8601 | Emission moment (t0) |
| `ticker` | string | Target asset symbol |
| `sentiment` | enum | BULLISH / BEARISH / NEUTRAL |
| `conviction` | enum | Model self-reported (HIGH / NEUTRAL) |
| `status` | enum | SUCCESS (real judgment) / FALLBACK (parse-failure default) |
| `is_real_judgment` | bool | Excludes fallbacks from scoring |
| `headline_snippet` | string | Source text excerpt |
| `current_price` | float | Close at t0 |
| `chg_3m / chg_15m / chg_60m / chg_3pm` | float % | Forward returns |
| `score` | float 0–10 | Direction-signed performance score (NaN for fallbacks) |

---

## 6. Reproducibility & scope

The statistics above are derived from a single multi-week validation run. The sample sizes are explicitly small; the findings are framed as **hypotheses with measured uncertainty**, not established alpha. The methodology — first-emission deduplication, direction-signed scoring, confidence-interval reporting, and refusal to act on transaction-cost-negative point estimates — is the transferable contribution.

## License

Documentation released under MIT. No warranty; nothing here is financial advice.
