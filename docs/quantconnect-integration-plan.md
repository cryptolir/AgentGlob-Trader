# QuantConnect Integration Plan

> Status: Proposed  
> Scope: AgentGlob Trader + QuantConnect + Hyperliquid  
> Last updated: 2026-09-30

## 1. Goal

Add QuantConnect as a **quant research, strategy-generation, backtesting, and optimization layer** for AgentGlob trading agents, while keeping the existing AgentGlob Trader architecture as the only path for live Hyperliquid execution.

The intended system is:

```text
                         AgentGlob / OpenClaw Agent
                                  |
                    +-------------+-------------+
                    |                           |
             QuantConnect MCP          AgentGlob Trader MCP
                    |                           |
       research / projects /            Hyperliquid market
       backtests / optimization         + account + orders
                    |                           |
            QuantConnect Cloud            AgentGlob Runtime
                                                |
                                      keys + caps + signing
                                                |
                                           Hyperliquid
```

The key principle is:

**QuantConnect may help the agent decide what to trade. It must not bypass AgentGlob's execution controls.**

---

## 2. Why integrate QuantConnect

AgentGlob Trader already provides a strong execution and safety layer for Hyperliquid:

- market and account reads;
- bounded order placement;
- leverage control;
- delegated Hyperliquid trading keys;
- server-side asset allowlists;
- server-side order and daily caps;
- no withdrawal path;
- no private key inside the AI agent or MCP process.

What it does not provide is a full quantitative research environment.

QuantConnect adds that missing layer:

- create and manage algorithm projects;
- generate and edit LEAN strategies;
- run historical backtests;
- inspect performance;
- optimize parameters;
- compare strategy variants;
- use QuantConnect datasets and custom datasets;
- optionally deploy algorithms through QuantConnect-supported brokerages.

For AgentGlob, the initial integration should use QuantConnect for **research and validation only**. Hyperliquid execution remains in AgentGlob Trader.

---

## 3. Non-goals for the first release

The first release should **not**:

1. Give QuantConnect access to the Hyperliquid private key or delegated trading key.
2. Let QuantConnect send orders directly to Hyperliquid.
3. Reimplement AgentGlob's risk caps inside prompts, skills, or QuantConnect code.
4. Treat a successful backtest as automatic permission to trade.
5. Replace the existing Hyperliquid MCP.
6. Build a full custom LEAN Hyperliquid brokerage integration.
7. Automatically promote every generated strategy to live execution.
8. Allow the model to modify owner-defined trading caps.

A custom LEAN brokerage adapter can be considered later, but it is intentionally outside the MVP.

---

## 4. Current external constraint

As of 2026-09-30, Hyperliquid is not listed as a native QuantConnect live brokerage.

Therefore the initial architecture should be:

```text
QuantConnect
    |
research + backtest result
    |
AgentGlob agent
    |
AgentGlob Trader MCP
    |
AgentGlob Runtime
    |
Hyperliquid
```

and not:

```text
QuantConnect -> Hyperliquid
```

This separation is also desirable from a security and product-control perspective.

---

## 5. Integration architecture

### 5.1 Agent tool layer

An AgentGlob trading agent should be able to connect to both:

#### A. QuantConnect MCP

Purpose:

- create research/algorithm projects;
- write or modify LEAN strategy code;
- compile strategies;
- launch backtests;
- inspect backtest metrics and trade history;
- run optimization;
- compare variants;
- manage QuantConnect-side research artifacts.

The QuantConnect remote MCP endpoint is currently:

```text
https://www.quantconnect.com/api/v2/mcp
```

QuantConnect currently requires a paid plan for remote MCP access.

#### B. AgentGlob Trader Hyperliquid MCP

Purpose:

- read current Hyperliquid market data;
- read the agent's account and positions;
- inspect open orders and fills;
- set bounded leverage;
- place bounded orders;
- cancel orders;
- perform the narrowly supported funding/stablecoin actions.

The existing security model remains unchanged:

```text
AI agent
   |
Hyperliquid MCP
   |
AgentGlob Runtime
   |  server-side caps + delegated key + signing
Hyperliquid
```

