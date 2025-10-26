# Claude Trader - Product Requirements Document

## 1. Overview

Claude Trader is an AI-native trading analysis system built on the Claude Agent SDK. Users describe their trading strategies in natural language, and Claude generates custom AI agents, tools, and orchestration code tailored to that strategy. These generated agents autonomously gather market data, perform analysis, and generate trading insights at configurable intervals.

### Goals
- **AI-native strategy creation**: Claude builds custom agents and code from natural language descriptions
- **Self-generating architecture**: Each strategy generates its own specialized sub-agents, tools, and orchestration logic
- Leverage Claude's multi-agent capabilities for parallel data gathering and analysis
- Provide flexible, configurable analysis intervals (1m, 5m, 15m, 1h, 4h, 1D)
- Support multiple concurrent strategies with isolated execution contexts

### Core Insight
Traditional approach: Generic agents read static configuration files.
**Claude Trader approach**: Claude generates custom agents that generate analysis. **Claude builds Claude agents!**

## 2. User Workflow

### 2.1 Strategy Creation (Interactive)

**Step 1: Natural Language Input**
```
User: "I want to trade BTC based on RSI divergences and whale wallet movements.
       Buy when RSI shows bullish divergence AND whales are accumulating."
```

**Step 2: Strategy Builder Agent**

The **Strategy Builder Agent** (meta-agent) is launched and:

1. **Analyzes Requirements**
   - Identifies trading asset, signals, logic, risk parameters
   - Determines required data sources (price data, on-chain data, etc.)
   - Plans the agent architecture needed

2. **Asks Clarifying Questions**
   ```
   "I understand you want to track whale movements. Should I:
   1. Monitor top 100 BTC holders?
   2. Track specific wallet addresses you provide?
   3. Monitor exchange cold wallets?

   Also: RSI period? (default: 14), Timeframe? (1h, 4h, 1D?), Analysis interval?"
   ```

3. **Generates Complete Strategy Package**

   Creates `/strategies/btc-rsi-whale/`:
   ```
   strategy.md                    # Human-readable strategy documentation
   config.json                    # Intervals, risk params, data sources
   /agents/
     whale-monitor.ts             # GENERATED: Custom sub-agent code
     rsi-divergence-detector.ts   # GENERATED: Custom sub-agent code
     orchestrator.ts              # GENERATED: Strategy coordination logic
   /tools/
     blockchain-scanner.ts        # GENERATED: On-chain data fetcher
     rsi-calculator.ts            # GENERATED: RSI + divergence detection
   /data-sources/
     whale-addresses.json         # Wallets to monitor
     api-config.json              # API endpoints and configuration
   ```

4. **Generates Custom Sub-Agent Code**

   Example: `whale-monitor.ts`
   ```typescript
   import { Agent } from '@anthropic-ai/agent-sdk';

   export const whaleMonitorAgent = new Agent({
     name: 'whale-monitor',
     instructions: `You are a whale wallet monitoring specialist for Bitcoin.

     Your job: Track the wallets in whale-addresses.json and determine if they
     are accumulating or distributing BTC.

     Use the blockchain-scanner tool to fetch recent transactions.

     Return format:
     {
       accumulation_score: 0-100,
       recent_activity: string,
       notable_movements: array,
       confidence: 0-100
     }`,
     tools: ['blockchain-scanner']
   });
   ```

   Example: `orchestrator.ts`
   ```typescript
   // GENERATED ORCHESTRATION LOGIC
   export async function analyzeStrategy() {
     // Spawn sub-agents in parallel
     const [whaleData, rsiData] = await Promise.all([
       whaleMonitorAgent.run(),
       rsiDivergenceAgent.run()
     ]);

     // Strategy-specific synthesis logic
     let signal = 'HOLD';
     let confidence = 0;

     if (rsiData.divergence_type === 'bullish' &&
         whaleData.accumulation_score > 70) {
       signal = 'BUY';
       confidence = Math.min(rsiData.confidence, whaleData.confidence);
     }

     return { signal, confidence, whaleData, rsiData };
   }
   ```

5. **User Review & Iteration**
   ```
   Agent: "I've created your BTC RSI-Whale strategy. Here's what I built:

   - whale-monitor sub-agent: Tracks top 50 BTC wallets
   - rsi-divergence-detector sub-agent: Detects divergences on 4h timeframe
   - Custom orchestrator: Generates BUY when both conditions align
   - Analysis interval: 15 minutes

   Review generated code at /strategies/btc-rsi-whale/

   Ready to:
   1. Run test analysis
   2. Modify parameters
   3. Activate strategy"
   ```

   User can iterate: "Change to 1h timeframe" → Agent updates generated code

