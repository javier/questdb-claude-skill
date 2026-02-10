# QuestDB Common Mistakes

This is the most critical reference in the skill. Every row represents a pattern
that LLMs frequently hallucinate because they exist in PostgreSQL but NOT in
QuestDB, or because QuestDB's syntax differs from what an LLM would guess.

**Read this before writing any QuestDB SQL.**

## SQL Syntax Mistakes

| ❌ Don't | ✅ Do Instead | Why |
|---|---|---|
| `GROUP BY time_bucket('5m', ts)` | `SAMPLE BY 5m` | `time_bucket()` does not exist. SAMPLE BY is QuestDB's time aggregation. |
| `date_trunc('hour', ts)` | `timestamp_floor('1h', ts)` or `SAMPLE BY 1h` | `date_trunc()` does not exist. Use `timestamp_floor()` for expressions or SAMPLE BY for aggregation. |
| `generate_series(...)` | Use TICK bracket expansion or a helper table | `generate_series()` does not exist. |
| `SELECT DISTINCT ON (symbol) ...` | `SELECT * FROM t LATEST ON ts PARTITION BY symbol` | `DISTINCT ON` does not exist. LATEST ON is purpose-built and much faster. |
| `... HAVING avg(price) > 100` | CTE + `WHERE` on outer query | `HAVING` does not exist. Wrap in CTE/subquery, filter outside. |
| `... QUALIFY row_number() OVER (...) = 1` | CTE + `WHERE` on outer query | `QUALIFY` does not exist. Filter window results via CTE. |
| `INTERVAL '1 hour'` | `'1h'` or use `dateadd('h', 1, ts)` | PostgreSQL interval literals are not supported. |
| `ts >= NOW() - INTERVAL '1 day'` | `WHERE ts IN '$now - 1d..$now'` | Use TICK syntax for time ranges. |
| `BETWEEN '2025-01-01' AND '2025-01-31'` | `WHERE ts IN '2025-01-[01..31]'` | TICK is preferred. BETWEEN works but TICK is more expressive. |
| `ORDER BY ts ASC NULLS LAST` | `ORDER BY ts ASC` | `NULLS FIRST/LAST` is not supported. |
| `STRING` or `TEXT` | `VARCHAR` or `SYMBOL` | Use `VARCHAR` for unique strings, `SYMBOL` for repeated low-cardinality strings. |
| `BOOLEAN` column type | `BOOLEAN` exists but use with care | Supported, but prefer `SYMBOL` for filterable flag columns in high-volume tables. |
| `ALTER TABLE ... ADD CONSTRAINT` | Not supported | QuestDB has no constraints, foreign keys, or unique indexes. Use DEDUP for uniqueness. |
| `CREATE INDEX ON ...` | Not needed | QuestDB auto-indexes designated timestamp and SYMBOL columns. No manual index creation. |
| `UPSERT` / `ON CONFLICT` | `DEDUP UPSERT KEYS(...)` on CREATE TABLE | Dedup is declared at table creation, not at query time. |

## SAMPLE BY Mistakes

| ❌ Don't | ✅ Do Instead | Why |
|---|---|---|
| `SAMPLE BY '5 minutes'` | `SAMPLE BY 5m` | Units are single-letter suffixes: `s`, `m`, `h`, `d`, `M`, `y`. No quotes, no spelled-out words. |
| `SAMPLE BY 5m` without designated timestamp | Add `TIMESTAMP(col)` to CREATE TABLE | SAMPLE BY requires a designated timestamp on the table. |
| `SAMPLE BY 5m FILL(PREV) ALIGN TO CALENDAR` | `SAMPLE BY 5m FILL(PREV)` | FILL goes at the end, after SAMPLE BY (and after ALIGN TO CALENDAR if explicitly used). |
| `SELECT non_agg_col ... SAMPLE BY` without it being the timestamp | Aggregate or remove the column | Non-aggregated, non-timestamp columns in SAMPLE BY cause errors. Only the designated timestamp and aggregated expressions are valid. |
| Window functions + SAMPLE BY in same SELECT | Subquery: SAMPLE BY inside, window outside | Cannot mix window functions and SAMPLE BY at the same query level. |

## Materialized View Mistakes

| ❌ Don't | ✅ Do Instead | Why |
|---|---|---|
| `CREATE MATERIALIZED VIEW ... AS (...) REFRESH EVERY 5m` | `CREATE MATERIALIZED VIEW ... REFRESH EVERY 5m AS (...)` | REFRESH clause goes BEFORE AS, not after. |
| `INSERT INTO mat_view ...` | Insert into the BASE table | Never write directly to a materialized view. It refreshes from its source automatically. |
| `DROP TABLE mat_view` | `DROP MATERIALIZED VIEW mat_view` | Materialized views have their own DROP syntax. |
| `WITH BASE` on single-table views | Remove `WITH BASE` | WITH BASE is only needed for JOINs — it tells QuestDB which table triggers refresh. |
| Expecting cascading views to rebuild on backfill | Cascade from base → fine → coarse | Views only refresh on new inserts to the immediate source. Plan the cascade direction accordingly. |

