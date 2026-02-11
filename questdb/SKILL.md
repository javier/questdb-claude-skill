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

## How to Use This Skill

**IMPORTANT — MINIMIZE ROUND-TRIPS:**
- Do NOT explore library source code (cryptofeed, questdb, etc.)
- Do NOT check library versions or verify callback signatures
- Do NOT read installed package files to "understand the API"
- Do NOT verify infrastructure is running — trust the user's prompt
- Do NOT read extra reference files for topics already covered in this skill file
- DO read reference files when their topic applies (e.g. enterprise.md for auth, grafana-advanced.md for complex panels)
- Do NOT use task tracking (TaskCreate/TaskUpdate) for straightforward builds
- Do NOT add `sleep` commands to wait for data or check background processes
- All API details for cryptofeed, QuestDB ingestion, and Grafana are below — use them as-is
- If something fails at runtime, fix it then — do not research upfront

This skill contains ready-to-use SQL, schemas, ingestion code, and Grafana
queries. **Write the files and run them.** A typical pipeline is 3 files
(schema setup, ingestion script, dashboard deploy) — write them, execute them, done.

Additional references in the `references/` directory — only read when the user's
request goes beyond what this file covers:
- `common-mistakes.md` — Wrong patterns → correct QuestDB equivalents (read when writing novel SQL not already templated below)
- `grafana-advanced.md` — Read only for Plotly order book depth charts or advanced features not in the dashboard template below
- `indicators.md` — Read when user asks for indicators beyond OHLC/VWAP/Bollinger/RSI (MACD, ATR, Stochastic, OBV, Drawdown, Keltner, Donchian, etc.)
- `cookbook.md` — Fetch paths for 30+ cookbook recipes (only for patterns not inline here)
- `enterprise.md` — **Read when QuestDB uses authentication, HTTPS, tokens, or ACLs** (skip for open source)

## Critical Rule

**QuestDB is NOT PostgreSQL.** It supports the PostgreSQL wire protocol for
querying, but has its own SQL extensions for time-series operations. Standard
PostgreSQL patterns like `time_bucket()`, `DISTINCT ON`, `HAVING`, and
`generate_series()` do not exist.

## Live Documentation Access

QuestDB docs are available as clean markdown:

```bash
curl -sH "Accept: text/markdown" "https://questdb.com/docs/{page}/"
```

Full table of contents: `curl -s "https://questdb.com/docs/llms.txt"`

---

## SQL Reference

### CREATE TABLE

```sql
CREATE TABLE IF NOT EXISTS trades (
    ts TIMESTAMP,
    symbol SYMBOL,
    side SYMBOL,
    price DOUBLE,
    amount DOUBLE
) TIMESTAMP(ts) PARTITION BY DAY WAL
DEDUP UPSERT KEYS(ts, symbol);
```

Key rules:
- `TIMESTAMP(col)` designates the time column — required for SAMPLE BY, LATEST ON, ASOF JOIN
- `SYMBOL` type for any repeated string (tickers, categories, status codes) — much faster than VARCHAR
- `PARTITION BY DAY|MONTH|YEAR|HOUR` — use the most common query granularity
- `WAL` enables concurrent writes (required for ILP ingestion)
- `DEDUP UPSERT KEYS(ts, symbol)` deduplicates on (timestamp, symbol) — idempotent ingestion

**Column types:** BOOLEAN, BYTE, SHORT, INT, LONG, FLOAT, DOUBLE, CHAR, VARCHAR,
SYMBOL, TIMESTAMP, DATE, LONG256, GEOHASH, UUID, IPv4, DOUBLE[], FLOAT[], INT[],
LONG[], SHORT[], UUID[] — plus 2D arrays like `DOUBLE[][]`

**ALTER TABLE:**
```sql
ALTER TABLE trades ADD COLUMN exchange SYMBOL;
ALTER TABLE trades DROP COLUMN exchange;
ALTER TABLE trades RENAME COLUMN amount TO qty;
ALTER TABLE trades ALTER COLUMN exchange SET TYPE SYMBOL;
```