6. **Activation**
   ```
   User: "Activate it"
   Agent: "✓ Strategy activated. First analysis runs in 15 minutes."
   ```

### 2.2 Strategy Execution (Automated)

**At Each Interval (e.g., every 15 minutes):**

1. **Scheduler** triggers strategy `btc-rsi-whale`
2. **Strategy Orchestrator** executes:
   ```
   - Loads strategy context (with prompt caching)
   - Spawns whale-monitor sub-agent → fetches on-chain data
   - Spawns rsi-divergence-detector sub-agent → analyzes price/RSI
   - Sub-agents run in parallel
   - Orchestrator receives focused results (not full context)
   - Synthesizes signal using custom logic
   - Writes analysis report
   ```

3. **Output**: `/analysis/btc-rsi-whale/2025-10-26-14-30.md`
   ```markdown
   # BTC RSI-Whale Analysis - 2025-10-26 14:30

   **Signal**: HOLD
   **Confidence**: 65%

   ## RSI Divergence Analysis
   - Status: No divergence detected
   - RSI: 52 (neutral)
   - Timeframe: 4h
   - Confidence: 80%

   ## Whale Activity
   - Accumulation Score: 45/100
   - Recent Activity: Mixed (3 accumulating, 2 distributing)
   - Notable: Wallet 0x742d... moved 500 BTC to Binance (bearish signal)
   - Confidence: 75%

   ## Recommendation
   Conditions not met. Waiting for bullish divergence signal.
   Next check: 2025-10-26 14:45
   ```

### 2.3 Strategy Management

**Ongoing Interaction:**
- "Show active strategies" → List all running strategies
- "Why didn't btc-rsi-whale trigger?" → Detailed reasoning from last analysis
- "Make btc-rsi-whale more aggressive" → Strategy Builder modifies thresholds/prompts
- "Create similar strategy for ETH" → Strategy Builder clones and adapts
- "Backtest btc-rsi-whale" → Historical analysis (future enhancement)
- "Pause btc-rsi-whale" → Stop scheduled execution
- "Delete btc-rsi-whale" → Remove strategy package

## 3. Core Features

### 3.1 Strategy Builder Agent (Meta-Agent)
- **Natural Language Processing**: Parses user trading ideas into structured requirements
- **Intelligent Questioning**: Asks clarifying questions to fill gaps
- **Code Generation**: Writes custom sub-agents, tools, and orchestration logic
- **Iterative Refinement**: Modifies generated code based on user feedback
- **Strategy Cloning**: Adapts existing strategies for new assets/conditions

### 3.2 Generated Strategy Components

Each strategy generates its own:

1. **Custom Sub-Agents**
   - Specialized prompts tailored to specific analysis tasks
   - Only the tools needed for that sub-agent
   - Focused output format (not full context)

2. **Custom Tools**
   - Data fetchers for required sources (APIs, blockchain, news, etc.)
   - Calculators for indicators/metrics
   - Validators for data quality

3. **Orchestration Logic**
   - Custom coordination code per strategy
   - Strategy-specific synthesis logic
   - Parallel sub-agent spawning

4. **Configuration**
   - Analysis intervals
   - Risk parameters
   - Data source endpoints
   - API credentials

### 3.3 Multi-Strategy Execution Engine
- **Scheduler**: Manages intervals for all active strategies independently
- **Isolation**: Each strategy runs in its own context
- **Parallel Execution**: Multiple strategies run concurrently
- **Resource Management**: Rate limiting, API quotas, cost tracking

### 3.4 Context & Memory Management
- **Prompt Caching**: Cache strategy prompts and static context (5-min TTL)
- **Historical Context**: Recent analysis results for trend detection
- **Project Memory**: Global trading rules in `CLAUDE.md`
- **Strategy Memory**: Per-strategy state in `.context` files

## 4. Technical Architecture

