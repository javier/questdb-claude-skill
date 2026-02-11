# questdb-claude-skill

QuestDB Claude Skill. Experimental

Try it out with this prompt:

```
I have QuestDB and Grafana running locally with default ports. Grafana password is YOUR_PASSWORD. Build a real-time crypto market data pipeline
using cryptofeed (OKX exchange) ingesting trades and L2 order book data into QuestDB.
Symbols:BTC-USDT,ETH-USDT,SOL-USDT,XRP-USDT,ADA-USDT,AVAX-USDT,DOT-USDT,LINK-USDT,ATOM-USDT,UNI-USDT,LTC-USDT,NEAR-USDT,APT-USDT,ARB-USDT,OP-USDT,
POL-USDT,FIL-USDT,SUI-USDT. Then create a Grafana dashboard with OHLC candlesticks, VWAP, Bollinger Bands, and RSI panels, with a symbol
dropdown. Do this in python, please. I have a venv which should contain most (if not all) the dependencies you need at
/Users/j/prj/python/questdb_dataset_utils/venv. Please use it


Do NOT use Explore() on any library. Do NOT verify infrastructure is running. Do NOT read reference files beyond the skill itself. Do NOT use task
tracking. Just write the files and run them. Open the dashboard in my browser once finished, please. Do not ask for permissions to read local
files, skill reference files, or to create the python scripts.
```

All of these indicators are embedded into the skill and should be fast to generate. For other
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

