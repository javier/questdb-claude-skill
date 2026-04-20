# QuestDB SQL Grammar Reference

Authoritative index of keywords, functions, data types, and operators available
in QuestDB. Sourced from the sql-parser grammar and verified against live docs.

**Authoritative grammar source:**
https://github.com/questdb/sql-parser/tree/main/src/grammar
(`keywords.ts`, `functions.ts`, `dataTypes.ts`, `operators.ts`, `constants.ts`)

**When in doubt, fetch the doc page:**
```bash
curl -sH "Accept: text/markdown" "https://questdb.com/docs/query/functions/finance.md"
curl -s "https://questdb.com/docs/llms.txt"   # full index
```

---

## Notable Keywords

These keywords signal QuestDB-specific features that LLMs commonly miss or
assume don't exist. Standard SQL keywords (SELECT, FROM, WHERE, etc.) are
omitted.

### Time-series query keywords
| Keyword | Feature | Doc path |
|---------|---------|----------|
| `SAMPLE` | Time-bucketed aggregation (`SAMPLE BY 5m`) | `query/sql/sample-by.md` |
| `LATEST` | Most recent row per group (`LATEST ON ts PARTITION BY symbol`) | `query/sql/latest-on.md` |
| `ASOF` | Time-series join: nearest ts <= left | `query/sql/asof-join.md` |
| `LT` | Time-series join: strictly ts < left | `query/sql/join.md` |
| `SPLICE` | Interleave two time series | `query/sql/join.md` |
| `HORIZON` | ASOF at a grid of time offsets in one parallel pass | `query/sql/horizon-join.md` |
| `WINDOW` | Aggregate right-table rows within +- time window | `query/sql/window-join.md` |
| `LATERAL` | Per-outer-row subquery (top-N, dynamic filters) | `query/sql/lateral-join.md` |
| `FILL` | Gap-filling after SAMPLE BY (`FILL(PREV\|NULL\|LINEAR\|value)`) | `query/sql/sample-by.md` |
| `ALIGN` | `ALIGN TO CALENDAR` for SAMPLE BY bucket alignment | `query/sql/sample-by.md` |
| `CUMULATIVE` | Window frame shorthand for running totals | `query/functions/window-functions/overview.md` |

### Data reshaping
| Keyword | Feature | Doc path |
|---------|---------|----------|
| `PIVOT` | Reshape rows to columns | `query/sql/pivot.md` |
| `UNPIVOT` | Reshape columns to rows | `query/sql/unpivot.md` |
| `UNNEST` | Expand arrays/JSON arrays into rows | `query/sql/unnest.md` |

### HORIZON/WINDOW JOIN modifiers
| Keyword | Context |
|---------|---------|
| `RANGE` | HORIZON JOIN: `RANGE FROM -1m TO 5m STEP 30s`; WINDOW JOIN: `RANGE BETWEEN 5 seconds PRECEDING AND 5 seconds FOLLOWING` |
| `LIST` | HORIZON JOIN: explicit offsets `LIST (0, 5s, 30s, 1m)` |
| `STEP` | HORIZON JOIN: interval step in RANGE form |
| `PREVAILING` | WINDOW JOIN: `INCLUDE PREVAILING` (default) / `EXCLUDE PREVAILING` |
| `TOLERANCE` | ASOF JOIN tolerance bound |
| `PRECEDING` | Window/WINDOW JOIN: time or row boundary |
| `FOLLOWING` | Window/WINDOW JOIN: time or row boundary |
| `ORDINALITY` | `UNNEST ... WITH ORDINALITY` adds a 1-based index column |

