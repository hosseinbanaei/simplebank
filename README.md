# Simple Bank

A backend service for a simple bank, written in Go with PostgreSQL. It is still early: the project currently has the database schema, migrations, and type-safe Go query code generated with [sqlc](https://sqlc.dev).

## Features

- Database schema for accounts, balance entries, and transfers between accounts
- Versioned up/down migrations managed with [golang-migrate](https://github.com/golang-migrate/migrate)
- Go code generated from SQL queries with sqlc
- Makefile targets for running PostgreSQL in Docker and managing the database

## Tech Stack

| Component        | Tool                      |
| ---------------- | ------------------------- |
| Language         | Go                        |
| Database         | PostgreSQL (`postgres:alpine` Docker image) |
| Migrations       | golang-migrate            |
| Query generation | sqlc (v1.31.1)            |
| Task runner      | Make                      |

## Project Structure

```
.
├── Makefile                 # Database and code-generation tasks
├── sqlc.yml                 # sqlc configuration
└── db/
    ├── migration/           # SQL migrations (golang-migrate)
    │   ├── 000001_init_schema.up.sql
    │   └── 000001_init_schema.down.sql
    ├── query/               # SQL queries used by sqlc
    │   └── account.sql
    └── sqlc/                # Generated Go code (do not edit by hand)
        ├── db.go
        ├── models.go
        └── account.sql.go
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
- [Go](https://go.dev/dl/)
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

| Query           | Description                                           |
| --------------- | ----------------------------------------------------- |
| `CreateAccount` | Insert an account (`owner`, `balance`, `currency`) and return the created row |

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

- Schema, migrations, and the `CreateAccount` query are in place.
- There is no Go module (`go.mod`), application entry point, or test suite yet.
