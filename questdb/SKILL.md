---
name: questdb
description: >
  Use this skill whenever working with QuestDB — a high-performance time-series
  database. Trigger on any mention of QuestDB, time-series SQL with SAMPLE BY,
  LATEST ON, ASOF JOIN, ILP ingestion, or the questdb Python/Go/Java/Rust/.NET
  client libraries. Also trigger when writing Grafana queries against QuestDB,
  creating materialized views for time-series rollups, working with order book
  or financial market data in QuestDB, or any SQL that involves designated
  timestamps or time-partitioned tables. QuestDB extends SQL with unique
  time-series keywords — standard PostgreSQL or MySQL patterns will fail.
  Always read this skill before writing QuestDB SQL to avoid hallucinating
  incorrect syntax.
---

# QuestDB Skill

## Critical Rule

**QuestDB is NOT PostgreSQL.** It supports the PostgreSQL wire protocol for
querying, but has its own SQL extensions for time-series operations. Standard
PostgreSQL patterns like `time_bucket()`, `DISTINCT ON`, `HAVING`, and
`generate_series()` do not exist. Read this skill before writing any QuestDB SQL.

## Live Documentation Access

QuestDB docs are available as clean markdown. Use this for any syntax you're
unsure about:

```bash
curl -sH "Accept: text/markdown" "https://questdb.com/docs/{page}/"
```

Full table of contents (use as a lookup index):
```bash
curl -s "https://questdb.com/docs/llms.txt"
```

Every page listed in llms.txt returns markdown when requested with
`Accept: text/markdown`.

**Performance rule:** This skill contains inline SQL for the most common patterns
(OHLC, VWAP, Bollinger Bands, RSI, materialized views, Grafana queries). Use
them directly — do NOT fetch remote docs for patterns already covered here.
Only fetch docs for advanced or unusual queries not in this skill.

---

## SQL Reference

### CREATE TABLE

```sql
CREATE TABLE trades (
    ts TIMESTAMP,
    symbol SYMBOL,
    side SYMBOL,
    price DOUBLE,
    amount DOUBLE
) TIMESTAMP(ts) PARTITION BY DAY WAL
DEDUP UPSERT KEYS(ts, symbol);
```

- `TIMESTAMP(ts)` — designates the time column. Required for SAMPLE BY, LATEST ON, ASOF JOIN. Data is physically sorted by this column.
- `PARTITION BY DAY WAL` — time partitioning + write-ahead log. Always use WAL for concurrent writes.
- `SYMBOL` — optimized type for low-cardinality repeated strings (tickers, sides, exchanges, status codes). Use instead of VARCHAR for repeated values.
- `DEDUP UPSERT KEYS(ts, col1, ...)` — deduplication on ingest.
- Partition options: `HOUR`, `DAY`, `WEEK`, `MONTH`, `YEAR`

Arrays (e.g. for order book data) — prefer 2D arrays:
```sql
CREATE TABLE order_book (
    ts TIMESTAMP,
    symbol SYMBOL,
    bids DOUBLE[][],   -- [1]=prices, [2]=sizes
    asks DOUBLE[][]    -- [1]=prices, [2]=sizes
) TIMESTAMP(ts) PARTITION BY DAY WAL
DEDUP UPSERT KEYS(ts, symbol);
```

**QuestDB arrays are 1-indexed**, not 0-indexed:
- `bids[1]` = prices array, `bids[2]` = sizes array
- `bids[1][1]` = best bid price

Full reference: fetch `query/sql/create-table`
Data types reference: fetch `query/datatypes/overview`
Array reference: fetch `query/datatypes/array`

### SAMPLE BY (Time-Series Aggregation)

QuestDB's core time-series feature. Replaces PostgreSQL's `GROUP BY time_bucket()` or `date_trunc()`.

```sql
SELECT
    ts, symbol,
    first(price) AS open,
    max(price) AS high,
    min(price) AS low,
    last(price) AS close,
    sum(amount) AS volume
FROM trades
WHERE ts IN '$now - 1h..$now' AND symbol = 'BTC-USDT'
SAMPLE BY 5s;
```

Rules:
- Goes AFTER WHERE, before ORDER BY
- Units: `s` (seconds), `m` (minutes), `h` (hours), `d` (days), `M` (months), `y` (years)
- `ALIGN TO CALENDAR` aligns buckets to clock boundaries (this is the default, so it can be omitted)
- Table MUST have a designated timestamp
- Gap filling: `SAMPLE BY 1m FILL(PREV)` — options: `PREV`, `NULL`, `LINEAR`, `value`
- FILL goes immediately after SAMPLE BY

Full reference: fetch `query/sql/sample-by`
FILL reference: fetch `query/sql/fill`

### LATEST ON (Last Value Per Group)

Returns the most recent row for each distinct partition value:

```sql
SELECT * FROM trades
LATEST ON ts PARTITION BY symbol;
```

- Must reference the designated timestamp column
- Much faster than window function alternatives

Full reference: fetch `query/sql/latest-on`

### Joins

**ASOF JOIN** — matches each row to the nearest preceding timestamp:
```sql
SELECT t.ts, t.symbol, t.price, o.bids, o.asks
FROM trades t
ASOF JOIN order_book o ON (symbol);
```
- `ON` matches the equi-join key (typically SYMBOL)
- Optional: `TOLERANCE 5s` to limit lookback window

**WINDOW JOIN** — efficient time-based aggregation across related tables:
```sql
SELECT t.ts, t.symbol, t.price,
       avg(o.mid_price) AS avg_mid
FROM trades t
WINDOW JOIN order_book o ON (symbol)
OVER (ROWS BETWEEN 10 PRECEDING AND CURRENT ROW);
```

**Standard JOINs**: `JOIN`, `LEFT JOIN`, `CROSS JOIN` work as expected.

Full reference: fetch `query/sql/asof-join`, `query/sql/join`, `query/sql/window-join`

### Window Functions

```sql
-- Moving average
avg(price) OVER (PARTITION BY symbol ORDER BY ts
    RANGE BETWEEN 20 PRECEDING AND CURRENT ROW)

-- Native EMA — DO NOT implement manually
avg(price, 'period', 20) OVER (PARTITION BY symbol ORDER BY ts)

-- Native VWEMA (volume-weighted EMA)
avg(price, 'period', 20, volume) OVER (PARTITION BY symbol ORDER BY ts)

-- Cumulative (running total from start)
sum(volume) OVER (ORDER BY ts CUMULATIVE)

-- lag / lead
lag(price) OVER (PARTITION BY symbol ORDER BY ts)
lead(price, 1) OVER (PARTITION BY symbol ORDER BY ts)
```

Arithmetic on window results is supported:
```sql
SELECT price - avg(price) OVER (...) AS deviation FROM trades;
```

**Limitations:**
- Cannot filter window results in WHERE — no QUALIFY support. Wrap in CTE/subquery, filter in outer query.
- Cannot combine window functions with GROUP BY or SAMPLE BY in the same query level. Use a subquery:

```sql
-- ✅ SAMPLE BY in subquery, window functions outside
SELECT *, avg(close, 'period', 20) OVER (PARTITION BY symbol ORDER BY ts) AS ema20
FROM (
    SELECT ts, symbol, first(price) AS open, max(price) AS high,
           min(price) AS low, last(price) AS close, sum(amount) AS volume
    FROM trades
    WHERE ts IN '$today'
    SAMPLE BY 1m
);
```

Full reference: fetch `query/functions/window-functions/overview`, `query/functions/window-functions/reference`, `query/functions/window-functions/syntax`

### Materialized Views

Pre-computed aggregations. Insert into the base table — never into the view directly. Views can cascade (build on each other) for multi-resolution rollups.

**REFRESH IMMEDIATE (default — real-time):**
```sql
CREATE MATERIALIZED VIEW trades_ohlc_1m AS (
    SELECT ts, symbol,
        first(price) AS open, max(price) AS high,
        min(price) AS low, last(price) AS close,
        sum(amount) AS volume
    FROM trades
    SAMPLE BY 1m
) PARTITION BY DAY;
```

**REFRESH EVERY (periodic):**
```sql
CREATE MATERIALIZED VIEW trades_ohlc_1h
REFRESH EVERY 10m AS (
    SELECT ts, symbol, avg(price) AS avg_price
    FROM trades SAMPLE BY 1h
) PARTITION BY DAY;
```

**REFRESH MANUAL:**
```sql
CREATE MATERIALIZED VIEW trades_daily
REFRESH MANUAL AS (
    SELECT ts, symbol, avg(price) AS avg_price
    FROM trades SAMPLE BY 1d
) PARTITION BY MONTH;
-- Trigger with: REFRESH MATERIALIZED VIEW trades_daily;
```

**TTL (data lifecycle):**
```sql
CREATE MATERIALIZED VIEW trades_ohlc_5s AS (
    SELECT ts, symbol, first(price) AS open, max(price) AS high,
           min(price) AS low, last(price) AS close, sum(amount) AS volume
    FROM trades SAMPLE BY 5s
) PARTITION BY DAY TTL 7 DAYS;
```

TTL on views is independent of the base table's TTL. Fine-grained views (5s) can have short TTL while coarser views (1h, 1d) retain longer.

