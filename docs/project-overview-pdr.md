# Project Overview & PDR

**Blockscout v10.0.6** — Open-source blockchain explorer for EVM chains
**License:** GPL v3.0
**Umbrella App:** 6 OTP apps, ~2600 source files, 320 DB migrations

---

## Project Purpose

Blockscout is a **transparent, open-source alternative** to centralized block explorers (Etherscan, Etherscan Classic). Provides transaction search, account inspection, smart contract verification, token tracking, and multichain support for Ethereum, L2s, sidechains, testnets, and private networks.

**Key Differentiation:**
- Open-source (GPL v3.0) for blockchain transparency and community audit
- Supports **400+ chains** (Ethereum, Optimism, Arbitrum, zkSync, Gnosis, Polygon, Scroll, Shibarium, zkEVM, Celo, Filecoin, etc.)
- Multiple API generations (Etherscan-compatible RPC, v1 semi-REST, v2 modern REST)
- GraphQL + WebSocket real-time capabilities
- Smart contract verification (Solidity, Vyper, Stylus)
- Proxy detection (EIP-1167, 1822, 1967, 2535, 7702)
- L2-specific semantics (Arbitrum batch submissions, Optimism output roots, zkSync proofs, etc.)

---

## Target Users

1. **Blockchain Networks** — Self-host for transparency + community trust
2. **Developers** — Smart contract verification, transaction debugging, ABI interaction
3. **Users** — Search addresses, verify tokens, inspect contract code
4. **Analytics Platforms** — Multichain indexing and data export
5. **Enterprises** — Private network deployments with custom branding

---

## Key Features

### Search & Discovery
- Transaction search by hash, address, method signature
- Address inspection (balances, holdings, transaction history)
- Token discovery (ERC-20, ERC-721, ERC-1155)
- Smart contract catalog with verified source code
- Multichain search across deployments
- Tag-based address cataloging (exchanges, contracts, etc.)

### Real-time Indexing
- Block indexing (realtime via WebSocket or polling)
- Transaction indexing with status tracking
- Internal transactions (calls, creates, selfdestruct)
- Event logs with decoded parameters
- Token transfers (ERC-20, 721, 1155)
- Validator withdrawals (EIP-4895)
- Account abstraction (ERC-4337 user operations)
- Signed authorizations (EIP-7702)
- Reorg handling + orphan marking

### Smart Contracts
- Verification: Solidity, Vyper, Stylus (WASM)
- Multi-file compilation (standard JSON)
- Proxy detection (6 proxy patterns)
- ABI parsing & method decoding
- State reading via JSONRPC
- Write preparation (unsigned txs)
- Bytecode matching (eth-bytecode-db integration)

### APIs
- **Etherscan-compatible RPC** — Drop-in replacement
- **REST v1** — Semi-structured endpoints
- **REST v2** — OpenAPI-documented modern API
- **GraphQL** — Absinthe with Relay pagination
- **WebSocket** — Real-time block/tx/token/rate updates
- **Rate limiting** — Tiered API-key plans (Hammer + Redis)

### Market Data
- Coin pricing (configurable sources: CoinGecko, CoinMarketCap, CryptoCompare, DIA, etc.)
- Token pricing
- Historical OHLC data
- In-memory caching via con_cache

### Authentication
- SIWE (Sign-In with Ethereum)
- Auth0 OAuth
- Keycloak support
- API key auth

### L2 Chain Support
- **Arbitrum:** Batch submissions, cross-chain messages, DA metrics
- **Optimism:** Output roots, finality proofs, interop messages
- **zkSync:** ZK proof submissions, Merkle proofs
- **Scroll:** Batch finality tracking
- **Polygon zkEVM:** Sequence submission proofs
- **Celo:** Epoch rewards, validator elections
- **Filecoin:** Pending actor operations
- **Shibarium:** Bridge tracking
- **Zilliqa:** Custom consensus fields

---

## Technology Stack

### Backend
- **Language:** Elixir ~1.19 (functional, distributed, fault-tolerant)
- **Framework:** Phoenix 1.6 (real-time, WebSocket, API)
- **Database:** PostgreSQL with migrations (320 total)
- **ORM:** Ecto with Ecto.Multi for bulk upserts
- **Caching:**
  - Redis (rate limiting, session, distributed cache)
  - Con_cache (in-memory OHLC, health metrics)
