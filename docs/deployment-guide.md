# Deployment Guide

**Blockscout v10.0.6** — Installation, configuration, and deployment

---

## Quick Start (Docker Compose)

### Prerequisites

- Docker + Docker Compose (v20.10+)
- PostgreSQL 12+ (or use compose service)
- Redis 5+ (or use compose service)
- JSON-RPC endpoint (Geth, Nethermind, Erigon, etc.)

### Minimal Setup (5 minutes)

```bash
# Clone repository
git clone https://github.com/blockscout/blockscout.git
cd blockscout

# Copy environment template
cp docker-compose/.env.example .env

# Edit .env with your settings
nano .env

# Start services
docker-compose -f docker-compose/docker-compose.yml up -d

# Verify
curl http://localhost:4000/api/v2/blocks
```

---

## Environment Variables Reference

**Critical (Required):**

| Variable | Example | Purpose |
|----------|---------|---------|
| `ETHEREUM_JSONRPC_HTTP` | `http://localhost:8545` | JSON-RPC endpoint (HTTP) |
| `ETHEREUM_JSONRPC_HTTP_TIMEOUT` | `20` | Request timeout (seconds) |
| `DATABASE_URL` | `postgresql://postgres:password@db:5432/blockscout` | Primary PostgreSQL connection |
| `COIN_SYMBOL` | `ETH` | Native currency symbol |
| `CHAIN_ID` | `1` | EVM chain ID |

**Database:**

| Variable | Default | Purpose |
|----------|---------|---------|
| `POOL_SIZE` | `10` | Connection pool size |
| `DATABASE_SSL` | `false` | Use SSL for DB connection |
| `REPLICA_ENABLED` | `false` | Enable read replica |
| `REPLICA_URL` | (none) | Read-only replica connection string |

**RPC Configuration:**

| Variable | Default | Purpose |
|----------|---------|---------|
| `ETHEREUM_JSONRPC_VARIANT` | `geth` | Client variant (geth, nethermind, erigon, besu, rsk, anvil, filecoin, zilliqa) |
| `ETHEREUM_JSONRPC_HTTP_SCHEMA` | `http` | HTTP or HTTPS |
| `ETHEREUM_JSONRPC_HTTP_HOST` | `localhost` | RPC host |
| `ETHEREUM_JSONRPC_HTTP_PORT` | `8545` | RPC port |
| `ETHEREUM_JSONRPC_TRANSPORT` | `http` | Transport (http, websocket, ipc) |
| `ETHEREUM_JSONRPC_WS_URL` | (none) | WebSocket URL (if using WS for realtime) |

**Indexing:**

| Variable | Default | Purpose |
|----------|---------|---------|
| `INDEXER_DISABLE_BLOCK_REWARD_FETCHER` | `false` | Skip block reward fetching |
| `INDEXER_DISABLE_INTERNAL_TRANSACTIONS_FETCHER` | `false` | Skip internal tx fetching |
| `INDEXER_DISABLE_PENDING_TRANSACTIONS_FETCHER` | `false` | Skip mempool fetching |
| `BLOCK_TRANSFORMER` | `base` | Base transformer (base, optimism, arbitrum, etc.) |
| `FETCH_COIN_BALANCE_ON_DEMAND` | `true` | Warm balance cache on-demand |

**Chain Type:**

| Variable | Default | Purpose |
|----------|---------|---------|
| `CHAIN_TYPE` | `ethereum` | ethereum, optimism, arbitrum, zksync, polygon_zkevm, scroll, shibarium, celo, filecoin, zilliqa, etc. |

**API Configuration:**

| Variable | Default | Purpose |
|----------|---------|---------|
| `DISABLE_GRAPHQL` | `false` | Disable GraphQL endpoint |
| `DISABLE_READ_API` | `false` | Disable REST API (v2) |
| `DISABLE_WRITE_API` | `false` | Disable contract write APIs |
| `API_RATE_LIMIT` | `50` | Requests per second (unauthenticated) |
| `API_RATE_LIMIT_WITH_TOKEN` | `500` | Requests per second (authenticated) |

**Authentication:**

