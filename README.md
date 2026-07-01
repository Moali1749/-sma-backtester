# SMA Momentum Backtester (AAPL)

A backtest of a simple moving-average (SMA) momentum strategy on Apple stock, built from scratch in Python. The goal of this project was not to find a profitable strategy, but to build an **honest** backtest and understand exactly what every line does — including the subtle mistakes that make most beginner backtests misleading.

## The Strategy

A classic trend-following rule based on a 20-day simple moving average:

- **Price above SMA-20** → the stock is trending up → **hold the stock** (position = 1)
- **Price below SMA-20** → the stock is trending down → **stay in cash** (position = 0)

The SMA acts as a direction indicator, not a "fair price." The idea rests on *momentum*: trends tend to persist, so the strategy rides an uptrend and steps aside when it breaks.

## Avoiding Look-Ahead Bias

The most important design decision in the project. A signal based on today's **closing** price is only known *after* the market closes, so it cannot be acted on until the **next** day. The signal is therefore shifted forward by one day:

```python
df["signal"]   = (df["close"] > df["sma20"]).astype(int)
df["position"] = df["signal"].shift(1)   # act on tomorrow, not today
```

Skipping this shift is the single most common beginner error — it lets the backtest "trade" on information it wouldn't have had in real time, producing impressive but fake returns.

## Results (2015–2024)

Growth of $1 invested at the start of 2015:

| Approach | Final value | Avg. daily return |
|---|---|---|
| Buy & Hold (AAPL) | ×9.46 | 0.109% |
| SMA Strategy | ×12.17 | 0.106% |

The interesting part: the strategy's **average daily return was lower**, yet it **finished with more capital**.

## The Key Insight

Capital compounds multiplicatively, not additively — final wealth is the *product* of daily `(1 + r)` factors, not their sum. Large drawdowns hurt this product disproportionately: a 50% loss requires a 100% gain just to recover.

By sitting in cash during major downturns (2018, the 2020 COVID crash, 2022), the strategy gave up some upside but avoided the deepest drops. Because drawdowns damage compounded capital nonlinearly, **avoiding big losses outweighed the missed gains** — which is why a strategy with lower average daily return still won on total growth.

This is the central lesson of the project: *average return is not the same as final outcome, and controlling drawdowns can matter more than maximizing returns.*

## Tech Stack

- **Python**
- **yfinance** — downloading historical price data
- **pandas** — time-series data handling, rolling windows, vectorized returns
- **matplotlib** — visualizing the equity curves

## How It Works (pipeline)

1. **Data** — download adjusted OHLCV data for AAPL via `yfinance` (`auto_adjust=True` to correct for splits and dividends).
2. **Compute** — daily returns via `pct_change()`; 20-day SMA via `rolling(20).mean()`.
3. **Signal** — compare price to SMA, convert to 1/0, shift by one day.
4. **Backtest** — strategy return = `position × return`; build equity curves with `cumprod()`.
5. **Visualize** — plot strategy vs. buy & hold.

## Honest Limitations

This is a single, deliberately simple experiment, and the result should be read as an illustration of a principle, not proof of an edge:

- Tested on **one stock**, **one SMA window (20)**, **one time period** — not robust across assets or parameters.
- **No transaction costs, slippage, or taxes** — real trading would erode returns, especially given frequent entries/exits.
- Trend-following strategies are known to **bleed in sideways markets** (whipsaw), which a single trending stock like AAPL doesn't fully stress-test.

## Future Work

- Test across many tickers and SMA windows to check robustness (avoid cherry-picking).
- Add transaction costs and measure how sensitive the edge is to them.
- Report risk metrics: maximum drawdown, Sharpe ratio, volatility — not just final return.
- Compare against a mean-reversion variant, which profits in the sideways regimes where momentum struggles.