### 5.2 Responsibility split

| Component | Responsibility |
|---|---|
| AgentGlob/OpenClaw agent | Orchestrates research, interprets results, evaluates live conditions |
| QuantConnect MCP | Quant research, LEAN projects, backtesting, optimization |
| QuantConnect Cloud | Runs research/backtests and stores QC project artifacts |
| AgentGlob Trader MCP | Typed Hyperliquid capability surface |
| AgentGlob Runtime | Authentication, keys, risk controls, signing, execution |
| Hyperliquid | Market data, account state, order matching |

No QuantConnect component should become part of AgentGlob's security boundary in the MVP.

---

## 6. Proposed agent workflow

Example owner request:

> Research a BTC momentum strategy using 15-minute candles. Backtest it over the last three years. Only consider it eligible if it meets the configured strategy-validation rules. If current Hyperliquid conditions match the validated strategy, propose or execute the trade according to this agent's execution policy.

Recommended flow:

### Step 1 — Define the research brief

The agent converts the request into a structured research brief:

- market/symbol;
- timeframe;
- signal hypothesis;
- entry rule;
- exit rule;
- leverage assumption;
- fee/slippage assumptions;
- backtest period;
- validation thresholds;
- whether optimization is permitted;
- whether live execution is permitted or human confirmation is required.

### Step 2 — Run QuantConnect research

Using QuantConnect MCP, the agent:

1. creates or selects a QuantConnect project;
2. writes the LEAN strategy;
3. compiles it;
4. runs the backtest;
5. reads results;
6. fixes implementation errors if needed;
7. optionally runs parameter optimization;
8. records the final strategy definition and result.

### Step 3 — Validate the result

The agent must not reason from one headline metric.

At minimum, the strategy-validation layer should consider:

- total return;
- Sharpe or equivalent risk-adjusted return;
- maximum drawdown;
- number of trades;
- win/loss distribution;
- average trade;
- exposure;
- turnover;
- estimated trading fees;
- estimated slippage;
- parameter sensitivity;
- in-sample vs out-of-sample performance;
- evidence of overfitting;
- test period and data provenance.

The validation rules belong in an AgentGlob skill/policy, not in the execution MCP.

### Step 4 — Reconcile with current Hyperliquid state

The agent uses the existing Hyperliquid MCP to read:

- current market price;
- relevant candles/order book/funding;
- account balance;
- current positions;
- open orders;
- current leverage;
- trading readiness.

This is important: **the QuantConnect backtest result is historical research, while the Hyperliquid MCP provides the live execution context.**

### Step 5 — Generate a trade intent

The agent creates an internal trade intent such as:

```json
{
  "strategy_id": "qc-project/backtest-reference",
  "coin": "BTC",
  "side": "long",
  "entry_reason": "validated momentum entry condition",
  "desired_notional": 250,
  "desired_leverage": 2,
  "execution": "limit",
  "reduce_only": false
}
```

This object is advisory. It is **not** an authorization token and cannot override owner limits.

### Step 6 — Execute through AgentGlob Trader

The agent calls the existing tools:

```text
hl_account_status
hl_account
hl_market_data
hl_set_leverage
hl_place_order
```

The AgentGlob Runtime independently enforces:

- key validity;
- allowed assets;
- maximum leverage;
- maximum order size;
- daily notional allowance;
- other owner-defined constraints.

A rejected order remains rejected regardless of the QuantConnect result.

### Step 7 — Record provenance

For every strategy-driven trade, store enough provenance to reconstruct why it happened:

- QuantConnect project ID;
- backtest ID;
- strategy/code version or hash;
- backtest period;
- metrics used for validation;
- optimization ID if applicable;
- Hyperliquid market snapshot/time;
- resulting AgentGlob order request;
- runtime approval/refusal;
- Hyperliquid order/fill IDs.

This should make it possible to answer:

> Which backtest and strategy version caused this trade?

---

## 7. Hyperliquid-native backtesting

A generic BTC/ETH backtest from another venue will not perfectly reproduce Hyperliquid perpetual trading.

Differences may include:

