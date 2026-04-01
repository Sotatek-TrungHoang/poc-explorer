# Code Standards

**Blockscout v10.0.6** — Elixir coding standards and conventions

---

## Elixir Conventions

### Module Organization

**File structure:**
```
lib/explorer/chain/address.ex
  ├── defmodule Explorer.Chain.Address do
  │   ├── use Ecto.Schema
  │   ├── import Ecto.Changeset
  │   │
  │   ├── @moduledoc (brief description)
  │   ├── @typedoc (type documentation)
  │   │
  │   ├── schema "addresses" do  (Ecto schema)
  │   ├── def changeset/2        (Validation)
  │   ├── def validate_*/2       (Validators)
  │   └── # No query logic here — use Explorer.Chain
```

**Separation of concerns:**
- **Schemas** (`lib/explorer/chain/`) — Structure + validation only
- **Queries** (`lib/explorer/chain.ex`) — All read queries; entry point for consumers
- **Imports** (`lib/explorer/chain/import/runner/`) — Bulk write operations
- **Logic** (`lib/explorer/smart_contract/`, etc.) — Business logic modules

### Naming Patterns

| Item | Pattern | Example |
|------|---------|---------|
| Modules | PascalCase (nested) | `Explorer.Chain.Address` |
| Functions | snake_case | `def fetch_address/1` |
| Variables | snake_case | `address_hash`, `tx_count` |
| Schema table | snake_case (plural) | `table "addresses"` |
| Schema field | snake_case | `field :hash` |
| Database column | snake_case | `address_hash CHAR(42)` |
| Constants | SCREAMING_SNAKE_CASE | `@max_batch_size 100` |
| Booleans | `is_` or `has_` prefix | `is_contract`, `has_logs` |
| Behaviour modules | Verb/Type prefix | `Explorer.Repo.Behaviour` |
| Test modules | Append `_test` | `test/explorer/chain/address_test.exs` |

### Function Style

**Public vs Private:**
```elixir
# Public API — well-documented
@doc "Fetches address by hash with balance and transaction count"
@spec get_address(Explorer.Chain.Address.hash()) :: {:ok, Address.t()} | :not_found
def get_address(hash), do: ...

# Private helpers — minimal docs
defp normalize_hash(hash) do ...
```

**Guard clauses over conditionals:**
```elixir
# Preferred
def fetch_blocks(limit) when limit > 0 and limit <= 1000, do: ...

# Avoid (when guard not suitable)
def fetch_blocks(limit) do
  if limit > 0 and limit <= 1000 do
    ...
  else
    :error
  end
end
```

**Pattern matching over explicit checks:**
```elixir
# Preferred
def handle_transaction({:ok, %{"hash" => hash}}) do
  {:ok, hash}
end
def handle_transaction(:error) do
  :error
end

# Avoid
def handle_transaction(response) do
  if is_map(response) and Map.has_key?(response, "hash") do
    {:ok, response["hash"]}
  else
    :error
  end
end
```

### Pipe Operator Usage

**For data transformations:**
```elixir
addresses
|> Enum.filter(&blockchain_addressable?/1)
|> Enum.map(&to_address_map/1)
|> Enum.sort_by(&(&1.balance))
|> Enum.reverse()
```

**Avoid piping for function calls with side effects:**
```elixir
# Better (explicit error handling)
case Repo.get(Address, hash) do
  nil -> :not_found
  addr -> {:ok, addr}
end

# Avoid piping when control flow matters
hash
|> Repo.get(Address)  # Returns nil or %Address, not tuple
|> handle_result()    # Now need unwrapping logic
```

---

## Module Organization

### Supervisor Trees (application.ex pattern)

