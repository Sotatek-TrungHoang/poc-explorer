# Scout Report: apps/indexer + apps/ethereum_jsonrpc

## Indexer App Overview

- **Purpose**: Fetches blockchain data from on-chain node, indexes it to the Explorer DB
- **Version**: 10.0.6 (umbrella app, Elixir ~> 1.19)
- **Entry point**: `Indexer.Application` → `Indexer.Supervisor`
- **Architecture**: OTP supervision tree; `use Indexer.Fetcher` macro auto-generates supervisor + TaskSupervisor boilerplate per fetcher. All fetchers individually toggle-able via `Application.get_env(:indexer, Module)[:enabled]`.

---

## Indexer Key Modules

### Block Layer (`lib/indexer/block/`)
| Module | Role |
|---|---|
| `Block.Catchup.Fetcher` | Fills `MissingBlockRange` gaps from latest → genesis (Task-based, batch + concurrency configurable) |
| `Block.Catchup.BoundIntervalSupervisor` | Adaptive polling scheduler for catchup |
| `Block.Catchup.MissingRangesCollector` | Computes gap ranges from DB |
| `Block.Catchup.MassiveBlocksFetcher` | Separate pipeline for oversized blocks |
| `Block.Realtime.Fetcher` | WebSocket `eth_subscribe newHeads`; falls back to HTTP polling ≥1s; tracks last 10 blocks for reorg detection |
| `Block.Fetcher` (shared) | Core `fetch_and_import_range/2`; dispatches async sub-fetchers post-import |

### Fetchers (`lib/indexer/fetcher/`)
| Fetcher | Description |
|---|---|
| `InternalTransaction` | Traces (per-tx or per-block depending on client variant) |
| `CoinBalance.Catchup/Realtime` | Address coin balance updates |
| `TokenBalance.Current/Historical` | ERC-20/721/1155 balances |
| `TokenInstance.Realtime/Retry/Refetch/Sanitize/SanitizeERC1155/SanitizeERC721` | NFT metadata with retry queues |
| `Token/TokenUpdater/TokenTotalSupplyUpdater` | Token metadata sync |
| `ContractCode` | Bytecode for newly created contracts |
| `BlockReward` | Miner/uncle rewards |
| `UncleBlock` | Uncle block fetching |
| `PendingTransaction` | Mempool |
| `Withdrawal` | EIP-4895 validator withdrawals |
| `SignedAuthorizationStatus` | EIP-7702 auth statuses |
| `AddressNonceUpdater` | On-demand nonce refresh |
| `RollupL1ReorgMonitor` | L1 reorg detection for rollups |
| `OnDemand.*` | CoinBalance, TokenBalance, ContractCode, TokenInstance on-demand |
| **L2/Chain-specific** | Optimism, Arbitrum, PolygonZkEvm, Scroll, ZkSync, Shibarium, Celo, Filecoin, Zilliqa, Stability, Blackfort |
| `MultichainSearchDb.*` | Export queues (main, balances, counters, token info) for multichain search |

### Transformers (`lib/indexer/transform/`)
Shape raw RPC responses into DB-importable params:
- Core: `Blocks`, `Addresses`, `AddressCoinBalances`, `AddressCoinBalancesDaily`, `AddressTokenBalances`, `TokenTransfers`, `TokenInstances`, `MintTransfers`, `TransactionActions`, `SignedAuthorizations`
- Chain-specific: Arbitrum, Celo, Optimism, PolygonZkEvm, Scroll, Shibarium, Stability

### Other Notable Modules
- `Indexer.BufferedTask` — batching buffer for async work queues
- `Indexer.Memory` — memory pressure monitoring; triggers fetcher pausing
- `Indexer.Prometheus` — metrics instrumentation
- `Indexer.Migrator.*` — background DB migration workers (WETH recovery, uncataloged transfers, uncles without index)
- `Indexer.Supervisor` — root supervisor, chain-type-aware child configuration

---

## Indexing Strategies

### Dual-Mode
| Mode | Trigger | Direction |
|---|---|---|
| **Realtime** | WS `eth_subscribe newHeads` or HTTP polling (≥1s min) | Latest block forward |
| **Catchup** | `BoundIntervalSupervisor` scheduler | Latest → genesis via `MissingBlockRange` |

### Catchup Details
- `MissingBlockRange` DB table is source of truth for gaps
- Adaptive backoff: `BoundIntervalSupervisor` grows/shrinks interval based on whether blocks remain
- `MassiveBlocksFetcher` isolates large blocks to prevent batch stalls
- Batch size × concurrency = parallelism (e.g. 10 batches × 10 concurrent)

### Realtime + Reorg Handling
- WS primary; HTTP polling fallback
- Tracks `previous_number`; gap detected → fetches in-between blocks
- Non-consensus (orphaned) blocks marked in DB; sub-chain data invalidated
- Publishes `chain_event` via `Explorer.Chain.Events.Publisher` for live UI