| Variable | Default | Purpose |
|----------|---------|---------|
| `AUTH0_DOMAIN` | (none) | Auth0 tenant domain |
| `AUTH0_CLIENT_ID` | (none) | Auth0 client ID |
| `AUTH0_CLIENT_SECRET` | (none) | Auth0 secret |
| `KEYCLOAK_ENABLED` | `false` | Enable Keycloak auth |

**Smart Contract Verification:**

| Variable | Default | Purpose |
|----------|---------|---------|
| `SOLC_BINARY` | `/usr/bin/solc` | Solc compiler path |
| `VYPER_BINARY` | `/usr/bin/vyper` | Vyper compiler path |
| `RUST_VERIFIER_MICROSERVICE` | (none) | Rust verifier endpoint (sc-verifier) |
| `ETHEREUM_BYTECODE_DB_URL` | (none) | eth-bytecode-db endpoint |

**Market Data:**

| Variable | Default | Purpose |
|----------|---------|---------|
| `COIN_MARKET_CAP_ENABLED` | `false` | Enable CoinMarketCap pricing |
| `COIN_MARKET_CAP_API_KEY` | (none) | CMC API key |
| `COINGECKO_COIN_ID` | `ethereum` | CoinGecko coin ID |
| `MARKET_HISTORY_FETCH_INTERVAL` | `3600` | Update interval (seconds) |

**Branding & UI:**

| Variable | Default | Purpose |
|----------|---------|---------|
| `BLOCKSCOUT_HOST` | `localhost:4000` | Public domain + port |
| `LOGO` | (blockscout) | Logo URL or path |
| `LOGO_TEXT` | `Blockscout` | Logo text |
| `FOOTER_CHAIN_LOGO_URL` | (none) | Chain logo for footer |
| `NETWORK_NAME` | `Ethereum` | Chain name displayed |
| `NETWORK_ICON` | (none) | Network icon URL |

**Redis:**

| Variable | Default | Purpose |
|----------|---------|---------|
| `REDIS_URL` | `redis://localhost:6379/0` | Redis connection |
| `REDIS_CACHE_ENABLED` | `true` | Use Redis for cache |

**Monitoring:**

| Variable | Default | Purpose |
|----------|---------|---------|
| `DATADOG_ENABLED` | `false` | Enable Datadog tracing |
| `DATADOG_API_KEY` | (none) | Datadog API key |
| `PROMETHEUS_ENABLED` | `true` | Enable Prometheus metrics |
| `PROMETHEUS_PORT` | `4002` | Metrics endpoint port |

---

## Docker Compose Stack

### Minimal Stack (docker-compose/docker-compose.yml)

```yaml
version: '3.9'

services:
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: blockscout
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  backend:
    image: blockscout/blockscout:latest
    depends_on:
      - db
      - redis
    environment:
      ETHEREUM_JSONRPC_HTTP: http://host.docker.internal:8545
      DATABASE_URL: postgresql://postgres:postgres@db:5432/blockscout
      REDIS_URL: redis://redis:6379/0
      COIN_SYMBOL: ETH
      CHAIN_ID: 1
    ports:
      - "4000:4000"
    volumes:
      - ./config/runtime.exs:/app/config/runtime.exs:ro

  frontend:
    image: blockscout/frontend:latest
    depends_on:
      - backend
    environment:
      NEXT_PUBLIC_API_HOST: http://backend:4000
    ports:
      - "3000:3000"

volumes:
  postgres_data:
```

### Production Stack

Add services for:
- nginx (reverse proxy, SSL termination)
- PostgreSQL replicas (read scaling)
- Redis cluster (high availability)
- Monitoring (Prometheus, Datadog agent)
- Backups (pg_dump scheduler)

---

## Database Setup

### Initial Migration

```bash
# Start services without backend
docker-compose up -d db redis

# Run migrations
docker-compose run backend mix ecto.migrate

# (Or in OTP release)
./blockscout/bin/blockscout eval "Explorer.ReleaseTasks.create_and_migrate()"
```

### Backup & Restore

```bash
# Backup
pg_dump -U postgres -h localhost -d blockscout > blockscout_backup.sql

# Restore
psql -U postgres -h localhost -d blockscout < blockscout_backup.sql
```

