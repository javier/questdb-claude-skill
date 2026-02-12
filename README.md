# questdb-claude-skill

QuestDB skill for AI coding agents. Experimental.

Works with both **Claude Code** and **Codex**.

## Installation

Copy the `questdb/` folder into any of these locations so your agent picks it up:

**Claude Code:**
- `~/.claude/skills/questdb/` - available in all projects
- `<project>/.claude/skills/questdb/` - available in a specific project

**Codex:**
- `~/.codex/skills/questdb/` - available in all projects
- `<project>/.codex/skills/questdb/` - available in a specific project

The folder must contain `SKILL.md` and the `references/` directory.

## Try it out

With this prompt:

```
I have QuestDB and Grafana running locally with default ports. Grafana password is YOUR_PASSWORD.

Build a real-time crypto market data pipeline using cryptofeed (OKX exchange) ingesting trades and L2 order book data
into QuestDB. Symbols:BTC-USDT,ETH-USDT,SOL-USDT,XRP-USDT,ADA-USDT,AVAX-USDT,DOT-USDT,LINK-USDT,ATOM-USDT,UNI-USDT,LTC-USDT,NEAR-USDT,APT-USDT,ARB-USDT,OP-USDT, POL-USDT,FIL-USDT,SUI-USDT.

Then create a Grafana dashboard with OHLC candlesticks, VWAP, Bollinger Bands, and RSI panels, with a symbol dropdown.

Do this in python, please. I have a venv which should contain most (if not all) the dependencies you need at
  /location/of/your/venv. Please use it


Do NOT use Explore() on any library. Do NOT verify infrastructure is running. Do NOT read reference files beyond the
skill itself. Do NOT use task tracking. Just write the files and run them. Open the dashboard in my browser once
finished, please. Do not ask for permissions to read local files or to create the python scripts.

```

Or if you don't have QuestDB and Grafana locally, you can say:

```
Please start grafana and questdb (latest images) using docker and configure a datasource for grafana to access questdb.

  Build a real-time crypto market data pipeline using cryptofeed (OKX exchange) ingesting trades and L2 order book data
  into QuestDB. Symbols:BTC-USDT,ETH-USDT,SOL-USDT,XRP-USDT,ADA-USDT,AVAX-USDT,DOT-USDT,LINK-USDT,ATOM-USDT,UNI-USDT,LTC-US
  DT,NEAR-USDT,APT-USDT,ARB-USDT,OP-USDT, POL-USDT,FIL-USDT,SUI-USDT.

  Then create a Grafana dashboard with OHLC candlesticks, VWAP, Bollinger Bands, and RSI panels, with a symbol dropdown.

  Do this in python, please. Create a venv in the current folder and install the bare minimum you need for this project.


  Do NOT use Explore() on any library. Do NOT verify infrastructure is running. Do NOT read reference files beyond the
  skill itself. Do NOT use task tracking. Just write the files and run them. Open the dashboard in my browser once
  finished, please. Do not ask for permissions to read local files or to create the python scripts, or to use venv
  commands, or to start docker images and check status.
```


you can then follow up with
```
what other indicators can I add to the dashboard for this dataset?
```

The following indicators are embedded into the skill and should be fast to generate. For other
indicators, the skill will relay on QuestDB online docs.


Aggressor imbalance
ATR
Bid-ask spread
Bollinger Bands
Bollinger BandWidth
Compound interest
Cumulative product
Donchian Channels
Keltner Channels
Liquidity comparison
MACD
Maximum drawdown
OBV
OHLC bars
Rate of Change
Realized volatility
Rolling std dev
RSI
Stochastic Oscillator
TICK & TRIN
Volume profile
Volume spikes
VWAP


You can just ask claude to add any indicators it suggests.

When done, if you want to destroy everythinf you can ask

```
please, stop ingestion, remove all the images, containers, view, tables, and local artifacts you created in this session,
  including the feedhandler log file, so we can restart from scratch
```