### DDL / table management
| Keyword | Feature |
|---------|---------|
| `WAL` | Write-ahead log (concurrent writes) |
| `DEDUP` | Deduplication on insert (`DEDUP UPSERT KEYS(...)`) |
| `UPSERT` | Part of `DEDUP UPSERT KEYS(...)` |
| `MATERIALIZED` | Materialized views (`CREATE MATERIALIZED VIEW ... AS (... SAMPLE BY ...)`) |
| `TTL` | Automatic partition expiry |
| `PARTITION` | `PARTITION BY DAY\|MONTH\|YEAR\|HOUR` |
| `DECLARE` | SQL variables (`DECLARE @x := ...`) |
| `DETACH` / `ATTACH` | Partition management |
| `SQUASH` | Merge WAL partitions |

### Enterprise / auth
| Keyword | Feature |
|---------|---------|
| `GRANT` / `REVOKE` | Permission management |
| `USER` / `USERS` | User management |
| `SERVICE` | Service accounts |
| `TOKEN` | Auth tokens |
| `ASSUME` | Assume service account identity |

---

## Data Types

Complete list. Use `SYMBOL` for any repeated low-cardinality string (tickers,
categories, status codes) - it is indexed and much faster than VARCHAR.

| Type | Description |
|------|-------------|
| `BOOLEAN` | true/false |
| `BYTE` | 8-bit signed integer |
| `SHORT` | 16-bit signed integer |
| `INT` / `INTEGER` | 32-bit signed integer |
| `LONG` | 64-bit signed integer |
| `LONG128` | 128-bit signed integer |
| `LONG256` | 256-bit unsigned integer |
| `FLOAT` | 32-bit IEEE 754 |
| `DOUBLE` | 64-bit IEEE 754 |
| `DECIMAL` | Arbitrary-precision decimal |
| `CHAR` | Single UTF-16 character |
| `VARCHAR` | Variable-length UTF-8 string |
| `STRING` | Legacy alias for VARCHAR |
| `SYMBOL` | Indexed, interned string (use for repeated values) |
| `TIMESTAMP` | Microsecond-precision UTC timestamp |
| `TIMESTAMP_NS` | Nanosecond-precision UTC timestamp |
| `DATE` | Millisecond-precision date |
| `INTERVAL` | Time interval (pair of timestamps) |
| `UUID` | 128-bit UUID |
| `IPv4` | IPv4 address |
| `GEOHASH` | Variable-precision geohash |
| `BINARY` | Binary data |
| `DOUBLE[]` | 1D double array |
| `DOUBLE[][]` | 2D double array (order books) |
| `FLOAT[]`, `INT[]`, `LONG[]`, `SHORT[]`, `UUID[]` | Typed arrays |

---

## Functions by Category

### Finance / Market Data

Native builtins for order book and execution analysis. **Use these instead of
manual formulas when they fit.**

| Function | Signature | Description |
|----------|-----------|-------------|
| `vwap` | `vwap(price, volume)` | Volume-weighted average price (aggregate) |
| `twap` | `twap(price, timestamp)` | Time-weighted average price using step-function integration (aggregate). Supports SAMPLE BY + FILL. |
| `mid` | `mid(bid, ask)` | Midpoint of bid and ask. Returns NULL if either is NaN/NULL. |
| `spread` | `spread(bid, ask)` | Bid-ask spread (`ask - bid`) |
| `spread_bps` | `spread_bps(bid, ask)` | Spread in basis points: `spread/mid * 10000` |
| `wmid` | `wmid(bidSize, bidPrice, askPrice, askSize)` | Size-weighted mid (imbalance-adjusted) |
| `l2price` | `l2price(target_size, size_array, price_array)` or `l2price(target_size, size1, price1, size2, price2, ...)` | Average fill price for a market order of target_size against an L2 book |
| `regr_slope` | `regr_slope(y, x)` | Slope of linear regression line. Use for beta, trend. |
| `regr_intercept` | `regr_intercept(y, x)` | Y-intercept of linear regression line |

### Aggregation - Basic

