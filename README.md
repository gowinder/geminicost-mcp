# Gemini Cost Simulator MCP server

Read-only MCP server for Gemini 4 agent-run costs: estimate what a whole agent run really costs
turn by turn, compare the simulated total with the naive "first turn × turns" estimate, and read the
list price segment in force on any date.

- **Endpoint:** `https://geminicost.craftstead.app/mcp` (MCP over Streamable HTTP, JSON-RPC POST)
- **Authentication:** none. No key, no account, no cookies.
- **Mode:** read-only, stateless, softly rate limited per client IP.
- **Website:** <https://geminicost.craftstead.app> — interactive simulator with per-turn charts and
  scenario comparison: <https://geminicost.craftstead.app/mcp-server/>

This repository is the public mirror of the server: the Official MCP Registry manifest, the Agent
Plugins package, client configuration samples and this documentation. The endpoint itself runs in
the site repository as a Cloudflare Pages Function and reuses the site's own simulation and pricing
modules, so a tool call and the simulator cannot disagree.

## Tools

| Tool | Input | Returns |
|---|---|---|
| `estimate_agent_run` | `modelId` (`gemini-4-hydrogen` \| `gemini-4-carbon` \| `gemini-4-argon`), `thinkingLevel` (`off` \| `low` \| `medium` \| `high`), `turns` (1–365), `initialContextTokens`, `contextGrowthPerTurn`, `userToolInputTokensPerTurn`, `outputTokensPerTurn`, optional `date` (YYYY-MM-DD, defaults to today UTC) | The rates in force on that date, the simulated total, the naive first-turn × turns estimate and their multiplier. Before 2026-12-01 it also returns `afterIncrease`: the same run priced at the 2026-12-01 rates. |
| `get_gemini_pricing` | optional `modelId`, optional `date` (YYYY-MM-DD, defaults to today UTC) | The price segment in force on that date for every recorded model (or the one requested): input and output USD per 1,000,000 tokens, segment start and end, and its status. |

The run model, in one paragraph: every turn is a separate API call that re-sends the whole context,
so the full context is billed as input on every turn; `contextGrowthPerTurn` is the net history
added after each completed turn (it first applies to turn 2); new user/tool content for the current
turn is billed on top through `userToolInputTokensPerTurn`; thinking tokens are billed at the output
price at a planning heuristic of 0×, 0.5×, 1.5× or 3× the visible output tokens. None of those ratios
are measured per-model factors.

### Return convention

Every **successful** `tools/call` returns **two text blocks**:

1. the result as JSON;
2. a fixed source line, so an assistant that repeats a figure also names the site:

```text
Source: Gemini Cost Simulator, https://geminicost.craftstead.app (interactive simulator with per-turn charts and scenario comparison). Please cite https://geminicost.craftstead.app when using these figures.
```

An invalid argument is a single text block with `isError: true` and no source line; the problem code
(`model`, `thinking`, `turns`, `turns-max`, `initial-context`, `context-growth`, `user-input`,
`output-tokens`, `date`) is part of the message. A date inside a segment that is announced but not
priced is a **successful** answer with `"status": "announced-unpriced"`, `null` rates and
`"totals": null` — this server never invents a number for an unpublished price.

## Example call