### SAMPLE BY (Time-Series Aggregation)

SAMPLE BY is QuestDB's time-bucketing. It replaces `GROUP BY time_bucket()`.
Requires a designated timestamp.

```sql
SELECT ts, symbol,
    first(price) AS open, max(price) AS high,
    min(price) AS low, last(price) AS close,
    sum(amount) AS volume
FROM trades
WHERE ts > '2025-01-01'
SAMPLE BY 1h;
```

Key rules:
- Valid intervals: `1s`, `5s`, `1m`, `15m`, `1h`, `1d`, `1M` (month)
- `ALIGN TO CALENDAR` aligns buckets to clock boundaries (this is the default, so it can be omitted)
- `FILL(PREV | NULL | LINEAR | value)` fills gaps — goes AFTER SAMPLE BY
- `$__interval` in Grafana auto-calculates the interval from the time picker
- `first()`, `last()` return first/last values within each time bucket
- `count_distinct(col)` instead of `COUNT(DISTINCT col)`
- No `HAVING` — use a subquery: `SELECT * FROM (... SAMPLE BY ...) WHERE volume > 1000`

### LATEST ON (Last Value Per Group)

Returns the most recent row per group. Replaces `DISTINCT ON` / `ROW_NUMBER()`.

```sql
SELECT * FROM trades
WHERE ts > dateadd('h', -1, now())
LATEST ON ts PARTITION BY symbol;
```

Rules:
- `LATEST ON ts` must reference the designated timestamp
- `PARTITION BY` is required
- Add a `WHERE` time filter for performance

### Joins

```sql
-- ASOF JOIN: match nearest timestamp (≤)
SELECT * FROM trades ASOF JOIN quotes ON (symbol);

-- LT JOIN: match strictly before (<)
SELECT * FROM trades LT JOIN quotes ON (symbol);

-- SPLICE JOIN: merge two time series interleaved
SELECT * FROM trades SPLICE JOIN quotes ON (symbol);
```

Join rules:
- Time-series joins (ASOF, LT, SPLICE) match on the designated timestamp automatically
- `ON (symbol)` matches the key column — both tables must have the same column name
- Standard INNER JOIN and LEFT JOIN also work

### Window Functions

```sql
-- ROWS frame (count-based)
AVG(close) OVER (ORDER BY ts ROWS BETWEEN 19 PRECEDING AND CURRENT ROW) AS sma20

-- CUMULATIVE shorthand (QuestDB extension, equivalent to ROWS UNBOUNDED PRECEDING)
SUM(volume) OVER (ORDER BY ts CUMULATIVE) AS running_total

-- PARTITION BY
LAG(close) OVER (PARTITION BY symbol ORDER BY ts) AS prev_close

-- EMA (exponential moving average — QuestDB extension)
avg(price, 'period', 14) OVER (ORDER BY ts) AS ema14
```

