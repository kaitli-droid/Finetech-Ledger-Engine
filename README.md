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

Documentation is at: `https://finetech-ledger.onrender.com/swagger/index.html#`

---

## Key Engineering Concepts Demonstrated

### Double-Entry Ledger

Every transaction produces two or more entries whose amounts must sum to zero.

```go
// Example: Transfer $100 from Alice to Bob 
entries := []models.EntryInput{
    {AccountID: aliceID, Amount: -100},  // Alice loses $100
    {AccountID: bobID, Amount: 100},     // Bob gains $100
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
fintech-ledger/
├── cmd/server/              # HTTP server entry point
├── internal/
│   ├── api/                 # HTTP handlers and request routing
│   ├── db/                  # Database connection and executor interface
│   ├── fx/                  # Foreign exchange conversion and rates
│   ├── ledger/              # Core ledger logic (transactions, audit)
│   ├── models/              # Data structures and types
│   ├── wallet/              # Account management and transfers
│   ├── payments/            # Payment gateway integration (NEW)
│   │   ├── mpesa/          # M-Pesa (Daraja) provider
│   │   ├── bank/           # Bank transfer provider
│   │   ├── cards/          # Card payment provider  
│   │   └── paypal/         # PayPal provider
│   ├── approval/            # Approval workflow engine
│   ├── batch/               # Batch transfer processing
│   ├── consolidation/       # Account consolidation
│   ├── archival/            # GDPR compliance and archival
│   ├── cost-allocation/     # Cost allocation and analytics
│   ├── disputes/            # Dispute resolution (Feature 16)
│   ├── export/              # Data export and reporting
│   ├── limits/              # Transfer limits and controls
│   ├── reconciliation/      # Account reconciliation
│   ├── scheduler/           # Scheduled operations
│   ├── webhooks/            # Webhook management
│   └── many others...       # 16+ features implemented
├── migrations/              # Database schema migrations
└── docker-compose.yml       # PostgreSQL container setup
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
psql -h localhost -U ledger -d ledger -f migrations/002_constraints.sql
psql -h localhost -U ledger -d ledger -f migrations/003_system_accounts.sql
psql -h localhost -U ledger -d ledger -f migrations/004_multi_leg_transactions.sql
....
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
accountID, err := wallet.CreateWalletAccount(ctx, db, userID, "USD")
```

### Simple Transfer

```go
err := wallet.Transfer(ctx, db, fromID, toID, 10000, "USD", "transfer-ref-001")
```

### Transfer with Fees

```go
err := wallet.TransferWithFees(ctx, db, fromID, toID, 10000, 500, feeAccountID, "USD", "transfer-ref-002")
```

### Cross-Currency Transfer

```go
err := wallet.TransferCrossCurrency(ctx, db, fromID, toID, 10000, fxRateID, "fx-transfer-ref-001")
```

### Reverse a Transaction

```go
err := wallet.ReverseTransaction(ctx, db, originalTxID, "reversal-ref-001")
```

### Audit and Compliance

```go
// Rebuild balance from audit trail
finalBalance, inconsistencies, err := ledger.ReplayAuditLog(ctx, db, accountID)

// Verify audit log integrity
isIntegral, issues, err := ledger.VerifyAuditIntegrity(ctx, db)

// Get balance history
trail, err := ledger.GetAuditTrail(ctx, db, accountID, 100)
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

Server runs on `http://localhost:9090` with full router, middleware stack, and JWT authentication.

### API Features

- **45+ HTTP endpoints** covering all features
- **JWT authentication** with scope-based authorization
- **OAuth2 integration** for third-party auth
- **CORS support** for cross-origin requests
- **Rate limiting** (token bucket) to prevent abuse
- **Security headers** to protect against common attacks
- **Request logging** for debugging and auditing
- **Webhook support** for payment provider integrations

### Endpoint Categories

#### Authentication & Authorization (Public)
- **POST /auth/login** - Get JWT token
- **POST /auth/register** - Create new user
- **POST /oauth/initiate** - Start OAuth2 flow
- **POST /oauth/callback** - Complete OAuth2

#### Wallets & Transfers
- **POST /wallets** - Create wallet account
- **GET /wallets/:id** - Get wallet balance
- **POST /transfers** - Execute immediate transfer
- **POST /scheduled-transfers** - Schedule transfer for future
- **POST /batch-transfers** - Create batch job
- **POST /batch-transfers/:id** - Add items to batch
- **POST /batch-transfers/process** - Execute batch

#### Limits & Approvals
- **POST /transfer-limits** - Set transfer limit
- **GET /transfer-limits/:id** - Get limit status
- **POST /approval-requests** - Create approval request
- **PATCH /approve-request/:id** - Submit approval decision

#### Foreign Exchange
- **GET /fx-rates/:pair** - Get conversion rate (e.g., USD-EUR)

#### Data Management
- **POST /exports** - Create export job (CSV, JSON, Excel)
- **GET /exports/:id** - Get export status and download
- **POST /reconciliations** - Start reconciliation process

#### Account Consolidation
- **POST /consolidation-groups** - Create consolidation group
- **POST /consolidation-groups/:id** - Add member to group

#### Cost Allocation
- **POST /profit-centers** - Create profit center
- **POST /cost-transactions** - Record cost transaction

#### Payment Gateways
- **POST /payments/initiate** - Initiate payment (PayPal, M-Pesa, Bank, Card)
- **POST /payments/confirm** - Confirm payment
- **GET /payments/:id** - Check payment status
- **POST /payments/refund** - Refund payment
- **POST /payment-methods/save** - Save payment method
- **GET /payment-methods** - List saved payment methods

#### Webhooks (Provider Integration)
- **POST /webhooks/payments** - Receive payment webhooks with signature verification

#### Dispute Resolution (Feature 16)
- **POST /disputes** - Open dispute
- **GET /disputes/:id** - Get dispute details  
- **PATCH /disputes/update** - Update dispute status
- **POST /disputes/evidence** - Upload evidence file
- **POST /disputes/timeline** - Record timeline event

#### Operational
- **GET /health** - Health check (public, no auth required)

### Response Format

All endpoints return JSON with consistent structure:

**Success Response:**
```json
{
  "data": { /* response payload */ },
  "code": 200
}
```

**Error Response:**
```json
{
  "error": "error description",
  "code": 400
}
```

### Authentication

Protected endpoints require JWT token in Authorization header:

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

Get token via login:
```bash
curl -X POST http://localhost:9090/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"pass"}'
```

### Example Requests

Create wallet:
```bash
TOKEN="<your_jwt_token>"
curl -X POST http://localhost:9090/wallets \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "currency": "USD"
  }'
```

Create transfer:
```bash
curl -X POST http://localhost:9090/transfers \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "from_id": "wallet-uuid-1",
    "to_id": "wallet-uuid-2",
    "amount": 10000,
    "currency": "USD",
    "reference": "transfer-ref-001"
  }'
```

Open dispute:
```bash
curl -X POST http://localhost:9090/disputes \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "transaction_id": "tx-uuid",
    "reason": "unauthorized_transaction",
    "description": "I did not authorize this transfer",
    "reporter_id": "wallet-uuid"
  }'
```

Initiate payment:
```bash
curl -X POST http://localhost:9090/payments/initiate \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 5000,
    "currency": "USD",
    "provider": "paypal",
    "description": "Product purchase"
  }'
```

## License

MIT
