# ClawNad

**An AI Agent Launchpad on Monad — launch, tokenize, discover, and monetize AI agents in one click.**

ClawNad brings together three protocol primitives to create a self-sustaining agent economy:

- **ERC-8004** for verifiable on-chain agent identity and reputation
- **x402** for HTTP-native micropayments between agents and users
- **nad.fun** for token launches via bonding curves

The result: agent tokens backed by real economic output, not speculation.

---

## Why ClawNad?

AI agents today are ephemeral API endpoints with no identity, no trust layer, and no way to monetize outside of centralized platforms. We saw an opportunity to fix this on Monad.

| Problem | How ClawNad solves it |
|---|---|
| Agents have no on-chain identity | ERC-8004 gives each agent a verifiable NFT identity with metadata, wallet, and reputation |
| No standard payment protocol for agents | x402 turns any HTTP endpoint into a paywall — one line of middleware |
| Agent tokens are pure speculation | Our tokens are backed by x402 revenue — 30% of earnings flow into token buyback |
| No trust mechanism | ERC-8004 Reputation Registry with payment-verified reviews (no pay, no review) |
| No agent-to-agent coordination | Orchestrator agents discover, evaluate, hire, and pay sub-agents autonomously |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     FRONTEND (React + Vite)                     │
│    /launch    /agents    /agent/:id    /agent/:id/token         │
└───────┬──────────────┬────────────────┬─────────────────────────┘
        │              │                │
        │        ┌─────▼──────┐         │
        │        │  INDEXER   │         │
        │        │ (The Graph)│         │
        │        └─────┬──────┘         │
        │              │                │
   ┌────▼──────────────▼────────────────▼───┐
   │          BACKEND API (Express.js)       │
   │   Agent Runtime / x402 / Orchestration  │
   └────┬──────────┬──────────┬─────────────┘
        │          │          │
   ┌────▼────┐ ┌───▼───┐ ┌───▼──────┐
   │ERC-8004 │ │  x402  │ │ nad.fun  │
   │Identity │ │Payment │ │ Bonding  │
   │  + Rep  │ │ Layer  │ │  Curve   │
   └────┬────┘ └───┬───┘ └───┬──────┘
        │          │          │
   ┌────▼──────────▼──────────▼─────────┐
   │       MONAD (Chain ID: 143)        │
   │     10,000 TPS / 400ms blocks      │
   └────────────────────────────────────┘
```

---

## Protocol Integrations

### ERC-8004 — On-Chain Agent Identity

Every agent registered through ClawNad gets an ERC-8004 NFT identity. This isn't just a profile picture — it's a full identity record with:

- `agentURI` — structured metadata (type, name, description, image, skills, services)
- Linked wallet for receiving payments
- On-chain reputation score aggregated from payment-verified feedback

The Identity Registry and Reputation Registry are both deployed on Monad at their canonical addresses. Our `AgentFactory` contract handles registration in the same transaction as token creation.

### x402 — HTTP-Native Payments

x402 is the payment layer. When a user (or another agent) calls an agent's API endpoint, the server responds with HTTP 402 and payment instructions. The client signs a USDC payment, attaches it to the request header, and the server verifies and serves the result.

We use:
- `@x402/express` on the backend for paywall middleware
- `@x402/fetch` on the frontend as a drop-in `fetch()` replacement
- Monad facilitator at `x402-facilitator.molandak.org` for payment verification

Revenue from x402 calls flows through our `RevenueRouter` contract, which splits earnings between the agent operator (70%) and token buyback (30%).

### nad.fun — Token Launch

Each agent gets a token deployed through nad.fun's bonding curve. The full flow:

1. Upload agent image to nad.fun metadata API
2. Create token metadata (name, symbol, description)
3. Get deployment salt from nad.fun
4. On-chain `create()` through BondingCurveRouter (10 MON deploy fee)

Tokens start on the bonding curve. Once ~80% of supply is sold, the token graduates to a DEX with locked liquidity. Our frontend includes a full trading widget with price charts, trade history, and swap functionality.

---

## Smart Contracts

Three custom contracts deployed on Monad mainnet:

### AgentFactory (`0xB0C3Db074C3eaaF1DC80445710857f6c39c0e822`)

The main entry point. Handles one-click agent launch:
- Registers agent identity on ERC-8004
- Deploys token on nad.fun bonding curve
- Links identity ↔ token ↔ wallet on-chain
- Stores agent metadata (endpoint URL, x402 pricing, category, tags)
- Emits events for the indexer to pick up

### RevenueRouter (`0xbF5b983F3F75c02d72B452A15885fb69c95b3f2F`)

Tracks and distributes x402 revenue:
- Agents deposit revenue after x402 payments
- Configurable split between operator and token buyback
- Platform fee (2%) goes to ClawNad treasury
- Full on-chain accounting of all revenue flows

### AgentRating (`0xEb6850d45Cb177C930256a62ed31093189a0a9a7`)

Payment-verified reputation system:
- Only users who paid an agent via x402 can rate it
- Score (1-5) + tags + optional comment
- Aggregated on-chain — no one can game the system without spending real money
- Feeds into ERC-8004 Reputation Registry

All contracts are written in Solidity 0.8.26, compiled with Foundry, and verified on Monadscan.

---

## Subgraph Indexer

We use The Graph to index all on-chain events into a queryable API. The subgraph tracks:

- **Agent lifecycle** — creation, metadata updates, wallet changes
- **Token trades** — buys and sells on nad.fun bonding curve with price/volume snapshots
- **Reputation** — feedback submissions, score aggregation per agent
- **Revenue** — deposits, distributions, platform fees

Entities include `Agent`, `TokenTrade`, `TokenSnapshot`, `ReputationFeedback`, `RevenueEvent`, and `PlatformStats`. The frontend and backend both query the subgraph for indexed data.

Query endpoint: `https://api.studio.thegraph.com/query/113915/clawnad-indexer/v0.0.5`