**Cascading views (5s → 1m → 1h):**
```sql
CREATE MATERIALIZED VIEW trades_ohlc_5s AS (
    SELECT ts, symbol, first(price) AS open, max(price) AS high,
           min(price) AS low, last(price) AS close, sum(amount) AS volume
    FROM trades SAMPLE BY 5s
) PARTITION BY DAY TTL 3 DAYS;

CREATE MATERIALIZED VIEW trades_ohlc_1m AS (
    SELECT ts, symbol, first(open) AS open, max(high) AS high,
           min(low) AS low, last(close) AS close, sum(volume) AS volume
    FROM trades_ohlc_5s SAMPLE BY 1m
) PARTITION BY DAY TTL 30 DAYS;

CREATE MATERIALIZED VIEW trades_ohlc_1h AS (
    SELECT ts, symbol, first(open) AS open, max(high) AS high,
           min(low) AS low, last(close) AS close, sum(volume) AS volume
    FROM trades_ohlc_1m SAMPLE BY 1h
) PARTITION BY MONTH TTL 1 YEARS;
```

**WITH BASE — only needed for JOINs.** Tells QuestDB which table triggers refresh:
```sql
CREATE MATERIALIZED VIEW trades_with_metadata
WITH BASE trades AS (
    SELECT t.ts, t.symbol, m.sector, avg(t.price) AS avg_price
    FROM trades t JOIN instruments m ON t.symbol = m.symbol
    SAMPLE BY 1h
) PARTITION BY DAY;
```
Changes to `instruments` will not trigger refresh — only `trades` will.

**PERIOD clause** — for fixed-interval data, prevents refreshing incomplete periods:
```sql
CREATE MATERIALIZED VIEW trades_hourly
REFRESH EVERY 15m PERIOD (LENGTH 1h DELAY 5m) AS (
    SELECT ts, symbol, avg(price) AS avg_price
    FROM trades SAMPLE BY 1h
) PARTITION BY DAY;
```

Full reference: fetch `query/sql/create-mat-view`
Concepts: fetch `concepts/materialized-views`

### Timestamp Filtering — TICK Syntax (Preferred)

TICK (Temporal Interval Calendar Kit) is the preferred way to filter by time. Use instead of `dateadd()` and `BETWEEN` for most cases.

**Syntax order:** `date [T time] @timezone #dayFilter ;duration`

```sql
WHERE ts IN '$now - 1h..$now'                    -- last hour
WHERE ts IN '$now - 30m..$now'                    -- last 30 minutes
WHERE ts IN '$today'                              -- full day
WHERE ts IN '$yesterday'                          -- yesterday
WHERE ts IN '2025-02-09'                          -- specific day
WHERE ts IN '2025-02-01..$today'                  -- date range
WHERE ts IN '$today - 5bd..$today - 1bd'          -- last 5 business days
WHERE ts IN '2025-02-[01..28]T09:30@America/New_York#workday;6h30m'  -- NYSE hours
WHERE ts IN '2025-02-09T[09:00,12:00,18:00];1h'  -- multiple windows
```

**Units:** `y` (years), `M` (months), `w` (weeks), `d` (days), `bd` (business days), `h` (hours), `m` (minutes), `s` (seconds)

**Bracket expansion:** `[a,b,c]` for lists, `[a..b]` for ranges, `[5,10..12,20]` for mixed

**When you still need dateadd():** Computed timestamps from other column values, programmatic offsets in expressions, or combining with function results.

Full TICK reference: fetch `query/operators/tick`

### DECLARE (Variables)

```sql
DECLARE @sym := 'BTC-USDT'
SELECT * FROM trades WHERE symbol = @sym AND ts IN '$today';
```

### No HAVING

QuestDB does NOT support HAVING. Filter aggregation results with a CTE:

```sql
-- ❌ WRONG: HAVING does not exist
SELECT symbol, avg(price) AS avg_price FROM trades
SAMPLE BY 1h HAVING avg_price > 50000;

-- ✅ CORRECT: CTE + WHERE
WITH agg AS (
    SELECT symbol, avg(price) AS avg_price FROM trades
    WHERE ts IN '$today' SAMPLE BY 1h
)
SELECT * FROM agg WHERE avg_price > 50000;
```

### SQL Execution Order

`FROM → WHERE → SAMPLE BY → GROUP BY → SELECT → DISTINCT → ORDER BY → LIMIT`

No HAVING. SAMPLE BY executes before GROUP BY.

### Key Functions

| Function | Purpose |
|---|---|
| `first(col)` | First value in group (open) |
| `last(col)` | Last value in group (close) |
| `sum(col)` | Sum (volume) |
| `avg(col)` | Average |
| `min(col)` / `max(col)` | Low / high |
| `count()` | Row count |
| `l2price(size, sizes[], prices[])` | L2 order book execution price |
| `mid(bid, ask)` | Midpoint price |
| `wmid(bidSz, bidPx, askPx, askSz)` | Weighted mid-price |
| `now()` | Current server timestamp |
| `dateadd('unit', n, ts)` | Add/subtract time |
| `datediff('unit', ts1, ts2)` | Time difference |
| `timestamp_floor('interval', ts)` | Floor to interval |