```elixir
# apps/explorer/lib/explorer/application.ex
defmodule Explorer.Application do
  use Application

  def start(_type, _args) do
    children = [
      # Repo (mandatory)
      {Explorer.Repo, []},
      {Explorer.Repo.Replica1, []},

      # Caching
      {Con.Cache, [name: :market_cache, ttl_check_interval: :timer.seconds(10)]},
      {Redix, [name: :redix]},

      # Workers
      {GenServer, name: Explorer.Vault},
      {GenServer, name: Explorer.Market.Fetcher.Coin},
      {Supervisor, name: Explorer.Chain.Events.Listener},

      # Registry (for PubSub)
      {Registry, keys: :duplicate, name: Registry.ChainEvents},
    ]

    opts = [strategy: :one_for_one, name: Explorer.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

**Key principles:**
- One supervisor file per app (`application.ex`)
- List children in order: dependencies first (Repo), then services, then workers
- Use tuples `{Module, [opts]}` for simple starts; `{:via, Registry, {:name, key}}` for named children

### Schema Structure (Ecto pattern)

```elixir
defmodule Explorer.Chain.Address do
  use Ecto.Schema
  import Ecto.Changeset

  @moduledoc """
  Represents an account/contract on the blockchain.
  """

  # Type definitions
  @type t :: %__MODULE__{
    hash: binary(),
    is_contract: boolean(),
    ...
  }

  # Table definition
  schema "addresses" do
    field :hash, :string, primary_key: true  # 20-byte address as hex string
    field :nonce, :integer, default: 0
    field :is_contract, :boolean, default: false
    field :contract_code, :string
    field :inserted_block_number, :integer

    has_many :transactions, Explorer.Chain.Transaction, foreign_key: :from_address_hash
    belongs_to :smart_contract, Explorer.Chain.SmartContract, foreign_key: :hash, type: :string

    timestamps()
  end

  # Changesets
  @doc "Validates and prepares changeset for insert/update"
  def changeset(address, attrs) do
    address
    |> cast(attrs, [:hash, :nonce, :is_contract])
    |> validate_required([:hash])
    |> unique_constraint(:hash)
    |> validate_hash_format(:hash)
  end

  # Validators
  defp validate_hash_format(changeset, field) do
    validate_change(changeset, field, fn _, hash ->
      if String.match?(hash, ~r/^0x[0-9a-f]{40}$/i) do
        []
      else
        [{field, "must be valid Ethereum address"}]
      end
    end)
  end
end
```

---

## Testing Patterns

### Test Structure

```elixir
# test/explorer/chain/address_test.exs
defmodule Explorer.Chain.AddressTest do
  use ExUnit.Case

  alias Explorer.Chain.Address
  import Explorer.Factory  # Factories for test data

  describe "changeset/2" do
    test "validates required fields" do
      params = %{"hash" => "0x" <> String.duplicate("0", 40)}
      changeset = Address.changeset(%Address{}, params)
      assert changeset.valid?
    end

    test "rejects invalid hash format" do
      params = %{"hash" => "invalid"}
      changeset = Address.changeset(%Address{}, params)
      refute changeset.valid?
      assert "must be valid Ethereum address" in errors_on(changeset).hash
    end
  end

  describe "fetch_address/1" do
    test "returns address when it exists" do
      address = insert(:address)
      assert {:ok, fetched} = Explorer.Chain.get_address(address.hash)
      assert fetched.hash == address.hash
    end

    test "returns :not_found when address doesn't exist" do
      assert :not_found == Explorer.Chain.get_address("0x" <> String.duplicate("0", 40))
    end
  end
end
```

**Key practices:**
- Use `describe/2` for grouping related tests
- Use factories (`insert(:address)`) not fixtures
- One assertion per test (or logically related)
- Meaningful test names (describe what should happen, not how)
- Test edge cases and error paths

### Integration Tests

```elixir
# test/block_scout_web/api/v2/address_controller_test.exs
defmodule BlockScoutWeb.API.V2.AddressControllerTest do
  use BlockScoutWeb.ConnCase

  setup do
    {:ok, conn: build_conn(), address: insert(:address)}
  end

  test "GET /api/v2/addresses/:hash returns address details", %{address: address} do
    conn = get(conn, "/api/v2/addresses/#{address.hash}")
    assert json_response(conn, 200)["result"]["hash"] == address.hash
  end

  test "GET /api/v2/addresses/invalid_hash returns 422", %{conn: conn} do
    conn = get(conn, "/api/v2/addresses/invalid")
    assert json_response(conn, 422)
  end
