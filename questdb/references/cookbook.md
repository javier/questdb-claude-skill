# QuestDB Cookbook Recipes

**IMPORTANT:** The most common recipes (OHLC, VWAP, Bollinger Bands, RSI) are
already provided as complete, ready-to-use SQL in the main SKILL.md. **Do NOT
fetch these from the cookbook — use the inline versions.**

Only fetch cookbook recipes for patterns NOT already covered in SKILL.md:

```bash
curl -sH "Accept: text/markdown" "https://questdb.com/docs/{path}/"
```

## Capital Markets Recipes

| Recipe | Fetch path | Use for |
|---|---|---|
| OHLC bars | `cookbook/sql/finance/ohlc` | Candlestick data from tick trades |
| VWAP | `cookbook/sql/finance/vwap` | Volume-weighted average price (cumulative window) |
| Bollinger Bands | `cookbook/sql/finance/bollinger-bands` | 20-period SMA ± 2σ volatility bands |
| Bollinger BandWidth | `cookbook/sql/finance/bollinger-bandwidth` | Squeeze detection for breakout signals |
| Rolling Std Dev | `cookbook/sql/finance/rolling-stddev` | Expanding/rolling standard deviation |
| Volume Profile | `cookbook/sql/finance/volume-profile` | Volume distribution across price bins |
| Volume Spike Detection | `cookbook/sql/finance/volume-spike` | Current vs previous volume via LAG |
| Aggressor Volume Imbalance | `cookbook/sql/finance/aggressor-volume-imbalance` | Buy vs sell order flow |
| Liquidity Comparison | `cookbook/sql/finance/liquidity-comparison` | Effective spreads via L2Price across instruments |
| TICK and TRIN | `cookbook/sql/finance/tick-trin` | Market breadth indicators (ARMS Index) |
| Cumulative Product | `cookbook/sql/finance/cumulative-product` | Random walk / stock price simulation |
| Compound Interest | `cookbook/sql/finance/compound-interest` | Compound interest with POWER + windows |

## Time-Series Pattern Recipes

| Recipe | Fetch path | Use for |
|---|---|---|
| Force designated timestamp | `cookbook/sql/time-series/force-designated-timestamp` | Restore designated timestamp after UNION, CAST, or joins |
| Latest N per partition | `cookbook/sql/time-series/latest-n-per-partition` | Top N recent rows per group via window functions |
| Session windows | `cookbook/sql/time-series/session-windows` | Detect sessions and elapsed time between events |
| Latest activity window | `cookbook/sql/time-series/latest-activity-window` | Last N minutes of activity via subquery + LIMIT -1 |
| Filter by week number | `cookbook/sql/time-series/filter-by-week` | ISO week filtering via week_of_year() |
| Distribute discrete values | `cookbook/sql/time-series/distribute-discrete-values` | Spread cumulative measurements across intervals |
| Epoch timestamps | `cookbook/sql/time-series/epoch-timestamps` | Query with epoch values (microseconds default) |
| Right interval bound | `cookbook/sql/time-series/sample-by-interval-bounds` | Shift SAMPLE BY to right-aligned timestamps |
| Remove outliers | `cookbook/sql/time-series/remove-outliers` | Filter OHLC outliers via moving average comparison |
| Fill from one column | `cookbook/sql/time-series/fill-from-one-column` | Propagate values across SAMPLE BY columns |
| Fill with historical values | `cookbook/sql/time-series/fill-prev-with-history` | FILL(PREV) with filler row for history carryover |
| Fill keyed arbitrary interval | `cookbook/sql/time-series/fill-keyed-arbitrary-interval` | FILL with keyed SAMPLE BY across custom date ranges |
| Sparse sensor join strategies | `cookbook/sql/time-series/sparse-sensor-data` | Compare CROSS/LEFT/ASOF JOIN for sparse data |

## Advanced SQL Recipes

| Recipe | Fetch path | Use for |
|---|---|---|
| Rows before/after current | `cookbook/sql/advanced/rows-before-after-value-match` | LAG/LEAD to access surrounding rows |
| Local min/max | `cookbook/sql/advanced/local-min-max` | Min/max within a time range around each row |
| Top N plus others | `cookbook/sql/advanced/top-n-plus-others` | Top N + aggregated "Others" row via rank() |
| Pivot with others | `cookbook/sql/advanced/pivot-with-others` | Pivot specific values + "Others" column |
| Unpivot | `cookbook/sql/advanced/unpivot-table` | Wide to long format via UNION ALL |
| Sankey / funnel diagrams | `cookbook/sql/advanced/sankey-funnel` | Session analytics for conversion funnels |
| Conditional aggregates | `cookbook/sql/advanced/conditional-aggregates` | Multiple CASE-based aggregates in one query |
| General + sampled aggregates | `cookbook/sql/advanced/general-and-sampled-aggregates` | Overall stats + time-bucketed via CROSS JOIN |
| Consistent histogram buckets | `cookbook/sql/advanced/consistent-histogram-buckets` | Fixed-boundary histograms for distributions |
| Arrays from string literals | `cookbook/sql/advanced/array-from-string` | Cast strings to array types |

## Grafana Integration Recipes

| Recipe | Fetch path | Use for |
|---|---|---|
| Dynamic table queries | `cookbook/integrations/grafana/dynamic-table-queries` | Query multiple tables via Grafana variables |
| Read-only user | `cookbook/integrations/grafana/read-only-user` | Secure Grafana with read-only PG user |
| Variable dropdown | `cookbook/integrations/grafana/variable-dropdown` | Dropdown with display name ≠ query value |
| Overlay time shift | `cookbook/integrations/grafana/overlay-timeshift` | Yesterday vs today on same chart |

## Demo Data Schema

QuestDB's live demo instance at demo.questdb.io has FX and crypto datasets:

Fetch path: `cookbook/demo-data-schema`

## Concepts Reference

For deeper understanding, fetch these pages:

| Topic | Fetch path |
|---|---|
| Designated timestamp | `concepts/designated-timestamp` |
| Materialized views | `concepts/materialized-views` |
| SYMBOL type | `concepts/symbol` |
| Deduplication | `concepts/deduplication` |
| Time partitions | `concepts/partitions` |
| Write-ahead log (WAL) | `concepts/write-ahead-log` |
| TTL (data retention) | `concepts/ttl` |
| SQL extensions overview | `concepts/deep-dive/sql-extensions` |
| SQL execution order | `query/sql-execution-order` |
| Schema design essentials | `schema-design-essentials` |