Full aggregation reference: fetch `query/functions/aggregation`
Finance functions: fetch `query/functions/finance`
Date/time functions: fetch `query/functions/date-time`
Array functions: fetch `query/functions/array`

---

## Cookbook: Common Query Recipes

**IMPORTANT:** The most common recipes (OHLC, VWAP, Bollinger Bands, RSI) are
already provided as complete, ready-to-use SQL in the "Grafana Integration" section
above. **Do NOT fetch these from the cookbook — use the inline versions.**

Only fetch cookbook recipes for patterns NOT already covered inline. Use this:

```bash
curl -sH "Accept: text/markdown" "https://questdb.com/docs/{path}/"
```

### Capital Markets Recipes

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

### Time-Series Pattern Recipes

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

### Advanced SQL Recipes

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

### Grafana Integration Recipes

| Recipe | Fetch path | Use for |
|---|---|---|
| Dynamic table queries | `cookbook/integrations/grafana/dynamic-table-queries` | Query multiple tables via Grafana variables |
| Read-only user | `cookbook/integrations/grafana/read-only-user` | Secure Grafana with read-only PG user |
| Variable dropdown | `cookbook/integrations/grafana/variable-dropdown` | Dropdown with display name ≠ query value |
| Overlay time shift | `cookbook/integrations/grafana/overlay-timeshift` | Yesterday vs today on same chart |

### Demo data schema

QuestDB's live demo instance at demo.questdb.io has FX and crypto datasets. Fetch the schema reference for interactive testing:

Fetch path: `cookbook/demo-data-schema`

---

## Data Ingestion

### ILP (InfluxDB Line Protocol) — Primary Method

Use ILP for all high-throughput ingestion. **Never use INSERT INTO for streaming data.**

The default client is Python. All official clients share the same API patterns.
Use Python unless the project specifies otherwise.

**Python:**
```python
from questdb.ingress import Sender, TimestampNanos
import numpy as np

# Open Source (no auth). For Enterprise auth, see "Enterprise: Authentication & ACL" section.
conf = "tcp::addr=localhost:9009;protocol_version=2;"

with Sender.from_conf(conf) as sender:
    sender.row(
        'trades',
        symbols={'symbol': 'BTC-USDT', 'side': 'buy'},
        columns={'price': 42000.50, 'amount': 1.5},
        at=TimestampNanos.now()
    )
    sender.flush()
```

Key patterns:
- `Sender.from_conf()` does NOT connect — use `with` context manager or call `sender.establish()`
- `symbols={}` for SYMBOL columns, `columns={}` for everything else
- Arrays MUST be `np.float64` numpy arrays, not Python lists
- TCP config requires `protocol_version=2` for array support
- HTTP transport: `http::addr=localhost:9000;`
- TCP transport: `tcp::addr=localhost:9009;`

**2D array ingestion (order books):**
```python
sender.row(
    'order_book',
    symbols={'symbol': 'BTC-USDT'},
    columns={
        'bids': [
            np.array([41999.0, 41998.5, 41998.0], dtype=np.float64),  # prices [1]
            np.array([2.1, 5.3, 10.0], dtype=np.float64),            # sizes  [2]
        ],
        'asks': [
            np.array([42001.0, 42001.5, 42002.0], dtype=np.float64),
            np.array([1.8, 4.2, 8.5], dtype=np.float64),
        ],
    },
    at=TimestampNanos.now()
)
```

**Other client libraries:** Go, Java, Rust, Node.js, C/C++, .NET
All documented at: fetch `ingestion/clients/{language}`
Configuration string format (shared across clients): fetch `ingestion/clients/configuration-string`

### INSERT INTO — For Low Volume Only

Use for DDL, backfills, or one-off inserts. Not for streaming:

```sql
INSERT INTO trades (ts, symbol, side, price, amount)
VALUES ('2025-02-09T10:00:00.000000Z', 'BTC-USDT', 'buy', 42000.50, 1.5);
```

### Querying via PG Wire

Connect with any PostgreSQL client library on port 8812:

```python
import psycopg as pg
conn = pg.connect("user=admin password=quest host=localhost port=8812 dbname=qdb")
```

Language-specific query examples: fetch `query/pgwire/{language}`
(Python, Go, Java, Rust, Node.js, .NET, PHP, R, C/C++)

### HTTP REST API

- **Query**: `GET http://localhost:9000/exec?query=URL_ENCODED_SQL`
- **POST is not supported** for the exec endpoint — use GET only
- Returns JSON: `{ "columns": [...], "dataset": [...] }`

Full reference: fetch `query/rest-api`

---

## Grafana Integration

QuestDB has a dedicated Grafana datasource plugin (`questdb-questdb-datasource`).
Connects via PG wire on port 8812.