end
```

---

## Configuration Patterns

### Runtime Configuration (config/runtime.exs)

```elixir
# config/runtime.exs
config :explorer, Explorer.Repo,
  url: System.get_env("DATABASE_URL") || raise("DATABASE_URL not set"),
  pool_size: String.to_integer(System.get_env("POOL_SIZE", "10")),
  ssl: String.to_existing_atom(System.get_env("DATABASE_SSL", "false"))

config :block_scout_web, BlockScoutWeb.Endpoint,
  url: [
    scheme: System.get_env("ETHEREUM_JSONRPC_HTTP_SCHEME", "http"),
    host: System.get_env("ETHEREUM_JSONRPC_HTTP_HOST"),
    port: String.to_integer(System.get_env("ETHEREUM_JSONRPC_HTTP_PORT", "8545"))
  ]

# Feature flags
enable_graphql = System.get_env("ENABLE_GRAPHQL", "true") == "true"
config :block_scout_web, :graphql_enabled, enable_graphql
```

**Principles:**
- Use `System.get_env/2` with defaults for optional vars
- Use `System.get_env/1` (no default) for required vars + `raise`
- Parse types explicitly (String.to_integer, etc.)
- Group by app + module for clarity

### Application Configuration (per-app config/)

```elixir
# apps/explorer/config/config.exs
config :explorer, :market_data_sources, [
  Explorer.Market.Fetcher.CoinGecko,
  Explorer.Market.Fetcher.CoinMarketCap
]

config :explorer, :smart_contract_verifiers, [
  Explorer.SmartContract.Solidity,
  Explorer.SmartContract.Vyper
]
```

---

## Error Handling

### Result Tuples (not exceptions)

```elixir
# Preferred: Return {:ok, data} or {:error, reason}
def fetch_transaction(hash) do
  case Repo.get(Transaction, hash) do
    nil -> {:error, :not_found}
    tx -> {:ok, tx}
  end
end

# Consumer code
case fetch_transaction(hash) do
  {:ok, tx} -> render_transaction(tx)
  {:error, :not_found} -> send_404(conn)
end
```

### Pattern Matching on Results

```elixir
# Validate + extract in one step
with {:ok, addr} <- Explorer.Chain.get_address(hash),
     {:ok, balance} <- fetch_balance(addr),
     true <- balance > 0 do
  {:ok, {addr, balance}}
else
  :not_found -> {:error, :address_not_found}
  {:error, reason} -> {:error, reason}
  false -> {:error, :zero_balance}
end
```

### Raising Exceptions (rare)

Only raise for **unrecoverable bugs**, not expected failures:

```elixir
# Good: Unrecoverable misconfiguration
def start_link(_) do
  endpoint = System.get_env("ETHEREUM_JSONRPC_HTTP") || raise("ETHEREUM_JSONRPC_HTTP not set")
  # ...
end

# Bad: Expected network error, should return {:error, reason}
def fetch_block(number) do
  case HTTPoison.get(endpoint, ...) do
    {:ok, resp} -> {:ok, parse(resp)}
    {:error, reason} -> raise("Network error: #{reason}")  # WRONG
  end
end
```

---

## Documentation

### Module Documentation

```elixir
defmodule Explorer.Chain.Transaction do
  @moduledoc """
  Represents a transaction on the blockchain.

  ## Fields
  - `hash` — Transaction hash (unique)
  - `nonce` — Account nonce (used for ordering)
  - `from_address_hash` — Sender address
  - `to_address_hash` — Recipient address (nil for contract creation)
  - `value` — ETH transferred (in wei)
  - `gas` — Gas limit for execution
  - `gas_price` — Gas price (wei per unit)
  - `input` — Call data / bytecode for contract creation
  - `status` — Execution status (1 = success, 0 = revert)

  ## Examples

      iex> Explorer.Chain.get_transaction("0xabcd...")
      {:ok, %Explorer.Chain.Transaction{hash: "0xabcd...", status: 1}}

      iex> Explorer.Chain.get_transaction("0xinvalid")
      {:error, :not_found}
  """
