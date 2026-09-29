# Trading skills

Three skills that give an agent the *judgement* to use the Hyperliquid MCP —
what the numbers mean, what the platform cannot do, and how to stay inside the
limits its owner set instead of discovering each one by being refused.

They are plain Markdown with YAML frontmatter, in the
[Agent Skills](https://modelcontextprotocol.io) format used by
[openclaw](https://github.com/cryptolir/openclaw) and AgentGlob. Any agent
runtime that loads a `SKILL.md` by its `description` can use them.

| Skill | Read it for |
|---|---|
| [`hyperliquid-monitor`](hyperliquid-monitor/SKILL.md) | Reading the market and the account. Read-only — places nothing, moves nothing. |
| [`hyperliquid-trading`](hyperliquid-trading/SKILL.md) | Placing, closing and cancelling orders, funding the perp account, and converting a stablecoin into USDC. |
| [`hyperliquid-risk`](hyperliquid-risk/SKILL.md) | Sizing, leverage, and what actually counts against a daily cap. |

## Why these are worth reading on their own

Most of what goes wrong when an LLM trades is not syntax. It is confidently
acting on a wrong mental model. These encode the corrections:

- **`ok: true` does not mean the order is live.** Hyperliquid reports each
  order's outcome *inside* a successful response — a rejection arrives with no
  error code at all. An agent that trusts the envelope reports a trade that
  never happened.
- **Every order costs its full size against the daily budget when it is sent** —
  refunded only if the request fails outright. A resting order you cancel still
  spent its notional. So an agent can burn its whole allowance opening positions
  and then be unable to close them.
- **`sz` is in units of the asset, never dollars.** Confusing the two is how an
  order comes out a thousand times too big.
- **`szi` carries the direction in its sign.** `-0.5` on ETH is short half an
  ether, not long.
- **There is no stop-loss, no take-profit and no modify.** An agent must never
  imply a position is protected while it is not looking.
- **Cancelling is gated too.** If trading is switched off, or an asset is taken
  off the allowlist, the agent cannot cancel its own resting orders. That needs
  to be said out loud, immediately, to a human.

## Installing them

**On AgentGlob:** add them from the skill catalog on the agent's Tools tab.

**Elsewhere:** copy the directory into wherever your runtime reads skills from.
The frontmatter declares the two environment variables the MCP needs, so a
runtime that checks requirements can tell you what is missing before the agent
tries to trade.

## A note on tone

These are written to be read by a model at decision time, so they are blunt,
they repeat the dangerous points, and they say "you cannot" rather than
"this is not currently supported". That is deliberate. Hedged documentation
produces hedged behaviour, and an agent that is unsure whether it can set a
stop-loss will imply that it did.
