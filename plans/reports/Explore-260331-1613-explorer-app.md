# Scout Report: apps/explorer

**Date:** 2026-03-31  
**Version:** 10.0.6

---

## App Overview

**Purpose:** Core data layer — read/write access to indexed blockchain data stored in PostgreSQL. Provides Ecto schemas, query logic, caching, background workers, market data, and smart contract verification.

**Key deps:**
- `ecto` + `ecto_sql` + `postgrex` — PostgreSQL ORM
- `ethereum_jsonrpc` (in_umbrella) — RPC calls
- `con_cache` + `redix` — in-memory & Redis caching
- `cloak_ecto` — encrypted fields (vault)
- `spandex` + `spandex_datadog` — distributed tracing
- `prometheus_ecto` — query metrics
- `bamboo` — email (account notifications)
- `siwe`, `ueberauth`, `ueberauth_auth0`, `joken` — Web3 + OAuth2 auth
- `libcluster` — distributed Erlang clustering
- `hammer` + `hammer_backend_redis` — rate limiting
- `que` — job queue
- `timex`, `decimal`, `math` — numeric/time utils
- `cbor`, `ex_keccak`, `blake2` — crypto primitives
- `ton`, `mint` — TON blockchain & HTTP client

---

## Directory Structure

```
lib/explorer/
├── chain/            # Core blockchain data — schemas, caches, import runners, fetchers
├── account/          # User accounts, watchlists, custom ABIs, notifications
├── admin/            # Admin recovery
├── application/      # OTP Application + supervisor tree
├── arbitrum/         # Arbitrum L2 specifics
├── etherscan/        # Etherscan-compatible API helpers
├── graphql/          # GraphQL (Absinthe) context/dataloader
├── history/          # Historical data (tx count, market)
├── market/           # Exchange rates — fetchers + sources + cache
├── microservice_interfaces/ # External microservice clients
├── migrator/         # Background data migration GenServers
├── prometheus/       # Metrics instrumentation
├── repo/             # Ecto Repo + replica + prometheus logger
├── smart_contract/   # Verification (Solidity/Vyper/Stylus), proxy detection
├── stats/            # Hot smart contracts stats
├── tags/             # Address tags cataloger
├── third_party_integrations/ # Auth0, Keycloak, Sourcify, SolidityScan, Noves.fi
├── token/            # Token metadata retriever, balance reader
├── token_instance_owner_address_migration/ # NFT ownership backfill
├── utility/          # Rate limiter, replica manager, event notifications, version updater
└── validator/        # Validator metadata processor
```

---

## Key Schemas / Models

### Chain (Ecto schemas)
| File | Schema |
|------|--------|
| `chain/address.ex` | Addresses (hash, contract flag, coin balance, nonce) |
| `chain/address/coin_balance.ex` | Address coin balance per block |
| `chain/address/coin_balance_daily.ex` | Daily aggregated coin balance |
| `chain/address/current_token_balance.ex` | Latest ERC-20/721/1155 balances |
| `chain/address/token_balance.ex` | Historical token balances |
| `chain/address/name.ex` | Address names/labels |
| `chain/block.ex` | Blocks (number, hash, miner, gas, difficulty, consensus) |
| `chain/block/emission_reward.ex` | Block emission rewards |
| `chain/block/reward.ex` | Block/uncle rewards |
| `chain/transaction.ex` | Transactions (hash, status, gas, input, method) |
| `chain/transaction/fork.ex` | Forked (uncle) transaction records |
| `chain/internal_transaction.ex` | Internal transactions (calls, creates, selfdestruct) |
| `chain/log.ex` | Event logs |
| `chain/token.ex` | Token metadata (name, symbol, decimals, type) |
| `chain/token_transfer.ex` | ERC-20/721/1155 transfer events |
| `chain/token/instance.ex` | NFT token instances |
| `chain/token/fiat_value.ex` | Token fiat price |
| `chain/smart_contract.ex` | Verified contracts (ABI, source, bytecode) |
| `chain/smart_contract_additional_source.ex` | Multi-file verification sources |
| `chain/contract_method.ex` | Decoded ABI methods |
| `chain/withdrawal.ex` | EIP-4895 validator withdrawals |
| `chain/user_operation.ex` | ERC-4337 account abstraction user ops |
| `chain/signed_authorization.ex` | EIP-7702 signed authorizations |
| `chain/transaction_action.ex` | Decoded tx actions (DeFi protocol interactions) |
| `chain/mud.ex` | MUD framework entities |
| `chain/pending_block_operation.ex` | Tracks blocks pending processing |
| `chain/pending_transaction_operation.ex` | Tracks txs pending processing |
| `market/market_history.ex` | Historical OHLC market data |

