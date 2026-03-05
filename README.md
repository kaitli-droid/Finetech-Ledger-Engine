# Fintech Ledger Engine 

A production-style double-entry ledger engine written in Go.
This project demonstrates how modern fintech systems implement **transaction integrity, auditability, and concurrency-safe balance management**.

The system models how payment platforms, exchanges, and digital wallets manage financial state safely under high concurrency.

---

## Why This Project Exists

Financial software cannot rely on simple balance updates.
Real systems must guarantee that:

* Every transaction is **balanced**
* No money can be **created or destroyed**
* All changes are **auditable**
* Concurrent requests **cannot corrupt balances**

This project implements those guarantees using **double-entry accounting principles**, **PostgreSQL transactional integrity**, and **carefully designed service boundaries**.

---

## Key Engineering Concepts Demonstrated

### Double-Entry Ledger

Every transaction produces two or more entries whose amounts must sum to zero.

```go
entries := []models.EntryInput{
    {AccountID: fromID, Amount: -10000, Currency: "USD", Description: "transfer out"},
    {AccountID: toID,   Amount:  10000, Currency: "USD", Description: "transfer in"},
}
```

The invariant enforced by the ledger:

```
sum(entries.amount) == 0
```

This ensures that funds cannot be accidentally created or lost.

---

### Transaction Integrity

All ledger operations execute inside a database transaction:

* Row-level locking (`SELECT FOR UPDATE`)
* Idempotent references
* Atomic entry creation
* Rollback safety

This prevents race conditions such as double-spending.

---

### Multi-Currency Transfers

Cross-currency transactions use clearing accounts to maintain balance per currency.

Example FX transaction:

```
User USD account      -100 USD
FX USD clearing       +100 USD

FX EUR clearing        -92 EUR
User EUR account       +92 EUR
```

Each currency remains internally balanced.

---

### Reversal-Based Corrections

Ledger entries are **immutable**.

Incorrect transactions are corrected using **reversal transactions**, ensuring the full audit history remains intact.

---

### Audit Replay

Account balances can be rebuilt entirely from ledger history:

```
SELECT SUM(amount)
FROM ledger_entries
WHERE account_id = ?
```

This makes it possible to verify system integrity at any time.

---

## System Architecture

```
fintech-ledger
│
├── cmd/server
│   └── HTTP server entrypoint
│
├── internal
│   ├── ledger       # double-entry transaction engine
│   ├── wallet       # account transfers
│   ├── fx           # foreign exchange conversion
│   ├── db           # database layer
│   ├── api          # REST API
│   └── models       # domain types
│
└── migrations
    └── PostgreSQL schema
```

The architecture keeps the **ledger core isolated** so that all financial state changes pass through a single invariant-enforcing layer.

---

## Running the Project

Start PostgreSQL with Docker:

```
docker compose up -d
```

Run database migrations:

```
psql -h localhost -U ledger -d ledger -f migrations/001_init.sql
```

Run tests:

```
go test ./...
```

Start the API server:

```
go run cmd/server/main.go
```

---

## Example Operations

Create a wallet:

```go
walletID, err := wallet.CreateWalletAccount(ctx, db, userID, "USD")
```

Transfer funds:

```go
err := wallet.Transfer(ctx, db, fromID, toID, 10000, "USD", "transfer-001")
```

Reverse a transaction:

```go
err := wallet.ReverseTransaction(ctx, db, txID, "reversal-001")
```

---

## Testing Strategy

The project uses invariant-based tests to validate financial correctness:

* Ledger entries always balance
* Currency consistency per transaction
* Idempotent reference behavior
* Concurrency safety
* Audit replay correctness

---

## What This Project Demonstrates

This repository showcases implementation of:

* Financial ledger architecture
* Transaction safety patterns
* Multi-currency accounting
* Concurrency control with SQL
* Audit-grade data modeling

---

## License

MIT