## Window Function Mistakes

| ❌ Don't | ✅ Do Instead | Why |
|---|---|---|
| `WHERE row_num = 1` in same query as `ROW_NUMBER() OVER (...) AS row_num` | CTE: compute window in inner query, filter in outer | Cannot filter on window function results in the same query level. |
| `GROUP BY symbol` + `avg(...) OVER (...)` in same SELECT | Subquery: GROUP BY inside, window outside | Cannot mix GROUP BY/SAMPLE BY with window functions at same level. |
| Manual EMA: `price * 2/(n+1) + prev * (1 - 2/(n+1))` | `avg(price, 'period', 20) OVER (...)` | QuestDB has native EMA. Don't implement manually. |
| `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` for running total | `sum(col) OVER (ORDER BY ts CUMULATIVE)` | QuestDB has a `CUMULATIVE` shorthand. Either syntax works, but CUMULATIVE is cleaner. |
| `SELECT ts AS time, ... OVER (ORDER BY ts ...)` | Put all OVER() in CTEs where `ts` is unaliased. Final SELECT only does `ts AS time` with no OVER. | Once `ts` is aliased as `time`, OVER in the same SELECT cannot reference `ts`. Move window functions to CTEs. |

## TICK Syntax Mistakes

| ❌ Don't | ✅ Do Instead | Why |
|---|---|---|
| `WHERE ts >= dateadd('h', -1, now())` | `WHERE ts IN '$now - 1h..$now'` | TICK is preferred for time ranges. More readable and optimized. |
| `WHERE ts IN '$now - 1 hour'` | `WHERE ts IN '$now - 1h'` | TICK units are single-letter: `h` not `hour`. |
| `WHERE ts IN $now - 1h` (no quotes) | `WHERE ts IN '$now - 1h..$now'` | TICK expressions must be in single quotes. Range requires `..` between start and end. |
| `WHERE ts = '$today'` | `WHERE ts IN '$today'` | Use `IN` not `=` for TICK expressions. |
| `WHERE ts IN '$now - 5 business days'` | `WHERE ts IN '$now - 5bd..$now'` | Business days use `bd` suffix. |

## Array Mistakes

| ❌ Don't | ✅ Do Instead | Why |
|---|---|---|
| `bids[0]` | `bids[1]` | QuestDB arrays are **1-indexed**. `[1]` is the first element. |
| `bids[0][0]` for best bid price | `bids[1][1]` | 2D: `bids[1]` = prices array, `bids[2]` = sizes array. `bids[1][1]` = best bid price. |
| Python: `columns={'bids': [[41999.0], [2.1]]}` | `columns={'bids': [np.array([41999.0], dtype=np.float64), np.array([2.1], dtype=np.float64)]}` | ILP client requires numpy arrays with explicit `np.float64` dtype, not Python lists. |
| `DOUBLE[]` for order book | `DOUBLE[][]` (2D) | Use 2D arrays: dimension 1 = prices, dimension 2 = sizes. |

## Ingestion Mistakes

| ❌ Don't | ✅ Do Instead | Why |
|---|---|---|
| `INSERT INTO trades VALUES (...)` for streaming | Use ILP client (`questdb.ingress.Sender`) | INSERT is slow for streaming. ILP is 10-100x faster and supports batching. |
| `Sender.from_conf(conf)` then immediately `.row(...)` | Use `with Sender.from_conf(conf) as sender:` | `from_conf()` does not connect. The context manager (or `.establish()`) opens the connection. |
| `sender.row(... symbols={'price': 42000})` | `columns={'price': 42000.0}` | `symbols={}` is for SYMBOL type columns only (strings). Numeric values go in `columns={}`. |
| HTTP POST to `/exec` for queries | HTTP GET to `/exec?query=...` | The exec endpoint only accepts GET requests. |
| `import psycopg2` | `import psycopg as pg` | Use `psycopg` (v3), not `psycopg2`. |
| `dbname='questdb'` in PG connection | `dbname='qdb'` | QuestDB's database name is `qdb`, not `questdb`. |
| `sender.flush()` after every row | Batch rows, flush periodically | Flushing per-row kills throughput. Flush every N rows or on a timer. |
| TCP without `protocol_version=2` when using arrays | `tcp::addr=...;protocol_version=2;` | Array support requires protocol version 2. |

## Grafana Mistakes

