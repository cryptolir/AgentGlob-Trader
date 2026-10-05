# AgentGlob Trader

[![CI](https://github.com/cryptolir/AgentGlob-Trader/actions/workflows/ci.yml/badge.svg)](https://github.com/cryptolir/AgentGlob-Trader/actions/workflows/ci.yml)

**Open-source trading tools for AI agents.** An MCP server and a set of agent
skills that let an AI agent read markets and trade
[Hyperliquid](https://hyperliquid.xyz) perpetual futures — without ever holding
a private key.

Built and used in production by [AgentGlob](https://agentglob.com).

---

## The idea: the agent never holds the key

Most "AI trading bot" setups hand the model an API key and hope the prompt holds.
This one does not.

```
   AI agent  ──►  MCP server  ──►  AgentGlob runtime  ──►  Hyperliquid
  (the model)    (this repo)      (keys, caps, signing)     (exchange)
     no key         no key          the only signer
```

The MCP server in this repo **holds no credential and never contacts the
exchange**. It calls an authenticated runtime API, which owns the delegated
trading key, enforces the owner's limits, and signs. The model's job is to
*ask* — and every refusal is decided somewhere the model cannot reach.

Practical consequences:

- **A prompt injection cannot lift a limit.** Caps live behind the API, not in
  the context window. An order over the cap comes back `403 order_cap_exceeded`
  no matter how convincingly the model was asked.
- **There is no withdrawal path in the code at all.** Not disabled by a flag —
  absent. No function builds one, so nothing can sign one. Funds cannot leave
  the account through these tools.
- **Trading uses a delegated key, not your wallet key.** Hyperliquid's API
  wallets can sign orders for an account but cannot move funds off it.

## What is here

| | |
|---|---|
| [`mcp/hyperliquid/`](mcp/hyperliquid) | The MCP server — 9 typed tools for market data, account reads, orders, leverage, funding and stablecoin conversion |
| [`skills/`](skills) | Three agent skills: how to read the market, how to trade, and how to size a trade so it stays inside its limits |

### The tools

| Tool | Does |
|---|---|
| `hl_market_data` | Mid prices, order book, candles, funding rates, perp metadata |
| `hl_account` | This agent's own positions, margin, balances, open orders, fills |
| `hl_place_order` | Limit order, long or short, with reduce-only and time-in-force |
| `hl_cancel_order` | Cancel one resting order by id |
| `hl_set_leverage` | Per-asset leverage, cross or isolated |
| `hl_transfer` | Move USDC from spot to perp so it can back a trade |
| `hl_transfer_status` | Reconcile an in-flight transfer against the exchange ledger |
| `hl_swap` | Convert USDH, USDT0 or USDE into USDC — never below $0.99 |
| `hl_account_status` | Trading readiness: key present, approval valid, expiry |

Account reads are always **this agent's own account**. There is no parameter for
someone else's address; the server fills it in.

### The skills are the interesting half

The MCP gives a model *capability*. The skills give it *judgement* — and most of
what an agent gets wrong about trading is judgement, not syntax.

- **[`hyperliquid-monitor`](skills/hyperliquid-monitor/SKILL.md)** — reading the
  market and the account. Which number actually answers "how much is in there",
  why `szi` carries the direction in its sign, and why a shared rate limit means
  your polling loop is not yours alone.
- **[`hyperliquid-trading`](skills/hyperliquid-trading/SKILL.md)** — placing and
  cancelling. Why `ok: true` **does not mean the order is live**, why there are
  no market orders, no stop-losses and no modify, and what every refusal code
  means.
- **[`hyperliquid-risk`](skills/hyperliquid-risk/SKILL.md)** — sizing and
  leverage. Why every order costs its full size against the daily budget *even if
  it never fills*, why that means an agent can spend its whole allowance opening
  positions and then be unable to close them, and why liquidation distance is the
  one number worth reporting unprompted.

They are plain Markdown. Read them even if you never run this code — they
document a lot of hard-won detail about agents and this exchange.

## Using it

This repo has three parts, and each one travels differently:

| Part | Works with other agents? |
|---|---|
| **The skills** | **Yes, any agent.** Plain Markdown that any skill-aware runtime can load. |
| **The MCP server** | **Yes, any MCP client** — Claude Code, Claude Desktop, Cursor, OpenClaw and others. It needs a backend to talk to. |
| **AgentGlob's backend** | **Only for agents running on AgentGlob.** |

That last line is on purpose. AgentGlob's backend only accepts calls from the
agents it hosts. If someone copies an agent's token onto a laptop, every call
from there is refused. So a leaked token is useless to anyone else.

### On AgentGlob (works out of the box)

[AgentGlob](https://agentglob.com) gives an AI agent a persistent home: its own
container, its own wallet, its own memory, and a dashboard where a human stays in
charge of what it may do.

1. Create an agent in the dashboard.
2. **Wallet tab** → generate a wallet, then provision a **Trading Key**. The
   trading key is a Hyperliquid API wallet: it signs orders and cannot withdraw.
   Your wallet key never enters the agent's container.
3. Set the caps — per-order size, daily total, maximum leverage, and which
   assets are allowed. The agent cannot change these and cannot read them.
4. **Tools tab** → add the Hyperliquid MCP. Add the skills from the catalog.

The agent can now trade, inside limits you set, and you can watch every order and
change the rules at any time — including switching trading off mid-position.

Every agent in a workspace can have it. Each one gets its own Trading Key and its
own limits, so a cautious agent and an active one can sit side by side.

### Skills in any agent

The skills are useful even without these tools: they teach an agent how
Hyperliquid really behaves (see [`skills/`](skills)).

**Claude Code** — copy them into your skills folder:

```bash
git clone https://github.com/cryptolir/AgentGlob-Trader
cp -r AgentGlob-Trader/skills/hyperliquid-* ~/.claude/skills/
```

**Other runtimes** — copy the three `hyperliquid-*` folders to wherever your
runtime reads skills from. Each is a `SKILL.md` with the standard `name` and
`description` frontmatter.

One thing to adapt: when something needs a human, the skills tell the agent to
ask its owner to use the AgentGlob dashboard (the Wallet tab, for example).
Elsewhere, that means whoever runs your backend.

### The MCP server in any MCP client

The server needs Node 20 or newer, and two settings:

| Setting | Value |
|---|---|
| `AGENTGLOB_RUNTIME_URL` | Your backend's address |
| `AGENTGLOB_RUNTIME_TOKEN` | The token your backend expects |

**Claude Code:**

```bash
claude mcp add hyperliquid \
  -e AGENTGLOB_RUNTIME_URL=https://your-backend.example \
  -e AGENTGLOB_RUNTIME_TOKEN=your-token \
  -- npx -y github:cryptolir/AgentGlob-Trader
```

**Claude Desktop, Cursor and most other clients** take the same thing as JSON —
Claude Desktop in `claude_desktop_config.json` (Settings → Developer → Edit
Config), Cursor in `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "hyperliquid": {
      "command": "npx",
      "args": ["-y", "github:cryptolir/AgentGlob-Trader"],
      "env": {
        "AGENTGLOB_RUNTIME_URL": "https://your-backend.example",
        "AGENTGLOB_RUNTIME_TOKEN": "your-token"
      }
    }
  }
}
```

`npx` downloads the repo, builds it and starts the server. If a setting is
missing, it stops at once with a message saying which.

**The backend is yours to write.** It is small — eight endpoints, all listed in
[`mcp/hyperliquid/README.md`](mcp/hyperliquid/README.md#the-runtime-contract).
It is also where every safety rule lives: the key, the limits and the signing.
Do not move them into the MCP server — the whole point is that the agent can
reach the server, so the server cannot be the thing that protects you.

Or skip writing a backend: run the agent on AgentGlob.

### From source

```bash
git clone https://github.com/cryptolir/AgentGlob-Trader
cd AgentGlob-Trader
npm install   # also builds, into dist/
npm test
```

`npm start` runs the server once the two settings are set.

## The safety model, stated plainly

- **Caps are server-side.** Per-order notional, daily notional, max leverage,
  asset allowlist. The agent cannot read them or raise them; an order that tries
  to set its own is refused as an attempt, not a typo.
- **An empty allowlist allows nothing.** Absence is never permission.
- **Funding moves one direction only** — spot to perp. Perp back to spot is not
  restricted, it is not built: there is no code that constructs it.
- **The one spot action is narrow by construction.** `hl_swap` converts three
  hand-picked stablecoins into USDC and nothing else — it cannot buy a coin
  whose price moves. Its limit price *is* its floor, so it never sells below
  $0.99; a coin that has lost its $1 value simply does not sell. It reuses the
  ordinary order action, so it adds nothing new that a key could sign.
- **Mainnet only, small sizes.** No paper mode. The safety comes from the caps
  and from position sizes a person chose, not from a sandbox.

### What it still cannot protect you from

Worth saying out loud, because the skills say it to the agent too:

- **There is no stop-loss.** Hyperliquid has no resting protective order here,
  and nothing closes a losing position while the agent is not looking.
- **An agent can lose money inside its caps.** The caps bound the size of a
  mistake, not whether one happens.
- **Nothing here is financial advice**, and the skills instruct the agent to
  refuse to give any.


## Integration plans

- [QuantConnect integration](docs/quantconnect-integration-plan.md) — use QuantConnect for strategy research, backtesting and optimization while keeping Hyperliquid execution behind AgentGlob's existing runtime controls.
- [Platform research](docs/platform-research.md) — short, plain-English summaries of outside platforms (QuantConnect, Binance, LEAN, Hummingbot) and how they could work with an AgentGlob trading agent.

## Contributing

Issues and pull requests welcome — especially additional venues. The shape here
(thin typed MCP + skills that carry the judgement + server-side caps) is meant to
be reusable for other exchanges.

`npm install` builds and `npm test` runs the tests — the same checks run on
every pull request.

## License

Apache License 2.0 — see [LICENSE](LICENSE).