### Performance Tuning

**PostgreSQL (postgresql.conf):**

```
shared_buffers = 256MB          # 25% of system RAM
effective_cache_size = 1GB      # 50% of system RAM
work_mem = 16MB                 # shared_buffers / max_connections
maintenance_work_mem = 64MB
wal_buffers = 16MB
random_page_cost = 1.1          # SSD optimized
```

**Connection Pool:**

```elixir
# config/runtime.exs
config :explorer, Explorer.Repo,
  pool_size: 20,                # Adjust per workload
  timeout: 30_000,              # Query timeout
  queue_target: 50,             # Backpressure tuning
  queue_interval: 5_000
```

---

## Microservices Configuration

### Smart Contract Verifier (sc-verifier)

```bash
# Deploy as separate service
docker run -d \
  -p 8050:8050 \
  ghcr.io/blockscout/sc-verifier:latest

# Configure in Blockscout
export RUST_VERIFIER_MICROSERVICE=http://verifier:8050
```

### Signature Provider (sig-provider)

```bash
# Deploy for function signature lookup
docker run -d \
  -p 8051:8051 \
  ghcr.io/blockscout/sig-provider:latest

# Configure
export SIG_PROVIDER_URL=http://sig-provider:8051
```

### Bytecode Database (eth-bytecode-db)

```bash
# Deploy for contract bytecode matching
docker run -d \
  -p 8052:8052 \
  ghcr.io/blockscout/eth-bytecode-db:latest

# Configure
export ETHEREUM_BYTECODE_DB_URL=http://bytecode-db:8052
```

---

## Kubernetes Deployment

### Helm Chart (Recommended)

```bash
# Add repository
helm repo add blockscout https://charts.blockscout.com
helm repo update

# Install
helm install blockscout blockscout/blockscout \
  --namespace blockscout --create-namespace \
  --set chain.type=ethereum \
  --set ethereum.rpcEndpoint=http://geth:8545 \
  --set database.host=postgres \
  --set database.database=blockscout \
  --set image.tag=v10.0.6

# Verify
kubectl get pods -n blockscout
```

### StatefulSet (Custom)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: blockscout
spec:
  serviceName: blockscout
  replicas: 3
  selector:
    matchLabels:
      app: blockscout
  template:
    metadata:
      labels:
        app: blockscout
    spec:
      containers:
      - name: blockscout
        image: blockscout/blockscout:v10.0.6
        ports:
        - containerPort: 4000
        - containerPort: 4002  # Prometheus
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: blockscout-secrets
              key: database-url
        - name: ETHEREUM_JSONRPC_HTTP
          value: http://geth:8545
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        livenessProbe:
          httpGet:
            path: /api/v2/blocks
            port: 4000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /api/v2/blocks
            port: 4000
          initialDelaySeconds: 10
          periodSeconds: 5
```

---

## Monitoring & Logs

### Prometheus Metrics

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'blockscout'
    static_configs:
      - targets: ['localhost:4002']

  - job_name: 'postgres'
    static_configs:
      - targets: ['localhost:9187']
```

**Key Metrics:**
- `indexer_blocks_fetched_total` — Blocks indexed
- `indexer_import_duration_seconds` — Import latency
- `explorer_repo_query_duration_seconds` — DB query time
- `explorer_chain_blocks_behind` — Sync lag

### Logging

```bash
# View logs (Docker)
docker-compose logs -f backend

# View logs (Kubernetes)
kubectl logs -f statefulset/blockscout -n blockscout

# Export to ELK/Datadog
export DATADOG_ENABLED=true
export DATADOG_API_KEY=<key>
```

---

## Scaling Strategies

### Horizontal Scaling

**Read-heavy setup:**
```
1. Primary Blockscout (write)
2. Read replicas (query-only instances)
3. Load balancer (nginx, HAProxy)
4. Redis cluster (distributed cache)
```

**Deployment:**
```bash
# Instance 1: Indexer + API
REPLICA_ENABLED=false INDEXER_DISABLE_*=false

# Instance 2+: API only (query mode)
REPLICA_ENABLED=true REPLICA_URL=postgresql://replica:5432/blockscout
INDEXER_DISABLE_*=true  # Disable all fetchers
```

