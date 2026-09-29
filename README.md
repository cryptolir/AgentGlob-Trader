# AgentGlob Trader

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
| [`mcp/hyperliquid/`](mcp/hyperliquid) | The MCP server — 8 typed tools for market data, account reads, orders, leverage and funding |
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

## Running it

The MCP is a stdio server. It needs two environment variables:

```bash
AGENTGLOB_RUNTIME_URL=https://your-runtime
AGENTGLOB_RUNTIME_TOKEN=<per-agent token>
```

**Be clear about what this repo is and is not.** The MCP is the thin half: it
speaks to a runtime API that holds the keys, enforces the caps and signs. That
runtime is part of the AgentGlob platform and is not in this repo. You can point
the client at your own implementation of the same `/api/runtime/hyperliquid/*`
endpoints — the contract is small and fully visible in
[`runtime-client.ts`](mcp/hyperliquid/runtime-client.ts) — but out of the box,
these tools expect AgentGlob behind them.

### The easy way — run it on AgentGlob

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

## The safety model, stated plainly

- **Caps are server-side.** Per-order notional, daily notional, max leverage,
  asset allowlist. The agent cannot read them or raise them; an order that tries
  to set its own is refused as an attempt, not a typo.
- **An empty allowlist allows nothing.** Absence is never permission.
- **Funding moves one direction only** — spot to perp. Perp back to spot is not
  restricted, it is not built: there is no code that constructs it.
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

## Contributing

Issues and pull requests welcome — especially additional venues. The shape here
(thin typed MCP + skills that carry the judgement + server-side caps) is meant to
be reusable for other exchanges.

## License

Apache License 2.0 — see [LICENSE](LICENSE).