- funding;
- fees;
- spread;
- liquidity;
- slippage;
- mark/index-price behavior;
- leverage/margin mechanics;
- liquidation behavior;
- available contracts;
- listing history.

Therefore the integration should support two research modes.

### Mode A — Fast research

Use QuantConnect-native crypto/market data.

Purpose:

- idea generation;
- signal experimentation;
- broad strategy comparison.

Result must be labelled as **venue-approximate**.

### Mode B — Hyperliquid-specific validation

Use Hyperliquid historical data inside QuantConnect through a custom dataset.

QuantConnect supports custom external data through LEAN custom data types. This lets us import Hyperliquid-derived historical information without making Hyperliquid a native QuantConnect brokerage.

Recommended dataset fields:

```text
timestamp
coin
open
high
low
close
volume
mark_price
index_price
funding_rate
open_interest
best_bid
best_ask
```

Not every field is required for v1.

### Minimum useful v1 dataset

Start with:

```text
timestamp
coin
open
high
low
close
volume
funding_rate
```

Initial symbols:

- BTC
- ETH

Initial resolutions:

- 1 minute or 5 minute raw source;
- strategies can consolidate upward.

### Data service boundary

Do not make QuantConnect fetch through the trading/signing runtime.

Create a separate read-only data path:

```text
Hyperliquid public data
       |
AgentGlob data collector
       |
versioned historical dataset
       |
QuantConnect custom data
```

The execution runtime should remain focused on account-scoped operations and signing.

---

## 8. Repository changes

Recommended additions to this repository:

```text
AgentGlob-Trader/
├── docs/
│   └── quantconnect-integration-plan.md
│
├── skills/
│   ├── hyperliquid-monitor/
│   ├── hyperliquid-risk/
│   ├── hyperliquid-trading/
│   ├── quant-strategy-research/          # Phase 2
│   │   └── SKILL.md
│   └── quant-strategy-validation/        # Phase 2
│       └── SKILL.md
│
└── mcp/
    └── hyperliquid/                      # unchanged security boundary
```

The QuantConnect MCP itself does **not** need to be copied into this repo. AgentGlob should configure the external QuantConnect MCP as an additional tool server.

---

## 9. New AgentGlob skills

### 9.1 `quant-strategy-research`

Purpose: teach the agent how to turn a trading hypothesis into reproducible QuantConnect research.

Responsibilities:

- convert user intent into a research brief;
- select suitable timeframe and dataset;
- define explicit entry/exit logic;
- include fees/slippage assumptions;
- prevent look-ahead bias;
- separate training/optimization and evaluation periods;
- use QuantConnect MCP to run the work;
- return structured research output.

Suggested output:

```json
{
  "strategy": "...",
  "project_id": "...",
  "backtest_id": "...",
  "period": {"from": "...", "to": "..."},
  "data_source": "...",
  "metrics": {},
  "assumptions": [],
  "limitations": []
}
```

### 9.2 `quant-strategy-validation`

Purpose: stop the agent from treating a visually attractive backtest as proof that a strategy is safe or robust.

Responsibilities:

- evaluate risk-adjusted metrics;
- inspect drawdown;
- check trade count/sample size;
- compare in/out-of-sample performance;
- check sensitivity to parameter changes;
- flag likely overfitting;
- identify data/venue mismatch;
- produce `eligible`, `not_eligible`, or `needs_review`.

Important:

**Eligibility is a research decision, not execution permission.**

Even an eligible strategy still goes through AgentGlob Runtime risk controls.

---

## 10. Configuration

Agent configuration should support a QuantConnect connection without exposing credentials inside strategy prompts.

Conceptually:

```text
Tools
├── Hyperliquid MCP
└── QuantConnect MCP
```

QuantConnect authentication should be managed by the AgentGlob tool/gateway layer or the MCP authorization flow, not written into:

- SOUL.md;
- SKILL.md;
- agent memory;
- project notes;
- chat history.

Suggested AgentGlob configuration metadata:

```json
{
  "quantconnect": {
    "enabled": true,
    "execution_enabled": false,
    "hyperliquid_custom_data": false
  }
}
```

