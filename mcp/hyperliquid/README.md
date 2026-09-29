# Hyperliquid MCP server

A stdio [MCP](https://modelcontextprotocol.io) server exposing Hyperliquid
market data, account reads and bounded trading as eight typed tools.

It **holds no credential and never contacts `api.hyperliquid.xyz`**. Every call
goes to an authenticated runtime API that owns the trading key, enforces the
owner's caps and does the signing.

## Files

| File | |
|---|---|
| `server.ts` | The MCP server: tool listing, dispatch, and error mapping |
| `tools.ts` | The eight tool definitions — schemas and handlers |
| `runtime-client.ts` | Thin HTTP client for the runtime API. The whole contract is here |
| `tools.test.ts` | Tests for the direction guard on `hl_transfer` |

## Environment

```bash
AGENTGLOB_RUNTIME_URL=https://your-runtime
AGENTGLOB_RUNTIME_TOKEN=<per-agent token>
```

Both are required and validated at startup, so a missing credential fails
immediately instead of surfacing as a confusing `401` on the first trade.

## Design notes

**Tools are grouped, not one-per-endpoint.** A single `hl_market_data` with a
typed `kind` beats seven near-identical tools — better for model accuracy and
for review surface. Same for `hl_account`.

**This layer enforces nothing.** Typed schemas stop a model inventing
parameters; they do not stop it wanting the wrong thing. Caps, the asset
allowlist and the key all live behind the runtime API, which an agent can reach
directly — so that API is the security boundary and this file is ergonomics.
It is deliberately not treated as a boundary.

**Refusals are surfaced verbatim, with their code.** A cap refusal is an
*answer*, not a fault: the model needs to see `order_cap_exceeded` and stop,
rather than retry differently.

**A non-JSON response is an error, not a success.** An HTML login redirect
parses as nothing useful; treating it as a result would let a misrouted call
read as a completed trade.

**`hl_transfer` validates its own direction.** The advertised enum does not
bind — the MCP protocol hands raw arguments to the handler. Hardcoding the
direction would turn a "move it back to spot" request into another deposit
*into* perp: the opposite fund movement, silently. It refuses instead of
rewriting. `tools.test.ts` pins this.

## The runtime contract

If you want to run these tools against your own backend, implement these
endpoints. `runtime-client.ts` is the complete specification.

| Method | Path | Body |
|---|---|---|
| POST | `/api/runtime/hyperliquid/info` | Hyperliquid info request; `user` is filled in server-side |
| POST | `/api/runtime/hyperliquid/order` | `{ coin, isBuy, px, sz, reduceOnly?, tif? }` |
| POST | `/api/runtime/hyperliquid/cancel` | `{ coin, oid }` |
| POST | `/api/runtime/hyperliquid/leverage` | `{ coin, leverage, isCross? }` |
| POST | `/api/runtime/hyperliquid/transfer` | `{ amount, direction: "spot_to_perp" }` |
| GET | `/api/runtime/hyperliquid/transfer` | — reconcile an in-flight transfer |
| GET | `/api/runtime/hyperliquid/status` | — trading readiness |

Authentication is `Authorization: Bearer <AGENTGLOB_RUNTIME_TOKEN>`.

Errors should return a non-2xx status with `{ error, code }`. The `code` is what
the agent reasons about — the skills in this repo document the full set.