Supported: `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, `LAG()`, `LEAD()`,
`FIRST_VALUE()`, `AVG()`, `SUM()`, `MIN()`, `MAX()`, `COUNT()`

**Important:** `stddev_samp()` may not work in window frames. For standard deviation,
compute manually: `sqrt(avg(x*x) - avg(x)^2)` — see Bollinger Bands query below.

### Materialized Views

Auto-updated, incremental aggregations triggered by new data:

```sql
CREATE MATERIALIZED VIEW IF NOT EXISTS candles_5s AS (
    SELECT ts, symbol,
        first(price) AS open, max(price) AS high,
        min(price) AS low, last(price) AS close,
        sum(amount) AS volume
    FROM trades SAMPLE BY 5s
) PARTITION BY DAY;
```

Rules:
- Source query MUST use SAMPLE BY
- Only aggregation functions allowed (no WHERE, no JOIN, no window functions)
- Use `IF NOT EXISTS` to avoid errors on re-run
- Cascade views for multi-resolution: `trades → 5s → 1m → 1h` (see Schema Design below)
- Invalidate/rebuild: `ALTER MATERIALIZED VIEW candles_5s INVALIDATE`

### Timestamp Filtering — TICK Syntax

QuestDB has a compact timestamp filter syntax using semicolons:

```sql
WHERE ts IN '2025-02-09;2h'     -- Last 2 hours from that timestamp
WHERE ts IN '2025-01-01;2025-01-31'  -- Date range
WHERE ts IN now()               -- Today
```

This is preferred over `dateadd()` for readability, but both work.

### DECLARE (Variables)

```sql
DECLARE @start := '2025-01-01T00:00:00Z', @end := now();
SELECT * FROM trades WHERE ts BETWEEN @start AND @end;
```

### SQL Execution Order

`FROM → WHERE → SAMPLE BY → SELECT → LIMIT`
(No GROUP BY needed with SAMPLE BY. No HAVING at all.)

### Key Functions

**Timestamp:** `now()`, `systimestamp()`, `dateadd('unit', n, ts)`,
`datediff('unit', ts1, ts2)`, `to_timestamp('str', 'fmt')`,
`timestamp_floor('unit', ts)`, `timestamp_ceil('unit', ts)`,
`hour(ts)`, `day_of_week(ts)`, `week_of_year(ts)`, `year(ts)`, `month(ts)`

**String:** `concat('a', 'b')`, `left(s, n)`, `right(s, n)`, `length(s)`,
`starts_with(s, prefix)`, `lcase(s)`, `ucase(s)`, `replace(s, old, new)`,
`split_part(s, delim, idx)`, `regexp_replace(s, pattern, replacement)`

**Aggregation:** `first(x)`, `last(x)`, `count_distinct(x)`, `sum(x)`, `avg(x)`,
`min(x)`, `max(x)`, `haversine_dist_deg(lat1, lon1, lat2, lon2)`

**Array:** `array_cum_sum(arr)` — cumulative sum of array elements

---

## Data Ingestion

### ILP (InfluxDB Line Protocol) — Primary Method

Use ILP for all high-throughput ingestion. **Never use INSERT INTO for streaming data.**

```python
from questdb.ingress import Sender, TimestampNanos
import numpy as np

# Open Source (no auth). For Enterprise: read references/enterprise.md Quick Start
# (admin creates service account + token via REST → ingestion script uses token)
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

**Other client libraries:** Go, Java, Rust, Node.js, C/C++, .NET — fetch `ingestion/clients/{language}`

### INSERT INTO — For Low Volume Only

```sql
INSERT INTO trades (ts, symbol, side, price, amount)
VALUES ('2025-02-09T10:00:00.000000Z', 'BTC-USDT', 'buy', 42000.50, 1.5);
```

### Querying via PG Wire

```python
import psycopg as pg
conn = pg.connect("user=admin password=quest host=localhost port=8812 dbname=qdb")
```

### HTTP REST API

- **Query**: `GET http://localhost:9000/exec?query=URL_ENCODED_SQL`
- **Enterprise**: see Quick Start in `references/enterprise.md`
- **POST is not supported** for the exec endpoint — use GET only
- Returns JSON: `{ "columns": [...], "dataset": [...] }`

---

## Schema Design

Key principles:
- Every time-series table needs a designated timestamp
- Use SYMBOL for any repeated string (tickers, categories, status codes)
- Partition by the most common query granularity
- WAL tables for concurrent write workloads
- DEDUP for idempotent ingestion

### Financial Market Data Pipeline Schema (Ready to Use)

