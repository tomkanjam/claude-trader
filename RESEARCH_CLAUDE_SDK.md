# Claude Agent SDK Research - For Claude Trader

**Research Date:** 2025-10-26
**Purpose:** Understand Claude Agent SDK capabilities for building the Strategy Builder meta-agent and sub-agent system

---

## 1. Installation & Setup

```bash
npm install @anthropic-ai/claude-agent-sdk
```

**Requirements:**
- Node.js 18+
- TypeScript support
- Anthropic API key

---

## 2. Core API: `query()` Function

The main entry point for creating agents:

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

async function basicAgent() {
  for await (const msg of query({
    prompt: "Analyze Bitcoin price trends",
    options: {
      maxTurns: 3
    }
  })) {
    if (msg.type === "result" && msg.subtype === "success") {
      console.log(msg.result);
    }
  }
}
```

**Key Features:**
- Returns an `AsyncGenerator<SDKMessage, void>`
- Streams messages as the agent works
- Supports options for configuration

---

## 3. Custom Tools with `createSdkMcpServer()`

Custom tools extend Claude's capabilities:

```typescript
import { query, tool, createSdkMcpServer } from "@anthropic-ai/claude-agent-sdk";
import { z } from "zod";

// Define custom tool server
const marketDataServer = createSdkMcpServer({
  name: "market-data",
  version: "1.0.0",
  tools: [
    tool(
      "fetch_price",
      "Fetch current cryptocurrency price",
      {
        symbol: z.string().describe("Trading pair symbol (e.g., BTCUSDT)"),
        exchange: z.enum(["binance", "coinbase"]).default("binance")
      },
      async (args) => {
        // Call exchange API
        const response = await fetch(
          `https://api.binance.com/api/v3/ticker/price?symbol=${args.symbol}`
        );
        const data = await response.json();

        return {
          content: [{
            type: "text",
            text: JSON.stringify(data, null, 2)
          }]
        };
      }
    ),

    tool(
      "calculate_rsi",
      "Calculate RSI indicator",
      {
        prices: z.array(z.number()).describe("Array of closing prices"),
        period: z.number().default(14).describe("RSI period")
      },
      async (args) => {
        // RSI calculation logic
        const rsi = calculateRSI(args.prices, args.period);

        return {
          content: [{
            type: "text",
            text: `RSI(${args.period}): ${rsi.toFixed(2)}`
          }]
        };
      }
    )
  ]
});

// Use the custom tools
async function analyzeWithCustomTools() {
  for await (const msg of query({
    prompt: "Get BTC price and calculate RSI",
    options: {
      mcpServers: {
        "market-data": marketDataServer
      },
      allowedTools: [
        "mcp__market-data__fetch_price",
        "mcp__market-data__calculate_rsi"
      ]
    }
  })) {
    // Handle messages
  }
}
```

**Tool Naming Convention:**
- Format: `mcp__<server-name>__<tool-name>`
- Example: `mcp__market-data__fetch_price`

**Tool Requirements:**
- Must return `{ content: [{ type: "text", text: string }] }`
- Use Zod schemas for type-safe parameters
- Handler function must be async

---

## 4. Sub-Agents (Critical for Claude Trader!)

### Why Sub-Agents?

1. **Parallelization**: Run multiple analysis tasks simultaneously
2. **Context Isolation**: Each sub-agent has its own context window
3. **Focused Output**: Sub-agents return only relevant data to orchestrator

### AgentDefinition Type

```typescript
type AgentDefinition = {
  description: string;           // What this sub-agent does
  tools?: string[];              // Tools it can use
  prompt: string;                // Specialized instructions
  model?: 'sonnet' | 'opus' | 'haiku' | 'inherit';
}
```

### Creating Sub-Agents Programmatically

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

async function orchestrateStrategy() {
  const agents = {
    "whale-monitor": {
      description: "Monitor whale wallet movements",
      prompt: `You are a whale wallet monitoring specialist for Bitcoin.

      Your job: Track large BTC wallets and determine if they are
      accumulating or distributing.

      Use the blockchain-scanner tool to fetch recent transactions.

      Return ONLY a JSON object:
      {
        "accumulation_score": 0-100,
        "recent_activity": "description",
        "confidence": 0-100
      }`,
      tools: ["mcp__blockchain__scan_wallet"],
      model: "sonnet"
    },

    "rsi-analyzer": {
      description: "Detect RSI divergences",
      prompt: `You are an RSI divergence detection specialist.

      Your job: Analyze price and RSI data to identify bullish or bearish divergences.

      Use the price-fetcher and rsi-calculator tools.

      Return ONLY a JSON object:
      {
        "divergence_type": "bullish" | "bearish" | "none",
        "strength": 0-100,
        "confidence": 0-100,
        "reasoning": "explanation"
      }`,
      tools: [
        "mcp__market-data__fetch_price",
        "mcp__market-data__calculate_rsi"
      ],
      model: "sonnet"
    }
  };

  // Main orchestrator prompt that spawns sub-agents
  const orchestratorPrompt = `You are the strategy orchestrator for BTC-RSI-Whale strategy.

  Steps:
  1. Use the /agent whale-monitor command to analyze whale activity
  2. Use the /agent rsi-analyzer command to detect RSI divergences
  3. Synthesize both results into a trading signal

  Return a final JSON with:
  {
    "signal": "BUY" | "SELL" | "HOLD",
    "confidence": 0-100,
    "reasoning": "detailed explanation",
    "whale_data": {...},
    "rsi_data": {...}
  }`;

  for await (const msg of query({
    prompt: orchestratorPrompt,
    options: {
      agents,
      mcpServers: {
        "market-data": marketDataServer,
        "blockchain": blockchainServer
      },
      maxTurns: 10
    }
  })) {
    if (msg.type === "result" && msg.subtype === "success") {
      const analysis = JSON.parse(msg.result);
      return analysis;
    }
  }
}
```

