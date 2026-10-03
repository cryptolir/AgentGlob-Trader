# Platform research

Notes on outside trading platforms and how they could work with an AgentGlob trading agent. Each section is a short, high-level summary, not a promise of a feature.

## QuantConnect

*Checked October 2026. For the detailed build plan, see [QuantConnect integration plan](quantconnect-integration-plan.md).*

### What QuantConnect offers

- A lab for writing strategies and testing them on years of past prices, plus a big data library. **Free.**
- Live running of a strategy, with fake money (paper trading) or real money. **Paid.**
- An API, and an MCP for AI tools, that can drive all of it.
- **The missing piece:** Hyperliquid is not one of its official exchanges.

### Best fit: QuantConnect decides, the AgentGlob agent trades

```
QuantConnect strategy  ──signal──►  AgentGlob  ──►  trading agent  ──►  Hyperliquid
(tested, running with              "SOL: 20%")      checks the owner's limits,
 fake money)                                        places the real order
```

1. Build and test the strategy in QuantConnect (free).
2. Run it there in paper mode. It never touches real money or keys.
3. When it wants to trade, it sends a signal (a web request) to AgentGlob.
4. The agent places the real order with its own Trading Key, inside the owner's limits.

**Why this shape:** keys stay in AgentGlob, the owner's limits and off switch still work, and QuantConnect never touches the money.

**Cost and limits:** about $28/month (smallest paid plan plus one live server). That plan allows 20 signals per hour, which is fine for slow strategies such as a daily rotation, but not for fast trading.

**What would need building:** a signal inbox for each agent, locked with a secret the owner pastes into QuantConnect, plus an on/off switch in the dashboard. One choice for the owner: the agent either follows each signal exactly, or treats it as advice and asks first.

### Second use: a research lab for the agent (free)

The agent could use QuantConnect's API to test an idea on past prices and show the results before suggesting a trade. QuantConnect's standalone MCP is marked deprecated (they now ship it inside their VS Code extension), so this would mean wrapping their API in a small tool, the same way this repo wraps the Hyperliquid runtime.

### Not recommended: QuantConnect trading Hyperliquid directly

Only a third-party add-on supports it, and the exchange keys would go to outside code. The owner's limits and dashboard control would be lost.

### One thing to know

QuantConnect's crypto prices come from big exchanges such as Binance and Coinbase, not Hyperliquid. Test results will be close to what the agent sees live, but not exact, because fees and funding differ.

### Sources

