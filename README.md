# Simple Bank

A backend service for a simple bank, written in Go with PostgreSQL. It is still early: the project currently has the database schema, migrations, and type-safe Go CRUD queries for accounts, entries, and transfers generated with [sqlc](https://sqlc.dev), plus a transactional money transfer between accounts.

Module path: `github.com/hosseinbanaei/simaplebank`

## Features

- Database schema for accounts, balance entries, and transfers between accounts
- Versioned up/down migrations managed with [golang-migrate](https://github.com/golang-migrate/migrate)
- Go code generated from SQL queries with sqlc (create, get, list, update, delete)
- Money transfer that records the transfer, both entries, and both balance updates in one database transaction
- Integration tests for every query, run against a real PostgreSQL database
- Makefile targets for running PostgreSQL in Docker, managing the database, and running tests

## Tech Stack

| Component        | Tool                      |
| ---------------- | ------------------------- |
| Language         | Go 1.27.1                 |
| Database         | PostgreSQL (`postgres:alpine` Docker image) |
| Migrations       | golang-migrate            |
| Query generation | sqlc (v1.31.1)            |
| Database driver  | [lib/pq](https://github.com/lib/pq) |
| Testing          | Go `testing` + [testify](https://github.com/stretchr/testify) |
| Task runner      | Make                      |

## Project Structure

```
.
├── go.mod / go.sum          # Go module definition and dependencies
├── Makefile                 # Database and code-generation tasks
├── sqlc.yml                 # sqlc configuration
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

- check that the source account has enough balance (balances can become negative),
- check that both accounts use the same currency,
- update the two accounts in a consistent order, so concurrent transfers in opposite directions between the same two accounts may deadlock.

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
| `store_test.go`    | `TransferTx` with 5 concurrent transfers in one direction: checks each result and the final balances |

## Current Status

- Schema, migrations, Go module, and CRUD queries for accounts, entries, and transfers are in place.
- `TransferTx` performs a money transfer in a single transaction.
- All queries and `TransferTx` have integration tests.
- There is no application entry point yet.