Key macros:
- `$__timeFilter(ts)` — time range from Grafana's time picker
- `$__interval` — auto-calculated SAMPLE BY interval

Symbol dropdown variable query: `SELECT DISTINCT symbol FROM trades`

### Discovering the QuestDB Datasource UID

Before creating dashboards programmatically, find the datasource UID:
```python
import requests
r = requests.get("http://localhost:3000/api/datasources",
                 auth=("admin", "YOUR_PASSWORD"))
for ds in r.json():
    if ds["type"] == "questdb-questdb-datasource":
        print(ds["uid"])
```

### Ready-to-Use Grafana Queries

These are complete, tested SQL queries for common financial panels.
**Use them directly — no need to fetch cookbook recipes.**

**OHLC Candlestick (from raw trades):**
```sql
SELECT ts AS time,
    first(price) AS open, max(price) AS high,
    min(price) AS low, last(price) AS close,
    sum(amount) AS volume
FROM trades
WHERE $__timeFilter(ts) AND symbol = '$symbol'
SAMPLE BY $__interval;
```

**OHLC Candlestick (from materialized view — faster for large datasets):**
```sql
SELECT ts AS time,
    first(open) AS open, max(high) AS high,
    min(low) AS low, last(close) AS close,
    sum(volume) AS volume
FROM candles_1m
WHERE $__timeFilter(ts) AND symbol = '$symbol'
SAMPLE BY $__interval;
```

**VWAP (cumulative volume-weighted average price):**
```sql
WITH ohlc AS (
    SELECT ts, symbol,
        first(open) AS open, max(high) AS high,
        min(low) AS low, last(close) AS close,
        sum(volume) AS volume
    FROM candles_1m
    WHERE $__timeFilter(ts) AND symbol = '$symbol'
    SAMPLE BY $__interval
)
SELECT ts AS time,
    close,
    sum((high + low + close) / 3 * volume) OVER (ORDER BY ts CUMULATIVE)
    / sum(volume) OVER (ORDER BY ts CUMULATIVE) AS vwap
FROM ohlc;
```

**Bollinger Bands (20-period SMA ± 2σ):**
Uses manual variance calculation which is more compatible than `stddev_samp` in
window frames:
```sql
WITH ohlc AS (
    SELECT ts, symbol,
        first(open) AS open, max(high) AS high,
        min(low) AS low, last(close) AS close,
        sum(volume) AS volume
    FROM candles_1m
    WHERE $__timeFilter(ts) AND symbol = '$symbol'
    SAMPLE BY $__interval
),
stats AS (
    SELECT ts, close,
        AVG(close) OVER (
            ORDER BY ts ROWS BETWEEN 19 PRECEDING AND CURRENT ROW
        ) AS sma20,
        AVG(close * close) OVER (
            ORDER BY ts ROWS BETWEEN 19 PRECEDING AND CURRENT ROW
        ) AS avg_close_sq
    FROM ohlc
)
SELECT ts AS time, close, sma20,
    sma20 + 2 * sqrt(avg_close_sq - (sma20 * sma20)) AS upper_band,
    sma20 - 2 * sqrt(avg_close_sq - (sma20 * sma20)) AS lower_band
FROM stats;
```

**RSI (14-period, SMA-smoothed):**
RSI is NOT in the QuestDB cookbook. Uses SMA via ROWS BETWEEN, which is simpler
and proven in production:
```sql
WITH ohlc AS (
    SELECT ts, symbol,
        first(open) AS open, max(high) AS high,
        min(low) AS low, last(close) AS close,
        sum(volume) AS volume
    FROM candles_1m
    WHERE $__timeFilter(ts) AND symbol = '$symbol'
    SAMPLE BY $__interval
),
changes AS (
    SELECT ts, close,
        close - LAG(close) OVER (ORDER BY ts) AS change
    FROM ohlc
),
gains_losses AS (
    SELECT ts, close,
        CASE WHEN change > 0 THEN change ELSE 0 END AS gain,
        CASE WHEN change < 0 THEN ABS(change) ELSE 0 END AS loss
    FROM changes
),
avg_gl AS (
    SELECT ts, close,
        AVG(gain) OVER (ORDER BY ts
            ROWS BETWEEN 13 PRECEDING AND CURRENT ROW) AS avg_gain,
        AVG(loss) OVER (ORDER BY ts
            ROWS BETWEEN 13 PRECEDING AND CURRENT ROW) AS avg_loss
    FROM gains_losses
)
SELECT ts AS time,
    CASE WHEN avg_loss = 0 THEN 100
         ELSE 100 - (100 / (1 + avg_gain / NULLIF(avg_loss, 0)))
    END AS rsi
FROM avg_gl;
```
Note: For EMA-smoothed RSI, replace the AVG...ROWS BETWEEN with
`avg(gain, 'period', 14) OVER (ORDER BY ts)` (QuestDB native EMA).

