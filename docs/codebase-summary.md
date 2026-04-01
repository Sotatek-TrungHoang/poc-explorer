# Codebase Summary

**Blockscout v10.0.6** — Elixir OTP umbrella app with 6 focused applications

---

## Umbrella Architecture

Blockscout uses Mix's **umbrella project** pattern: 6 independent OTP apps under `apps/` sharing root `mix.exs`.

```
blockscout/                    (Umbrella root)
├── mix.exs                    (Shared deps, releases config)
├── config/                    (Shared config)
├── apps/
│   ├── block_scout_web/       (901 files) Phoenix web layer
│   ├── explorer/              (1235 files) Core data layer
│   ├── indexer/               (319 files) Blockchain pipeline
│   ├── ethereum_jsonrpc/      (139 files) JSON-RPC client
│   ├── utils/                 (11 files) Shared utilities
│   └── nft_media_handler/     (8 files) NFT media processing
├── docker/                    (Dockerfile templates)
├── docker-compose/            (Compose stacks: local, staging, prod)
└── rel/                       (Release config)
```

---

## App Breakdown

### 1. block_scout_web (Web Layer, 901 files)

**Purpose:** Phoenix 1.6 web server; HTTP APIs (Etherscan-compat, v1, v2, GraphQL), WebSocket subscriptions, Web UI rendering.

**Key Dirs:**
- `lib/block_scout_web/`
  - `controllers/` — REST endpoints for addresses, txs, blocks, tokens, smart contracts, accounts
  - `channels/` — Phoenix Channels for real-time updates (blocks, txs, token balances, rates)
  - `views/` — JSON renderers for each controller
  - `plugs/` — Middleware: rate limiting, auth, error handling, chain context
  - `api/` — API versioning (etherscan, v1, v2, graphql)
  - `graphql/` — Absinthe schema, types, resolvers, dataloaders
  - `router.ex` — Route definitions

**APIs:**
- **Etherscan-compatible:** `GET /api?module=...&action=...` (blocks, transactions, accounts, contracts, tokens, stats, logs)
- **v1 semi-REST:** `GET /api/v1/addresses/:hash` (address details), `/api/v1/transactions/:hash`
- **v2 modern REST:** `GET /api/v2/addresses/:hash` with OpenAPI specs
- **GraphQL:** `POST /graphql` with Relay cursor pagination, batched queries
- **WebSocket:** `wss://` for real-time (blocks, txs, address balance updates, token transfers, exchange rates)

**Features:**
- Redis-backed rate limiting (Hammer) with API-key tiers
- Auth0 OAuth + SIWE + API key auth
- CSRF protection via Phoenix
- Datadog tracing (Spandex)
- Prometheus metrics

**Frontend Integration:**
- Webpack 5 bundler
- jQuery + Redux for state
- Bootstrap 4 UI
- Web3.js for Ethereum calls
- Chart.js for analytics
- Phoenix LiveView for dynamic pages

---

### 2. explorer (Data Layer, 1235 files)

**Purpose:** Core Ecto schemas, chain queries, caching, background workers, market data, smart contract verification, account management.

**Key Dirs:**
- `lib/explorer/`
  - `chain/` — 60+ Ecto schemas (addresses, blocks, txs, internal_txs, logs, tokens, contracts, withdrawals, user_operations, signed_authorizations)
  - `chain/import/` — Bulk upsert runners via `Ecto.Multi` (blocks, transactions, internal_txs, logs, token_transfers, token_instances, etc.)
  - `chain/search.ex` — Cross-entity search logic
  - `smart_contract/` — Verification (Solidity, Vyper, Stylus), proxy detection, ABI parsing, bytecode matching
  - `market/` — Price fetchers (7 sources), historical OHLC, coin/token pricing
  - `account/` — User accounts, watchlists, custom ABIs, notifications
  - `arbitrum/`, `optimism/`, `polygon_zkevm/`, etc. — L2-specific schemas & logic
  - `graphql/` — GraphQL context, dataloaders
  - `migrator/` — Background migration workers (18 GenServers for data backfill)
  - `repo/` — Ecto Repo + Replica1 + Prometheus instrumentation

**Schemas (70+ total):**