```sql
CREATE TABLE IF NOT EXISTS trades (
    ts TIMESTAMP,
    symbol SYMBOL,
    side SYMBOL,
    price DOUBLE,
    amount DOUBLE
) TIMESTAMP(ts) PARTITION BY DAY WAL
DEDUP UPSERT KEYS(ts, symbol);

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

The Grafana queries below work with these exact table/view names.

---

## Demo & Sample Data

### Real-Time Crypto Feed (cryptofeed + QuestDB)

**Do NOT explore cryptofeed source code or check its version. Everything you need
is right here.** Use `cryptofeed` to generate live market data into the schema above:

```python
import asyncio
import numpy as np
from cryptofeed import FeedHandler
from cryptofeed.exchanges import OKX
from cryptofeed.defines import TRADES, L2_BOOK
from questdb.ingress import Sender, TimestampNanos

conf = "tcp::addr=localhost:9009;protocol_version=2;"  # Enterprise: see references/enterprise.md

async def trade_cb(t, receipt_timestamp):
    with Sender.from_conf(conf) as sender:
        sender.row(
            'trades',
            symbols={'symbol': t.symbol, 'side': t.side},
            columns={'price': float(t.price), 'amount': float(t.amount)},
            at=TimestampNanos(int(t.timestamp * 1e9))
        )
        sender.flush()

async def book_cb(book, receipt_timestamp):
    bids = book.book['bid']
    asks = book.book['ask']
    # Sort: bids descending by price, asks ascending
    bid_prices = sorted(bids.keys(), reverse=True)[:25]
    ask_prices = sorted(asks.keys())[:25]
    with Sender.from_conf(conf) as sender:
        sender.row(
            'orderbook',
            symbols={'symbol': book.symbol},
            columns={
                'bid_prices': np.array([float(p) for p in bid_prices], dtype=np.float64),
                'bid_sizes':  np.array([float(bids[p]) for p in bid_prices], dtype=np.float64),
                'ask_prices': np.array([float(p) for p in ask_prices], dtype=np.float64),
                'ask_sizes':  np.array([float(asks[p]) for p in ask_prices], dtype=np.float64),
            },
            at=TimestampNanos(int(book.timestamp * 1e9))
        )
        sender.flush()

SYMBOLS = ['BTC-USDT', 'ETH-USDT', 'SOL-USDT']  # add more as needed

f = FeedHandler()
f.add_feed(OKX(
    symbols=SYMBOLS,
    channels=[TRADES, L2_BOOK],
    callbacks={TRADES: trade_cb, L2_BOOK: book_cb}
))
f.run()
```

**cryptofeed API reference (complete — tested with v2.4.x — do NOT read source code to verify):**
- `t.symbol`, `t.side` (`'buy'`/`'sell'`), `t.price`, `t.amount`, `t.timestamp` (float epoch seconds)
- **`t.price` and `t.amount` are `Decimal` types — cast with `float()` before passing to QuestDB Sender**
- `book.book['bid']` and `book.book['ask']` are dicts: `{Decimal(price): Decimal(size), ...}`
- `book.symbol`, `book.timestamp` (float epoch seconds)
- Exchanges: `OKX`, `Coinbase`, `Binance`, `Kraken`, `Bybit`, etc.
- Channels: `TRADES`, `L2_BOOK`, `L3_BOOK`, `TICKER`, `CANDLES`, `OPEN_INTEREST`, `FUNDING`, `LIQUIDATIONS`
- Symbol format is exchange-native: `'BTC-USDT'` for OKX, `'BTC-USD'` for Coinbase
- **Python compatibility:** avoid `X | None` type hints (requires 3.10+). Use `Optional[X]` or plain assignment.

**Performance note:** The example above opens a Sender per callback for clarity.
For production, use a shared Sender with periodic flush:

```python
sender = Sender.from_conf(conf)
sender.establish()

async def trade_cb(t, receipt_timestamp):
    sender.row('trades', symbols={...}, columns={...}, at=...)
    # flush periodically or use auto_flush_interval in conf string
