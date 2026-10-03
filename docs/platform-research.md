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