| Category | Count | Examples |
|----------|-------|----------|
| Core blockchain | 15 | Address, Block, Transaction, InternalTransaction, Log, Token, TokenTransfer, SmartContract, Withdrawal, UserOperation, SignedAuthorization |
| Token-related | 8 | Token, TokenInstance, CurrentTokenBalance, TokenBalance, TokenFiatValue, ContractMethod |
| L2-specific | 20+ | Arbitrum (batch, DA), Optimism (output root), zkSync (batch), Scroll, Polygon zkEVM, Celo, Filecoin, Shibarium, etc. |
| Market & cache | 5 | MarketHistory, GasPriceOracle, CacheBlock, CacheAddress, CacheToken |
| Account | 7 | User, Watchlist, CustomABI, AddressTag, TransactionTag, Notification |

**Data Access:**
- `Explorer.Repo` — Primary read/write PostgreSQL connection
- `Explorer.Repo.Replica1` — Read-only replica for heavy queries
- `ReplicaAccessibilityManager` — Dynamic failover if replica unhealthy
- `Explorer.Chain` — Massive facade module (~500+ functions) for all chain queries
- Import runners — Bulk upsert with `Ecto.Multi` transactions

**Caching:**
- **Con_cache:** In-memory OHLC history, gas prices
- **Redis via Redix:** Session, rate limit counters, distributed cache
- **GenServer counters:** 25+ persistent counters (block/tx/address counts, gas metrics, token holders)

**Workers & Migrations:**
- `Market.Fetcher.*` — Periodic coin/token/history fetchers
- `Validator.MetadataProcessor` — Validator info processor
- `Tags.AddressTag.Cataloger` — Address tag sync
- 18 background migrators — Data consistency backfills

---

### 3. indexer (Blockchain Pipeline, 319 files)

**Purpose:** Fetches blockchain data from JSON-RPC, indexes to Explorer DB. Dual-mode: realtime (WS `eth_subscribe newHeads`) + catchup (fills gaps via `MissingBlockRange` table).

**Key Dirs:**
- `lib/indexer/`
  - `block/` — Block fetching (realtime & catchup), reorg handling
  - `fetcher/` — Per-entity fetchers (internal_txs, coin_balance, token_balance, token_instance, contract_code, block_reward, withdrawal, signed_authorization, L2-specific)
  - `transform/` — RPC response parsers (blocks, addresses, token_transfers, token_instances, transaction_actions, etc.)
  - `supervisor.ex` — Root OTP supervision tree

**Fetcher List (30+):**
- Core: Block (realtime/catchup), InternalTransaction, CoinBalance, TokenBalance, TokenInstance, ContractCode, BlockReward, UncleBlock, Withdrawal, SignedAuthorizationStatus
- Upgrades: PendingTransaction, Token, TokenUpdater, AddressNonceUpdater
- L2-specific: Arbitrum, Optimism, PolygonZkEvm, Scroll, ZkSync, Shibarium, Celo, Filecoin, Zilliqa, Stability, Blackfort
- On-demand: CoinBalance, TokenBalance, ContractCode, TokenInstance (cache warm)
- Multichain: MultichainSearchDb export queues

**Indexing Strategy:**
- **Realtime mode:** WebSocket `eth_subscribe newHeads` subscribed to latest blocks; HTTP polling fallback (≥1s min)
- **Catchup mode:** `MissingBlockRange` table tracks gaps; `BoundIntervalSupervisor` scheduler fetches genesis → latest; `MassiveBlocksFetcher` isolates large blocks
- **Async pattern:** After block import, dispatcher spawns sub-fetchers for internal-txs, balances, code, rewards (each managed by supervised queue)
- **Reorg handling:** Tracks `previous_number`; detects gaps; marks orphan blocks; publishes `chain_event` for UI

**Memory & Concurrency:**
- `BufferedTask` — Batching queue with configurable batch-size × concurrency
- `Memory` monitor — Pauses fetchers under memory pressure
- Configurable parallelism per fetcher (e.g., 10 batches × 10 concurrent = 100 parallel requests)

---

### 4. ethereum_jsonrpc (JSON-RPC Client, 139 files)

**Purpose:** Transport-agnostic Ethereum JSON-RPC client library. Pluggable transport (HTTP, WS, IPC) + variant (Geth, Nethermind, Erigon, Besu, etc.).

**Key Modules:**
- `EthereumJSONRPC` — Public API (`json_rpc/2`, `fetch_blocks_by_range/2`, `fetch_transaction_receipts/2`, etc.)
- `Transport.*` — HTTP (batch, chunked), WebSocket (GenServer, reconnect), IPC
- `Variant.*` — Chain-specific methods (Geth, Nethermind, Erigon, Besu, RSK, Anvil, Filecoin, Zilliqa)
- `RequestCoordinator` — Timeout backoff + rate limiting via `RollingWindow` ETS
- `Utility.EndpointAvailabilityObserver` — Multi-endpoint fallback health tracking