`execution_enabled` should remain `false` for this integration. Hyperliquid live execution belongs to AgentGlob Trader.

---

## 11. Security requirements

The existing AgentGlob Trader guarantees remain mandatory.

### MUST

- Keep Hyperliquid delegated trading keys in the AgentGlob runtime.
- Keep owner caps server-side.
- Treat QuantConnect output as untrusted decision input.
- Require every live trade to pass existing AgentGlob runtime checks.
- Maintain an audit link between research and execution.
- Separate Hyperliquid public historical-data ingestion from signing infrastructure.
- Sanitize strategy identifiers and external metadata before storing them.
- Apply timeouts/retry limits to QuantConnect research loops.

### MUST NOT

- Send a Hyperliquid private key to QuantConnect.
- Send the delegated Hyperliquid signing key to QuantConnect.
- Add owner caps to prompts as the primary enforcement mechanism.
- Give QuantConnect a bypass endpoint to AgentGlob's runtime.
- Automatically increase execution limits because a backtest performs well.
- Treat MCP schema validation as a security boundary.
- Let an agent edit the controls that authorize its own trades.

---

## 12. Failure handling

### QuantConnect unavailable

The agent may continue to monitor Hyperliquid but must not claim a strategy was newly validated.

### QuantConnect backtest fails

Return the compilation/runtime error and correct the research project. Do not convert a failed backtest into a trade signal.

### Weak or incomplete result

Mark the research as `needs_review` or `not_eligible`.

### Hyperliquid runtime refuses execution

Surface the runtime refusal code exactly, consistent with the existing AgentGlob Trader design.

Do not try to route around the refusal.

### Research/live data disagreement

Prefer the current Hyperliquid market/account state for execution decisions.

A stale QuantConnect result must never override live state.

---

## 13. Implementation phases

## Phase 1 — Dual MCP proof of concept

**Objective:** One AgentGlob agent can use QuantConnect for backtesting and AgentGlob Trader for live Hyperliquid tools in the same session.

Tasks:

- add QuantConnect MCP configuration support to AgentGlob;
- authenticate an AgentGlob/OpenClaw agent to QuantConnect;
- verify project creation;
- verify compile;
- verify backtest creation;
- verify backtest result retrieval;
- keep current Hyperliquid MCP unchanged;
- create a simple end-to-end BTC research workflow;
- require explicit human approval before any resulting live trade during POC.

Acceptance criteria:

- agent can create and backtest a strategy using QuantConnect;
- agent can quote the resulting backtest ID and core metrics;
- agent can independently read current Hyperliquid state;
- no QuantConnect credential reaches Hyperliquid;
- no Hyperliquid key reaches QuantConnect;
- a trade still fails when AgentGlob Runtime caps reject it.

---

## Phase 2 — Research and validation skills

**Objective:** Make strategy research reproducible rather than prompt-dependent.

Tasks:

- add `skills/quant-strategy-research/SKILL.md`;
- add `skills/quant-strategy-validation/SKILL.md`;
- define standard research brief;
- define standard result schema;
- define validation statuses;
- add anti-look-ahead and anti-overfitting guidance;
- record QuantConnect project/backtest provenance;
- add tests/examples for common workflows.

Acceptance criteria:

- two agents given the same brief follow the same research stages;
- live trade intent cannot be generated from a failed backtest;
- validation output records assumptions and limitations;
- every trade intent can point back to a QuantConnect result.

---

## Phase 3 — Hyperliquid historical dataset

**Objective:** Reduce venue mismatch between research and execution.

Tasks:

- implement read-only Hyperliquid historical collector;
- persist normalized candles and funding;
- version datasets;
- expose dataset files through a read-only source suitable for QuantConnect;
- add LEAN custom data class/project template;
- tag backtests with dataset version;
- compare QuantConnect-native BTC backtests with Hyperliquid-native backtests.

Acceptance criteria:

- historical Hyperliquid data can be consumed by a QuantConnect project;
- backtests are reproducible from a dataset version;
- funding can be included in strategy evaluation;
- strategy reports clearly state whether data is venue-native or approximate.