### 4.1 Backend Server Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Client (Frontend/API)                       │
│                   Next.js / Mobile App / CLI                    │
└────────────────┬────────────────────────────────────────────────┘
                 │ HTTPS/WSS
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│                    API Gateway / Load Balancer                  │
│                  (ALB, Nginx, or Cloud Provider)                │
└────────────────┬────────────────────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│                  Backend Server (Node.js/Fastify)               │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  REST API Layer                                           │ │
│  │  ├─ POST   /api/strategies (create via Builder Agent)    │ │
│  │  ├─ GET    /api/strategies (list all)                    │ │
│  │  ├─ GET    /api/strategies/:id (details)                 │ │
│  │  ├─ PATCH  /api/strategies/:id (modify)                  │ │
│  │  ├─ DELETE /api/strategies/:id (remove)                  │ │
│  │  ├─ POST   /api/strategies/:id/activate                  │ │
│  │  ├─ GET    /api/analysis/:strategyId (results)           │ │
│  │  └─ GET    /health (health check)                        │ │
│  └───────────────────────────────────────────────────────────┘ │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  WebSocket Layer                                          │ │
│  │  ├─ Real-time strategy status updates                    │ │
│  │  ├─ Live analysis results streaming                      │ │
│  │  └─ Builder Agent conversation (Q&A)                     │ │
│  └───────────────────────────────────────────────────────────┘ │
└────────────────┬────────────────────────────────────────────────┘
                 │
    ┌────────────┼────────────┐
    │            │            │
    ↓            ↓            ↓
┌────────┐  ┌─────────┐  ┌────────────┐
│PostgreSQL  │ Redis   │  │ Job Queue  │
│+TimeScale│  │ Cache   │  │ (BullMQ)   │
└────────┘  └─────────┘  └──────┬─────┘
                                 │
                                 ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Strategy Execution Workers                   │
│  (Containerized - one per strategy or pooled)                  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  Layer 1: Strategy Builder Agent (Meta-Agent)            │ │
│  │  ├─ Receives user natural language input via API         │ │
│  │  ├─ Generates complete strategy packages:                │ │
│  │  │  ├─ Custom sub-agents (specialized prompts)           │ │
│  │  │  ├─ Custom tools (data fetchers, calculators)         │ │
│  │  │  ├─ Orchestration code (TypeScript)                   │ │
│  │  │  └─ Configuration files                               │ │
│  │  └─ Writes to /strategies/[strategy-name]/               │ │
│  └───────────────────────────────────────────────────────────┘ │
│                             ↓                                   │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  Layer 2: Strategy Orchestrators (Per-Strategy)          │ │
│  │  ├─ Load generated TypeScript code                       │ │
│  │  ├─ Spawn strategy-specific sub-agents (parallel)        │ │
│  │  ├─ Coordinate data gathering                            │ │
│  │  └─ Synthesize analysis with custom logic                │ │
│  └───────────────────────────────────────────────────────────┘ │
│                             ↓                                   │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  Layer 3: Scheduler                                       │ │
│  │  ├─ Manages intervals per strategy (node-cron)           │ │
│  │  ├─ Pushes jobs to BullMQ queue                          │ │
│  │  ├─ Distributed locking via Redis                        │ │
│  │  └─ Resource limits and throttling                       │ │
│  └───────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 System Layers (Agent Architecture)

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 1: Strategy Builder (Meta-Agent)                    │
│  ├─ Takes natural language user input                      │
│  ├─ Generates complete strategy packages:                  │
│  │  ├─ Custom sub-agents (with specialized prompts)        │
│  │  ├─ Custom tools (data fetchers, calculators)           │
│  │  ├─ Orchestration code                                  │
│  │  └─ Configuration files                                 │
│  └─ Writes to /strategies/[strategy-name]/                 │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Layer 2: Strategy Orchestrators (Per-Strategy)            │
│  ├─ Each strategy has its own custom orchestrator          │
│  ├─ Spawns strategy-specific sub-agents                    │
│  ├─ Coordinates parallel data gathering                    │
│  └─ Synthesizes analysis using custom logic                │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Layer 3: Scheduler                                         │
│  ├─ Manages intervals per strategy independently           │
│  ├─ Triggers orchestrators at configured times             │
│  └─ Handles execution queue and resource limits            │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Layer 4: Management & Dashboard                            │
│  ├─ View all active strategies                             │
│  ├─ Modify strategies (via Strategy Builder Agent)         │
│  ├─ View analysis history                                  │
│  └─ Monitor performance and costs                          │
└─────────────────────────────────────────────────────────────┘
```

### 4.3 Container Deployment Model

**Option A: Per-Strategy Containers** (Recommended for MVP)
- Each active strategy runs in its own Docker container
- Complete isolation (security, resource limits)
- Easy horizontal scaling
- Higher overhead but maximum safety

**Option B: Pooled Workers** (For scale optimization)
- Worker pool executes strategies from queue
- Share container resources
- Lower overhead, higher throughput
- Require careful isolation between strategies

**Hybrid Approach** (Production)
- High-frequency strategies (1m, 5m) → pooled workers
- Long-running/resource-intensive strategies → dedicated containers

### 4.4 Claude Agent SDK Integration

**Strategy Builder Agent**
```typescript
import { Agent } from '@anthropic-ai/agent-sdk';