**Combined VWAP + Bollinger (single panel, multiple series):**
```sql
WITH ohlc AS (
    SELECT ts, symbol,
        first(open) AS open, max(high) AS high,
        min(low) AS low, last(close) AS close,
        sum(volume) AS volume
    FROM candles_1m
    WHERE $__timeFilter(ts) AND symbol = '$symbol'
    SAMPLE BY $__interval
),
stats AS (
    SELECT ts, open, high, low, close, volume,
        AVG(close) OVER (
            ORDER BY ts ROWS BETWEEN 19 PRECEDING AND CURRENT ROW
        ) AS sma20,
        AVG(close * close) OVER (
            ORDER BY ts ROWS BETWEEN 19 PRECEDING AND CURRENT ROW
        ) AS avg_close_sq
    FROM ohlc
)
SELECT ts AS time, close, sma20,
    sma20 + 2 * sqrt(avg_close_sq - (sma20 * sma20)) AS upper_band,
    sma20 - 2 * sqrt(avg_close_sq - (sma20 * sma20)) AS lower_band,
    sum((high + low + close) / 3 * volume) OVER (ORDER BY ts CUMULATIVE)
    / sum(volume) OVER (ORDER BY ts CUMULATIVE) AS vwap
FROM stats;
```

### Recommended Materialized View Cascade

See the complete schema in the "Schema Design" section below, which includes
the cascading `candles_5s → candles_1m → candles_1h` views. The Grafana queries
above are designed to query `candles_1m` by default.

### Advanced Grafana Dashboard Techniques

These patterns come from production QuestDB financial dashboards.

**Multiple indicators in one candlestick panel:**
Use separate queries with distinct `refId` values, then set `includeAllFields: true`
on the candlestick panel. This overlays VWAP, RSI, and Bollinger on top of OHLC candles.

```json
{
  "type": "candlestick",
  "options": {
    "mode": "candles+volume",
    "includeAllFields": true,
    "fields": {
      "open": "open", "high": "high", "low": "low",
      "close": "close", "volume": "total_volume"
    }
  },
  "targets": [
    {"refId": "OHLC", "rawSql": "SELECT ... FROM candles_15m ..."},
    {"refId": "VWAP", "rawSql": "SELECT ... cumulative VWAP ..."},
    {"refId": "RSI", "rawSql": "SELECT ... RSI calculation ..."},
    {"refId": "Bollinger", "rawSql": "SELECT ... Bollinger Bands ..."}
  ]
}
```

**Per-query axis and styling overrides (byFrameRefID):**
Use `fieldConfig.overrides` to give each refId its own axis, unit, or color:
```json
{
  "overrides": [
    {
      "matcher": {"id": "byFrameRefID", "options": "RSI"},
      "properties": [
        {"id": "unit", "value": "RSI %"},
        {"id": "custom.axisPlacement", "value": "auto"},
        {"id": "custom.axisSoftMax", "value": 100}
      ]
    },
    {
      "matcher": {"id": "byFrameRefID", "options": "Bollinger"},
      "properties": [
        {"id": "color", "value": {"fixedColor": "light-blue", "mode": "palette-classic-by-name"}},
        {"id": "custom.showPoints", "value": "always"}
      ]
    }
  ]
}
```

**Per-panel time range overrides:**
Different panels can show different time windows, independent of the dashboard picker:
```json
{
  "timeFrom": "1m",
  "hideTimeOverride": true
}
```
Common values: `"1m"` for real-time tables, `"30m"` for volume charts, `"now/d"` for
daily indicators. Set `hideTimeOverride: true` to keep the panel title clean.

**Time shift for near-real-time data:**
Offset the time range slightly to account for ingestion latency:
```json
{
  "timeFrom": "1m",
  "timeShift": "5s"
}
```

**Repeating panels per symbol:**
Set `repeat` on a panel or row to auto-duplicate per symbol variable value:
```json
{
  "repeat": "SYMBOL",
  "repeatDirection": "h",
  "maxPerRow": 2
}
```
For section headers, use a collapsed row with `"type": "row"` and `"repeat": "SYMBOL"`.

**Symbol dropdown variable using LATEST ON:**
Get only symbols with recent data, not all historical symbols:
```sql
SELECT DISTINCT symbol FROM (
    SELECT symbol FROM trades
    WHERE timestamp > dateadd('d', -4, now())
    LATEST ON timestamp PARTITION BY symbol
) ORDER BY symbol;
```