---

## Phase 4 — Automation and monitoring

**Objective:** Allow controlled recurring strategy research and revalidation.

Examples:

- rerun a strategy weekly;
- revalidate after a material drawdown;
- rerun after strategy parameters change;
- compare current live performance with backtest expectations.

Requirements:

- research automation cannot change trading caps;
- a revalidated strategy does not automatically get a larger allocation;
- failures are recorded and surfaced;
- repeated optimization must not silently replace the approved strategy version.

---

## Phase 5 — Optional custom LEAN Hyperliquid brokerage

Only consider this if there is a strong need for LEAN-native execution.

Possible architecture:

```text
LEAN
  |
custom AgentGlob brokerage adapter
  |
AgentGlob Runtime
  |
Hyperliquid
```

Even in this model, LEAN should call AgentGlob Runtime instead of holding Hyperliquid signing keys.

A complete integration would need to model, at minimum:

- symbol mapping;
- order submission/cancellation;
- fills;
- account/position state;
- leverage;
- fees;
- live market data;
- historical data;
- funding;
- brokerage model constraints;
- reconnect/reconciliation behavior.

This phase is intentionally deferred because the dual-MCP architecture delivers most of the product value with much less security and maintenance surface.

---

## 14. Suggested first end-to-end test

Use one deliberately simple strategy.

### Research brief

- Venue target: Hyperliquid
- Symbol: BTC
- Resolution: 15 minutes
- Strategy: simple momentum/trend rule
- Backtest period: enough history to cover multiple market regimes
- No optimization on the first run
- Include fee/slippage assumptions
- Execution: disabled until result is manually reviewed

### Test flow

```text
Owner
  |
"Research BTC momentum"
  |
AgentGlob Agent
  |
QuantConnect MCP
  |
create -> compile -> backtest -> analyze
  |
structured validation
  |
Hyperliquid MCP
  |
read current BTC + account state
  |
trade proposal
  |
human approval
  |
hl_place_order
  |
AgentGlob Runtime checks
  |
Hyperliquid
```

### POC success condition

The test is successful when the agent can explain, with identifiers:

1. which strategy it tested;
2. which QuantConnect backtest produced the evidence;
3. which validation rules were applied;
4. what the current Hyperliquid state was;
5. what trade it requested;
6. whether AgentGlob Runtime accepted or rejected it;
7. the resulting Hyperliquid order/fill identifier if accepted.

---

## 15. Recommended product behavior

In the AgentGlob Dashboard, the owner should ultimately see separate concepts:

### Research

```text
QuantConnect
Connected

Last strategy:
BTC Momentum v3

Backtest:
2026-09-30 / <id>

Status:
Validated / Needs Review / Rejected
```

### Execution

```text
Hyperliquid
Connected

Trading key:
Active

Allowed assets:
BTC, ETH

Max leverage:
...

Per-order cap:
...

Daily cap:
...
```

Keeping research and execution visually separate reinforces the actual security architecture.

---

## 16. Final architecture decision

For the first production integration, use:

```text
QuantConnect = research engine
AgentGlob agent = orchestration + reasoning
AgentGlob Trader = Hyperliquid capability layer
AgentGlob Runtime = security + signing + execution
Hyperliquid = trading venue
```

Do **not** make QuantConnect the direct Hyperliquid execution layer for the MVP.

This gives AgentGlob agents access to a serious quantitative research environment without weakening the most important property of the current system:

> **The agent may request a trade, but the agent is never the authority that decides whether the trade is allowed.**

---

## References

- QuantConnect MCP Server: https://www.quantconnect.com/docs/v2/ai-assistance/mcp-server
- QuantConnect MCP key concepts: https://www.quantconnect.com/docs/v2/ai-assistance/mcp-server/key-concepts
- QuantConnect live brokerages: https://www.quantconnect.com/docs/v2/cloud-platform/live-trading/brokerages
- QuantConnect custom historical data: https://www.quantconnect.com/docs/v2/writing-algorithms/historical-data/custom-data
- AgentGlob Trader Hyperliquid MCP contract: ../mcp/hyperliquid/README.md