### L2 / Chain-specific schemas
- `chain/arbitrum/` — Arbitrum DA, batch submissions, L1→L2 messages
- `chain/optimism/` — Output roots, interop messages, finalization periods
- `chain/polygon_zkevm/` — ZK proof submissions
- `chain/zksync/` — ZkSync batches
- `chain/scroll/` — Scroll batch data
- `chain/shibarium/` — Shibarium bridge
- `chain/beacon/` — EIP-4788 beacon chain blobs
- `chain/celo/` — Celo epoch rewards, elections
- `chain/filecoin/` — Filecoin pending ops
- `chain/zilliqa/` — Zilliqa specific
- `chain/neon/` — Neon EVM
- `chain/stability/` — Stability validators
- `chain/blackfort/` — Blackfort validators

---

## Data Access Patterns

- **Primary Repo:** `Explorer.Repo` (read/write) — wraps Ecto `Repo`
- **Replica:** `Explorer.Repo.Replica1` — read-only replica for heavy queries
- **`ReplicaAccessibilityManager`** — dynamically routes reads to replica if healthy
- **`Explorer.Chain`** (`chain.ex`) — massive facade module, entry point for all chain queries (blocks, txs, addresses, tokens, logs, contracts)
- **`Explorer.Chain.Search`** (`chain/search.ex`) — cross-entity search
- **Import runners** (`chain/import/runner/`) — bulk upsert via `Ecto.Multi`:
  - `addresses.ex`, `blocks.ex`, `transactions.ex`, `internal_transactions.ex`, `logs.ex`, `token_transfers.ex`, `token_instances.ex`, `signed_authorizations.ex`
  - L2-specific runners: arbitrum, beacon, optimism, polygon_zkevm, scroll, shibarium, stability, celo
- **Paging:** `PagingOptions` pattern used across queries (cursor pagination)
- **Prometheus:** All Ecto queries instrumented via `prometheus_ecto` + `PrometheusLogger`
- **Tracing:** `spandex_ecto` traces queries to Datadog

---

## Business Logic

### Smart Contracts
- **`SmartContract.Reader`** — reads contract state via JSONRPC calls
- **`SmartContract.Writer`** — prepares write txs to contracts
- **`SmartContract.Helper`** — ABI helpers, bytecode matching
- **`SmartContract.EthBytecodeDbInterface`** — lookup via eth-bytecode-db microservice
- **`SmartContract.RustVerifierInterface`** — Rust-based verifier microservice (Sourcify/Hardhat)
- **`SmartContract.SigProviderInterface`** — function/event signature lookup
- **Solidity:** `solidity/` — multi-file compilation, standard JSON
- **Vyper:** `vyper/` — Vyper compiler integration, `VyperDownloader`
- **Stylus:** `stylus/` — Rust/WASM contract verification
- **Proxy detection** (`smart_contract/proxy/`):  
  EIP-1167 (minimal proxy), EIP-1822 (UUPS), EIP-1967 (transparent), EIP-2535 (diamond), EIP-7702, ERC-7760, MasterCopy, clone-with-immutable-args

### Token Standards
- ERC-20 (fungible) — tracked via `TokenTransfer`, `CurrentTokenBalance`
- ERC-721 (NFT) — `TokenInstance`, ownership via `CurrentTokenBalance`
- ERC-1155 (multi-token) — same schemas, distinguished by `token_type` field
- **`Token.MetadataRetriever`** — fetches token URI metadata (IPFS, HTTP)
- **`Token.BalanceReader`** — on-chain balance reads via JSONRPC batch calls

### Market Data
- **`Market.Fetcher.Coin`** — fetches coin price (configurable source)
- **`Market.Fetcher.Token`** — fetches token prices
- **`Market.Fetcher.History`** — fetches historical OHLC
- **Sources:** CoinGecko, CoinMarketCap, CryptoCompare, CryptoRank, DeFiLlama, DIA, Mobula
- **`Market.MarketHistoryCache`** — `con_cache`-backed cache for history