**2D array indexing for order book in Grafana queries:**
```sql
-- bids[1,1] = best bid price, bids[2,1] = best bid size
-- asks[1,1] = best ask price, asks[2,1] = best ask size
SELECT timestamp,
    avg(asks[1,1] - bids[1,1]) AS spread,
    sum(bids[1,1] * bids[2,1]) AS bid_volume,
    sum(asks[1,1] * asks[2,1]) AS ask_volume
FROM market_data
WHERE $__timeFilter(timestamp) AND symbol = '${SYMBOL}'
SAMPLE BY 1s;
```

**Market depth with Plotly plugin (`ae3e-plotly-panel`):**
Use the `ae3e-plotly-panel` plugin for order book depth charts. Query returns
full bid/ask arrays with cumulative volumes:
```sql
WITH snapshot AS (
    SELECT timestamp, bids, asks
    FROM orderbook
    WHERE $__timeFilter(timestamp) AND symbol = '${SYMBOL}'
    ORDER BY timestamp DESC
    LIMIT 1
)
SELECT timestamp,
    bids[1] AS bprices, bids[2] AS bvolumes, array_cum_sum(bids[2]) AS bcumvolumes,
    asks[1] AS aprices, asks[2] AS avolumes, array_cum_sum(asks[2]) AS acumvolumes
FROM snapshot;
```

### Grafana Dashboard Deployment via API

Create dashboards programmatically using the Grafana HTTP API:
```python
import requests, json

GRAFANA_URL = "http://localhost:3000"
GRAFANA_AUTH = ("admin", "YOUR_PASSWORD")

# Find QuestDB datasource UID
ds = requests.get(f"{GRAFANA_URL}/api/datasources", auth=GRAFANA_AUTH).json()
questdb_uid = next(d["uid"] for d in ds if d["type"] == "questdb-questdb-datasource")

# Create/update dashboard
dashboard = {
    "dashboard": {
        "title": "My Dashboard",
        "uid": "my-dashboard-uid",
        "templating": {"list": [{
            "name": "symbol", "type": "query",
            "query": "SELECT DISTINCT symbol FROM trades",
            "datasource": {"uid": questdb_uid, "type": "questdb-questdb-datasource"},
        }]},
        "panels": [
            # ... panel definitions with SQL queries above
        ],
    },
    "overwrite": True,
}
requests.post(f"{GRAFANA_URL}/api/dashboards/db", auth=GRAFANA_AUTH,
              headers={"Content-Type": "application/json"}, json=dashboard)
```

Always reference the datasource by UID and type, never by display name.

For Grafana-specific patterns not covered above, fetch the cookbook recipes listed
in the Cookbook section.

---

## Enterprise: Authentication & ACL

QuestDB Enterprise requires user/service account setup with explicit grants.
Skip this section entirely for QuestDB Open Source (no auth needed).

Full ACL reference: fetch `query/sql/acl/create-user`, `query/sql/acl/create-service-account`, `query/sql/acl/alter-user`

### 1. Create User or Service Account

```sql
-- Interactive user (for humans / Grafana)
CREATE USER analyst;

-- Service account (for applications / pipelines)
CREATE SERVICE ACCOUNT ingestor;
```

Use a service account for automated pipelines. Use a user for interactive access.

### 2. Grant Permissions

```sql
-- Table & view creation (creator auto-gets all permissions on what they create)
GRANT CREATE TABLE TO analyst;
GRANT CREATE MATERIALIZED VIEW TO analyst;

-- Insert on pre-existing tables (not needed if the user created the table)
GRANT INSERT TO ingestor;

-- Protocol access — grant only what's needed
GRANT PGWIRE TO analyst;     -- PostgreSQL wire protocol (port 8812)
GRANT HTTP TO analyst;        -- REST API + HTTP ingestion (port 9000)
GRANT ILP TO ingestor;        -- ILP ingestion (port 9009)

-- Admin
GRANT LIST USERS TO admin_user;  -- list users and service accounts
```

### 3. Create Auth Tokens

**REST API token** (for HTTP ingestion and REST queries):
```sql
ALTER USER api_user CREATE TOKEN TYPE REST WITH TTL '3600d' REFRESH;
-- or for service accounts:
ALTER SERVICE ACCOUNT ingestor CREATE TOKEN TYPE REST WITH TTL '3600d' REFRESH;
```

Returns:
| name | token | expires_at | refresh |
|---|---|---|---|
| api_user | qt1Kd8Rqmy5jb_YdIJeJ... | 2035-12-20T09:44:30Z | true |

**JWK token** (for ILP/TCP ingestion):
```sql
ALTER USER sensor_writer CREATE TOKEN TYPE JWK;
-- or for service accounts:
ALTER SERVICE ACCOUNT ingestor CREATE TOKEN TYPE JWK;
```

Returns three values needed for ILP auth:
| name | public_key_x | public_key_y | private_key |
|---|---|---|---|
| sensor_writer | TQPRohJ4IKMUo... | D4JGDU5FT2jSo... | Herob8IUAJ6-d... |

### 4. Connection Strings with Auth