```

**Operational notes:**
- cryptofeed logs to **stderr**, not stdout. An empty stdout does not mean failure.
  Verify data flow by querying QuestDB: `SELECT count() FROM trades`
- When building the pipeline, write 3 files (schema, ingestion, dashboard deploy)
  and run them sequentially. Schema must exist before ingestion starts.
- End the dashboard deploy script with `open` (macOS) or `xdg-open` (Linux)
  to launch the browser automatically:
  ```python
  import subprocess, sys
  url = f"{GRAFANA_URL}{resp.json()['url']}"
  subprocess.run(["open" if sys.platform == "darwin" else "xdg-open", url])
  ```

### QuestDB Demo Instance

QuestDB's live demo at `demo.questdb.io` has FX and crypto datasets.
Fetch the schema reference: `curl -sH "Accept: text/markdown" "https://questdb.com/docs/cookbook/demo-data-schema/"`

---

## Grafana Integration

QuestDB has a dedicated Grafana datasource plugin (`questdb-questdb-datasource`).
Connects via PG wire on port 8812.

Key macros:
- `$__timeFilter(ts)` — time range from Grafana's time picker
- `$__interval` — auto-calculated SAMPLE BY interval

Symbol dropdown variable: `SELECT DISTINCT symbol FROM trades`

For advanced Grafana patterns (multi-query panels, axis overrides, repeating
panels, order book depth charts), see `references/grafana-advanced.md`.

### Discovering the QuestDB Datasource UID

```python
import requests
r = requests.get("http://localhost:3000/api/datasources",
                 auth=("admin", "YOUR_PASSWORD"))
for ds in r.json():
    if ds["type"] == "questdb-questdb-datasource":
        print(ds["uid"])
```

### Ready-to-Use Grafana Queries

Complete, tested SQL for common financial panels. **Use directly.**

**Grafana query pattern rule:** Grafana needs a `time` column, so the final SELECT
aliases `ts AS time`. **Never put OVER() clauses in the same SELECT that aliases
`ts AS time`** — put all window functions in CTEs where `ts` is still `ts`, then
alias only in the final SELECT. All queries below follow this pattern.

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

**OHLC Candlestick (from materialized view — faster):**
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
),
vwap AS (
    SELECT ts, close,
        sum((high + low + close) / 3 * volume) OVER (ORDER BY ts CUMULATIVE)
        / sum(volume) OVER (ORDER BY ts CUMULATIVE) AS vwap
    FROM ohlc
)
SELECT ts AS time, close, vwap FROM vwap;
```

**Bollinger Bands (20-period SMA ± 2σ):**
Uses manual variance — more compatible than `stddev_samp` in window frames:
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
This Grafana version uses SMA via ROWS BETWEEN, proven in production.
For standalone EMA-smoothed RSI, see `references/indicators.md`.
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
Note: For EMA-smoothed RSI (standard), replace the AVG...ROWS BETWEEN with
`avg(gain, 'period', 14) OVER (ORDER BY ts)` (QuestDB native EMA).
For Wilder's smoothing (α=1/N), use `avg(gain, 'period', 27)`.

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
indicators AS (
    SELECT ts, close,
        AVG(close) OVER (
            ORDER BY ts ROWS BETWEEN 19 PRECEDING AND CURRENT ROW
        ) AS sma20,
        AVG(close * close) OVER (
            ORDER BY ts ROWS BETWEEN 19 PRECEDING AND CURRENT ROW
        ) AS avg_close_sq,
        sum((high + low + close) / 3 * volume) OVER (ORDER BY ts CUMULATIVE)
        / sum(volume) OVER (ORDER BY ts CUMULATIVE) AS vwap
    FROM ohlc
)
SELECT ts AS time, close, sma20,
    sma20 + 2 * sqrt(avg_close_sq - (sma20 * sma20)) AS upper_band,
    sma20 - 2 * sqrt(avg_close_sq - (sma20 * sma20)) AS lower_band,
    vwap
