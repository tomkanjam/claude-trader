# Python vs TypeScript for Claude Trader

## Recommendation: **Python** ✅

### Why Python Wins for Claude Trader

#### 1. Technical Indicators Are Essential
Your Strategy Builder will need to provide sub-agents with tools to calculate:
- RSI, MACD, Bollinger Bands, ATR, etc.
- **Python**: One line with `talib.RSI(prices, 14)`
- **TypeScript**: Implement 50+ indicators from scratch

#### 2. Exchange Integrations
```python
# Python: Works with 100+ exchanges
import ccxt
exchange = ccxt.binance()

# TypeScript: Manually integrate each one
```

#### 3. Data Analysis
```python
# Python: Pandas is unmatched
df['returns'] = df['close'].pct_change()
df['volatility'] = df['returns'].rolling(30).std()

# TypeScript: Manual array manipulation
```

#### 4. Strategy Builder Can Generate Better Code
Claude is better at generating Python for trading because:
- More training data (most trading code is Python)
- Libraries are well-documented
- Sub-agents can use `pandas`, `numpy` naturally

#### 5. Open Source Community
- Traders know Python >> TypeScript
- More contributors
- More strategy examples to learn from

#### 6. Future Backtesting
When you add backtesting, Python has mature frameworks:
- Backtrader
- Zipline
- VectorBT

TypeScript: Build from scratch

---

## Simplified Python Architecture

### Tech Stack

| Component | Python Choice | Why |
|-----------|---------------|-----|
| Runtime | Python 3.11+ | Fast, mature |
| AI SDK | `anthropic-ai/claude-agent-sdk` | Official support |
| Web Framework | FastAPI | Fast, async, type hints |
| Database | DuckDB | Same as TypeScript plan |
| Scheduler | `schedule` | Simple, Pythonic |
| Indicators | `TA-Lib` + `pandas-ta` | 150+ indicators |
| Exchange APIs | `ccxt` | 100+ exchanges |
| Data Analysis | `pandas` + `numpy` | Industry standard |
| Deployment | Docker → Fly.io | Same as TypeScript |

### Project Structure

```
claude-trader/
├── src/
│   ├── agents/
│   │   ├── strategy_builder.py      # Meta-agent
│   │   └── strategy_executor.py     # Runs strategies
│   ├── tools/
│   │   ├── market_data.py           # ccxt integration
│   │   ├── indicators.py            # TA-Lib wrappers
│   │   └── blockchain.py            # On-chain data
│   ├── scheduler/
│   │   └── cron.py                  # schedule library
│   ├── storage/
│   │   ├── duckdb.py                # DuckDB client
│   │   └── schema.sql
│   └── main.py
├── strategies/                      # Generated strategies
│   └── btc-rsi-whale/
│       ├── config.json
│       └── agents.json
├── data/
│   └── claude_trader.duckdb
├── requirements.txt
├── Dockerfile
└── fly.toml
```

### Example: Python MCP Tools

```python
from anthropic_agent import tool, create_sdk_mcp_server
import ccxt
import talib
import pandas as pd

# Market data tools with ccxt
market_tools = create_sdk_mcp_server(
    name="market",
    version="1.0.0",
    tools=[
        tool(
            name="get_price",
            description="Get cryptocurrency price",
            parameters={
                "symbol": {"type": "string", "description": "Trading pair (e.g., BTC/USDT)"},
                "exchange": {"type": "string", "default": "binance"}
            },
            handler=async def get_price(symbol, exchange="binance"):
                ex = getattr(ccxt, exchange)()
                ticker = ex.fetch_ticker(symbol)
                return {"price": ticker['last'], "volume": ticker['volume']}
        ),

        tool(
            name="calculate_rsi",
            description="Calculate RSI indicator",
            parameters={
                "symbol": {"type": "string"},
                "period": {"type": "integer", "default": 14},
                "timeframe": {"type": "string", "default": "1h"}
            },
            handler=async def calculate_rsi(symbol, period=14, timeframe="1h"):
                exchange = ccxt.binance()
                ohlcv = exchange.fetch_ohlcv(symbol, timeframe, limit=100)
                df = pd.DataFrame(ohlcv, columns=['time', 'open', 'high', 'low', 'close', 'volume'])

                # One line!
                rsi = talib.RSI(df['close'].values, timeperiod=period)

                return {
                    "current_rsi": float(rsi[-1]),
                    "overbought": rsi[-1] > 70,
                    "oversold": rsi[-1] < 30,
                    "trend": "bullish" if rsi[-1] > 50 else "bearish"
                }
        ),

        tool(
            name="detect_divergence",
            description="Detect RSI divergences",
            parameters={
                "symbol": {"type": "string"},
                "lookback": {"type": "integer", "default": 20}
            },
            handler=async def detect_divergence(symbol, lookback=20):
                exchange = ccxt.binance()
                ohlcv = exchange.fetch_ohlcv(symbol, '4h', limit=lookback)
                df = pd.DataFrame(ohlcv, columns=['time', 'open', 'high', 'low', 'close', 'volume'])

                # Calculate RSI
                df['rsi'] = talib.RSI(df['close'], timeperiod=14)

                # Detect bullish divergence (price lower low, RSI higher low)
                price_lower_low = df['close'].iloc[-1] < df['close'].iloc[-5]
                rsi_higher_low = df['rsi'].iloc[-1] > df['rsi'].iloc[-5]

                if price_lower_low and rsi_higher_low:
                    return {"divergence": "bullish", "strength": 75, "confidence": 80}

                return {"divergence": "none", "strength": 0, "confidence": 50}
        )
    ]
)
```