const strategyBuilderAgent = new Agent({
  name: 'strategy-builder',
  instructions: `You are the Strategy Builder for Claude Trader.

  Your job: Convert user natural language trading ideas into complete,
  executable strategy packages.

  For each strategy, you generate:
  1. Custom sub-agent code with specialized prompts
  2. Custom tools for data fetching and calculation
  3. Orchestration logic that coordinates sub-agents
  4. Configuration files

  Ask clarifying questions to ensure you understand:
  - Trading pairs and timeframes
  - Entry/exit criteria
  - Data sources needed
  - Risk parameters`,

  tools: [
    'file-system',      // Write generated code
    'code-generator',   // Generate TypeScript/Python code
    'web-search',       // Research data sources/APIs
    'ask-user'          // Clarifying questions
  ]
});
```

**Generated Strategy Orchestrator Example**
```typescript
// AUTO-GENERATED by Strategy Builder Agent
// Strategy: btc-rsi-whale
import { Agent } from '@anthropic-ai/agent-sdk';
import { whaleMonitorAgent } from './agents/whale-monitor';
import { rsiDivergenceAgent } from './agents/rsi-divergence-detector';

export async function execute() {
  // Parallel sub-agent execution
  const [whaleData, rsiData] = await Promise.all([
    whaleMonitorAgent.run({
      context: loadStrategyContext(),
      caching: true
    }),
    rsiDivergenceAgent.run({
      context: loadStrategyContext(),
      caching: true
    })
  ]);

  // Custom synthesis logic (generated per strategy)
  const signal = synthesizeSignal(whaleData, rsiData);

  // Write analysis
  await writeAnalysis(signal, whaleData, rsiData);

  return signal;
}