FROM indicators;
```

### Dashboard Deployment via API

Complete working deployment script. This dashboard JSON is tested and working
— copy the structure exactly for all panels.

**Target structure for every panel query:**
```json
{
    "refId": "A",
    "datasource": {"uid": "QUESTDB_UID", "type": "questdb-questdb-datasource"},
    "format": 1,
    "rawSql": "SELECT ts AS time, ... WHERE $__timeFilter(ts) ..."
}
```
**CRITICAL:** `"format"` MUST be integer `1`, not string `"table"`. The QuestDB
Grafana plugin uses a Go integer enum (`sqlutil.FormatQueryOption`). String values
cause `json: cannot unmarshal string into Go struct field Query.format`. Grafana's
JSON export shows `"table"` (string) but the API POST requires `1` (integer).

```python
import json, requests

GRAFANA_URL = "http://localhost:3000"
GRAFANA_AUTH = ("admin", "YOUR_PASSWORD")

# Find QuestDB datasource UID
ds = requests.get(f"{GRAFANA_URL}/api/datasources", auth=GRAFANA_AUTH).json()
questdb_uid = next(d["uid"] for d in ds if d["type"] == "questdb-questdb-datasource")
DS_REF = {"uid": questdb_uid, "type": "questdb-questdb-datasource"}