### Vertical Scaling

**Database tuning:**
- Increase `pool_size` (up to 100)
- Increase buffer pools (PostgreSQL config)
- Add SSD storage for better I/O

**Cache tuning:**
- Increase `con_cache` memory allocation
- Use Redis Cluster for multi-node cache
- Reduce GraphQL dataloader batch sizes

---

## Security Best Practices

### Network Security

```bash
# Firewall (iptables/ufw)
ufw allow 4000/tcp  # HTTP
ufw allow 443/tcp   # HTTPS
ufw deny 5432/tcp   # DB (internal only)
ufw deny 6379/tcp   # Redis (internal only)

# SSL/TLS (nginx)
server {
  listen 443 ssl;
  ssl_certificate /etc/letsencrypt/live/blockscout.example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/blockscout.example.com/privkey.pem;
}
```

### Database Security

```bash
# Rotate database password
ALTER USER postgres WITH PASSWORD 'new_strong_password';

# Restrict DB access
psql -U postgres -d blockscout -c "REVOKE CREATE ON SCHEMA public FROM PUBLIC;"

# Enable SSL for DB connections
DATABASE_URL="postgresql://user:pass@host:5432/blockscout?sslmode=require"
```

### Environment Variables

```bash
# Never commit secrets to Git
.env              # ← Add to .gitignore
.env.production   # ← Add to .gitignore

# Use secrets manager
export DATADOG_API_KEY=$(aws secretsmanager get-secret-value --secret-id blockscout/datadog_key)
```

---

## Troubleshooting

### Common Issues

**Issue: "Cannot connect to JSON-RPC endpoint"**
```bash
# Verify endpoint
curl http://localhost:8545 -X POST -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'

# Check firewall
telnet localhost 8545

# Update env
export ETHEREUM_JSONRPC_HTTP_TIMEOUT=30
```

**Issue: "Database connection pool exhausted"**
```bash
# Increase pool size
export POOL_SIZE=50

# Check active connections
psql -c "SELECT count(*) FROM pg_stat_activity WHERE datname='blockscout';"
```

**Issue: "GraphQL queries timing out"**
```bash
# Increase query timeout
export GRAPHQL_QUERY_TIMEOUT=30000  # milliseconds

# Check slow queries
SELECT query, mean_time FROM pg_stat_statements ORDER BY mean_time DESC;
```

**Issue: "WebSocket subscriptions not working"**
```bash
# Enable WS transport
export ETHEREUM_JSONRPC_TRANSPORT=websocket
export ETHEREUM_JSONRPC_WS_URL=ws://localhost:8546

# Check WebSocket connection
wscat -c ws://localhost:4000/socket
```

---

## Maintenance Tasks

### Regular Backups

```bash
# Daily backup (cron)
0 2 * * * pg_dump -U postgres blockscout | gzip > /backups/blockscout_$(date +\%Y\%m\%d).sql.gz
```

### Dependency Updates

```bash
# Check for updates
mix deps.update --all

# Review breaking changes (CHANGELOG.md)
git log --oneline | head -20

# Test in staging first
docker build -t blockscout:latest .
```

### Database Maintenance

```bash
# Vacuum (PostgreSQL cleanup)
VACUUM ANALYZE;

# Reindex (if query performance degrades)
REINDEX DATABASE blockscout;
```

---

## Performance Benchmarks

### Expected Performance (v10.0.6)

| Metric | Target | Notes |
|--------|--------|-------|
| Block indexing latency | <12s | From on-chain to indexed |
| Average API response time | <500ms | p50 for REST endpoints |
| p99 API response time | <2s | 99th percentile |
| Concurrent API clients | 10k+ | With rate limiting |
| Transactions per second (indexing) | 100+ | Depends on JSON-RPC endpoint |
| Uptime | 99.5%+ | Across 30-day window |

### Load Testing

```bash
# Apache Bench
ab -n 10000 -c 100 http://localhost:4000/api/v2/blocks

# Wrk (more realistic)
wrk -t12 -c400 -d30s --latency http://localhost:4000/api/v2/blocks
```