### Account / Auth
- User accounts with watchlists, custom ABIs, address tags, tx tags
- Auth: SIWE (Sign-In with Ethereum), Auth0, Keycloak
- Encrypted fields via `Vault` (cloak_ecto + AES)
- Email notifications via Bamboo

### Third-party integrations
- Sourcify, SolidityScan, Noves.fi (tx interpretation), BENS (blockchain ENS), Metadata microservice, TAC operation lifecycle, Multichain Search

---

## Workers & Background Jobs

### GenServers / Supervisors (from application.ex)
| Process | Role |
|---------|------|
| `Explorer.Repo` + `Repo.Replica1` | Ecto DB pools |
| `Explorer.Vault` | Encryption key server |
| `HistoryTaskSupervisor`, `MarketTaskSupervisor` | Task supervisors |
| `SmartContract.SolcDownloader` | Downloads Solc compiler versions |
| `SmartContract.VyperDownloader` | Downloads Vyper compiler versions |
| `Explorer.Chain.Health.Monitor` | Chain health heartbeat |
| `Registry.ChainEvents` | PubSub registry for chain events |
| `Chain.Events.Listener` | Listens to PG NOTIFY for real-time chain events |
| `GasPriceOracle` | Caches suggested gas prices |
| `Market.Fetcher.Coin/Token/History` | Periodic market data fetchers |
| `Validator.MetadataProcessor` | Fetches validator metadata |
| `Tags.AddressTag.Cataloger` | Syncs address tag catalog |
| `SmartContract.CertifiedSmartContractCataloger` | Catalogs certified contracts |
| `Chain.Fetcher.CheckBytecodeMatchingOnDemand` | On-demand bytecode check vs eth-bytecode-db |
| `Chain.Fetcher.LookUpSmartContractSourcesOnDemand` | On-demand source lookup |
| `Chain.Fetcher.FetchValidatorInfoOnDemand` | On-demand validator info |
| `TokenInstanceOwnerAddressMigration.Supervisor` | NFT ownership backfill supervisor |
| `Utility.ReplicaAccessibilityManager` | Monitors replica availability |
| `Utility.VersionConstantsUpdater` | Keeps version constants fresh |
| `Optimism.InteropMessage` | Optimism interop message handler |
| `Market.MarketHistoryCache` (con_cache) | In-memory market history cache |
| `Redix` | Redis connection |

### Background Migrators (indexer mode only)
- `TransactionsDenormalization` — denormalizes block/status onto txs
- `AddressCurrentTokenBalanceTokenType` / `AddressTokenBalanceTokenType` — backfill token type
- `SanitizeMissingBlockRanges` / `SanitizeIncorrectNFTTokenTransfers` / `SanitizeIncorrectWETHTokenTransfers`
- `TokenTransferTokenType`, `TokenTransferBlockConsensus`, `TransactionBlockConsensus`
- `RestoreOmittedWETHTransfers`, `FilecoinPendingAddressOperations`
- `CeloL2Epochs`, `CeloAccounts`, `CeloAggregatedElectionRewards`
- `SanitizeErc1155TokenBalancesWithoutTokenIds`
- `BackfillMultichainSearchDB`, `ArbitrumDaRecordsNormalization`, `ShrinkInternalTransactions`

### Cache Counters (persistent GenServers)
Blocks, Transactions, Addresses, Contracts (total/new/verified), Gas usage, Block fees (burnt/priority), Average block time, Token holders, Token transfers, Withdrawals sum, Address-level counters (tx count, token count, gas usage), Pending operations counts, L2-specific (Optimism output root size, Rootstock locked BTC, Blackfort/Stability validator counts, Celo epochs)

---

## Migrations Summary

- **Total:** 320 migrations
- **Earliest:** 2018-01-17 (`create_address`, `create_blocks`, `create_transactions`)
- **Early key:** `create_logs` (2018-02-12), `create_internal_transactions` (2018-02-21), `create_market_history` (2018-04-24), `create_smart_contract_verified` (2018-05-18)
- **Recent (2025-2026):**  
  - FHE operations table (2025-12)  
  - Contract verification status alterations  
  - NFT migration re-runs  
  - Internal TX PK constraints  
  - Multichain search DB queue vacuums  
  - Drop `is_vyper_contract` column (2026-02)  
  - `address_names` primary key change (2026-03)

---

## Unresolved Questions

- None for documentation purposes.