### How Sub-Agents Work

1. **Orchestrator calls sub-agent**: Main agent uses `/agent <name>` command
2. **Sub-agent executes**: Runs with its own context and tools
3. **Returns focused result**: Only relevant data goes back to orchestrator
4. **Parallel execution**: Multiple sub-agents can run simultaneously

---

## 5. Code Generation Pattern (For Strategy Builder)

The Strategy Builder meta-agent will **generate TypeScript files** for each strategy:

```typescript
async function strategyBuilderAgent(userIdea: string) {
  const builderPrompt = `You are the Strategy Builder for Claude Trader.

User's strategy idea: "${userIdea}"

Your job:
1. Analyze the trading strategy requirements
2. Ask clarifying questions if needed
3. Generate a complete strategy package:
   - config.json with strategy metadata
   - TypeScript files for sub-agents
   - TypeScript files for custom tools
   - orchestrator.ts that coordinates everything

Generate files in this structure:
/strategies/{strategy-name}/
  config.json
  /agents/
    sub-agent-1.ts
    sub-agent-2.ts
  /tools/
    tool-1.ts
    tool-2.ts
  orchestrator.ts

Use the file-system tools to write these files.
Each sub-agent should have a focused, specialized prompt.
Each tool should be a simple TypeScript function.`;

  for await (const msg of query({
    prompt: builderPrompt,
    options: {
      maxTurns: 20,
      // Allow file system access for code generation
      allowedTools: ["write_file", "read_file", "list_directory"]
    }
  })) {
    // Stream progress to user
    if (msg.type === "text") {
      console.log(msg.content);
    }

    if (msg.type === "result") {
      return msg.result; // Strategy created
    }
  }
}
```

---

## 6. Agent Patterns for Claude Trader

### Pattern 1: Strategy Builder (Meta-Agent)

```typescript
const strategyBuilder = {
  prompt: `You are the Strategy Builder meta-agent.

  Convert user trading ideas into executable strategy packages.

  Process:
  1. Parse natural language description
  2. Ask clarifying questions
  3. Generate config.json
  4. Generate sub-agent definitions
  5. Generate custom tools
  6. Create orchestrator logic
  7. Validate generated code`,

  tools: [
    "write_file",
    "read_file",
    "web_search", // Research APIs/data sources
    "bash"        // Validate TypeScript syntax
  ]
};
```

### Pattern 2: Strategy Orchestrator (Per-Strategy)

```typescript
// Generated by Strategy Builder
async function executeStrategy(strategyName: string) {
  const config = loadStrategyConfig(strategyName);
  const agents = loadSubAgents(strategyName);
  const mcpServers = loadCustomTools(strategyName);

  const orchestratorPrompt = generateOrchestratorPrompt(config);

  for await (const msg of query({
    prompt: orchestratorPrompt,
    options: {
      agents,           // Strategy-specific sub-agents
      mcpServers,       // Strategy-specific tools
      maxTurns: config.maxTurns || 10
    }
  })) {
    if (msg.type === "result") {
      return msg.result;
    }
  }
}
```

### Pattern 3: Parallel Sub-Agent Execution

```typescript
// Orchestrator instructs Claude to spawn multiple sub-agents
const prompt = `Execute these sub-agents in parallel:
1. /agent price-monitor - fetch latest BTC price
2. /agent sentiment-analyzer - analyze crypto news sentiment
3. /agent volume-analyzer - check trading volume trends

Wait for all results, then synthesize into a trading signal.`;
```

---

## 7. Key Insights for Claude Trader

### ✅ What Works Well

1. **Sub-agents for specialization**: Each indicator/data source = one sub-agent
2. **Custom tools for data fetching**: Wrap exchange APIs in MCP tools
3. **Code generation**: Strategy Builder can generate `.ts` files directly
4. **Parallel execution**: Multiple data sources analyzed simultaneously
5. **Context efficiency**: Sub-agents only return relevant data

