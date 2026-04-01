# System Architecture

**Blockscout v10.0.6** — High-level architecture overview and data flow

---

## High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                          External Services                       │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────────┐  │
│  │ JSON-RPC    │  │ Microservices│  │ Market Data Sources    │  │
│  │ Endpoint    │  │ (sc-verifier,│  │ (CoinGecko, CMC, etc)  │  │
│  │ (Geth, etc) │  │  sig-provider│  │                        │  │
│  └─────┬───────┘  └──────┬───────┘  └──────────┬─────────────┘  │
│        │                 │                     │                 │
└────────┼─────────────────┼─────────────────────┼─────────────────┘
         │                 │                     │
    ┌────▼──────────────────▼─────────────────────▼───────────┐
    │         Blockscout Application Layer (OTP Apps)          │
    │                                                            │
    │  ┌───────────────────────────────────────────────────┐   │
    │  │          block_scout_web (Phoenix)                │   │
    │  │  ┌──────────────────────────────────────────────┐ │   │
    │  │  │  HTTP APIs: Etherscan-compat, v1, v2         │ │   │
    │  │  │  GraphQL + WebSocket Real-time Subscriptions │ │   │
    │  │  │  Rate Limiting (Hammer + Redis)              │ │   │
    │  │  │  Auth: SIWE, OAuth, API Keys                 │ │   │
    │  │  └──────────────────────────────────────────────┘ │   │
    │  │  Frontend: Next.js, Web3.js, Redux                 │   │
    │  └───────────┬────────────────────────────────────────┘   │
    │              │ HTTP/WebSocket                              │
    │              │                                              │
    │  ┌───────────┼────────────────────────────────────────┐   │
    │  │           ▼  explorer (Data Layer)                 │   │
    │  │  ┌──────────────────────────────────────────────┐  │   │
    │  │  │  Ecto Schemas (70+ models)                   │  │   │
    │  │  │  - Core: Address, Block, Tx, Log, Token      │  │   │
    │  │  │  - L2-specific: Arbitrum, Optimism, etc      │  │   │
    │  │  │  Smart Contract Verification & Proxy         │  │   │
    │  │  │  Detection (6 patterns)                       │  │   │
    │  │  │  Market Data Fetchers (7 sources)            │  │   │
    │  │  │  Account: Users, Watchlists, Tags            │  │   │
    │  │  │  Caching: GenServers (25+), Redis, Con_cache │  │   │
    │  │  │  Background Migrants (18 workers)            │  │   │
    │  │  └────────────┬────────────────────────────────┘  │   │
    │  │               │ Ecto.Multi atomic imports         │   │
    │  └───────────────┼────────────────────────────────────┘   │
    │                  │                                         │
    │  ┌───────────────┼────────────────────────────────────┐   │
    │  │               ▼  indexer (Pipeline)               │   │
    │  │  ┌──────────────────────────────────────────────┐ │   │
    │  │  │  Block Fetcher: Realtime (WS) + Catchup     │ │   │
    │  │  │  Sub-fetchers: Internal Tx, Token Balance,  │ │   │
    │  │  │  Contract Code, Block Reward, Withdrawal    │ │   │
    │  │  │  Transformers: RPC → DB-compatible params   │ │   │
    │  │  │  Memory Monitor: Pause under pressure       │ │   │
    │  │  │  Reorg Handling: Gap detection, orphaning   │ │   │
    │  │  └──────────────────────────────────────────────┘ │   │
    │  └───────────────┬────────────────────────────────────┘   │
    │                  │ Bulk upsert (Ecto.Multi)               │
    │  ┌───────────────┼────────────────────────────────────┐   │
    │  │               ▼  ethereum_jsonrpc                 │   │
    │  │  ┌──────────────────────────────────────────────┐ │   │
    │  │  │  Transport: HTTP, WebSocket, IPC            │ │   │
    │  │  │  Variant: Geth, Nethermind, Erigon, etc     │ │   │
    │  │  │  Request Coordinator: Timeout backoff       │ │   │
    │  │  │  Endpoint Observer: Multi-URL fallback      │ │   │
    │  │  │  Rate Limiter: RollingWindow ETS            │ │   │
    │  │  └──────────────────────────────────────────────┘ │   │
    │  └───────────────────────────────────────────────────┘   │
    │                                                            │
    │  ┌───────────────────────────────────────────────────┐   │
    │  │  utils (Shared), nft_media_handler (Pipeline)   │   │
    │  └───────────────────────────────────────────────────┘   │
    └────┬──────────────────────────────────────────────────────┘
         │
    ┌────▼──────────────────────────────────────┐
    │     Data Persistence Layer                 │
    │  ┌──────────────────────────────────────┐  │
    │  │  PostgreSQL                          │  │
    │  │  - Primary: Read/Write (Explorer.Repo)│ │
    │  │  - Replica1: Read-only (heavy queries) │ │
    │  │  - 320 migrations (2018-2026)         │  │
    │  │  - Real-time events via NOTIFY       │  │
    │  └──────────────────────────────────────┘  │
    │  ┌──────────────────────────────────────┐  │
    │  │  Redis                                │  │
    │  │  - Rate limit counters                │  │
    │  │  - Session cache                      │  │
    │  │  - Distributed lock/cache             │  │
    │  └──────────────────────────────────────┘  │
    └──────────────────────────────────────────────┘
```

---

## OTP Supervision Trees

### Explorer Application (Core Data)

```
Explorer.Supervisor (one_for_one)
├── Explorer.Repo                       [Ecto primary DB pool]
├── Explorer.Repo.Replica1              [Ecto read-only replica]
├── Explorer.Vault                      [Encryption key server]
├── Con.Cache (name: market_cache)      [In-memory OHLC]
├── Redix (name: redix)                 [Redis connection]
├── HistoryTaskSupervisor               [Historical data fetcher]
├── MarketTaskSupervisor                [Market data fetcher]
├── SmartContract.SolcDownloader        [Solc compiler cache]
├── SmartContract.VyperDownloader       [Vyper compiler cache]
├── Chain.Health.Monitor                [Heartbeat + sync status]
├── Registry (name: ChainEvents)        [PubSub for real-time]
├── Chain.Events.Listener               [PG NOTIFY listener]
├── Market.Fetcher.Coin                 [Periodic coin pricing]
├── Market.Fetcher.Token                [Periodic token pricing]
├── Market.Fetcher.History              [Periodic OHLC history]
├── Validator.MetadataProcessor         [Validator info sync]
├── Tags.AddressTag.Cataloger           [Address tag catalog]
├── SmartContract.CertifiedCatalog      [Certified contracts]
├── TokenInstanceOwnerMigration.Super   [NFT ownership backfill]
├── Utility.ReplicaAccessibilityMgr     [Replica health monitor]
├── Utility.VersionConstantsUpdater     [Version refresh]
└── ... (18 background migrators)       [Data consistency]
```

### Indexer Application (Pipeline)

```
Indexer.Supervisor (one_for_one, chain_type: :ethereum)
├── Indexer.Block.Realtime.Fetcher      [Latest blocks WS/HTTP]
├── Indexer.Block.Catchup.Fetcher       [Missing ranges]
├── Indexer.Block.Catchup.BoundInterval [Adaptive scheduler]
├── Indexer.Fetcher.InternalTransaction [Traces per tx]
├── Indexer.Fetcher.CoinBalance         [Address balances]
├── Indexer.Fetcher.TokenBalance        [Token holdings]
├── Indexer.Fetcher.TokenInstance       [NFT metadata]
├── Indexer.Fetcher.ContractCode        [Bytecode fetch]
├── Indexer.Fetcher.BlockReward         [Miner rewards]
├── Indexer.Fetcher.UncleBlock          [Uncle data]
├── Indexer.Fetcher.PendingTransaction  [Mempool]
├── Indexer.Fetcher.Withdrawal          [EIP-4895]
├── Indexer.Fetcher.SignedAuth          [EIP-7702]
├── Indexer.Fetcher.OnDemand.*          [Cache warming]
├── Indexer.Fetcher.L2.*                [Chain-specific]
├── Indexer.Migrator.*                  [Background fixers]
└── Indexer.Memory                      [Memory pressure monitor]
```

### block_scout_web Application (Web Layer)

```
BlockScoutWeb.Supervisor (one_for_one)
├── BlockScoutWeb.Endpoint              [HTTP + WebSocket listener]
├── BlockScoutWeb.Telemetry             [Prometheus/Datadog]
└── ... (optional: cache connectors)
```

---

## Database Schema Overview

### Core Blockchain Tables

| Table | Rows | PK | Key Indexes |
|-------|------|----|----|
| `addresses` | ~1M+ | hash (20 bytes) | is_contract, nonce |
| `blocks` | ~20M+ | hash | number, miner_hash, consensus |
| `transactions` | ~1B+ | hash | from_address_hash, to_address_hash, block_hash, status |
| `internal_transactions` | ~10B+ | (tx_hash, idx) | from_address_hash, to_address_hash, type |
| `logs` | ~100B+ | (tx_hash, log_idx) | address_hash, topic0, topic1, topic2, topic3 |
| `tokens` | ~100k | address_hash | type (ERC-20/721/1155) |
| `token_transfers` | ~10B+ | (tx_hash, log_idx) | from_address_hash, to_address_hash, token_address_hash |
| `token_instances` | ~1B+ | (token_address, token_id) | owner_hash, metadata_updated_at |
| `smart_contracts` | ~1M | address_hash | is_verified, compiler_version |
| `smart_contract_additional_source` | ~100k | (address_hash, file_name) | contract verification support |
| `address_coin_balances` | ~100M+ | (address_hash, block_number) | address_hash, block_number |
| `current_token_balances` | ~1B+ | (address_hash, token_hash, token_id) | address_hash, token_hash |
| `withdrawals` | ~10M | (tx_hash, idx) | address_hash, block_number |
| `user_operations` | ~100M | tx_hash | sender, entry_point |
| `signed_authorizations` | ~100M | tx_hash | authority, address |

### L2-Specific Tables

| L2 | Tables | Purpose |
|----|--------|---------|
| Arbitrum | `arbitrum_batches`, `arbitrum_da_records` | Batch submissions, DA metrics |
| Optimism | `optimism_output_roots`, `optimism_interop_messages` | Output finality, interop |
| zkSync | `zksync_batches`, `zksync_l1_transactions` | Batch data, L1 anchor |
| Polygon zkEVM | `polygon_zkevm_batches` | Sequence submissions |
| Scroll | `scroll_batches` | Batch finality |
| Celo | `celo_epochs`, `celo_election_rewards` | Epoch rewards, elections |
| Filecoin | `filecoin_pending_operations` | Actor pending ops |
| Shibarium | `shibarium_bridges` | Bridge tracking |

### Cache & Performance Tables

| Table | TTL | Purpose |
|-------|-----|---------|
| `market_history` | Indefinite | OHLC data (1h, 1d candles) |
| `missing_block_ranges` | Until filled | Gap ranges for catchup |
| `pending_block_operations` | Until processed | Blocks awaiting sub-fetcher tasks |
| `contract_methods` | Indefinite | Decoded ABI methods cache |
| Address, Block, Token counters | GenServer in-memory | Real-time sync stats |

---

## Data Flow: Block Indexing

### Realtime Indexing (Latest Block)

```
1. Indexer.Block.Realtime.Fetcher subscribed to eth_subscribe newHeads
2. Receives new block header
3. Calls Block.Fetcher.fetch_and_import_range(block_num, block_num)
4. Fetches: block data, transactions, receipts, uncle blocks
5. Transforms via Indexer.Transform.Blocks
6. Ecto.Multi upsert: blocks, transactions, logs
7. Repo.transaction() executes atomically
8. On success: Dispatcher spawns async sub-fetchers
   ├── InternalTransaction.Fetcher (traces)
   ├── CoinBalance.Fetcher (address balances)
   ├── TokenBalance.Fetcher (token balances)
   ├── TokenInstance.Fetcher (NFT metadata)
   ├── ContractCode.Fetcher (bytecode)
   ├── BlockReward.Fetcher (miner rewards)
   ├── Withdrawal.Fetcher (validator exits)
   └── SignedAuthorization.Fetcher (EIP-7702)
9. Explorer.Chain.Events.Publisher publishes :block_received
10. WebSocket subscribers receive real-time update via Phoenix Channels
```

### Catchup Indexing (Missing Ranges)

```
1. MissingBlockRange table identifies gaps (e.g., blocks 1000000-1001000 missing)
2. BoundIntervalSupervisor scheduler triggers catchup
3. MassiveBlocksFetcher isolates blocks > 1MB (process separately)
4. Block.Catchup.Fetcher batches: [1000000-1000099, 1000100-1000199, ...]
5. Parallel fetcher tasks (concurrency configurable)
6. Each batch: fetch_and_import_range(start, end)
7. Same atomic upsert as realtime (Ecto.Multi)
8. Remove row from MissingBlockRange on success
9. Reorg detection: If gap detected in latest blocks, realtime fetcher fills it
```

### Reorg Handling

```
1. Realtime fetcher receives block N
2. Checks previous_number field vs stored block N-1
3. If gap detected: fetch intermediate blocks
4. If block N-1 exists but has different hash:
   - Mark orphaned block as non-consensus
   - Mark all descendant blocks as non-consensus
   - Republish PG NOTIFY for event listeners
   - UI subscribers receive reorg event
5. Continue indexing from new canonical tip
```

---

## API Layers

### 1. Etherscan-Compatible RPC

```
GET /api?module=account&action=balance&address=0x...
GET /api?module=account&action=txlist&address=0x...
GET /api?module=proxy&action=eth_blockNumber
POST /api? (JSON-RPC wrapped)
```

**Coverage:** 80+ methods (blocks, transactions, accounts, contracts, tokens, logs, stats)

### 2. v1 Semi-REST

```
GET /api/v1/addresses/:hash
GET /api/v1/addresses/:hash/transactions
GET /api/v1/transactions/:hash
GET /api/v1/blocks/:block_number_or_hash
```

### 3. v2 Modern REST (OpenAPI)

```
GET /api/v2/addresses/:hash
GET /api/v2/addresses/:hash/coin-balance-history
GET /api/v2/addresses/:hash/token-balances
GET /api/v2/transactions/:hash
GET /api/v2/blocks
GET /api/v2/smart-contracts/:address/methods-read
GET /api/v2/tokens/:address
GET /api/v2/search?q=0x...
```

**Features:**
- Pagination (cursor-based)
- Filtering (status, type, etc.)
- Sorting
- OpenAPI docs auto-generated

### 4. GraphQL (Absinthe)

```graphql
query {
  addresses(first: 10, after: "cursor...") {
    edges {
      node {
        hash
        balance
        transactions(first: 5) {
          edges { node { hash status } }
        }
      }
    }
  }
}
```

**Features:**
- Relay cursor pagination
- Dataloader for N+1 prevention
- Subscription support (via WebSocket)

### 5. WebSocket Real-time

```javascript
ws.send({
  event: "subscribe:address_balance_updates",
  payload: { address: "0x..." }
});

// Receives:
{ event: "address_balance_update", payload: { balance: "1000000000000000000" } }
```

**Subscriptions:** blocks, transactions, token_transfers, address_balance, exchange_rates

---

## Caching Strategy

### 1. In-Memory (Con_cache)

**Use:** Frequently accessed, small data (< 10MB)
- Market history (OHLC)
- Gas price oracle
- Validator metadata
- Address tag catalog

**TTL:** 1-3600 seconds

### 2. Redis (Distributed)

**Use:** Session, rate limiting, distributed cache
- API rate limit counters (Hammer backend)
- Session tokens
- Cross-instance cache coordination

**Data:** Simple scalars, JSON

### 3. GenServer Counters (Persistent)

**Use:** Live sync statistics (updated on every block)
- Block count, latest block number
- Transaction count, avg fee
- Address count, contract count
- Token holders count

**Location:** In-memory, persisted to DB on interval

### 4. Database Query Caching (Implicit)

**Use:** Ecto query results via preload
- Smart contract ABIs
- Token metadata
- Address labels

**TTL:** Until invalidated by new data

---

## Real-time Event System

### Architecture

```
Block Indexer → Ecto Insert → PostgreSQL NOTIFY
                                    ↓
                        Explorer.Chain.Events.Listener (GenServer)
                                    ↓
                        Explorer.Chain.Events.Publisher (broadcast)
                                    ↓
                        Registry.ChainEvents (PubSub)
                                    ↓
                        ┌──────────────────────────┐
                        │  Phoenix Channel Handler  │
                        │  (WebSocket subscriber)   │
                        └──────────────────────────┘
                                    ↓
                        Client (JavaScript)
```

### Event Types

- `:block_received` — New block indexed
- `:transaction_received` — New transaction indexed
- `:address_balance_updated` — Address balance changed
- `:token_transfer` — ERC-20/721/1155 transfer
- `:exchange_rate_updated` — Coin price changed
- `:reorg_detected` — Reorg occurred

---

## L2 Chain Support Pattern

### Configuration-Driven

```elixir
# config/runtime.exs
config :explorer, :chain_type, :optimism

# In application.ex, conditionally start L2-specific fetchers
if Application.get_env(:explorer, :chain_type) == :optimism do
  children = [
    {Indexer.Fetcher.Optimism.OutputRoot, []},
    {Indexer.Fetcher.Optimism.InteropMessage, []},
  ]
end
```

### Schema Extension

```elixir
# Core schema
defmodule Explorer.Chain.Block do
  schema "blocks" do
    field :hash, :string
    field :number, :integer
    # ...
    has_one :optimism_output_root, Explorer.Chain.Optimism.OutputRoot
  end
end

# L2-specific schema
defmodule Explorer.Chain.Optimism.OutputRoot do
  schema "optimism_output_roots" do
    field :output_root, :string
    field :l2_output_index, :integer
    belongs_to :block, Explorer.Chain.Block
  end
end
```

### Fetcher Pattern

```elixir
defmodule Indexer.Fetcher.Optimism.OutputRoot do
  use Indexer.Fetcher

  # Enabled only if chain_type == :optimism
  @enabled Application.get_env(:indexer, __MODULE__)[:enabled]

  def fetch_and_import_range(start_number, end_number) do
    # Fetch from Optimism L1 contract
    # Import optimism_output_roots table
  end
end
```

---

## Monitoring & Observability

### Prometheus Metrics

```
# Block indexing
indexer_blocks_fetched_total
indexer_blocks_imported_total
indexer_import_duration_seconds (histogram)

# Database
explorer_repo_query_duration_seconds
explorer_repo_connection_pool_utilization

# API
block_scout_web_request_duration_seconds
block_scout_web_requests_total

# Chain health
explorer_chain_blocks_behind (gauges for each instance)
explorer_syncing (0 = synced, 1 = syncing)
```

### Datadog Tracing (Spandex)

- Traces: `indexer.fetch_block`, `explorer_repo.query`, `web.http.request`
- Resource tags: block_number, tx_hash, address_hash
- Sampling: 10% of requests (configurable)

---

## Security & Authentication

### API Rate Limiting (Hammer + Redis)

```
Tier 1 (Free):  50 requests/sec
Tier 2 (Paid):  500 requests/sec
Tier 3 (Pro):   5000 requests/sec

Keyed by: API key or IP address
Backend: Redis (distributed)
Retry-After: Returned in 429 response
```

### Authentication Modes

1. **SIWE (Sign-In with Ethereum)** — Web3 wallet sign message
2. **OAuth (Auth0/Keycloak)** — Traditional OAuth flow
3. **API Keys** — Static keys for programmatic access
4. **CSRF Protection** — Phoenix default token rotation

### Data Encryption

- **Vault (cloak_ecto)** — Encrypts sensitive fields
  - API keys
  - User emails
  - Custom ABIs
- **Encryption:** AES-256-GCM
- **Key:** Environment variable (rotate on deployment)

