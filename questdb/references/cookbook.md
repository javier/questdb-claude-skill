# QuestDB Cookbook Recipe Index

Pointers to official SQL recipes in the QuestDB docs. **Every recipe is fetchable
as plain markdown** by appending `.md` to its URL — no HTML, no scraping.

```bash
# Fetch any recipe
curl -sH "Accept: text/markdown" "https://questdb.com/docs/cookbook/sql/finance/markout.md"

# Fetch the finance cookbook index page
curl -sH "Accept: text/markdown" "https://questdb.com/docs/cookbook/sql/finance.md"

# Full docs index (every page with its .md URL)
curl -s "https://questdb.com/docs/llms.txt"
```

Use `llms.txt` to discover any page. Every regular doc URL has a `.md` twin at
the same path — this is the authoritative, LLM-friendly way to read QuestDB docs.

---

## Finance / Capital Markets Recipes

Index: https://questdb.com/docs/cookbook/sql/finance.md

All paths below are relative to `https://questdb.com/docs/` — e.g. the full URL
for markout is `https://questdb.com/docs/cookbook/sql/finance/markout.md`.

### Price-Based Indicators
| Recipe | Path | Description |
|--------|------|-------------|
| OHLC Aggregation | `cookbook/sql/finance/ohlc.md` | Aggregate tick data into candlestick bars |
| VWAP | `cookbook/sql/finance/vwap.md` | Volume-Weighted Average Price |
| TWAP | `cookbook/sql/finance/twap.md` | Time-Weighted Average Price |
| Bollinger Bands | `cookbook/sql/finance/bollinger-bands.md` | Price channels based on standard deviation |
| Bollinger BandWidth | `cookbook/sql/finance/bollinger-bandwidth.md` | Measure band expansion and contraction |

### Momentum Indicators
| Recipe | Path | Description |
|--------|------|-------------|
| RSI | `cookbook/sql/finance/rsi.md` | Relative Strength Index for overbought/oversold |
| MACD | `cookbook/sql/finance/macd.md` | Moving Average Convergence Divergence |
| Stochastic Oscillator | `cookbook/sql/finance/stochastic.md` | Close vs. price range |
| Rate of Change | `cookbook/sql/finance/rate-of-change.md` | Percentage price change over N periods |

### Volatility Indicators
| Recipe | Path | Description |
|--------|------|-------------|
| ATR | `cookbook/sql/finance/atr.md` | Average True Range |
| Rolling Std Dev | `cookbook/sql/finance/rolling-stddev.md` | Moving stddev of returns |
| Donchian Channels | `cookbook/sql/finance/donchian-channels.md` | High/low price channels |
| Keltner Channels | `cookbook/sql/finance/keltner-channels.md` | EMA-based volatility channels |
| Realized Volatility | `cookbook/sql/finance/realized-volatility.md` | Historical volatility from returns |

### Volume & Order Flow
| Recipe | Path | Description |
|--------|------|-------------|
| OBV | `cookbook/sql/finance/obv.md` | On-Balance Volume |
| Volume Profile | `cookbook/sql/finance/volume-profile.md` | Volume distribution by price level |
| Volume Spike | `cookbook/sql/finance/volume-spike.md` | Detect abnormal volume |
| Aggressor Imbalance | `cookbook/sql/finance/aggressor-volume-imbalance.md` | Buy vs sell pressure |
| VPIN | `cookbook/sql/finance/vpin.md` | Volume-synchronized informed trading probability |

### Risk Metrics
| Recipe | Path | Description |
|--------|------|-------------|
| Maximum Drawdown | `cookbook/sql/finance/maximum-drawdown.md` | Peak-to-trough decline |

### Market Microstructure
| Recipe | Path | Description |
|--------|------|-------------|
| Bid-Ask Spread | `cookbook/sql/finance/bid-ask-spread.md` | Spread metrics |
| Gamma Scalping Signal | `cookbook/sql/finance/gamma-scalping-signal.md` | Vol-spread ratio |
| Liquidity Comparison | `cookbook/sql/finance/liquidity-comparison.md` | Compare liquidity across instruments |

### Post-Trade Analysis (Execution Quality / TCA)

**These recipes all use `HORIZON JOIN` — the idiomatic QuestDB pattern for
analysing trade fills against quotes at one or many time offsets. Use them
(or the patterns in them) rather than hand-rolling ASOF JOIN + self-joins.**

| Recipe | Path | Description |
|--------|------|-------------|
| Slippage per fill | `cookbook/sql/finance/slippage.md` | Measure execution slippage per fill |
| Slippage (aggregated) | `cookbook/sql/finance/slippage-aggregated.md` | Compare slippage across venues/counterparties |
| Markout analysis | `cookbook/sql/finance/markout.md` | Post-trade price reversion and adverse selection |
| Last-look detection | `cookbook/sql/finance/last-look.md` | Millisecond-granularity markout for last-look |
| Implementation shortfall | `cookbook/sql/finance/implementation-shortfall.md` | Cost decomposition into effective spread, permanent, and temporary impact (HORIZON JOIN + PIVOT) |
| Implementation shortfall (order) | `cookbook/sql/finance/implementation-shortfall-order.md` | Total IS per order vs arrival mid |
| ECN scorecard | `cookbook/sql/finance/ecn-scorecard.md` | Dashboard-style venue comparison |