**HTTP Transport:**
- Single request mode: `POST /` with Content-Type application/json
- Batch mode: Array of requests, chunked to avoid oversized payloads
- Chunked batch: Split large batches into configurable chunk sizes

**WebSocket Transport:**
- GenServer-based connection; supports `subscribe/unsubscribe` for `eth_subscribe` (newHeads, logs, etc.)
- Reconnect worker on disconnection
- Per-request correlation via JSON-RPC ID

**Chain Variants:**

| Variant | Client | Trace Method | Beneficiaries |
|---------|--------|--------------|---------------|
| Geth | go-ethereum | `debug_traceTransaction` (custom JS tracer) | `:ignore` |
| Nethermind | Nethermind | `:ignore` (block-level only) | `trace_replayBlockTransactions` |
| Erigon | Erigon | `:ignore` (block-level only) | `trace_replayBlockTransactions` |
| Besu | Hyperledger Besu | `debug_traceTransaction` | `trace_replayBlockTransactions` |
| RSK | Rootstock | `:ignore` | `:ignore` |
| Anvil | Foundry | Geth-like | — |
| Filecoin | Filecoin EVM | Custom actor/CID | — |
| Zilliqa | Zilliqa EVM | Custom quorum cert | — |

**Data Types:**
- Block, Blocks, Transaction, Transactions, Receipt, Receipts, Log, Logs, Uncle, Uncles
- Withdrawal, Withdrawals, Nonce, Nonces, FetchedBalance, FetchedBeneficiary, FetchedCode
- PendingTransaction, Subscription, SignedAuthorization

---

### 5. utils (Shared Utilities, 11 files)

**Purpose:** Shared utilities for env-var parsing, runtime helpers, token instance MIME detection, HTTP clients.

**Modules:**
- `Utils.ConfigHelper` — Environment variable parsing with defaults + validation
- `Utils.RuntimeEnvHelper` — Runtime env accessors (macros for generated functions)
- `Utils.TokenInstanceHelper` — MIME type detection for NFT metadata
- HTTP client wrappers for consistency

---

### 6. nft_media_handler (NFT Media Processing, 8 files)

**Purpose:** Pipeline for fetching, resizing, and uploading NFT media (images, metadata).

**Pipeline:**
1. Fetch metadata URL (on-chain or IPFS)
2. Download media file
3. Resize via Vix/libvips (configurable dimensions)
4. Upload to Cloudflare R2 or S3 (configurable backend)
5. Update DB with CDN URL

**Configuration:** Configurable concurrency, batch sizes, timeout, storage backend.

---

## File Naming Conventions

### Elixir Modules
- **snake_case** for files: `explorer/chain/address.ex`, `indexer/fetcher/block.ex`
- **PascalCase** for module names: `Explorer.Chain.Address`, `Indexer.Fetcher.Block`
- **Behaviour modules** prefix with verb or type: `EthereumJSONRPC.Transport`, `EthereumJSONRPC.Variant`

### Database Migrations
- **Ecto timestamp + verb + entity:** `20180117_create_address.exs`, `20260331_add_fhe_operations.exs`
- Located in each app's `priv/repo/migrations/`

### Tests
- Mirror source structure: `test/explorer/chain/address_test.exs` for `lib/explorer/chain/address.ex`
- Test files end with `_test.exs`

### Configuration
- `config/config.exs` — Shared config (dev, test, prod)
- `config/runtime.exs` — Runtime env-var config (secrets, feature flags)
- `apps/{app}/config/` — App-specific config

---

## Key Patterns

### OTP Supervision Trees
Each app has `lib/{app}/application.ex`:
- Defines `start/2` callback
- Specifies supervised children (Repos, GenServers, TaskSupervisors, Registry)
- Chain type drives which fetchers/workers start

### Ecto Schemas & Changesets
- Schemas in `lib/explorer/chain/` with associations
- Changesets in schemas with validation logic
- Migrations in `priv/repo/migrations/`

### Import Runners (Bulk Upsert Pattern)
Located in `lib/explorer/chain/import/runner/`:
```
Multi
  |> Runner.Blocks.run(blocks_params)
  |> Runner.Transactions.run(tx_params)
  |> Runner.InternalTransactions.run(internal_tx_params)
  |> Repo.transaction()
```
Ensures atomic consistency across related entities.