| Function | Description |
|----------|-------------|
| `avg(x)` | Arithmetic mean |
| `count()` / `count(col)` | Row count / non-NULL count |
| `count_distinct(col)` | Exact distinct count (use instead of `COUNT(DISTINCT col)`) |
| `sum(x)` | Sum |
| `min(x)` / `max(x)` | Minimum / maximum |
| `first(x)` / `last(x)` | First / last value by designated timestamp |
| `first_not_null(x)` / `last_not_null(x)` | First / last non-NULL value |
| `arg_max(value, key)` / `arg_min(value, key)` | Value at row where key is max/min |
| `ksum(x)` | Kahan compensated sum (better float precision) |
| `nsum(x)` | Neumaier sum (better float precision) |

### Aggregation - Statistical

| Function | Description |
|----------|-------------|
| `stddev(x)` / `stddev_samp(x)` | Sample standard deviation |
| `stddev_pop(x)` | Population standard deviation |
| `var_samp(x)` / `variance(x)` | Sample variance |
| `var_pop(x)` | Population variance |
| `corr(x, y)` | Pearson correlation coefficient |
| `covar_samp(x, y)` | Sample covariance |
| `covar_pop(x, y)` | Population covariance |
| `mode(x)` | Most frequent value |
| `geomean(x)` | Geometric mean (positive values) |
| `weighted_avg(val, weight)` | Weighted mean |
| `weighted_stddev(val, weight)` | Weighted stddev (reliability weights) |
| `weighted_stddev_freq(val, weight)` | Weighted stddev (frequency weights) |
| `weighted_stddev_rel(val, weight)` | Weighted stddev (reliability weights, alias) |

### Aggregation - Approximate

| Function | Description |
|----------|-------------|
| `approx_count_distinct(col, precision?)` | HyperLogLog distinct count estimate |
| `approx_median(val, precision?)` | Approximate median via HdrHistogram |
| `approx_percentile(val, percentile, precision?)` | Approximate percentile via HdrHistogram |

### Aggregation - String / Boolean / Bitwise

| Function | Description |
|----------|-------------|
| `string_agg(val, delim)` | Concatenate with delimiter |
| `string_distinct_agg(val, delim)` | Concatenate distinct values |
| `bool_and(x)` / `bool_or(x)` | All-true / any-true |
| `bit_and(x)` / `bit_or(x)` / `bit_xor(x)` | Bitwise aggregates |

### Aggregation - Array (element-wise across rows)

| Function | Description |
|----------|-------------|
| `array_elem_avg(arr)` | Element-wise average across arrays |
| `array_elem_sum(arr)` | Element-wise sum |
| `array_elem_min(arr)` / `array_elem_max(arr)` | Element-wise min/max |
| `array_agg(x)` | Collect values into an array |

### Window Functions

All work with `OVER (PARTITION BY ... ORDER BY ... ROWS/RANGE ...)`.

**Frame-respecting** (affected by ROWS BETWEEN / RANGE):
`avg`, `count`, `sum`, `ksum`, `min`, `max`, `first_value`, `last_value`,
`stddev`, `stddev_pop`, `stddev_samp`, `var_pop`, `var_samp`, `variance`,
`corr`, `covar_samp`, `covar_pop`

**Frame-ignoring** (use full partition):
`row_number`, `rank`, `dense_rank`, `percent_rank`, `lag`, `lead`

**QuestDB extensions:**
- `avg(price, 'period', N) OVER (ORDER BY ts)` - native EMA (exponential moving average)
- `sum(x) OVER (ORDER BY ts CUMULATIVE)` - shorthand for running total
- `first_value` / `last_value` support `IGNORE NULLS` / `RESPECT NULLS`
- `lag` / `lead` support `IGNORE NULLS` / `RESPECT NULLS`

### Array Functions (per-row)