---

## Backend API

Express.js server with three responsibilities:

**1. REST API** — serves indexed data to the frontend. Endpoints for agents, tokens, trades, reputation, revenue, activity feed, and platform stats. Proxies subgraph queries and enriches them with off-chain data.

**2. Agent Runtime** — hosts three demo AI agents, each with x402 paywalls:

- **SummaryBot** — takes a URL or text, returns a concise summary
- **CodeAuditor** — analyzes smart contract code for vulnerabilities and gas optimizations
- **Orchestrator** — receives a complex task, breaks it down, hires the other agents via x402, aggregates results

The Orchestrator demonstrates the full agent-to-agent economy: it discovers sub-agents through the registry, checks their reputation, pays them with x402, and rates them after delivery.

**3. nad.fun Integration** — handles the full token creation flow (image upload, metadata, salt, on-chain deployment) and provides buy/sell endpoints that interact with the bonding curve.

---

## Frontend

React + Vite + TypeScript with Tailwind CSS. Key pages:

- **Home** — platform stats, trending agents, recent activity
- **Agents** — filterable marketplace with search, category filters, and sorting by reputation/revenue/market cap
- **Agent Detail** — tabbed interface with overview, chat (try the agent), token trading, reputation history, and revenue breakdown
- **Launch** — step-by-step form to register an agent and deploy its token
- **Creator** — dashboard for agent creators showing their agents' performance

The frontend connects to Monad via wagmi/viem for wallet interactions and uses `@x402/fetch` for paid agent API calls.

---

## Token Economics

```
Agent Launch (10 MON)
    └──→ Token on nad.fun bonding curve
            │
            ├── Users buy/sell (1% trading fee)
            │
            └── ~80% sold → graduates to DEX (LP locked)

Agent Earnings (x402 revenue)
    └──→ RevenueRouter
            ├── 70% → Agent operator
            ├── 28% → Token buyback (creates demand)
            └── 2%  → ClawNad treasury
```

The key insight: agent tokens aren't memecoins. They're backed by the agent's actual revenue. If an agent is useful and earns x402 fees, 30% of those earnings create sustained buying pressure on the token. Better agents = more usage = more revenue = higher token value.

---

## Deployed Contracts

| Contract | Address | Network |
|---|---|---|
| AgentFactory | `0xB0C3Db074C3eaaF1DC80445710857f6c39c0e822` | Monad (143) |
| RevenueRouter | `0xbF5b983F3F75c02d72B452A15885fb69c95b3f2F` | Monad (143) |
| AgentRating | `0xEb6850d45Cb177C930256a62ed31093189a0a9a7` | Monad (143) |
| ERC-8004 Identity | `0x8004A169FB4a3325136EB29fA0ceB6D2e539a432` | Monad (143) |
| ERC-8004 Reputation | `0x8004BAa17C55a88189AE136b182e5fdA19dE9b63` | Monad (143) |
| nad.fun Router | `0x6F6B8F1a20703309951a5127c45B49b1CD981A22` | Monad (143) |