### Example: Strategy Executor

```python
from anthropic_agent import query
import json

async def execute_strategy(strategy_id: str):
    # Load strategy
    with open(f'strategies/{strategy_id}/config.json') as f:
        config = json.load(f)

    with open(f'strategies/{strategy_id}/agents.json') as f:
        agents = json.load(f)

    # Orchestrator prompt
    prompt = f"""You are the orchestrator for: {config['name']}

Strategy: {config['description']}

Steps:
1. Use /agent price-monitor to get current price
2. Use /agent rsi-analyzer to calculate RSI and detect divergences
3. Use /agent volume-analyzer to check volume trends

Synthesize results into a trading signal.

Return JSON:
{{
  "signal": "BUY" | "SELL" | "HOLD",
  "confidence": 0-100,
  "reasoning": "explanation"
}}"""

    # Execute with Claude SDK
    async for msg in query(
        prompt=prompt,
        options={
            "agents": agents,
            "mcp_servers": {
                "market": market_tools
            },
            "max_turns": 10
        }
    ):
        if msg.type == "result" and msg.subtype == "success":
            result = json.loads(msg.result)

            # Save to DuckDB
            db.execute("""
                INSERT INTO analysis_results (strategy_id, signal, confidence, reasoning, data)
                VALUES (?, ?, ?, ?, ?)
            """, [strategy_id, result['signal'], result['confidence'], result['reasoning'], json.dumps(result)])

            return result
```

### Example: Scheduler

```python
import schedule
import time
from datetime import datetime

def run_strategy(strategy_id):
    print(f"[{datetime.now()}] Running {strategy_id}")
    result = execute_strategy(strategy_id)
    print(f"Signal: {result['signal']}, Confidence: {result['confidence']}%")

# Load active strategies from DuckDB
strategies = db.execute("SELECT id, interval FROM strategies WHERE status = 'active'").fetchall()

for strategy_id, interval in strategies:
    if interval == '1m':
        schedule.every(1).minutes.do(run_strategy, strategy_id)
    elif interval == '5m':
        schedule.every(5).minutes.do(run_strategy, strategy_id)
    elif interval == '15m':
        schedule.every(15).minutes.do(run_strategy, strategy_id)
    elif interval == '1h':
        schedule.every().hour.do(run_strategy, strategy_id)

# Run scheduler
while True:
    schedule.run_pending()
    time.sleep(1)
```

---

## When TypeScript Would Be Better

1. **Heavy web UI focus** - If 80% of the app is a React dashboard
2. **No indicator calculations** - If you only fetch prices
3. **Team knows TypeScript** - If contributors don't know Python
4. **Microservices** - If splitting into many services (but we're not)

---

## Recommendation

### Go with Python ✅

**Why:**
1. Technical indicators are essential for trading strategies
2. Exchange integrations via `ccxt` save months of work
3. Strategy Builder will generate better Python code
4. Open source community is larger in Python for trading
5. Future backtesting requires Python libraries
6. Data analysis with pandas is unmatched

**Trade-offs:**
- Web UI might be slightly harder (but FastAPI is excellent)
- Slightly more memory usage than Node.js
- Need to learn Python SDK (but it's very similar to TypeScript)

### Migration Path

1. Keep all architecture decisions (DuckDB, self-contained, Fly.io)
2. Replace TypeScript → Python in examples
3. Add `ccxt`, `talib`, `pandas` for trading capabilities
4. Use FastAPI if you need web API (optional)
5. Everything else stays the same!

---

## Next Steps

Should I:
1. **Rewrite ARCHITECTURE.md for Python**? (Update all code examples)
2. **Research Python Claude Agent SDK**? (Like we did for TypeScript)
3. **Keep TypeScript**? (If you have strong reasons)

Let me know!
