# Simple Bank

A backend service for a simple bank, written in Go with PostgreSQL. It is still early: the project currently has the database schema, migrations, and type-safe Go CRUD queries for accounts, entries, and transfers generated with [sqlc](https://sqlc.dev).

Module path: `github.com/hosseinbanaei/simaplebank`

## Features

- Database schema for accounts, balance entries, and transfers between accounts
- Versioned up/down migrations managed with [golang-migrate](https://github.com/golang-migrate/migrate)
- Go code generated from SQL queries with sqlc (create, get, list, update, delete)
- Makefile targets for running PostgreSQL in Docker and managing the database

## Tech Stack

| Component        | Tool                      |
| ---------------- | ------------------------- |
| Language         | Go 1.27.1                 |
| Database         | PostgreSQL (`postgres:alpine` Docker image) |
| Migrations       | golang-migrate            |
| Query generation | sqlc (v1.31.1)            |
| Task runner      | Make                      |

## Project Structure

```
.
├── go.mod                   # Go module definition
├── Makefile                 # Database and code-generation tasks
├── sqlc.yml                 # sqlc configuration
└── db/
    ├── migration/           # SQL migrations (golang-migrate)
    │   ├── 000001_init_schema.up.sql
    │   └── 000001_init_schema.down.sql
    ├── query/               # SQL queries used by sqlc
    │   ├── account.sql
    │   ├── entry.sql
    │   └── transfer.sql
    └── sqlc/                # Generated Go code (do not edit by hand)
        ├── db.go
        ├── models.go
        ├── account.sql.go
        ├── entry.sql.go
        └── transfer.sql.go
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
| `ListAccounts`   | `account.sql`  | List accounts ordered by `id`, paginated with `limit` / `offset`    |
| `UpdateAccount`  | `account.sql`  | Set an account's `balance` by `id` and return the updated row       |
| `DeleteAccount`  | `account.sql`  | Delete an account by `id`                                           |
| `CreateEntry`    | `entry.sql`    | Insert an entry (`account_id`, `amount`) and return it              |
| `GetEntry`       | `entry.sql`    | Get an entry by `id`                                                |
| `ListEntries`    | `entry.sql`    | List an account's entries ordered by `id`, paginated                |
| `CreateTransfer` | `transfer.sql` | Insert a transfer (`from_account_id`, `to_account_id`, `amount`) and return it |
| `GetTransfer`    | `transfer.sql` | Get a transfer by `id`                                              |
| `ListTransfers`  | `transfer.sql` | List transfers where `from_account_id` or `to_account_id` matches the given values, ordered by `id`, paginated |

These queries are plain database operations. They do not yet enforce business rules such as checking balances or moving money atomically between accounts.

Example usage of the generated code:

```go
queries := db.New(conn) // conn is a *sql.DB or *sql.Tx

account, err := queries.CreateAccount(ctx, db.CreateAccountParams{
	Owner:    "alice",
	Balance:  100,
	Currency: "USD",
})
```

## Current Status

- Schema, migrations, Go module, and CRUD queries for accounts, entries, and transfers are in place.
- There is no application entry point or test suite yet.