**HTTP ingestion (recommended — returns errors immediately):**
```python
# Open Source (no auth)
conf = f"http::addr={host}:9000;auto_flush_interval={auto_flush_interval};"

# Enterprise (REST token + TLS)
conf = f"https::addr={host}:9000;token={token};tls_verify=unsafe_off;auto_flush_interval={auto_flush_interval};"
```

**TCP/ILP ingestion (faster, but errors are silent — risk of data loss on disconnect):**
```python
# Open Source (no auth)
conf = f"tcp::addr={host}:9009;protocol_version=2;auto_flush_interval={auto_flush_interval};"

# Enterprise (JWK token + TLS)
conf = f"tcps::addr={host}:9009;username={ilp_user};token={private_key};token_x={public_key_x};token_y={public_key_y};tls_verify=unsafe_off;protocol_version=2;auto_flush_interval={auto_flush_interval};"
```

**Protocol choice:**
- **HTTP** (recommended): Returns errors immediately. Slightly slower. Use `http::` (open source) or `https::` (enterprise).
- **TCP/ILP**: Fastest throughput. Errors are NOT communicated back — data loss can happen silently on disconnect. Use `tcp::` (open source) or `tcps::` (enterprise).

**TLS / self-signed certificates:**
When the server uses `https` or `tcps`, the client verifies the TLS certificate by default. For self-signed certificates (common in dev/staging), add `tls_verify=unsafe_off;` to the connection string. **Always ask the user** whether the server uses a self-signed certificate before adding this flag — never add it silently in production configs.

```python
# Self-signed cert example
conf = f"https::addr={host}:9000;token={token};tls_verify=unsafe_off;auto_flush_interval={auto_flush_interval};"
```

### 5. PG Wire with Auth (Grafana, psycopg)

```python
import psycopg as pg

# Open Source
conn_str = f"user=admin password=quest host=localhost port=8812 dbname=qdb"

# Enterprise
conn_str = f"user={user} password={password} host={host} port=8812 dbname=qdb"

conn = pg.connect(conn_str)
```

No `sslmode` needed — QuestDB handles PG wire auth via its own user/password system.

---

## Schema Design

Key principles:
- Every time-series table needs a designated timestamp
- Use SYMBOL for any repeated string (tickers, categories, status codes)
- Partition by the most common query granularity
- WAL tables for concurrent write workloads
- DEDUP for idempotent ingestion

For advanced schema questions, fetch: `schema-design-essentials`

### Financial Market Data Pipeline Schema (Ready to Use)

This is a complete schema for tick trades + L2 order book + cascading OHLCV candles:

```sql
-- Raw trades
CREATE TABLE IF NOT EXISTS trades (
    ts TIMESTAMP,
    symbol SYMBOL,
    side SYMBOL,
    price DOUBLE,
    amount DOUBLE
) TIMESTAMP(ts) PARTITION BY DAY WAL
DEDUP UPSERT KEYS(ts, symbol);

-- L2 order book snapshots (2D arrays: [1]=prices, [2]=sizes)
CREATE TABLE IF NOT EXISTS orderbook (
    ts TIMESTAMP,
    symbol SYMBOL,
    bid_prices DOUBLE[],
    bid_sizes DOUBLE[],
    ask_prices DOUBLE[],
    ask_sizes DOUBLE[]
) TIMESTAMP(ts) PARTITION BY DAY WAL
DEDUP UPSERT KEYS(ts, symbol);

-- Cascading materialized views: trades → 5s → 1m → 1h
CREATE MATERIALIZED VIEW IF NOT EXISTS candles_5s AS (
    SELECT ts, symbol,
        first(price) AS open, max(price) AS high,
        min(price) AS low, last(price) AS close,
        sum(amount) AS volume
    FROM trades SAMPLE BY 5s
) PARTITION BY DAY;

CREATE MATERIALIZED VIEW IF NOT EXISTS candles_1m AS (
    SELECT ts, symbol,
        first(open) AS open, max(high) AS high,
        min(low) AS low, last(close) AS close,
        sum(volume) AS volume
    FROM candles_5s SAMPLE BY 1m
) PARTITION BY DAY;

CREATE MATERIALIZED VIEW IF NOT EXISTS candles_1h AS (
    SELECT ts, symbol,
        first(open) AS open, max(high) AS high,
        min(low) AS low, last(close) AS close,
        sum(volume) AS volume
    FROM candles_1m SAMPLE BY 1h
) PARTITION BY MONTH;
```

Use this schema as-is for any financial market data pipeline. The Grafana queries
in this skill are designed to work with these exact table/view names.

---

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

---

## Common Mistakes

Read `references/common-mistakes.md` in this skill directory for a comprehensive
table of wrong patterns and their correct QuestDB equivalents. This is the
single most important reference for avoiding hallucinated SQL.