| ❌ Don't | ✅ Do Instead | Why |
|---|---|---|
| `WHERE ts >= $__from AND ts <= $__to` | `WHERE $__timeFilter(ts)` | Use the QuestDB datasource macro. It handles the time range correctly. |
| `SAMPLE BY 1m` (hardcoded) with Grafana | `SAMPLE BY $__interval` | Let Grafana auto-calculate the interval based on the visible time range and panel width. |
| PostgreSQL datasource for QuestDB | QuestDB datasource plugin (`questdb-questdb-datasource`) | The QuestDB plugin has optimized macros and better compatibility. |
| `datasource: 'QuestDB'` (by name) in provisioned dashboards | `datasource: { uid: 'questdb-ds-uid', type: 'questdb-questdb-datasource' }` | Always reference datasources by UID and type, not display name. |

## Schema Design Mistakes

| ❌ Don't | ✅ Do Instead | Why |
|---|---|---|
| `VARCHAR` for ticker symbols | `SYMBOL` | SYMBOL is indexed and optimized for repeated low-cardinality strings. VARCHAR is not indexed. |
| Table without `TIMESTAMP(col)` | Always designate a timestamp column | Without it, you cannot use SAMPLE BY, LATEST ON, or ASOF JOIN. |
| Table without `WAL` | Add `WAL` to CREATE TABLE | WAL enables concurrent writes. Without it, only one writer is allowed. |
| Table without `PARTITION BY` | Add `PARTITION BY DAY` (or appropriate granularity) | Unpartitioned tables cannot use TTL and have worse query performance on large datasets. |
| Separate `date` and `time` columns | Single `TIMESTAMP` column | QuestDB is optimized for a single designated timestamp. Split date/time prevents using time-series features. |

## Enterprise Auth Mistakes

| ❌ Don't | ✅ Do Instead | Why |
|---|---|---|
| Skip auth setup on Enterprise | Create user/service account + grants + tokens | Enterprise requires explicit auth. No anonymous access. |
| `GRANT INSERT` to a user who creates their own tables | Skip it — creator auto-gets all permissions | Creating a table automatically grants full permissions on it to the creator. |
| `http::` with a REST token on Enterprise | `https::` (note the `s`) | Enterprise with tokens requires TLS. Use `https::` for HTTP, `tcps::` for TCP. |
| `tcp::` with JWK tokens on Enterprise | `tcps::` (note the `s`) | Same — TLS required for authenticated connections. |
| Use TCP/ILP without error handling | Prefer HTTP ingestion, or add monitoring for TCP | TCP/ILP does not return errors to the client. Data loss happens silently on disconnect. |
| Hardcode tokens in source code | Use environment variables or secrets manager | Tokens are sensitive credentials. Pass via `QDB_TOKEN`, `QDB_ILP_USER`, etc. |
| Pass REST token params to ILP config | REST uses `token=`. ILP uses `username=`, `token=` (private key), `token_x=`, `token_y=`. | REST and JWK tokens have completely different connection string parameters. |
| Forget `protocol_version=2` with `tcps::` | Always include `protocol_version=2;` in TCP config | Required for array support and modern protocol features, regardless of auth. |
| `tls_verify=unsafe_off` in production without asking | Ask the user if the server uses a self-signed certificate | Only disable TLS verification for self-signed certs. Never add silently for production. |
| Omit `tls_verify=unsafe_off` with self-signed certs | Add `tls_verify=unsafe_off;` to the connection string | Without it, the client will reject the self-signed certificate and fail to connect. |

## Grafana Dashboard Mistakes

| ❌ Don't | ✅ Do Instead | Why |
|---|---|---|
| Put OHLC + VWAP + RSI + Bollinger in separate panels | Use one candlestick panel with multiple queries (refIDs) and `includeAllFields: true` | Overlaying indicators on candles in one panel is standard for financial dashboards. |
| Use `stddev_samp()` in window functions | Compute manually: `sqrt(avg(x*x) - avg(x)^2)` | `stddev_samp` may not work inside window frames. Manual variance is more compatible. |
| Use a single time range for all panels | Use per-panel `timeFrom` overrides (e.g. `"1m"`, `"30m"`, `"now/d"`) | Different panels need different ranges: real-time tables need seconds, indicators need hours. |
| Forget `hideTimeOverride: true` on panels with `timeFrom` | Always set `hideTimeOverride: true` | Otherwise Grafana shows an ugly time override label in the panel title. |
| Create one panel per symbol manually | Use `repeat: "SYMBOL"` on panels or rows with `repeatDirection: "h"` | Auto-duplicates per symbol. Much cleaner and scales with symbol count. |
| Use `SELECT DISTINCT symbol FROM trades` for dropdown | Use LATEST ON to get only symbols with recent data | `LATEST ON` avoids returning dead symbols with no recent data. |
| Reference datasource by name | Always use UID and type: `{"uid": "...", "type": "questdb-questdb-datasource"}` | Names can change. UIDs are stable. |
| Assume all indicators share the same Y-axis | Use `byFrameRefID` overrides to give RSI its own axis and unit | RSI is 0-100, price is in dollars. They need separate axes. |

