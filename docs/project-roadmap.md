# Project Roadmap

**Blockscout v10.0.6** — Development roadmap and current status

---

## Current Status

**Version:** 10.0.6
**Release Date:** 2026-Q1
**Supported Chains:** 400+
**DB Migrations:** 320 (2018-2026-03)
**License:** GPL v3.0
**Maintainers:** Blockscout Core Team + Community

---

## Phase 1: Core Stability (Completed)

**Status:** ✅ **COMPLETE**

Core blockchain indexing, data model, and API layers.

**Milestones:**
- ✅ Ecto schemas for core entities (address, block, transaction, token)
- ✅ Realtime block indexing via WebSocket
- ✅ Catchup indexing for missing block ranges
- ✅ Etherscan-compatible API
- ✅ PostgreSQL + replica read failover
- ✅ Reorg handling and orphan marking
- ✅ Smart contract verification (Solidity, Vyper)

**Key Components:**
- 6 OTP umbrella apps
- 70+ Ecto schemas
- ~2600 source files

---

## Phase 2: API & Real-time Features (Completed)

**Status:** ✅ **COMPLETE**

Multi-API support and real-time subscriptions.

**Milestones:**
- ✅ v1 semi-REST API
- ✅ v2 modern REST API with OpenAPI docs
- ✅ GraphQL with Relay pagination and dataloader
- ✅ WebSocket real-time subscriptions (blocks, txs, balances, rates)
- ✅ Rate limiting (Hammer + Redis)
- ✅ API key management + tiered access
- ✅ SIWE authentication + OAuth integration

**Key Components:**
- 3 API generations in block_scout_web
- Absinthe GraphQL server
- Phoenix Channels WebSocket

---

## Phase 3: L2 Support (In Progress)

**Status:** 🟡 **~80% COMPLETE** (Active Development)

Multi-chain L2 and sidechain semantics.

**Completed:**
- ✅ Arbitrum batch submissions + DA metrics + cross-chain messages
- ✅ Optimism output roots + finality + interop messages
- ✅ zkSync batches + L1 anchors
- ✅ Polygon zkEVM sequence submissions
- ✅ Scroll batch data
- ✅ Celo epoch rewards + validator elections
- ✅ Filecoin pending operations
- ✅ Shibarium bridge tracking
- ✅ Zilliqa custom consensus fields
- ✅ Stability + Blackfort validator metadata

**In Progress:**
- 🟡 FHE (Fully Homomorphic Encryption) operations indexing (migration 2025-12)
- 🟡 EIP-7702 signed authorizations (recent migration 2026-02)
- 🟡 Multichain search optimization (queue vacuums added 2026-03)

**Planned:**
- 🔵 MUD framework entity tracking (partial support)
- 🔵 More rollup variants (Metis, Mantle, Linea)
- 🔵 Testnet chain support enhancements

---

## Phase 4: Smart Contract Features (In Progress)

**Status:** 🟡 **~90% COMPLETE** (Maintenance + Enhancements)

Contract verification, bytecode matching, proxy detection.

**Completed:**
- ✅ Solidity verification (multi-file, standard JSON)
- ✅ Vyper verification with compiler integration
- ✅ Stylus (WASM) verification
- ✅ Proxy detection (6 patterns: EIP-1167, 1822, 1967, 2535, 7702, ERC-7760)
- ✅ ABI parsing and method decoding
- ✅ Bytecode matching via eth-bytecode-db
- ✅ Function signature lookup via sig-provider
- ✅ Contract state reader (JSONRPC calls)
- ✅ Write preparation (unsigned tx builder)

**In Progress:**
- 🟡 Certified contract catalog sync
- 🟡 Contract method caching optimization

**Planned:**
- 🔵 Cairo/Move contract verification (future EVM extensions)
- 🔵 LLVM bytecode matching
- 🔵 More proxy patterns (upcoming EIPs)

---

## Phase 5: Token & NFT Support (Completed)

**Status:** ✅ **COMPLETE** (Maintenance Mode)

ERC-20, ERC-721, ERC-1155 tracking and metadata.