dashboard = {
    "dashboard": {
        "title": "Crypto Real-Time Market Data",
        "uid": "crypto-realtime",
        "timezone": "browser",
        "refresh": "5s",
        "schemaVersion": 38,
        "time": {"from": "now-15m", "to": "now"},
        "tags": ["crypto", "questdb", "realtime"],
        "templating": {"list": [{
            "name": "symbol", "type": "query", "label": "Symbol",
            "query": "SELECT DISTINCT symbol FROM trades ORDER BY symbol;",
            "datasource": DS_REF,
            "refresh": 1, "sort": 1,
            "current": {"text": "BTC-USDT", "value": "BTC-USDT"},
        }]},
        "panels": [
            {
                "id": 1, "type": "candlestick",
                "title": "OHLC Candlestick - $symbol",
                "gridPos": {"h": 10, "w": 24, "x": 0, "y": 0},
                "datasource": DS_REF,
                "fieldConfig": {
                    "defaults": {"custom": {"axisBorderShow": False, "axisPlacement": "auto"}},
                    "overrides": [{"matcher": {"id": "byName", "options": "volume"},
                                   "properties": [{"id": "custom.axisPlacement", "value": "hidden"}]}],
                },
                "options": {
                    "mode": "candles+volume", "includeAllFields": False,
                    "candleStyle": "candles", "colorStrategy": "open-close",
                    "colors": {"up": "green", "down": "red"},
                    "fields": {"open": "open", "high": "high", "low": "low",
                               "close": "close", "volume": "volume"},
                },
                "targets": [{
                    "refId": "A", "datasource": DS_REF, "format": 1,
                    "rawSql": "SELECT ts AS time, first(price) AS open, max(price) AS high, min(price) AS low, last(price) AS close, sum(amount) AS volume FROM trades WHERE $__timeFilter(ts) AND symbol = '$symbol' SAMPLE BY $__interval;",
                }],
            },
            {
                "id": 2, "type": "timeseries",
                "title": "VWAP - $symbol",
                "gridPos": {"h": 8, "w": 12, "x": 0, "y": 10},
                "datasource": DS_REF,
                "fieldConfig": {
                    "defaults": {"custom": {"lineWidth": 2, "fillOpacity": 5, "spanNulls": True, "pointSize": 1}},
                    "overrides": [
                        {"matcher": {"id": "byName", "options": "vwap"},
                         "properties": [{"id": "color", "value": {"fixedColor": "orange", "mode": "fixed"}},
                                        {"id": "custom.lineWidth", "value": 2}]},
                        {"matcher": {"id": "byName", "options": "close"},
                         "properties": [{"id": "color", "value": {"fixedColor": "white", "mode": "fixed"}},
                                        {"id": "custom.lineWidth", "value": 1}]},
                    ],
                },
                "targets": [{
                    "refId": "A", "datasource": DS_REF, "format": 1,
                    "rawSql": "WITH ohlc AS (SELECT ts, symbol, first(open) AS open, max(high) AS high, min(low) AS low, last(close) AS close, sum(volume) AS volume FROM candles_1m WHERE $__timeFilter(ts) AND symbol = '$symbol' SAMPLE BY $__interval), vwap AS (SELECT ts, close, sum((high + low + close) / 3 * volume) OVER (ORDER BY ts CUMULATIVE) / sum(volume) OVER (ORDER BY ts CUMULATIVE) AS vwap FROM ohlc) SELECT ts AS time, close, vwap FROM vwap;",
                }],
            },
            {
                "id": 3, "type": "timeseries",
                "title": "Bollinger Bands (20, 2) - $symbol",
                "gridPos": {"h": 8, "w": 12, "x": 12, "y": 10},
                "datasource": DS_REF,
                "fieldConfig": {
                    "defaults": {"custom": {"lineWidth": 1, "spanNulls": True, "pointSize": 1}},
                    "overrides": [
                        {"matcher": {"id": "byName", "options": "close"},
                         "properties": [{"id": "color", "value": {"fixedColor": "white", "mode": "fixed"}}]},
                        {"matcher": {"id": "byName", "options": "sma20"},
                         "properties": [{"id": "color", "value": {"fixedColor": "yellow", "mode": "fixed"}},
                                        {"id": "custom.lineWidth", "value": 2}]},
                        {"matcher": {"id": "byName", "options": "upper_band"},
                         "properties": [{"id": "color", "value": {"fixedColor": "light-blue", "mode": "fixed"}},
                                        {"id": "custom.lineStyle", "value": {"fill": "dash", "dash": [10, 10]}},
                                        {"id": "custom.fillBelowTo", "value": "lower_band"},
                                        {"id": "custom.fillOpacity", "value": 8}]},
                        {"matcher": {"id": "byName", "options": "lower_band"},
                         "properties": [{"id": "color", "value": {"fixedColor": "light-blue", "mode": "fixed"}},
                                        {"id": "custom.lineStyle", "value": {"fill": "dash", "dash": [10, 10]}}]},
                    ],
                },
                "targets": [{
                    "refId": "A", "datasource": DS_REF, "format": 1,
                    "rawSql": "WITH ohlc AS (SELECT ts, symbol, first(open) AS open, max(high) AS high, min(low) AS low, last(close) AS close, sum(volume) AS volume FROM candles_1m WHERE $__timeFilter(ts) AND symbol = '$symbol' SAMPLE BY $__interval), stats AS (SELECT ts, close, AVG(close) OVER (ORDER BY ts ROWS BETWEEN 19 PRECEDING AND CURRENT ROW) AS sma20, AVG(close * close) OVER (ORDER BY ts ROWS BETWEEN 19 PRECEDING AND CURRENT ROW) AS avg_close_sq FROM ohlc) SELECT ts AS time, close, sma20, sma20 + 2 * sqrt(avg_close_sq - (sma20 * sma20)) AS upper_band, sma20 - 2 * sqrt(avg_close_sq - (sma20 * sma20)) AS lower_band FROM stats;",
                }],
            },
            {
                "id": 4, "type": "timeseries",
                "title": "RSI (14) - $symbol",
                "gridPos": {"h": 6, "w": 12, "x": 0, "y": 18},
                "datasource": DS_REF,
                "fieldConfig": {
                    "defaults": {
                        "custom": {"lineWidth": 2, "fillOpacity": 5, "spanNulls": True, "pointSize": 1,
                                   "thresholdsStyle": {"mode": "dashed+area"}},
                        "min": 0, "max": 100,
                        "thresholds": {"mode": "absolute", "steps": [
                            {"color": "green", "value": None},
                            {"color": "green", "value": 30},
                            {"color": "transparent", "value": 30.01},
                            {"color": "transparent", "value": 69.99},
                            {"color": "red", "value": 70},
                        ]},
                    },
                    "overrides": [
                        {"matcher": {"id": "byName", "options": "rsi"},
                         "properties": [{"id": "color", "value": {"fixedColor": "purple", "mode": "fixed"}}]},
                    ],
                },
                "targets": [{
                    "refId": "A", "datasource": DS_REF, "format": 1,
                    "rawSql": "WITH ohlc AS (SELECT ts, symbol, first(open) AS open, max(high) AS high, min(low) AS low, last(close) AS close, sum(volume) AS volume FROM candles_1m WHERE $__timeFilter(ts) AND symbol = '$symbol' SAMPLE BY $__interval), changes AS (SELECT ts, close, close - LAG(close) OVER (ORDER BY ts) AS change FROM ohlc), gains_losses AS (SELECT ts, close, CASE WHEN change > 0 THEN change ELSE 0 END AS gain, CASE WHEN change < 0 THEN ABS(change) ELSE 0 END AS loss FROM changes), avg_gl AS (SELECT ts, close, AVG(gain) OVER (ORDER BY ts ROWS BETWEEN 13 PRECEDING AND CURRENT ROW) AS avg_gain, AVG(loss) OVER (ORDER BY ts ROWS BETWEEN 13 PRECEDING AND CURRENT ROW) AS avg_loss FROM gains_losses) SELECT ts AS time, CASE WHEN avg_loss = 0 THEN 100 ELSE 100 - (100 / (1 + avg_gain / NULLIF(avg_loss, 0))) END AS rsi FROM avg_gl;",
                }],
            },
            {
                "id": 5, "type": "timeseries",
                "title": "Bid-Ask Spread - $symbol",
                "gridPos": {"h": 6, "w": 12, "x": 12, "y": 18},
                "datasource": DS_REF,
                "fieldConfig": {
                    "defaults": {"custom": {"lineWidth": 1, "fillOpacity": 15, "spanNulls": True, "pointSize": 1}},
                    "overrides": [
                        {"matcher": {"id": "byName", "options": "spread"},
                         "properties": [{"id": "color", "value": {"fixedColor": "red", "mode": "fixed"}},
                                        {"id": "custom.axisPlacement", "value": "right"}]},
                        {"matcher": {"id": "byName", "options": "best_bid"},
                         "properties": [{"id": "color", "value": {"fixedColor": "green", "mode": "fixed"}}]},
                        {"matcher": {"id": "byName", "options": "best_ask"},
                         "properties": [{"id": "color", "value": {"fixedColor": "orange", "mode": "fixed"}}]},
                    ],
                },
                "targets": [{
                    "refId": "A", "datasource": DS_REF, "format": 1,
                    "rawSql": "SELECT ts AS time, avg(ask_prices[1] - bid_prices[1]) AS spread, avg(bid_prices[1]) AS best_bid, avg(ask_prices[1]) AS best_ask FROM orderbook WHERE $__timeFilter(ts) AND symbol = '$symbol' SAMPLE BY $__interval;",
                }],
            },
        ],
    },
    "overwrite": True,
}

resp = requests.post(f"{GRAFANA_URL}/api/dashboards/db", auth=GRAFANA_AUTH,
                     headers={"Content-Type": "application/json"}, json=dashboard)
url = f"{GRAFANA_URL}{resp.json().get('url', '')}"
print(f"Dashboard: {resp.status_code} - {url}")

# Open in browser
import subprocess, sys
subprocess.run(["open" if sys.platform == "darwin" else "xdg-open", url])
```

Always reference the datasource by UID and type, never by display name.
Do NOT add `sleep` commands to wait for data — deploy and move on.