### Async Sub-Fetcher Pattern
After block import, `Block.Fetcher` dispatches async tasks for: internal txs, coin balances, token balances, token instances, contract codes, block rewards, uncles, withdrawals, signed authorizations, EIP-4844 blobs, L2-specific ops — each backed by its own supervised GenServer queue.

---

## Ethereum JSON-RPC App Overview

- **Purpose**: Transport-agnostic Ethereum JSON-RPC client library
- **Version**: 10.0.6 (standalone umbrella app, dep of `indexer`)
- **Architecture**: Two orthogonal behaviours: `Transport` (how to talk) + `Variant` (what client-specific methods to call); both pluggable via runtime config

---

## JSON-RPC Key Modules

### Core API
| Module | Role |
|---|---|
| `EthereumJSONRPC` | Public API: `json_rpc/2`, `fetch_blocks_by_range/2`, `fetch_transaction_receipts/2`, etc. |
| `Transport` | Behaviour: `json_rpc/2`, `subscribe/3`, `unsubscribe/2` |
| `Variant` | Behaviour: `fetch_internal_transactions/2`, `fetch_block_internal_transactions/2`, `fetch_beneficiaries/2`, `fetch_pending_transactions/1`, `fetch_first_trace/2` |

### Transport Implementations
| Module | Transport |
|---|---|
| `HTTP` | HTTP with pluggable adapter; supports single request, batch list, chunked batch |
| `HTTP.HTTPoison` | HTTPoison adapter |
| `HTTP.Tesla` | Tesla adapter |
| `WebSocket` | WS-based transport (GenServer); supports `subscribe/unsubscribe` |
| `WebSocket.WebSocketClient` | Low-level WS; `registration.ex`, `retry_worker.ex` for reconnect |
| `IPC` | Unix socket (dev/test) |

### Batch Request Handling
- `HTTP.json_rpc([requests], opts)` → `chunked_json_rpc/3`: splits into chunks, sends as JSON arrays, merges responses
- `WebSocket.json_rpc/2`: per-request concurrent calls over one connection
- All response IDs correlated back to requests

### Request Management
| Module | Role |
|---|---|
| `RequestCoordinator` | Timeout-based backoff + rate limiting via `RollingWindow` ETS; adds jitter |
| `RollingWindow` | Sliding window event counter (ETS-backed) |
| `Utility.EndpointAvailabilityObserver` | Endpoint health tracking; selects fallback URL on failure |
| `Utility.EndpointAvailabilityChecker` | Periodic health probe |
| `Utility.CommonHelper` | Duration parsing, misc helpers |
| `Utility.RangesHelper` | Block range arithmetic |

### Data Type Modules
`Block/Blocks`, `Transaction/Transactions`, `Log/Logs`, `Receipt/Receipts`, `Uncle/Uncles`, `Withdrawal/Withdrawals`, `Nonce/Nonces`, `FetchedBalance/s`, `FetchedBeneficiary/ies`, `FetchedCode/s`, `PendingTransaction`, `Subscription`, `SignedAuthorization`

---

## Chain Variants

| Module | Client | Internal TX method | Beneficiaries |
|---|---|---|---|
| `Geth` | go-ethereum | `debug_traceTransaction` (custom JS tracer) | `:ignore` |
| `Nethermind` | Nethermind | `:ignore` (block-level only) | `trace_replayBlockTransactions` |
| `Erigon` | Erigon | `:ignore` (block-level only) | `trace_replayBlockTransactions` |
| `Besu` | Hyperledger Besu | `debug_traceTransaction` | `trace_replayBlockTransactions` |
| `RSK` | Rootstock | `:ignore` | `:ignore` |
| `Anvil` | Foundry Anvil | Geth-like | — |
| `Filecoin` | Filecoin EVM | Custom actor/CID | — |
| `Zilliqa` | Zilliqa EVM | Custom quorum cert fields in Blocks | — |

Chain-specific dirs in ethereum_jsonrpc: `arbitrum/`, `zksync/`, `zilliqa/`

### Configuration Pattern
```elixir
# config/config.exs
config :ethereum_jsonrpc, :variant, EthereumJSONRPC.Geth
config :ethereum_jsonrpc, :transport, EthereumJSONRPC.HTTP
config :explorer, :chain_type, :optimism  # affects supervisor + transforms
```

Chain type drives: which fetchers start, which transformers run, which block fields are expected.

---

## Unresolved Questions

1. Max reorg depth tolerated in realtime fetcher? Is there a configurable uncle depth limit?
2. Dead-letter handling for permanently failing sub-fetcher tasks (malformed trace responses)?
3. `IPC` transport — still actively used in production or legacy/test-only?
4. `MultichainSearchDb` — what external service consumes these export queues?
5. `Indexer.Memory` monitor — what's the eviction/pausing policy when memory pressure is high?