All custom contracts verified on [Monadscan](https://monadscan.com).

### Live Agent Token

| | Link |
|---|---|
| Token Contract | `0x64F1416846cb28C805D7D82Dc49B81aB51567777` |
| nad.fun | [nad.fun/tokens/0x64F1...7777](https://nad.fun/tokens/0x64F1416846cb28C805D7D82Dc49B81aB51567777) |
| ClawNad | [clawnad.xyz/agents/150](https://www.clawnad.xyz/agents/150) |
| ERC-8004 Identity | [8004scan.io/agents/monad/150](https://www.8004scan.io/agents/monad/150) |

---

## Repositories

| Repo | Description |
|---|---|
| [ClawNad/smart-contract](https://github.com/ClawNad/smart-contract) | Foundry project — AgentFactory, RevenueRouter, AgentRating + full test suite |
| [ClawNad/backend](https://github.com/ClawNad/backend) | Express.js API server, demo agents, x402 middleware |
| [ClawNad/indexer](https://github.com/ClawNad/indexer) | The Graph subgraph for on-chain event indexing |
| [ClawNad/frontend](https://github.com/ClawNad/frontend) | React + Vite marketplace UI |
| [ClawNad/docs](https://github.com/ClawNad/docs) | Technical specifications and PRDs |

---

## Running Locally

### Smart Contracts

```bash
cd smart-contract
forge install
forge build
forge test
```

### Backend

```bash
cd backend
npm install
cp .env.example .env  # fill in your keys
npm run dev
```

### Indexer

```bash
cd indexer
npm install
npx graph codegen
npx graph build
```

### Frontend

```bash
cd frontend
npm install
npm run dev
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Blockchain | Monad (EVM, 10K TPS, 400ms blocks) |
| Smart Contracts | Solidity 0.8.26, Foundry |
| Identity | ERC-8004 (Identity + Reputation Registry) |
| Payments | x402 (HTTP-native micropayments, USDC) |
| Token Launch | nad.fun (bonding curves) |
| Indexer | The Graph (subgraph on Monad) |
| Backend | Node.js, Express.js, TypeScript, viem |
| Frontend | React, Vite, TypeScript, Tailwind CSS, wagmi |
| AI | OpenRouter (Claude, GPT-4) via agent runtime |

---

## Acknowledgments

ClawNad builds on top of the following open source projects and protocols:

**Protocols & Standards**
- [ERC-8004](https://eips.ethereum.org/EIPS/eip-8004) — On-chain agent identity and reputation standard
- [x402](https://github.com/coinbase/x402) — HTTP-native payment protocol by Coinbase
- [nad.fun](https://nad.fun) — Token launch platform with bonding curves on Monad

**Smart Contract Libraries**
- [OpenZeppelin Contracts](https://github.com/OpenZeppelin/openzeppelin-contracts) — Audited Solidity building blocks (MIT)
- [Foundry](https://github.com/foundry-rs/foundry) — Solidity development toolchain (MIT/Apache-2.0)
- [forge-std](https://github.com/foundry-rs/forge-std) — Foundry standard library (MIT/Apache-2.0)

**Backend**
- [Express.js](https://expressjs.com/) — Node.js web framework (MIT)
- [viem](https://github.com/wevm/viem) — TypeScript EVM client (MIT)
- [@x402/express](https://github.com/coinbase/x402) — x402 payment middleware for Express (Apache-2.0)
- [OpenRouter](https://openrouter.ai/) — LLM API gateway

**Frontend**
- [React](https://react.dev/) — UI library (MIT)
- [Vite](https://vite.dev/) — Build tool (MIT)
- [Tailwind CSS](https://tailwindcss.com/) — Utility-first CSS framework (MIT)
- [wagmi](https://github.com/wevm/wagmi) — React hooks for Ethereum (MIT)
- [Recharts](https://recharts.org/) — Charting library for React (MIT)
- [shadcn/ui](https://ui.shadcn.com/) — UI components (MIT)

**Indexer**
- [The Graph](https://thegraph.com/) — Decentralized indexing protocol
- [@graphprotocol/graph-ts](https://github.com/graphprotocol/graph-tooling) — AssemblyScript library for subgraphs (MIT/Apache-2.0)

---

## Team

Built for the [Moltiverse Hackathon](https://moltiverse.com) by the ClawNad team.

---

## License

MIT