- [QuantConnect MCP server (official, marked deprecated)](https://github.com/QuantConnect/mcp-server)
- [QuantConnect docs: live trading notifications](https://www.quantconnect.com/docs/v2/writing-algorithms/live-trading/notifications)
- [QuantConnect docs: live trading commands](https://www.quantconnect.com/docs/v2/writing-algorithms/live-trading/commands)
- [QuantConnect docs: plan quotas (nodes, notifications)](https://www.quantconnect.com/docs/v2/cloud-platform/organizations/resources)
- [Third-party LEAN add-on with Hyperliquid](https://github.com/ypsik/LeanSharedFuturesBrokerage)
- [QuantConnect pricing review 2026](https://newyorkcityservers.com/blog/quantconnect-review)

## Binance (official MCP)

*Checked October 2026. Binance launched its official MCP as part of "Agent OS" on August 20, 2026.*

### What Binance offers

- **One online address, nothing to install:** `https://agent.binance.com/mcp/agentic`.
- **Prices for free:** prices, order books, charts and funding rates need no login.
- **Trading:** spot, margin, Convert (swap one coin for another) and futures, where the account and country allow it.
- **No API keys:** the owner signs in with their Binance account in a browser and approves what the agent may do.
- **A separate agent account:** the agent trades only inside its own sub-account, which the owner funds by hand. It can never withdraw, and it cannot pull money from the main account.
- **One-click cut-off:** "Disconnect agents" on Binance cuts the agent off.

### Best fit: through AgentGlob, the same way as Hyperliquid

```
trading agent ──► AgentGlob (owner's limits, the Binance login) ──► Binance MCP ──► agent sub-account
```

1. In the dashboard, the owner clicks "Connect Binance" and approves on Binance's page. AgentGlob keeps the login; the agent never sees it.
2. The owner moves money into the agent's sub-account.
3. The agent asks AgentGlob to trade. AgentGlob checks the owner's limits first (size, daily total, allowed coins, leverage), then passes the order to Binance.
4. Two off switches: "Disconnect" in AgentGlob, and "Disconnect agents" on Binance.

**Why this shape:** two layers of safety. Binance caps the risk at what is in the sub-account, and AgentGlob caps each trade and each day. The trading skills in this repo (monitor, trading, risk) mostly carry over.

### Simpler, but weaker: the agent connects straight to Binance

Almost nothing to build. The catch: the login sits inside the agent, and AgentGlob's limits and dashboard are skipped, so Binance's sub-account is the only safety net. Fine for a quick test with a small amount.

### Things to check first

- **The login step is the big unknown.** Binance's docs describe the sign-in done in a desktop browser with the AI app open. It needs testing whether a server (AgentGlob) can complete that sign-in and keep it working. If not, only the simpler path works.
- **How long the login lasts** before it needs renewing is not documented.
- **Where Binance is available:** Binance.com is not open in some countries, and futures depend on the account.
- **Binance holds the money.** With Hyperliquid the funds sit in the owner's own wallet; with Binance the exchange holds them.

### Sources

- [Binance docs: Binance MCP Server](https://developers.binance.com/en/docs/agent-native/mcp-server/agentic)
- [Binance docs: MCP Server intro](https://developers.binance.com/en/docs/agent-native/mcp-server)
- [crypto.news: Binance launches Agent OS and MCP trading server](https://crypto.news/binance-launches-agent-os-and-mcp-trading-server/)
- [PR Newswire: Binance introduces Agent OS](https://www.prnewswire.com/apac/news-releases/binance-introduces-agent-os-to-connect-ai-applications-to-financial-infrastructure-302856314.html)

## LEAN (open-source engine)

*Checked October 2026. LEAN is the free, open-source trading engine underneath QuantConnect. The QuantConnect website is a hosted service built on it; LEAN is the same engine, run on your own servers.*

### What it is

- A program that runs trading strategies written in Python or C#.
- **Backtesting:** it tests a strategy on past prices.
- **Live trading:** it runs the strategy for real, or with fake money.
- Free and open (Apache 2.0, the same license as this repo), widely used (about 22k GitHub stars), and actively maintained.

### How to run it

- **The easy way:** the `lean` command-line tool runs everything in Docker. It needs a **paid QuantConnect plan**, and some of its data costs extra.
- **The hard way:** build the engine from source (.NET 10). No account needed, but you bring your own price data and do more setup.

**Crypto exchanges it trades on:** Binance, Bybit, Kraken, Coinbase, Bitfinex and dYdX. **Not Hyperliquid.** A one-person add-on adds Hyperliquid and 7 more exchanges, but it has no outside users and the author says it is built for their own use.

### How it could fit an AgentGlob trading agent

1. **Backtesting on our own server (best fit).** Run LEAN on AgentGlob's servers. The agent hands it a strategy, gets the test results, and shows them to the owner before suggesting trades. No money or keys are involved, and there are no per-signal limits like the QuantConnect website's. Cost: server time plus price data.
2. **LEAN decides, the agent trades.** LEAN runs the strategy live in paper mode and sends signals to AgentGlob, the same as the QuantConnect plan above but on our own servers. The owner's limits and dashboard still apply.
3. **LEAN trades directly (not recommended).** It would need the one-person Hyperliquid add-on, and exchange keys would go into LEAN. That skips the owner's limits and the dashboard.

**Compared with the QuantConnect website:** LEAN costs nothing per month and has no signal cap, but you run the servers and find the data yourself.

### Sources

- [LEAN on GitHub](https://github.com/QuantConnect/Lean)
- [LEAN CLI: getting started](https://www.lean.io/docs/v2/lean-cli/key-concepts/getting-started)
- [LEAN CLI: supported brokerages](https://www.lean.io/docs/v2/lean-cli/live-trading/brokerages)
- [Third-party LEAN add-on with Hyperliquid](https://github.com/ypsik/LeanSharedFuturesBrokerage)