**Milestones:**
- ✅ ERC-20 fungible token tracking
- ✅ ERC-721 NFT indexing + ownership
- ✅ ERC-1155 multi-token support
- ✅ Token instance metadata fetching (IPFS, HTTP)
- ✅ NFT media pipeline (Vix/libvips + R2/S3 upload)
- ✅ Token price fetching (7 market data sources)
- ✅ Historical token balance snapshots
- ✅ Address-level token holdings

**Recent Fixes:**
- 🔧 NFT ownership migration re-runs (2025)
- 🔧 WETH transfer recovery (2025)
- 🔧 ERC-1155 balance sanitization (2026)

---

## Phase 6: Market Data & Analytics (Completed)

**Status:** ✅ **COMPLETE** (Ongoing Operations)

Exchange rates, OHLC history, portfolio analytics.

**Features:**
- ✅ Coin pricing (CoinGecko, CoinMarketCap, CryptoCompare, CryptoRank, DeFiLlama, DIA, Mobula)
- ✅ Token pricing (configurable source)
- ✅ Historical OHLC candles (1h, 1d)
- ✅ Gas price oracle + suggestion
- ✅ Market history cache (con_cache)
- ✅ Real-time exchange rate subscriptions

**In Progress:**
- 🟡 Market data source reliability improvements

---

## Phase 7: Account & User Features (Completed)

**Status:** ✅ **COMPLETE** (Maintenance Mode)

User accounts, watchlists, custom ABIs, notifications.

**Features:**
- ✅ User accounts with encrypted data (Vault + AES-256-GCM)
- ✅ Address watchlists
- ✅ Custom ABI management
- ✅ Address tags (personal + community)
- ✅ Transaction tags
- ✅ Email notifications
- ✅ API key generation + rate limit tiers

---

## Phase 8: Advanced Indexing (In Progress)

**Status:** 🟡 **~85% COMPLETE** (Ongoing Optimizations)

Transaction actions, internal transaction handling, reorg robustness.

**Completed:**
- ✅ Internal transaction tracing (Geth, Nethermind, Erigon, Besu variants)
- ✅ Transaction action decoding (DeFi protocol interactions)
- ✅ Uncle block tracking
- ✅ Validator withdrawal tracking (EIP-4895)
- ✅ Account abstraction (ERC-4337 user operations)
- ✅ Reorg detection + orphan marking

**In Progress:**
- 🟡 Internal transaction performance optimization
- 🟡 Reorg depth tolerance analysis
- 🟡 Memory pressure monitoring refinements

**Planned:**
- 🔵 Blobs indexing (EIP-4844 full support)
- 🔵 MEV-Boost block builder tracking
- 🔵 Proposer-builder separation data

---

## Phase 9: Observability & Performance (In Progress)

**Status:** 🟡 **~75% COMPLETE** (Active Optimization)

Prometheus metrics, Datadog tracing, performance tuning.

**Completed:**
- ✅ Prometheus metrics (blocks, txs, queries, requests)
- ✅ Datadog tracing (Spandex integration)
- ✅ Health monitoring (chain sync status)
- ✅ Replica availability tracking
- ✅ Memory pressure monitoring
- ✅ Rate limiter observability

**In Progress:**
- 🟡 Query performance profiling + index optimization
- 🟡 Caching hit rate analysis
- 🟡 Distributed trace correlation improvements

**Planned:**
- 🔵 Custom dashboard templates
- 🔵 Alert rule recommendations
- 🔵 Query planner insights

---

## Phase 10: Deployment & Operations (In Progress)

**Status:** 🟡 **~80% COMPLETE** (Maintenance)

Docker, Docker Compose, Kubernetes, Ansible deployment.

**Completed:**
- ✅ Docker image (Elixir builder + runtime)
- ✅ Docker Compose stacks (local dev, staging, prod)
- ✅ Kubernetes manifests + StatefulSet config
- ✅ Ansible playbooks (infrastructure automation)
- ✅ Nginx reverse proxy config
- ✅ PostgreSQL backup automation

**In Progress:**
- 🟡 Helm chart standardization
- 🟡 GitOps (ArgoCD) integration examples

