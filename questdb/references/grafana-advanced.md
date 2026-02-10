# Advanced Grafana Dashboard Techniques

These patterns come from production QuestDB financial dashboards.

## Multiple Indicators in One Candlestick Panel

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

## Per-Query Axis and Styling Overrides (byFrameRefID)

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

## Per-Panel Time Range Overrides

Different panels can show different time windows, independent of the dashboard picker:
```json
{
  "timeFrom": "1m",
  "hideTimeOverride": true
}
```
Common values: `"1m"` for real-time tables, `"30m"` for volume charts, `"now/d"` for
daily indicators. Always set `hideTimeOverride: true` to keep the panel title clean.

## Time Shift for Ingestion Latency

Offset the time range slightly to account for near-real-time data:
```json
{
  "timeFrom": "1m",
  "timeShift": "5s"
}
```

## Repeating Panels Per Symbol

Set `repeat` on a panel or row to auto-duplicate per symbol variable value:
```json
{
  "repeat": "SYMBOL",
  "repeatDirection": "h",
  "maxPerRow": 2
}
```
For section headers, use a collapsed row with `"type": "row"` and `"repeat": "SYMBOL"`.

## Symbol Dropdown Variable Using LATEST ON

Get only symbols with recent data, not all historical symbols:
```sql
SELECT DISTINCT symbol FROM (
    SELECT symbol FROM trades
    WHERE timestamp > dateadd('d', -4, now())
    LATEST ON timestamp PARTITION BY symbol
) ORDER BY symbol;
```

## 2D Array Indexing for Order Book Queries

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

## Market Depth with Plotly Plugin (`ae3e-plotly-panel`)

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