### ⚠️ Limitations & Considerations

1. **No direct sub-agent API**: Must use `/agent <name>` command in prompt
2. **File system access**: May need custom MCP server for file operations
3. **TypeScript execution**: Generated code needs `tsx` or `ts-node` to run
4. **Cost**: Each sub-agent spawns a new Claude request (consider caching)
5. **Error handling**: Sub-agents can fail; orchestrator must handle errors

### 💡 Best Practices

1. **Focused sub-agent prompts**: Single responsibility (RSI, whale monitoring, etc.)
2. **JSON output format**: Easy to parse and synthesize
3. **Tool naming**: Clear `mcp__<domain>__<action>` convention
4. **Validation**: Validate generated code with TypeScript compiler
5. **Caching**: Cache strategy prompts and context (5-min TTL)

---

## 8. Architecture Implications

### Strategy Creation Flow

```
User Input
    ↓
Strategy Builder Agent (query with maxTurns=20)
    ↓
Generates:
    - config.json
    - sub-agent definitions (as TypeScript)
    - custom tools (as TypeScript)
    - orchestrator.ts
    ↓
Files written to /strategies/{name}/
    ↓
User reviews and activates
```

### Strategy Execution Flow

```
Scheduler triggers strategy
    ↓
Load strategy config + code
    ↓
Call query() with:
    - orchestrator prompt
    - agents (sub-agent definitions)
    - mcpServers (custom tools)
    ↓
Claude spawns sub-agents in parallel
    ↓
Sub-agents return focused results
    ↓
Orchestrator synthesizes signal
    ↓
Write analysis to DuckDB + markdown
```

### Simplified Tech Stack

```
Claude Agent SDK
    ├─ Strategy Builder (meta-agent)
    │   └─ Generates TypeScript code
    │
    ├─ Strategy Orchestrators (per-strategy)
    │   └─ Spawns sub-agents via /agent command
    │
    ├─ Custom MCP Tools
    │   ├─ Market data (Binance, CoinGecko)
    │   ├─ Blockchain data (Etherscan)
    │   └─ File system (read/write strategies)
    │
    └─ Sub-Agents (parallel execution)
        ├─ Price monitor
        ├─ RSI analyzer
        ├─ Whale tracker
        └─ Sentiment analyzer
```

---

## 9. Next Steps

### Immediate Research Needs

1. ✅ **Dynamic TypeScript execution** - How to `import()` generated `.ts` files
2. ⏳ **File system MCP server** - Read/write strategy files
3. ⏳ **Error handling** - How to catch and recover from sub-agent failures
4. ⏳ **Cost optimization** - Prompt caching strategies

### Prototypes to Build

1. **Simple tool example**: Market data fetcher MCP server
2. **Sub-agent test**: Spawn two parallel sub-agents and synthesize results
3. **Mini Strategy Builder**: Generate one `.ts` file from natural language
4. **Strategy executor**: Load and run a pre-generated strategy

---

## 10. Example: Complete Mini Strategy

```typescript
import { query, tool, createSdkMcpServer } from "@anthropic-ai/claude-agent-sdk";
import { z } from "zod";

// Custom tool server
const marketTools = createSdkMcpServer({
  name: "market",
  version: "1.0.0",
  tools: [
    tool("get_btc_price", "Get BTC price", {}, async () => {
      const res = await fetch("https://api.coinbase.com/v2/prices/BTC-USD/spot");
      const data = await res.json();
      return { content: [{ type: "text", text: JSON.stringify(data) }] };
    })
  ]
});

// Sub-agents
const agents = {
  "price-monitor": {
    description: "Monitor BTC price",
    prompt: "Use get_btc_price tool. Return JSON: {price: number, timestamp: string}",
    tools: ["mcp__market__get_btc_price"],
    model: "sonnet"
  }
};

// Execute strategy
async function runSimpleStrategy() {
  for await (const msg of query({
    prompt: "Use /agent price-monitor to get BTC price. If > $50000, say BUY, else HOLD.",
    options: {
      agents,
      mcpServers: { market: marketTools },
      maxTurns: 5
    }
  })) {
    if (msg.type === "result") {
      console.log("Signal:", msg.result);
    }
  }
}

runSimpleStrategy();
```

---

## 11. Open Questions

1. **Can Strategy Builder generate valid AgentDefinition objects?**
   - Yes, it can write them to `config.json` and we load them dynamically

2. **How to handle API keys for data sources?**
   - Pass via environment variables to MCP tool closures

3. **Can sub-agents use different models?**
   - Yes, via `model` property in `AgentDefinition`

4. **What's the cost of spawning multiple sub-agents?**
   - Each sub-agent = new Claude request; use prompt caching aggressively

5. **How to test generated strategies?**
   - Dry-run mode: Load strategy, execute once, don't persist results

---

**Status:** ✅ Ready to implement
**Next:** Create prototypes for tool creation and sub-agent spawning