function synthesizeSignal(whale, rsi) {
  if (rsi.divergence_type === 'bullish' &&
      whale.accumulation_score > 70) {
    return {
      action: 'BUY',
      confidence: Math.min(whale.confidence, rsi.confidence),
      reasoning: `Bullish RSI divergence detected with high whale accumulation`
    };
  }
  return { action: 'HOLD', confidence: 50, reasoning: 'Conditions not met' };
}
```

### 4.5 Key SDK Features Leveraged

1. **Sub-Agent System**: Each strategy spawns custom sub-agents in parallel
2. **Prompt Caching**: Cache strategy prompts and context (reduces costs 50%+)
3. **Code Generation**: Strategy Builder generates TypeScript code
4. **MCP Integration**: Connect to market data APIs, databases, blockchain explorers
5. **Custom Tools**: Generated per-strategy (calculators, data fetchers, validators)
6. **File System Tools**: Read/write strategy packages and analysis outputs

### 4.6 Technology Stack

**Primary Language: TypeScript**
- Chosen for production backend deployment
- 44% faster request/sec vs Python
- Better concurrency for I/O-heavy workloads (API calls, WebSockets)
- Non-blocking I/O perfect for parallel strategy execution
- Shared types across frontend/backend
- Lower memory footprint for containerized deployment

#### Backend Server
- **Runtime**: Node.js 20+ LTS
- **Framework**: Fastify (high-performance) or Express
- **API**: RESTful + WebSocket for real-time updates
- **SDK**: `@anthropic-ai/claude-agent-sdk` (TypeScript)
- **Code Generation**: TypeScript Compiler API for validation
- **Process Management**: PM2 or Docker containers

#### Frontend (Future)
- **Framework**: Next.js 15+ (React)
- **UI**: shadcn/ui + Tailwind CSS
- **State**: Zustand or React Query
- **Charts**: Lightweight Charts (TradingView), Recharts
- **Real-time**: Socket.io client

#### Data Sources
- **Market Data**: Binance, Coinbase, Kraken APIs (REST/WebSocket)
- **News/Sentiment**: NewsAPI, Reddit API, Twitter/X API
- **On-Chain**: Blockchain.com, Etherscan, Dune Analytics
- **Alternative Data**: Glassnode, CoinGlass, LunarCrush

#### Storage & Caching
- **Database**: PostgreSQL 16+ with TimescaleDB extension
  - Strategy metadata and configuration
  - Historical analysis results (time-series)
  - User accounts and permissions
- **Cache**: Redis 7+
  - Prompt caching for Claude SDK
  - Rate limiting (per-API, per-user)
  - Session management
  - Real-time data buffering
- **File Storage**:
  - Strategy Packages: `/strategies/[name]/` (generated TypeScript code)
  - Analysis Output: `/analysis/[name]/[timestamp].md`
  - S3-compatible storage for backups (optional)

#### Scheduling & Orchestration
- **Scheduler**: node-cron with Redis-based distributed locking
- **Queue**: BullMQ (Redis-backed job queue)
  - Strategy execution queue
  - Retry logic for failed analyses
  - Priority scheduling
- **Containerization**: Docker for strategy isolation

#### Monitoring & Observability
- **Metrics**: Prometheus + Grafana
  - API latency, throughput
  - Strategy execution times
  - Claude API usage and costs
  - Error rates
- **Logging**: Winston (structured JSON logs)
  - Separate logs per strategy
  - Centralized logging (CloudWatch, Datadog, or self-hosted)
- **Tracing**: OpenTelemetry (optional)
- **Health Checks**: `/health` endpoint with dependency checks

#### Deployment Architecture
- **Containers**: Docker + Docker Compose (local/staging)
- **Cloud Options**:
  - **AWS**: ECS Fargate + RDS + ElastiCache + ALB
  - **Railway/Render**: Simplified deployment (early stage)
  - **Fly.io**: Global edge deployment
- **CI/CD**: GitHub Actions
  - Automated testing on PRs
  - Deploy to staging on merge to `main`
  - Manual promotion to production
- **Infrastructure as Code**: Terraform (optional for AWS)

#### Security
- **API Keys**: Environment variables + AWS Secrets Manager / HashiCorp Vault
- **Authentication**: JWT tokens (frontend → backend)
- **Authorization**: RBAC for multi-user support
- **HTTPS**: TLS 1.3 via Let's Encrypt or AWS ACM
- **Rate Limiting**: Redis-based per-user/IP limits
- **Input Validation**: Zod schemas for all API inputs

## 5. Generated Strategy Package Structure

When the Strategy Builder Agent creates a strategy, it generates this structure:

```
/strategies/btc-rsi-whale/
├── strategy.md              # Human-readable documentation (AUTO-GENERATED)
├── config.json              # Strategy configuration (AUTO-GENERATED)
├── /agents/
│   ├── whale-monitor.ts     # Custom sub-agent (AUTO-GENERATED)
│   ├── rsi-divergence.ts    # Custom sub-agent (AUTO-GENERATED)
│   └── orchestrator.ts      # Coordination logic (AUTO-GENERATED)
├── /tools/
│   ├── blockchain-scanner.ts   # Data fetcher (AUTO-GENERATED)
│   ├── rsi-calculator.ts      # Calculator (AUTO-GENERATED)
│   └── tool-registry.ts       # Tool exports (AUTO-GENERATED)
├── /data-sources/
│   ├── whale-addresses.json   # Configuration data
│   └── api-config.json        # API endpoints/keys
└── .context                   # Strategy execution state
```

**strategy.md** (AUTO-GENERATED)
```markdown
# BTC RSI-Whale Accumulation Strategy

**Generated**: 2025-10-26 14:15
**Status**: Active
**Next Analysis**: 2025-10-26 14:30

## Overview
Monitors Bitcoin for bullish RSI divergences combined with whale wallet
accumulation patterns. Generates BUY signals when both conditions align.

## Trading Pair
BTC/USD

## Analysis Interval
15 minutes

## Sub-Agents
1. **whale-monitor**: Tracks top 50 BTC holder wallets for accumulation
2. **rsi-divergence-detector**: Identifies RSI divergences on 4h timeframe

## Signal Logic
BUY: Bullish RSI divergence + Accumulation score > 70
HOLD: Conditions not met
SELL: Not implemented (manual exit)