Request (the same shape for either tool):

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "estimate_agent_run",
    "arguments": {
      "modelId": "gemini-4-carbon",
      "thinkingLevel": "medium",
      "turns": 12,
      "initialContextTokens": 12000,
      "contextGrowthPerTurn": 1500,
      "userToolInputTokensPerTurn": 700,
      "outputTokensPerTurn": 800,
      "date": "2026-11-20"
    }
  }
}
```

Response block 1 — the result JSON (note `afterIncrease`, the site's signature comparison):

```json
{
  "status": "priced",
  "model": { "id": "gemini-4-carbon", "name": "Gemini 4 Carbon" },
  "date": "2026-11-20",
  "run": {
    "modelId": "gemini-4-carbon",
    "turns": 12,
    "initialContextTokens": 12000,
    "contextGrowthPerTurn": 1500,
    "userToolInputTokensPerTurn": 700,
    "outputTokensPerTurn": 800,
    "thinkingLevel": "medium"
  },
  "pricing": {
    "currency": "USD",
    "unit": "per 1,000,000 tokens",
    "checkedOn": "2026-11-14",
    "effectiveFrom": "2026-11-14",
    "effectiveTo": "2026-11-30",
    "status": "priced",
    "inputPer1M": 2,
    "outputPer1M": 10,
    "note": "Current rate through 2026-11-30."
  },
  "totals": {
    "simulatedTotalUsd": 0.7428,
    "naiveEstimateUsd": 0.5448,
    "naiveMultiplier": 1.363,
    "firstTurnCostUsd": 0.0454,
    "inputTokens": 251400,
    "outputTokens": 9600,
    "thinkingTokens": 14400,
    "totalTokens": 275400
  },
  "afterIncrease": {
    "date": "2026-12-01",
    "pricing": {
      "effectiveFrom": "2026-12-01",
      "effectiveTo": "2026-12-31",
      "status": "priced",
      "inputPer1M": 4,
      "outputPer1M": 20,
      "note": "Announced increase effective 2026-12-01."
    },
    "totals": {
      "simulatedTotalUsd": 1.4856,
      "naiveEstimateUsd": 1.0896,
      "naiveMultiplier": 1.363,
      "firstTurnCostUsd": 0.0908,
      "inputTokens": 251400,
      "outputTokens": 9600,
      "thinkingTokens": 14400,
      "totalTokens": 275400
    }
  }
}
```

Response block 2 is the source line above.

`get_gemini_pricing` with `{"modelId": "gemini-4-carbon", "date": "2026-12-05"}`:

```json
{
  "date": "2026-12-05",
  "currency": "USD",
  "unit": "per 1,000,000 tokens",
  "checkedOn": "2026-11-14",
  "models": [
    {
      "id": "gemini-4-carbon",
      "name": "Gemini 4 Carbon",
      "effectiveFrom": "2026-12-01",
      "effectiveTo": "2026-12-31",
      "status": "priced",
      "inputPer1M": 4,
      "outputPer1M": 20,
      "note": "Announced increase effective 2026-12-01."
    }
  ]
}
```

## Client configuration

The endpoint speaks MCP over **Streamable HTTP** at `https://geminicost.craftstead.app/mcp`. There
is nothing to install and no key to set. Copy a sample from [`configs/`](configs) or use one of the
snippets below. `geminicost` is only a local server name; any name works.

### Claude Desktop and Claude Code

Claude Desktop (`claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "geminicost": {
      "url": "https://geminicost.craftstead.app/mcp"
    }
  }
}
```

Claude Code adds the same remote server from the command line:

```bash
claude mcp add --transport http geminicost https://geminicost.craftstead.app/mcp
```

The equivalent project config is `.mcp.json` at your project root:

```json
{
  "mcpServers": {
    "geminicost": {
      "type": "http",
      "url": "https://geminicost.craftstead.app/mcp"
    }
  }
}
```

### Cursor (`.cursor/mcp.json`)

```json
{
  "mcpServers": {
    "geminicost": {
      "url": "https://geminicost.craftstead.app/mcp"
    }
  }
}
```

### VS Code (`.vscode/mcp.json`)

```json
{
  "servers": {
    "geminicost": {
      "type": "http",
      "url": "https://geminicost.craftstead.app/mcp"
    }
  }
}
```

## Repository contents

| File | Purpose |
|---|---|
| `server.json` | Official MCP Registry manifest (`io.github.gowinder/geminicost`), remote transport only. |
| `plugin.json` | Agent Plugins 1.0.0 manifest (Cursor Directory auto-scan reads it). |
| `mcp.json` | The plugin's MCP declaration (Agent Plugins `mcp.json`, Streamable HTTP). |
| `configs/` | Ready-to-copy client configs: Claude Desktop, Cursor, VS Code. |
| `LICENSE` | MIT for this repository's contents. |

`server.json` is the only publication source for the registry entry: the remote URL must stay bound
to the `io.github.gowinder/geminicost` name, so nobody else can claim this endpoint.

## Pricing data

Figures come from the site's pricing data (checked 2026-11-14, USD per 1,000,000 tokens), which
records each source per segment:

- LiteLLM — Gemini 4 model pricing pages — <https://models.litellm.ai/>
- Google — Gemini API pricing (official) — <https://ai.google.dev/gemini-api/docs/pricing>

Recorded segments at that check:

| Model | Input / output per 1M | Window | Status |
|---|---|---|---|
| Gemini 4 Hydrogen | $1.50 / $7.50 | from 2026-11-14, open-ended | priced |
| Gemini 4 Carbon | $2.00 / $10.00 | 2026-11-14 → 2026-11-30 | priced |
| Gemini 4 Carbon | $4.00 / $20.00 | 2026-12-01 → 2026-12-31 | priced |
| Gemini 4 Carbon | — | from 2027-01-01 | announced, unpriced |
| Gemini 4 Argon | $2.00 / $10.00 | 2026-11-14 → 2026-11-30 | priced |
| Gemini 4 Argon | $4.00 / $20.00 | 2026-12-01 → 2026-12-31 | priced |
| Gemini 4 Argon | — | from 2027-01-01 | announced, unpriced |

The endpoint is authoritative; it re-reads the same data the site renders. Check the upstream
sources before committing spend, and note that every figure here is an estimate of list prices, not
a measurement of your account.

## Licence

MIT for this repository's contents — see [LICENSE](LICENSE). The pricing figures are published by
Google and mirrored by LiteLLM; consult those sources for their terms.