### Market Breadth
| Recipe | Path | Description |
|--------|------|-------------|
| TICK & TRIN | `cookbook/sql/finance/tick-trin.md` | Market breadth indicators |

### Math Utilities
| Recipe | Path | Description |
|--------|------|-------------|
| Compound Interest | `cookbook/sql/finance/compound-interest.md` | Interest and growth calculations |
| Cumulative Product | `cookbook/sql/finance/cumulative-product.md` | Running product for returns |
| Log Returns | `cookbook/sql/finance/log-returns.md` | Log returns from consecutive prices |

---

## Time-Series Patterns

Common time-series operations that LLMs frequently get wrong because they reach
for PostgreSQL patterns that don't exist in QuestDB, or miss QuestDB-native
features like FILL, LATEST ON, and TICK.

| Recipe | Path | Description |
|--------|------|-------------|
| Elapsed time between rows | `cookbook/sql/time-series/elapsed-time.md` | `lag()` + `datediff()` pattern |
| Force designated timestamp | `cookbook/sql/time-series/force-designated-timestamp.md` | Explicit `TIMESTAMP(col)` in queries |
| Latest N per partition | `cookbook/sql/time-series/latest-n-per-partition.md` | Window functions for top-N per group |
| Session windows | `cookbook/sql/time-series/session-windows.md` | Detect state changes, compute elapsed time |
| Last N minutes of activity | `cookbook/sql/time-series/latest-activity-window.md` | Subquery with `LIMIT -1` |
| Filter by week number | `cookbook/sql/time-series/filter-by-week.md` | `week_of_year()` vs `dateadd()` |
| Distribute values across intervals | `cookbook/sql/time-series/distribute-discrete-values.md` | Spread cumulative measurements |
| Epoch timestamps | `cookbook/sql/time-series/epoch-timestamps.md` | Filtering with epoch values |
| Right interval bound (SAMPLE BY) | `cookbook/sql/time-series/sample-by-interval-bounds.md` | Shift bucket timestamps to right edge |
| Remove outliers from candles | `cookbook/sql/time-series/remove-outliers.md` | Window functions vs moving averages |
| FILL from another column | `cookbook/sql/time-series/fill-from-one-column.md` | Propagate values across columns |
| FILL PREV with historical data | `cookbook/sql/time-series/fill-prev-with-history.md` | Carry historical values into filtered ranges |
| FILL on keyed queries | `cookbook/sql/time-series/fill-keyed-arbitrary-interval.md` | Keyed FILL with arbitrary intervals using boundary rows |
| Sparse sensor join strategies | `cookbook/sql/time-series/sparse-sensor-data.md` | CROSS vs LEFT vs ASOF for multi-sensor data |

## Advanced SQL Patterns

| Recipe | Path | Description |
|--------|------|-------------|
| Rows before/after current | `cookbook/sql/advanced/rows-before-after-value-match.md` | LAG/LEAD window functions |
| Local min/max | `cookbook/sql/advanced/local-min-max.md` | Min/max within a time range around each row |
| Top N + others | `cookbook/sql/advanced/top-n-plus-others.md` | `rank()` + CASE for grouped results |
| Pivot with "Others" | `cookbook/sql/advanced/pivot-with-others.md` | CASE-based pivot with catch-all column |
| Unpivoting | `cookbook/sql/advanced/unpivot-table.md` | Wide to long format via UNION ALL |
| Conditional aggregates | `cookbook/sql/advanced/conditional-aggregates.md` | Multiple CASE-based aggregates in one query |
| General + sampled aggregates | `cookbook/sql/advanced/general-and-sampled-aggregates.md` | CROSS JOIN for overall + time-bucketed stats |
| Histogram buckets | `cookbook/sql/advanced/consistent-histogram-buckets.md` | Fixed-boundary distribution analysis |
| Arrays from string literals | `cookbook/sql/advanced/array-from-string.md` | Cast strings to array types |

---

## Demo Dataset Reference

The demo instance at `https://demo.questdb.io/` exposes the tables used throughout
the finance cookbook — fetch the schema reference before writing queries against it:

```bash
curl -sH "Accept: text/markdown" "https://questdb.com/docs/cookbook/demo-data-schema.md"
```

Key tables:
- `fx_trades` — FX trade executions (timestamp, symbol, ecn, side, price, quantity, counterparty, trade_id, order_id)
- `market_data` — consolidated best bid/ask (timestamp, symbol, best_bid, best_ask)
- `core_price` — ECN-level quotes (timestamp, symbol, ecn, bid_price, ask_price)
- `bbo_1s`, `bbo_1m`, `bbo_1h`, `bbo_1d` — BBO snapshot rollups
- `trades` — crypto trade data (separate from `fx_trades`)

---

## Related Reference Docs

Beyond the cookbook, the query-language docs also have `.md` twins. Fetch these
when you need exact syntax for a keyword:

- HORIZON JOIN — `query/sql/horizon-join.md`
- WINDOW JOIN — `query/sql/window-join.md`
- LATERAL JOIN — `query/sql/lateral-join.md`
- ASOF JOIN — `query/sql/asof-join.md`
- PIVOT — `query/sql/pivot.md`
- UNNEST — `query/sql/unnest.md`
- SAMPLE BY — `query/sql/sample-by.md`
- LATEST ON — `query/sql/latest-on.md`