**Planned:**
- 🔵 Multi-region deployment guides
- 🔵 Disaster recovery runbooks
- 🔵 Cost optimization playbooks

---

## Phase 11: Community & Ecosystem (Ongoing)

**Status:** 🟡 **~70% COMPLETE** (Active)

Open-source community, third-party integrations, ecosystem growth.

**Completed:**
- ✅ Public GitHub repository (GPL v3.0)
- ✅ Discord community (~10k members)
- ✅ Documentation site (docs.blockscout.com)
- ✅ Contributing guidelines + PR template
- ✅ 400+ public chain instances
- ✅ Third-party integrations: Sourcify, SolidityScan, Noves.fi, BENS, TAC

**In Progress:**
- 🟡 Community translator onboarding
- 🟡 Chain operator support program
- 🟡 Bug bounty program enhancements

**Planned:**
- 🔵 Bounty program formalization
- 🔵 Integration marketplace
- 🔵 Certified verifier program

---

## Recent Work (Last 90 Days)

**Completed (2025-Q4 through 2026-Q1):**
1. FHE operations table + indexing migration
2. EIP-7702 signed authorizations support
3. Multichain search database queue optimization
4. Address names PK constraint refactoring (2026-03)
5. Vyper contract flag deprecation (`is_vyper_contract` column drop)
6. Internal transaction PK optimization
7. NFT ownership migration re-runs for accuracy
8. Contract verification status tracking enhancements

**Dependencies Resolved:**
- phoenix ~1.6 (stable)
- ecto ~3.11 (stable)
- absinthe ~1.7 (GraphQL)
- libcluster ~3.3 (clustering)

---

## Upcoming (Next 90 Days)

**Priority 1 (High Impact):**
- [ ] Blob (EIP-4844) full integration + indexing
- [ ] EIP-4788 beacon chain state root verification
- [ ] Query performance optimization (missing index audit)
- [ ] Replica failover robustness improvements

**Priority 2 (Medium Impact):**
- [ ] More L2 variants (Metis, Mantle, Linea)
- [ ] Cairo contract verification (tentative)
- [ ] Consensus client diversity tracking (Prysm, Lighthouse, Nimbus)

**Priority 3 (Polish):**
- [ ] API documentation auto-generation (OpenAPI 3.1)
- [ ] Custom dashboard templates (Grafana)
- [ ] GraphQL subscriptions performance tuning
- [ ] Frontend modernization (Next.js 15 upgrade)

---

## Known Limitations & Future Work

### Technical Debt
- Internal transaction query performance on large chains (100B+ txs)
- Memory footprint of cache counters on multi-chain deployments
- Replica sync lag during high-throughput periods

### Not Supported (Unlikely in Near Term)
- Non-EVM chains (Cosmos, Polkadot, Solana) — architectural mismatch
- Sovereign rollups without EVM — requires custom variants
- Privacy-preserving chains (Monero, Zcash) — not transparent by design

### Community Requests (Backlog)
- Token holders endpoint optimization
- Address label bulk export API
- WebSocket subscription filters (reduce client-side processing)
- Custom branding enhancements (logo, colors, links) — partially done

---

## Success Metrics

**Current (v10.0.6):**
- 400+ supported chains
- 10k+ GitHub stars
- 10k+ Discord members
- ~100k+ DAU across public instances
- <2s p99 latency on majority of endpoints
- 99.5%+ uptime across instances

**2026 Goals:**
- 500+ supported chains
- 15k+ GitHub stars
- Reduce median sync lag to <6 seconds
- <1s p99 latency on 95% of endpoints
- 99.9%+ uptime for prod deployments

---

## Architecture Evolution

**v10.0.6 → v11.0.0 (Q2-Q3 2026):**
- Async/await refactoring (reduce GenServer boilerplate)
- QueryBuilder DSL (type-safe query composition)
- Event-driven architecture (reduce polling)
- Observability-as-code (dynamic metric registration)

**v11.0.0 → v12.0.0 (Q4 2026+):**
- Multiple storage backends (TimescaleDB for time-series)
- Federation support (query across instances)
- Plugin system (community-contributed fetchers/APIs)
- Mobile-first UI redesign