| Function | Description |
|----------|-------------|
| `array_avg(arr)` | Average of elements |
| `array_sum(arr)` | Sum of elements |
| `array_count(arr)` | Count of finite elements |
| `array_min(arr)` / `array_max(arr)` | Min/max element |
| `array_cum_sum(arr)` | Cumulative sum |
| `array_sort(arr, desc?, nullsFirst?)` | Sort elements |
| `array_reverse(arr)` | Reverse order |
| `array_position(arr, elem)` | 1-based index of element |
| `array_build(nArrays, size, filler...)` | Construct array at runtime |
| `array_stddev(arr)` / `array_stddev_pop(arr)` / `array_stddev_samp(arr)` | Stddev of elements |
| `dim_length(arr, dim)` | Length along dimension |
| `dot_product(a, b)` | Dot product of two arrays |
| `matmul(a, b)` | Matrix multiplication (2D) |
| `transpose(arr)` | Transpose (reverse coordinate order) |
| `flatten(arr)` | Flatten to 1D |
| `shift(arr, dist, fill?)` | Shift elements left/right |
| `insertion_point(arr, val, ahead?)` | Binary search insertion point |

### Date-Time Functions

**Current time:**
`now()`, `now_ns()`, `sysdate()`, `systimestamp()`, `systimestamp_ns()`,
`today()`, `tomorrow()`, `yesterday()` (each optionally with timezone arg)

**Extraction:**
`year(ts)`, `month(ts)`, `day(ts)`, `hour(ts)`, `minute(ts)`, `second(ts)`,
`micros(ts)`, `millis(ts)`, `nanos(ts)`, `day_of_week(ts)`,
`day_of_week_sunday_first(ts)`, `week_of_year(ts)`, `days_in_month(ts)`,
`is_leap_year(ts)`, `extract(unit, ts)`

**Arithmetic:**
`dateadd(period, n, ts, tz?)`, `datediff(period, ts1, ts2)`,
`timestamp_floor(interval, ts, offset?)`, `timestamp_ceil(unit, ts)`,
`date_trunc(unit, ts)`

**Conversion:**
`to_timestamp(str, fmt)`, `to_timestamp_ns(str, fmt)`, `to_date(str, fmt)`,
`to_str(ts, fmt)`, `to_timezone(ts, tz)`, `to_utc(ts, tz)`

**Interval:**
`interval(start, end)`, `interval_start(iv)`, `interval_end(iv)`

### String Functions

`concat(...)`, `left(s, n)`, `right(s, n)`, `length(s)`, `length_bytes(s)`,
`substring(s, start, len)`, `split_part(s, delim, idx)`, `position(s, sub)` /
`strpos(s, sub)`, `starts_with(s, prefix)`, `replace(s, old, new)`,
`regexp_replace(s, pattern, replacement)`, `lpad(s, len, fill?)`,
`rpad(s, len, fill?)`, `ltrim(s)`, `rtrim(s)`, `trim(s)`,
`to_lowercase(s)` / `lower(s)`, `to_uppercase(s)` / `upper(s)` / `ucase(s)`,
`quote_ident(s)`

### Numeric Functions

`abs(x)`, `ceil(x)` / `ceiling(x)`, `floor(x)`, `round(x, scale?)`,
`round_up(x, scale)`, `round_down(x, scale)`, `round_half_even(x, scale)`,
`sign(x)`, `sqrt(x)`, `power(x, exp)`, `exp(x)`, `ln(x)`, `log(x)`,
`greatest(...)`, `least(...)`, `size_pretty(bytes)`,
`pi()`, `sin(x)`, `cos(x)`, `tan(x)`, `cot(x)`, `asin(x)`, `acos(x)`,
`atan(x)`, `atan2(y, x)`, `degrees(x)`, `radians(x)`

### Conditional Functions

`CASE WHEN ... THEN ... ELSE ... END`, `coalesce(...)`, `nullif(a, b)`,
`ifnull(x, default)` / `nvl(x, default)`, `isnull(x)`

### JSON Functions

`json_extract(doc, jsonpath)::TYPE` - extract typed values from VARCHAR JSON.
JSONPath syntax: `$.field`, `$.array[0]`, `$.nested.path`.
Cast required for typed output (`::double`, `::varchar`, `::timestamp`, etc.).

### Spatial / Geo Functions