## Data Sources
- Blockchain.com API (whale transactions)
- Binance API (BTC/USD price data)

## Risk Parameters
- Max position size: $10,000
- Stop loss: 2%
- Take profit: 5%
```

**config.json** (AUTO-GENERATED)
```json
{
  "name": "btc-rsi-whale",
  "version": "1.0.0",
  "generated": "2025-10-26T14:15:00Z",
  "status": "active",
  "interval": "15m",
  "tradingPair": "BTC/USD",
  "subAgents": ["whale-monitor", "rsi-divergence-detector"],
  "tools": ["blockchain-scanner", "rsi-calculator"],
  "dataSources": {
    "blockchain": "https://blockchain.info/",
    "priceData": "https://api.binance.com/"
  },
  "riskParams": {
    "maxPositionSize": 10000,
    "stopLoss": 0.02,
    "takeProfit": 0.05
  }
}
```

## 6. Data Flow (New Architecture)

### Strategy Creation Flow
```
1. User inputs natural language strategy idea
   ↓
2. Strategy Builder Agent analyzes requirements
   ↓
3. Strategy Builder asks clarifying questions
   ↓
4. User provides answers
   ↓
5. Strategy Builder generates:
   - Custom sub-agent code (with specialized prompts)
   - Custom tools (data fetchers, calculators)
   - Orchestration logic (synthesis code)
   - Configuration files
   ↓
6. Strategy package written to /strategies/[name]/
   ↓
7. User reviews and approves
   ↓
8. Strategy activated and added to scheduler
```

### Strategy Execution Flow
```
1. Scheduler triggers strategy at configured interval
   ↓
2. Strategy orchestrator.ts executes
   ↓
3. Orchestrator spawns custom sub-agents in parallel:
   - whale-monitor.ts → runs with whale-specific prompt
   - rsi-divergence.ts → runs with RSI-specific prompt
   ↓
4. Sub-agents use generated tools:
   - blockchain-scanner.ts → fetches on-chain data
   - rsi-calculator.ts → calculates RSI + divergence
   ↓
5. Sub-agents return focused results:
   - whale-monitor → {accumulation_score, confidence, activity}
   - rsi-divergence → {divergence_type, strength, confidence}
   ↓
6. Orchestrator runs custom synthesis logic (generated per-strategy)
   ↓
7. Signal generated: {action: 'BUY'/'SELL'/'HOLD', confidence, reasoning}
   ↓
8. Analysis written to /analysis/[strategy-name]/[timestamp].md
   ↓
