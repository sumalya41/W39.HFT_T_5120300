# Signal from Liquidations

Statistical analysis of Binance and Bybit perpetual-futures data (BTCUSDT, ETHUSDT): data quality, maker markouts, liquidation cascades and a turnover-share rule, built as an out-of-core pipeline that runs inside Kaggle memory limits.

---

## 1. Overview

This notebook takes about 5.8 GB of raw tick data (trades, best bid/offer, liquidations) covering roughly three months from 2025-12-01 and answers four questions:

| # | Question | Where it is answered |
|---|----------|----------------------|
| Q1 | Are NaNs, empty books, time gaps and anomalies a market feature or a data-quality problem? | Data audit (part B) |
| Q2 | What are the maker markouts at 30 s, 120 s and 300 s (average, total, distribution)? | Markout analysis (part E) |
| Q3 | Are there liquidations and liquidation cascades, and how can they be detected? | Liquidations and cascades (parts D, Q) |
| Q4 | What minimum share of turnover should be kept so that clipped turnover is at least 500,000 USD per day? | Turnover constraint (part F) |

Extra analysis: simple-return time-series statistics, tail risk, copulas, distribution fitting and a check on whether the markout strategy can be improved.

---

## 2. Data

Expected input (Kaggle dataset):

```
/kaggle/input/datasets/sumalya32/kft-mrkdwn/liquidation_task/data
├── binance_booktickers/
├── binance_liquidations/
├── binance_trades/
├── bybit_liquidations/
└── description.md
```

- Timestamps are int64 microseconds.
- Bybit liquidation timestamps are shifted by +200 ms to align with Binance.
- Trade and liquidation `side` is the taker side (trades) or the side of the forced order (liquidations).
- The notebook looks for the data at the path above, then at `/kaggle/input/kft-mrkdwn/...`, then by recursive search, and finally by extracting the tar archive if needed.

---

## 3. Task definition

Maker PnL per trade, in basis points:

```
pnl_bps = -s * (m(tau) - p) / p * 1e4 + 0.5
```

- `s` = side, `p` = trade price, `m(tau)` = forward-filled BBO mid at `t + tau`.
- Trades whose `t + tau` falls outside the BBO range are excluded.
- Weight `w = min(notional, 100000)`.
- Score = PnL(kept) − PnL(all).
- Splits: train 2025-12-01 to 2026-01-31, validation February 2026, hidden test.

---

## 4. Method

**Memory-safe design**
- DuckDB splits the raw parquet files into per-day shards. The original row order is kept in a `frn` column.
- Only one UTC day is held in RAM at a time.
- Results are checkpointed, so a crashed session can resume.
- Three profiles via `LIQ_PROFILE`: `fast`, `balanced` (default), `lowmem`.
- Stage timing and peak RSS are logged to `runtime_profile.csv`.

**Statistics (simple returns `P_t / P_{t-h} - 1`, in bps)**
- Returns are computed on a clean 1-second mid grid. 60-second bars are used for distribution, tail and copula work.
- ACF correlogram, ADF and KPSS stationarity tests, variance ratio (Lo–MacKinlay).
- Tail risk: Hill estimator with plateau rule, POT/GPD with declustering, GEV block maxima, UTC-hour and weekday/weekend blocks.
- Copulas (Gaussian, Student-t, Clayton, survival Clayton, Gumbel) selected by AIC, plus empirical tail dependence and quintile lift tables.
- Distribution fitting, out of sample: Normal, Laplace, Logistic, Student-t, generalised normal, Johnson SU, NIG, Cauchy and a two-component normal mixture. Compared by BIC, held-out log-likelihood, KS and Anderson–Darling distance, tail-quantile error, Kupiec test and PIT.

**Liquidation cascades (Q3)**
- Time-clustering detection, with sensitivity to the cascade definition.
- Fano factor against an hour-preserving Poisson null.
- Cross-venue lead (Binance vs Bybit).
- Event study against hour-matched controls.
- Online detectors (Hawkes intensity ratio, event-count rule) reported with false alarms per day and detection lag.

**Strategy improvement**
- Causal features: liquidation state, signed past return, flow imbalance, size bucket.
- Cells are ranked on fit days by a lower confidence bound.
- The smallest cell set that keeps at least 600,000 USD on every fit day is chosen (500k with a 20% safety margin).
- Judged on separate days with a bootstrap CI and a positive-day share, against a drop-burst baseline and a look-ahead oracle.

---

## 5. Outputs

All outputs are written to `/kaggle/working`:

| File | Content |
|------|---------|
| `report.md` | Full written report with statistics and figure references |
| `signal_from_liquidations.pptx` | Slide deck with the main results and graphs |
| `figs/` | All figures (PNG) |
| `*.csv` | Summary tables (markouts, thresholds, cascades, fits) |
| `runtime_profile.csv` | Time and peak memory per stage |

---

## 6. How to run

1. Add the dataset `kft-mrkdwn` to the notebook as input.
2. Optional: set the profile before running, e.g. `LIQ_PROFILE=lowmem`.
3. Choose **Run All**.
4. Collect results from `/kaggle/working`.

Requirements: Python 3, `duckdb`, `pandas`, `numpy`, `scipy`, `statsmodels`, `matplotlib`, `python-pptx` (installed automatically if missing).

---

## 7. Key results

> Fill in after the final run (values are in `report.md`).

| Item | Result |
|------|--------|
| Data quality verdict (Q1) | |
| Mean markout 30 s / 120 s / 300 s (bps) | |
| Total markout 30 s / 120 s / 300 s | |
| Liquidation cascades found, detection rule | |
| Minimum turnover share for ≥ 500k USD/day (Q4) | |
| Validation score of the improved rule | |

---

## 8. Limitations

- Results depend on the cascade definition and thresholds chosen on the train period.
- Taker-cost assumptions in the strategy tests (`TAKER_FEE_BPS = 4.5`) are simplified.
- The hidden test period is not used for any tuning.

---

## 9. Repository layout

```
hft-liq-5gb.ipynb   main notebook (full pipeline)
outputs/            report.md, deck, figures, summary CSVs
README.md           this file
```
