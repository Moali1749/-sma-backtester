# SMA Momentum Backtester — From Strategy to Research

A two-part quantitative finance project in Python. Part 1 builds an honest backtest of an SMA momentum strategy on Apple stock. Part 2 subjects that result to the tests a quant researcher would apply — risk metrics, robustness checks, and transaction costs — and reaches a more sobering, more interesting conclusion.

**Notebooks:**
- `01_sma_backtester.ipynb` — the strategy and core backtest
- `02_research_extension.ipynb` — risk metrics, robustness grid, transaction costs

---

## Part 1 — The Strategy

A classic trend-following rule on a 20-day simple moving average:

- **Price above SMA-20** → uptrend → hold the stock (position = 1)
- **Price below SMA-20** → downtrend → stay in cash (position = 0)

### Avoiding look-ahead bias

A signal based on today's closing price is only known after the market closes, so it cannot be acted on until the next day. The signal is shifted forward by one day:

```python
signal   = (close > sma).astype(int)
position = signal.shift(1)   # act tomorrow, not today
```

Skipping this shift lets a backtest "trade" on information it wouldn't have had in real time — the most common beginner error, producing impressive but fake returns.

### Initial result (AAPL, 2015–2024, no costs)

| | Buy & Hold | Strategy |
|---|---|---|
| Growth of $1 | ×9.46 | **×12.17** |
| Avg. daily return | 0.109% | 0.106% |

The strategy finished higher despite a *lower* average daily return — because capital compounds multiplicatively, and avoiding deep drawdowns matters more than capturing every gain. At this point the strategy looked like a winner. Part 2 tests whether that holds up.

---

## Part 2 — The Research

### A. Risk metrics

Total return says nothing about the risk taken to earn it. Three standard metrics (AAPL, SMA-20, before costs):

| Metric | Buy & Hold | Strategy |
|---|---|---|
| Annualised volatility | higher | **lower** |
| Sharpe ratio | 0.9 | **1.4** |
| Max drawdown | −38% | **−22%** |

All three favour the strategy, and for the same reason: sitting in cash during major downturns (2018, COVID-2020, 2022) cuts both the variance of returns and the depth of drawdowns. The drawdown curve in the notebook shows this directly — the strategy stays near the surface while buy & hold dives.

### B. Robustness check — is it real or cherry-picked?

One good backtest on one stock with one parameter proves little: with enough combinations, some will look good by pure chance. The strategy was re-run across a 5×5 grid — five tickers (AAPL, MSFT, GOOGL, AMZN, KO) × five SMA windows (10, 20, 50, 100, 200).

**Result: out of 25 combinations, only one — the original AAPL + SMA-20 — achieved a Sharpe above 1.** Most of the grid sits well below; Coca-Cola (a low-trend stock) is weak at every window, and fast windows on several tickers produce *negative* Sharpe ratios.

This is the visual signature of **overfitting**: a single green cell in a red field (see the heatmap in the notebook). The strong original result was a lucky pairing of asset and parameter, not a market regularity.

### C. Transaction costs

The backtest above assumes trading is free. Adding a realistic 0.1% cost per trade:

| AAPL + SMA-20 | No costs | With costs |
|---|---|---|
| Sharpe | 1.40 | 1.33 |
| Growth of $1 | ×12.17 | **×9.90** |
| Number of trades | — | **207** |

207 trades over ten years — the strategy crosses its SMA roughly every 12 trading days. Each crossing costs money, and compounding turns a tiny 0.1% fee into a **19% haircut on final wealth**. After costs, the strategy's edge over buy & hold (×9.90 vs ×9.46) nearly vanishes. Fast SMA windows suffer most: more crossings, more fees, and several grid cells turn negative.

**Turnover matters:** an active strategy must beat the market by more than its trading costs — not just beat it.

---

## Conclusions

1. **The honest backtest still looked too good.** Even with look-ahead bias removed, the initial result (Sharpe 1.4, ×12.17) did not survive scrutiny.
2. **The edge was not robust.** Across 25 ticker/window combinations, only the original pair cleared Sharpe 1 — a classic overfitting pattern.
3. **Costs consumed most of what remained.** 207 trades at 0.1% each erased ~19% of final wealth and nearly all outperformance over buy & hold.
4. **The realistic takeaway:** a simple SMA momentum rule does not provide a reliable edge after robustness checks and costs. This is consistent with market efficiency — if such a simple rule worked reliably, it would be arbitraged away.

The project's real result is not a profitable strategy but a working research process: build honestly, measure risk (not just return), test for robustness before believing anything, and account for costs.

## Tech Stack

- **Python** — yfinance (data), pandas (time series, rolling windows), numpy (metrics), matplotlib & seaborn (equity curves, drawdown curve, heatmap)
- Core techniques: vectorised returns (`pct_change`), rolling windows (`rolling().mean()`), signal shifting (`shift`), running maximum for drawdowns (`cummax`), cumulative compounding (`cumprod`), parameter sweep with a reusable backtest function, pivot + heatmap for grid analysis

## Future Work

- Out-of-sample testing: fit parameters on one period, evaluate on another (walk-forward analysis).
- Compare against a mean-reversion variant, which profits in the sideways regimes where momentum bleeds.
- Model costs more realistically: spread, slippage, and their dependence on volatility.
- Extend the asset universe beyond US large-caps to test regime dependence.