9. Optional: Notification sent, trade logged (future)
```

## 7. Configuration

### 7.1 Global Config (`config.json`)
```json
{
  "anthropicApiKey": "sk-...",
  "defaultModel": "claude-sonnet-4-5",
  "enablePromptCaching": true,
  "outputDirectory": "./analysis",
  "maxConcurrentStrategies": 5,
  "strategyBuilder": {
    "model": "claude-sonnet-4-5",
    "temperature": 0.7,
    "maxIterations": 5
  },
  "defaultDataSources": {
    "marketData": "binance",
    "newsApi": "newsapi.org",
    "blockchain": "blockchain.com"
  },
  "globalRiskLimits": {
    "maxPositionSizePerStrategy": 10000,
    "maxTotalExposure": 50000,
    "maxDailyLoss": 500
  },
  "monitoring": {
    "prometheusPort": 9090,
    "logLevel": "info"
  }
}
```

### 7.2 Strategy-Level Config
- Auto-generated by Strategy Builder Agent in each strategy's `config.json`
- Can override global settings
- Includes custom data sources, intervals, risk params

## 8. MVP Deliverables

### Phase 1: Strategy Builder Agent (CRITICAL)
- [ ] Strategy Builder Agent implementation
- [ ] Natural language parsing for trading ideas
- [ ] Clarifying question system
- [ ] TypeScript code generation (sub-agents, tools, orchestrators)
- [ ] Strategy package file structure creation
- [ ] Basic validation and testing

### Phase 2: Core Execution Infrastructure
- [ ] Strategy orchestrator runtime
- [ ] Scheduler for interval-based execution
- [ ] File-based analysis output system
- [ ] Strategy activation/deactivation
- [ ] Basic error handling and logging

### Phase 3: Sample Strategy Templates
- [ ] Simple technical indicator strategy (RSI-based)
- [ ] Market data connector (Binance API)
- [ ] Basic data fetching tools
- [ ] Example sub-agent: Technical analyzer
- [ ] Demonstrate end-to-end flow

### Phase 4: Multi-Strategy Support
- [ ] Multiple concurrent strategies
- [ ] Per-strategy prompt caching
- [ ] Resource management (rate limiting, quotas)
- [ ] Strategy modification via Builder Agent
- [ ] Strategy cloning/adaptation

### Phase 5: Advanced Features
- [ ] Complex data sources (on-chain, sentiment, news)
- [ ] Advanced sub-agents (sentiment, risk analyzers)
- [ ] Historical analysis tracking
- [ ] Performance metrics and monitoring
- [ ] CLI dashboard for strategy management

## 9. Success Metrics

### Strategy Builder Agent
- **Generation Success Rate**: 95%+ strategies generate valid, runnable code
- **Question Clarity**: Users need < 3 iterations to finalize strategy
- **Code Quality**: Generated code passes TypeScript compilation
- **User Satisfaction**: Natural language → working strategy in < 5 minutes

### Strategy Execution
- **Performance**: Analysis completion time < 30 seconds per strategy
- **Cost Efficiency**: Prompt caching reduces API costs by 50%+
- **Accuracy**: Analysis includes all required data points per strategy
- **Reliability**: 99%+ uptime for scheduled analysis
- **Scalability**: Support 10+ concurrent strategies

## 10. Future Enhancements

### Immediate (Post-MVP)
- **Strategy Testing**: Test run before activation (dry-run mode)
- **Strategy Templates**: Pre-built strategy library (trend-following, mean-reversion, etc.)
- **CLI Dashboard**: Interactive terminal UI for strategy management
- **Alerts**: Notifications via email, Telegram, Discord

### Medium-term
- **Backtesting Engine**: Historical strategy testing with performance metrics
- **Paper Trading**: Simulated trade execution with portfolio tracking
- **Learning System**: Agent learns from successful/failed signals
- **Strategy Analytics**: Win rate, Sharpe ratio, drawdown analysis
- **Web Dashboard**: Real-time monitoring and controls

### Long-term
- **Live Trading**: Real trade execution via exchange APIs
- **Portfolio Management**: Multi-strategy portfolio optimization
- **Risk Management**: Dynamic position sizing, correlation analysis
- **Strategy Marketplace**: Share and discover community strategies
- **Multi-Exchange Support**: Cross-exchange arbitrage strategies

## 11. Security & Risk Considerations

### Code Generation Security
- **Sandboxed Execution**: Generated code runs in isolated environment
- **Code Review**: User approval required before strategy activation
- **Static Analysis**: Automated checks for unsafe patterns (eval, exec, etc.)
- **Dependency Pinning**: Lock down library versions for reproducibility

### API & Data Security
- **Credential Management**: Encrypted storage of API keys (keytar/keychain)
- **API Key Rotation**: Support for key rotation without strategy restart
- **Rate Limiting**: Respect API rate limits for all data sources
- **Data Validation**: Sanitize and validate all external data

### Financial Risk
- **Hard Limits**: Global and per-strategy position size limits
- **No Auto-Execution**: Manual review required for trades (MVP)
- **Audit Trail**: Log all analysis, signals, and decisions
- **Error Handling**: Graceful degradation when data sources fail
- **Circuit Breakers**: Stop all strategies on consecutive failures

### Operational Security
- **Logging**: No sensitive data in logs (API keys, wallet addresses)
- **Monitoring**: Alert on anomalous behavior (excessive API calls, errors)
- **Isolation**: Each strategy runs in isolated context (no cross-contamination)

## 12. Key Differentiators

**Claude Trader vs. Traditional Trading Bots:**
- ✅ **Natural language strategy creation** (no coding required)
- ✅ **Self-generating agents** (Claude writes custom agents per strategy)
- ✅ **Flexible data sources** (on-chain, sentiment, news, technical)
- ✅ **Transparent reasoning** (every decision explained in detail)
- ✅ **Easy modification** ("make this more aggressive" → code updated)
- ❌ No manual backtesting configuration
- ❌ No rigid templated strategies
- ❌ No black-box decisions

**This is AI-native trading:** The system programs itself based on your ideas.

---

**Document Version**: 2.0
**Last Updated**: 2025-10-26
**Status**: Ready for Implementation
**Next Steps**: Begin Phase 1 - Strategy Builder Agent