- **Job Queue:** Que (background workers)
- **Clustering:** libcluster (distributed Erlang)

### APIs & Integrations
- **JSON-RPC:** ethereum_jsonrpc (pluggable transport: HTTP, WS, IPC)
- **Variants:** Geth, Nethermind, Erigon, Besu, RSK, Anvil, Filecoin, Zilliqa
- **GraphQL:** Absinthe (Relay cursor pagination)
- **WebSocket:** Phoenix Channels (real-time subscriptions)

### Microservices (Optional)
- **sc-verifier** — Rust-based smart contract verifier (Sourcify, Hardhat)
- **sig-provider** — Function/event signature lookup
- **eth-bytecode-db** — Contract bytecode matching
- **visualizer** — Transaction visualization
- **nft-media-handler** — NFT media pipeline (Vix/libvips → R2/S3)
- **user-ops-indexer** — ERC-4337 indexing

### Monitoring & Observability
- **Prometheus** — Metrics (ETS counters, Ecto instrumentation)
- **Datadog** — Distributed tracing (Spandex)
- **Health monitoring** — Chain health heartbeat, replica availability

### Frontend (Separate)
- **Next.js** — Modern SSR framework
- **Web3.js** — Ethereum client
- **Redux** — State management
- **Bootstrap 4** — UI components
- **Chart.js** — Data visualization
- **Phoenix LiveView** — Real-time UI updates

### Deployment
- Docker (image-based, production-ready)
- Docker Compose (local + staging)
- Kubernetes (cloud-native)
- Ansible (infrastructure automation)

---

## Non-Functional Requirements

### Performance
- **Block indexing:** Near real-time (< 12s after on-chain confirmation)
- **Query latency:** Sub-second for common queries (addresses, txs, blocks)
- **Throughput:** Support thousands of concurrent API requests
- **Caching:** 25+ persistent counters (GenServers) + Redis + Con_cache

### Scalability
- **Umbrella OTP apps** — Isolated supervision trees
- **Read replicas** — Dynamic routing via ReplicaAccessibilityManager
- **Async sub-fetchers** — Parallel block, internal-tx, token-balance, contract-code processing
- **Batching:** BufferedTask queues with configurable batch size × concurrency
- **Memory pressure monitoring** — Pause fetchers under load

### Reliability
- **Reorg handling:** Gap detection, orphan marking, chain_event publishing
- **Multi-endpoint fallback:** EndpointAvailabilityObserver selects healthy URLs
- **Request backoff:** RequestCoordinator with jitter + exponential timeout
- **Distributed clustering:** libcluster for multi-node deployments
- **Job durability:** Que persists background jobs

### Security
- **Encryption:** Vault + cloak_ecto for sensitive fields (API keys, emails)
- **Auth:** SIWE, OAuth, API keys, CSRF protection
- **Rate limiting:** Hammer + Redis tiered plans
- **Smart contract verification:** Sandboxed compilers (Solc, Vyper)

### Maintainability
- **Code standards:** Elixir conventions, Credo linting
- **Testing:** ExUnit for core logic, integration tests
- **Documentation:** Inline comments, scout reports, API docs
- **Migrations:** Automated Ecto migration framework
- **Modularity:** 6 focused OTP apps

### Availability
- **Deployments:** Manual, Docker Compose, Kubernetes, Ansible
- **Multi-chain:** 400+ supported chains
- **Versioning:** Semantic versioning, backward-compatible APIs
- **Monitoring:** Prometheus metrics + Datadog tracing

---

## Success Metrics

- **Uptime:** ≥99.5% (SLA for production instances)
- **API latency:** p99 < 2s for 95% of endpoints
- **Block finality:** Sync within 1 block of chain head
- **Community adoption:** 400+ public instances, 100k+ DAU across ecosystem
- **Code quality:** Zero critical security CVEs, <5% unverified contracts
- **Developer experience:** < 15 min setup, comprehensive API docs

---

## Constraints

- **License:** GPL v3.0 (derivative works must be open-source)
- **Database:** PostgreSQL required (no other RDBMS tested)
- **Blockchain:** EVM-compatible only (Ethereum, L2s, sidechains, testnets, private)
- **Smart contract verification:** Limited to Solidity, Vyper, Stylus (no Move, Cairo)
- **Indexing lag:** Depends on JSON-RPC endpoint response time
