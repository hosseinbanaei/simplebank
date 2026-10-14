# Simple Bank

A backend service for a simple bank, written in Go with PostgreSQL. It is still early: the project currently has the database schema, migrations, and type-safe Go CRUD queries for accounts, entries, and transfers generated with [sqlc](https://sqlc.dev), plus a transactional money transfer between accounts and an HTTP JSON API for creating, getting, and listing accounts.

Module path: `github.com/hosseinbanaei/simaplebank`

## Features

- Database schema for accounts, balance entries, and transfers between accounts
- Versioned up/down migrations managed with [golang-migrate](https://github.com/golang-migrate/migrate)
- Go code generated from SQL queries with sqlc (create, get, list, update, delete)
- Money transfer that records the transfer, both entries, and both balance updates in one database transaction
- HTTP JSON API built with [Gin](https://github.com/gin-gonic/gin) for creating, getting, and listing accounts
- Integration tests for every query, run against a real PostgreSQL database
- GitHub Actions workflow that runs the migrations and tests on every push and pull request to `main`
- Makefile targets for running PostgreSQL in Docker, managing the database, running tests, and starting the server

## Tech Stack

| Component        | Tool                      |
| ---------------- | ------------------------- |
| Language         | Go 1.27.1                 |
| Database         | PostgreSQL (`postgres:alpine` Docker image) |
| Migrations       | golang-migrate            |
| Query generation | sqlc (v1.31.1)            |
| HTTP framework   | [Gin](https://github.com/gin-gonic/gin) v1.12.0 |
| Database driver  | [lib/pq](https://github.com/lib/pq) |
| Testing          | Go `testing` + [testify](https://github.com/stretchr/testify) |
| Task runner      | Make                      |
| CI               | GitHub Actions            |

## Project Structure

```
.
├── .github/workflows/
│   └── test.yml             # CI: run migrations and tests
├── main.go                  # Application entry point (HTTP server)
├── go.mod / go.sum          # Go module definition and dependencies
├── Makefile                 # Database, code-generation, test, and server tasks
├── sqlc.yml                 # sqlc configuration
├── api/
│   ├── server.go            # Gin router and route registration
│   └── account.go           # Account handlers
├── util/
│   └── random.go            # Random test data helpers
└── db/
    ├── migration/           # SQL migrations (golang-migrate)
    │   ├── 000001_init_schema.up.sql
    │   └── 000001_init_schema.down.sql
    ├── query/               # SQL queries used by sqlc
    │   ├── account.sql
    │   ├── entry.sql
    │   └── transfer.sql
    └── sqlc/                # Generated Go code (*.sql.go, db.go, models.go) and tests
        ├── db.go
        ├── models.go
        ├── account.sql.go
        ├── entry.sql.go
        ├── transfer.sql.go
        ├── store.go         # Store with transaction support and TransferTx
        ├── main_test.go     # Opens the test database connection
        ├── account_test.go
        ├── entry_test.go
        ├── transfer_test.go
        └── store_test.go
```

## Database Schema

| Table       | Purpose                                         | Columns                                                          |
| ----------- | ----------------------------------------------- | ---------------------------------------------------------------- |
| `accounts`  | Bank accounts                                   | `id`, `owner`, `balance`, `currency`, `created_at`               |
| `entries`   | Changes to an account's balance                 | `id`, `account_id`, `amount` (positive or negative), `created_at` |
| `transfers` | Money moved from one account to another         | `id`, `from_account_id`, `to_account_id`, `amount` (must be positive), `created_at` |

- `entries.account_id`, `transfers.from_account_id`, and `transfers.to_account_id` are foreign keys to `accounts.id`.
- Indexes exist on `accounts.owner`, `entries.account_id`, `transfers.from_account_id`, `transfers.to_account_id`, and the pair `(from_account_id, to_account_id)`.
- Amounts and balances are stored as `bigint`.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/)
- [Go](https://go.dev/dl/) 1.27.1 or later
- [golang-migrate CLI](https://github.com/golang-migrate/migrate/tree/master/cmd/migrate)
- [sqlc](https://docs.sqlc.dev/en/latest/overview/install.html)
- `make`

## Getting Started

1. Start a PostgreSQL container named `psb` on port `5432`:

   ```sh
   make postgres
   ```

2. Create the `simple_bank` database:

   ```sh
   make createdb
   ```

3. Apply the migrations:

   ```sh
   make migrateup
   ```

The Makefile uses the local development credentials `root` / `root`. They are meant only for a local container; do not reuse them anywhere else.

## Makefile Commands

| Command            | Description                                         |
| ------------------ | --------------------------------------------------- |
| `make postgres`    | Run a PostgreSQL container (`psb`) on port 5432     |
| `make createdb`    | Create the `simple_bank` database                   |
| `make dropdb`      | Drop the `simple_bank` database                     |
| `make migrateup`   | Apply all up migrations                             |
| `make migratedown` | Roll back all migrations                            |
| `make sqlc`        | Regenerate Go code from `db/query` with sqlc        |
| `make test`        | Run all tests with verbose output and coverage      |
| `make server`      | Start the HTTP server on port 4000                  |

## HTTP API

Start the database and apply the migrations (see [Getting Started](#getting-started)), then run:

```sh
make server
```

The server listens on `0.0.0.0:4000` and connects to `postgresql://root:root@localhost:5432/simple_bank?sslmode=disable`. Both values are constants in `main.go`.

| Method | Path            | Description                     |
| ------ | --------------- | ------------------------------- |
| `POST` | `/accounts`     | Create an account               |
| `GET`  | `/accounts/:id` | Get an account by `id`          |
| `GET`  | `/accounts`     | List accounts, one page at a time |

### `POST /accounts`

Request body:

| Field      | Type   | Rules                            |
| ---------- | ------ | -------------------------------- |
| `owner`    | string | Required                         |
| `currency` | string | Required; one of `USD`, `EUR`, `CAD` |

New accounts always start with a balance of `0`.

```sh
curl -X POST localhost:4000/accounts -d '{"owner":"alice","currency":"USD"}'
```

```json
{"id":1,"owner":"alice","balance":0,"currency":"USD","created_at":"2026-10-14T09:01:22.395181Z"}
```

### `GET /accounts/:id`

`id` must be an integer of at least `1`.

```sh
curl localhost:4000/accounts/1
```

Returns the account, or `404` if it does not exist.

### `GET /accounts`

Query parameters:

| Parameter   | Rules                          |
| ----------- | ------------------------------ |
| `page_id`   | Required; at least `1`         |
| `page_size` | Required; between `5` and `10` |

```sh
curl 'localhost:4000/accounts?page_id=1&page_size=5'
```

Returns a JSON array of accounts ordered by `id`.

### Responses

| Status | When                                                  |
| ------ | ----------------------------------------------------- |
| `200`  | Success                                               |
| `400`  | The request body, path, or query fails validation     |
| `404`  | `GET /accounts/:id` finds no account                  |
| `500`  | A database error                                      |

Errors are returned as `{"error": "<message>"}`, where the message is the underlying Go error text.

## Database Queries

Queries are written in `db/query/*.sql` and compiled by sqlc into the `db` package in `db/sqlc`. After adding or changing a query, run:

```sh
make sqlc
```

Available queries:

| Query            | File           | Description                                                         |
| ---------------- | -------------- | ------------------------------------------------------------------- |
| `CreateAccount`  | `account.sql`  | Insert an account (`owner`, `balance`, `currency`) and return it    |
| `GetAccount`     | `account.sql`  | Get an account by `id`                                              |
| `GetAccountForUpdate` | `account.sql` | Get an account by `id` and lock its row (`SELECT ... FOR UPDATE`) until the transaction ends |
| `ListAccounts`   | `account.sql`  | List accounts ordered by `id`, paginated with `limit` / `offset`    |
| `UpdateAccount`  | `account.sql`  | Set an account's `balance` by `id` and return the updated row       |
| `AddAccountBalance` | `account.sql` | Add `amount` (positive or negative) to an account's `balance` by `id` and return the updated row |
| `DeleteAccount`  | `account.sql`  | Delete an account by `id`                                           |
| `CreateEntry`    | `entry.sql`    | Insert an entry (`account_id`, `amount`) and return it              |
| `GetEntry`       | `entry.sql`    | Get an entry by `id`                                                |
| `ListEntries`    | `entry.sql`    | List an account's entries ordered by `id`, paginated                |
| `CreateTransfer` | `transfer.sql` | Insert a transfer (`from_account_id`, `to_account_id`, `amount`) and return it |
| `GetTransfer`    | `transfer.sql` | Get a transfer by `id`                                              |
| `ListTransfers`  | `transfer.sql` | List transfers where `from_account_id` or `to_account_id` matches the given values, ordered by `id`, paginated |

These queries are plain database operations and do not enforce business rules on their own. Atomic money transfers are handled by the store described below.

Example usage of the generated code:

```go
queries := db.New(conn) // conn is a *sql.DB or *sql.Tx

account, err := queries.CreateAccount(ctx, db.CreateAccountParams{
	Owner:    "alice",
	Balance:  100,
	Currency: "USD",
})
```

## Transactions

`db/sqlc/store.go` defines `Store`, which embeds `*Queries` (so every query is available on it) and adds transaction support:

- `NewStore(db *sql.DB)` creates a store from a database connection.
- `execTx` runs a function with a `*Queries` bound to a new transaction. It commits if the function succeeds and rolls back if it returns an error.
- `TransferTx` moves `amount` from `FromAccountID` to `ToAccountID` in a single transaction. It:
  1. creates a `transfers` row,
  2. creates an entry of `-amount` for the source account,
  3. creates an entry of `+amount` for the destination account,
  4. subtracts `amount` from the source account's balance,
  5. adds `amount` to the destination account's balance.

  Steps 4 and 5 always update the account with the smaller `id` first (via the `addMoney` helper), whichever direction the money moves. Because every transaction locks the two account rows in the same order, concurrent transfers in opposite directions between the same two accounts do not deadlock.

  It returns a `TransferTxResult` with the transfer, both entries, and both updated accounts.

```go
store := db.NewStore(conn) // conn is a *sql.DB

result, err := store.TransferTx(ctx, db.TransferTxParams{
	FromAccountID: 1,
	ToAccountID:   2,
	Amount:        10,
})
```

Balances are updated with a single `UPDATE ... SET balance = balance + amount` statement (`AddAccountBalance`), so concurrent transfers from the same account do not overwrite each other's changes.

`TransferTx` does not yet:

- check that the source account has enough balance (balances can become negative), or
- check that both accounts use the same currency.

## Testing

The tests in `db/sqlc` are integration tests: they run every query against the `simple_bank` database at `postgresql://root:root@localhost:5432/simple_bank?sslmode=disable`, which is set in `db/sqlc/main_test.go`. Start the database and apply the migrations first (see [Getting Started](#getting-started)), then run:

```sh
make test
```

Tests create accounts, entries, and transfers with random data from the `util` package and do not clean them up afterwards, so use a local development database only.

| Test file          | Covers                                                                  |
| ------------------ | ----------------------------------------------------------------------- |
| `account_test.go`  | Create, get, update, delete, and list accounts                          |
| `entry_test.go`    | Create, get, and list entries for an account                            |
| `transfer_test.go` | Create, get, and list transfers involving an account in either direction |
| `store_test.go`    | `TransferTx` with 5 concurrent transfers in one direction (checks each result and the final balances), and 10 concurrent transfers alternating between both directions (checks there is no deadlock and the balances end unchanged) |

## Continuous Integration

`.github/workflows/test.yml` runs on every push and pull request to `main`. It starts a `postgres:alpine` service with the `simple_bank` database, installs golang-migrate v4.20.1, runs `make migrateup`, and then runs `make test`.

## Current Status

- Schema, migrations, Go module, and CRUD queries for accounts, entries, and transfers are in place.
- `TransferTx` performs a money transfer in a single transaction.
- An HTTP API can create, get, and list accounts.
- All queries and `TransferTx` have integration tests, which run in CI. The HTTP API has no tests yet.
- There are no API endpoints yet for transfers, entries, updating, or deleting accounts.
- The database address and server address are hard-coded in `main.go` and `db/sqlc/main_test.go`.