`haversine_dist_deg(lat, lon, ts)` (aggregate: total distance),
`geo_distance_meters(lat1, lon1, lat2, lon2)`,
`geo_within_radius_latlon(lat, lon, clat, clon, radius_m)`,
`within_box(x, y, min_x, min_y, max_x, max_y)`,
`within_radius(x, y, cx, cy, r)`,
`make_geohash(lon, lat, bits)`

### Visualization (inline Unicode charts)

Render data as compact Unicode block charts directly in query results. Output is
`varchar` - works in psql, web console, JDBC, CSV.

**Note:** merged into main, shipping in the next release. Not yet available on
`demo.questdb.io` (which runs the latest stable release). Doc preview:
`https://preview-418--questdb-documentation.netlify.app/docs/query/functions/visualization.md`
(will move to `query/functions/visualization.md` on release).

| Function | Type | Signature | Description |
|----------|------|-----------|-------------|
| `bar` | Scalar | `bar(value, min, max, width)` | Horizontal bar proportional to value within [min, max]. Width = max characters. Uses `▏▎▍▌▋▊▉█` (8 sub-levels per character). Wraps aggregates: `bar(sum(amount), 0, 50, 30)`. |
| `sparkline` | Aggregate | `sparkline(value)` or `sparkline(value, min, max, width)` | Vertical block chart of values within a group. Uses `▁▂▃▄▅▆▇█`. Each value = one character. Optional `width` sub-samples into buckets. min/max can be NULL for auto-scaling. Pairs with SAMPLE BY. |

### Parquet / External Data

`read_parquet(file_path)` - read a Parquet file as a table (beta).
`generate_series(start, end)` - generate a sequence of integers.

### Hashing / Encoding

`md5(x)`, `sha1(x)`, `sha256(x)`, `base64(x)`

### System / Introspection

These are real functions but rarely needed in analytics queries:

`tables()`, `table_columns(name)`, `table_partitions(name)`,
`table_storage(name)`, `materialized_views()`, `views()`,
`wal_tables()`, `wal_transactions(name)`,
`server_version()`, `version()`, `current_database()`,
`query_activity()`, `reader_pool()`, `writer_pool()`,
`memory_metrics()`, `table_writer_metrics()`,
`typeOf(expr)`, `all_permissions()`, `permissions(user)`

### Random Data Generation (test data)

`rnd_boolean()`, `rnd_byte(min, max)`, `rnd_short(min, max)`,
`rnd_int(min, max, nanRate)`, `rnd_long(min, max, nanRate)`,
`rnd_float(nanRate)`, `rnd_double(nanRate)`, `rnd_double_array(len, nanRate)`,
`rnd_str(...)`, `rnd_symbol(...)`, `rnd_symbol_weighted(...)`,
`rnd_varchar(minLen, maxLen, nanRate)`, `rnd_char()`,
`rnd_date(min, max, nanRate)`, `rnd_timestamp(min, max, nanRate)`,
`rnd_timestamp_ns(min, max, nanRate)`,
`rnd_uuid4()`, `rnd_ipv4(...)`, `rnd_bin(minLen, maxLen, nanRate)`,
`rnd_geohash(bits)`, `rnd_long256()`,
`long_sequence(count)`, `timestamp_sequence(start, step)`,
`timestamp_sequence_ns(start, step)`, `timestamp_shuffle(ts1, ts2)`

---

## Operators

Standard SQL operators plus:

| Operator | Description |
|----------|-------------|
| `~` | Regex match |
| `!~` | Regex not match |
| `~=` | Regex match (case insensitive) |
| `ILIKE` | Case-insensitive LIKE |
| `IN` | Set membership / TICK interval syntax |
| `BETWEEN` | Range check |
| `IS [NOT] NULL` | NULL check |
| `PIVOT` / `UNPIVOT` | As operators in FROM clause |
| `MATCHED` | Used in MERGE statements |

All doc paths are relative to `https://questdb.com/docs/`.