### Fetcher Macro
Each fetcher uses `use Indexer.Fetcher` macro:
- Auto-generates supervisor
- Wraps fetcher module in GenServer
- Handles backoff, error recovery
- Task-based batch processing

### GraphQL Dataloader
- `Explorer.GraphQL.Dataloader` — Single query dataloader for N+1 prevention
- Resolvers use `:dataloader` batches

### Real-time Events
- `Explorer.Chain.Events.Publisher` — Publishes `chain_event` to Registry subscribers
- Listeners subscribe in web layer (Phoenix Channels)
- Events: `block_received`, `transaction_received`, `token_transfer`, etc.

---

## Configuration

All config via environment variables (runtime.exs):

**Categories:**
- **Endpoint:** Port, URL, CORS, TLS
- **Database:** URL, pool size, replica config
- **Chain:** Type (ethereum, optimism, arbitrum, etc.), chain ID, currency
- **JSON-RPC:** Endpoint URLs, variant, transport, request timeouts
- **API:** Rate limiting tiers, Etherscan API key
- **Feature flags:** GraphQL enabled, Solidity verification, etc.
- **Microservices:** Endpoints for sc-verifier, sig-provider, eth-bytecode-db
- **Market data:** CoinGecko API key, cache duration
- **Branding:** Logo, colors, links

See `config/runtime.exs` for full list (~100+ vars).

---

## Directory Layout Summary

```
blockscout/
├── apps/
│   ├── block_scout_web/lib/block_scout_web/
│   │   ├── controllers/    (API endpoints)
│   │   ├── channels/       (WebSocket real-time)
│   │   ├── views/          (JSON renderers)
│   │   ├── plugs/          (Middleware)
│   │   ├── api/            (API versions: etherscan, v1, v2, graphql)
│   │   ├── graphql/        (Absinthe schema)
│   │   ├── assets/         (Frontend: Webpack, JS, CSS)
│   │   └── router.ex       (Routes)
│   │
│   ├── explorer/lib/explorer/
│   │   ├── chain/          (Core schemas & queries)
│   │   ├── smart_contract/ (Verification, proxy detection)
│   │   ├── market/         (Price fetchers)
│   │   ├── account/        (Users, watchlists)
│   │   ├── graphql/        (GraphQL context)
│   │   ├── migrator/       (Background workers)
│   │   ├── {l2}/           (Arbitrum, Optimism, zkSync, etc.)
│   │   └── repo/           (Ecto Repo)
│   │
│   ├── indexer/lib/indexer/
│   │   ├── block/          (Realtime & catchup)
│   │   ├── fetcher/        (30+ fetchers)
│   │   ├── transform/      (RPC parsers)
│   │   └── supervisor.ex   (OTP tree)
│   │
│   ├── ethereum_jsonrpc/lib/ethereum_jsonrpc/
│   │   ├── http/           (HTTP transport)
│   │   ├── websocket/      (WS transport)
│   │   ├── geth/           (Geth variant)
│   │   └── ...             (Other variants)
│   │
│   ├── utils/lib/utils/    (Shared helpers)
│   └── nft_media_handler/lib/nft_media_handler/ (Media pipeline)
│
├── config/
│   ├── config.exs          (Dev/test/prod)
│   └── runtime.exs         (Runtime env vars)
│
├── docker/                 (Dockerfile templates)
├── docker-compose/         (Compose stacks)
└── rel/                    (Release config)
```

---

## Dependencies Summary

**Top-level Deps (mix.exs):**
- prometheus_ex, absinthe_plug, tesla, mint, ex_doc, number

**Per-App Deps (via mix.lock):**
- **ORM:** ecto, ecto_sql, postgrex
- **Web:** phoenix, phoenix_ecto, cowboy
- **GraphQL:** absinthe, absinthe_plug, absinthe_relay
- **Caching:** con_cache, redix, hammer, hammer_backend_redis
- **Auth:** siwe, ueberauth, ueberauth_auth0, joken
- **Crypto:** ex_keccak, blake2, cbor, ton
- **Monitoring:** prometheus, prometheus_ecto, spandex, spandex_datadog
- **Jobs:** que
- **Clustering:** libcluster

**See:** `mix.lock` (2300+ lines) for complete dependency tree.