end
```

### Function Documentation

```elixir
@doc """
Fetches a transaction by hash with associated data.

## Parameters
- `hash` — 66-character transaction hash (0x-prefixed hex)

## Returns
- `{:ok, tx}` — Transaction with blocks, logs, internal transactions preloaded
- `{:error, :not_found}` — Transaction not indexed yet

## Examples

    iex> Explorer.Chain.get_transaction("0x...")
    {:ok, %Transaction{hash: "0x...", status: 1}}
"""
@spec get_transaction(binary()) :: {:ok, Transaction.t()} | {:error, :not_found}
def get_transaction(hash) do
  # Implementation...
end
```

### Inline Comments

```elixir
def import_blocks(blocks) do
  # Ecto.Multi ensures atomic consistency: all or nothing
  Ecto.Multi.new()
  |> Ecto.Multi.insert_all(:blocks, Block, blocks, conflict_target: :hash)
  |> Ecto.Multi.run(:increment_counter, fn repo, %{blocks: {count, _}} ->
    {:ok, count}  # Return count for downstream use
  end)
  |> Repo.transaction()
end
```

**Avoid:**
- Comments that restate code: `x = 5  # Set x to 5` ❌
- Comments for obvious logic
- Outdated/stale comments

---

## Common Gotchas

### 1. Atom vs String Keys in Maps

```elixir
# JSON from HTTP has string keys
resp = Jason.decode!(body)  # %{"hash" => "0x...", "status" => 1}

# Schema params use atoms
Address.changeset(%Address{}, %{hash: "0x...", status: 1})

# Convert explicitly
params = Jason.decode!(body) |> atomize_keys()
```

### 2. Repo.all() vs Repo.stream()

```elixir
# Small queries: Repo.all returns list
addresses = Repo.all(Address)

# Large queries: Repo.stream returns stream (lazy, memory-efficient)
Repo.stream(Address)
|> Stream.filter(&has_transactions?/1)
|> Enum.into([])
```

### 3. Preload vs Join

```elixir
# Use preload for associations (2 queries, cleaner)
addresses = Repo.preload(addresses, [:smart_contract, :transactions])

# Use join for filtering (1 query, required if WHERE clause depends on association)
from a in Address,
  join: t in Transaction, on: t.from_address_hash == a.hash,
  where: t.status == 1
```

### 4. Timestamps

```elixir
# Ecto inserts NaiveDateTime; always use UTC
schema "blocks" do
  timestamps()  # Uses :inserted_at, :updated_at with :naive_datetime
end

# When querying: DateTime.utc_now() |> DateTime.to_naive()
```

### 5. Decimal for Financial Values

```elixir
# NEVER use floats for ETH/token amounts
field :value, :decimal  # Use Decimal, not :float

# When calculating
Decimal.mult(amount, price)  # Not: amount * price
```

---

## Linting & Formatting

### Credo (Static Analysis)

```bash
mix credo --strict
```

**Common rules:**
- Module names must be documented (`@moduledoc`)
- Function names must be documented (`@doc` for public)
- Avoid `cond` with many branches (use pattern matching instead)
- Avoid nested function definitions
- Avoid single-clause modules

### Elixir Formatter

```bash
mix format
```

Auto-formats:
- Indentation (2 spaces)
- Operator spacing
- Line length (fallback to 98 chars)

**Config:** `.formatter.exs` in each app

---

## Performance Considerations

### Database Queries

```elixir
# BAD: N+1 queries
addresses = Repo.all(Address)
for addr <- addresses do
  txs = Repo.all(from t in Transaction, where: t.from_address_hash == addr.hash)
  # ...
end

# GOOD: Preload
addresses = Repo.preload(Repo.all(Address), :transactions)
```

### Caching

```elixir
# Cache frequently-accessed data
def get_coin_price(symbol) do
  key = {:coin_price, symbol}

  case Con.Cache.get(key) do
    nil ->
      price = fetch_from_api(symbol)
      Con.Cache.put(key, price, ttl: 3600)
      price
    price ->
      price
  end
end
```

### Batch Operations

```elixir
# Use Ecto.Multi for atomic bulk inserts
Ecto.Multi.new()
|> Ecto.Multi.insert_all(:blocks, Block, blocks, on_conflict: :nothing)
|> Ecto.Multi.insert_all(:transactions, Transaction, txs, on_conflict: :nothing)
|> Repo.transaction()
```

