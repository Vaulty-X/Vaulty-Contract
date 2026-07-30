# Vaulty Backend

The Vaulty backend powers the platform's off-chain infrastructure, providing secure APIs, authentication, banking integrations, notifications, analytics, and communication with Stellar smart contracts.

Built with **Node.js**, **Express**, and **TypeScript**, the backend acts as the bridge between the frontend, the Stellar network, and third-party financial services, ensuring a secure, scalable, and reliable user experience.

---

# Overview

The backend is responsible for:

* User authentication and authorization
* Wallet management
* Savings vault management
* Transaction processing
* Nigerian bank integrations
* Fiat-to-USDT conversion workflows
* Lending and borrowing services
* Investment management
* Reward and streak calculations
* Notification delivery
* Analytics and reporting
* Communication with Soroban smart contracts

---

# Tech Stack

| Technology  | Purpose                |
| ----------- | ---------------------- |
| Node.js 20  | Runtime                |
| Express 4   | API Framework          |
| TypeScript  | Type Safety            |
| PostgreSQL  | Primary Database       |
| Redis       | Caching & Queues       |
| Prisma ORM  | Database Access        |
| JWT         | Authentication         |
| Stellar SDK | Blockchain Integration |
| BullMQ      | Background Jobs        |
| Docker      | Containerization       |

---

# Features

## Authentication

Provides secure user authentication and account management.

Features:

* User registration
* Login
* JWT authentication
* Refresh tokens
* Password reset
* Email verification
* Role-based access control

---

## Wallet Service

Manages user wallets and blockchain interactions.

Responsibilities:

* Wallet creation
* Wallet lookup
* Balance retrieval
* Transaction signing
* Stellar account synchronization

---

## Savings Service

Handles all savings-related operations.

Supports:

* Create vaults
* Deposit funds
* Withdraw funds
* Goal tracking
* Lock period validation
* Vault history

### Vault API

Base URL: `http://localhost:3000/api/v1/vaults`

All endpoints require authentication via Bearer token.

#### Create Vault

**Endpoint:** `POST /api/v1/vaults`

**Request Body:**
```json
{
  "name": "My Savings Vault",
  "description": "Saving for a new laptop",
  "targetAmount": "5000.00",
  "lockPeriod": 30,
  "assetCode": "USDC",
  "assetIssuer": "G...",
  "contractAddress": "G...",
  "onChainVaultId": "onc_vlt_123",
  "type": "PERSONAL",
  "goalDescription": "New MacBook Pro",
  "idempotencyKey": "idem-create-1"
}
```

**Response (201):**
```json
{
  "success": true,
  "data": {
    "vault": {
      "id": "vault_id",
      "userId": "user_id",
      "name": "My Savings Vault",
      "description": "Saving for a new laptop",
      "targetAmount": "5000.00",
      "currentAmount": "0.00",
      "type": "PERSONAL",
      "status": "ACTIVE",
      "targetDate": null,
      "lockPeriod": 30,
      "interestRate": null,
      "assetCode": "USDC",
      "assetIssuer": null,
      "contractAddress": null,
      "onChainVaultId": null,
      "lockedAt": null,
      "unlocksAt": null,
      "goalDescription": "New MacBook Pro",
      "createdAt": "2024-01-01T00:00:00.000Z",
      "updatedAt": "2024-01-01T00:00:00.000Z"
    }
  }
}
```

**Validation:**
- `name` is required, max 100 chars
- `targetAmount` is a positive decimal string
- `lockPeriod` is optional positive integer (days), max 3650
- `assetCode` defaults to `USDC` if omitted
- `type` defaults to `PERSONAL` if omitted
- `idempotencyKey` is required, max 64 chars

---

#### List Vaults

**Endpoint:** `GET /api/v1/vaults`

**Query Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| page | integer | 1 | Page number |
| limit | integer | 20 | Items per page (max 100) |
| status | enum | optional | ACTIVE, LOCKED, or CLOSED |
| type | enum | optional | PERSONAL, GROUP, or GOAL_ORIENTED |

**Response (200):**
```json
{
  "success": true,
  "vaults": [ /* vault objects */ ],
  "pagination": {
    "total": 3,
    "page": 1,
    "limit": 20,
    "pages": 1
  }
}
```

---

#### Get Vault Detail

**Endpoint:** `GET /api/v1/vaults/:vaultId`

**Response (200):**
```json
{
  "success": true,
  "data": {
    "vault": { /* vault object */ }
  }
}
```

Returns 404 if the vault does not exist or belongs to another user.

---

#### Deposit to Vault

**Endpoint:** `POST /api/v1/vaults/:vaultId/deposit`

**Request Body:**
```json
{
  "amount": "200.00",
  "description": "Monthly savings",
  "idempotencyKey": "idem-dep-1"
}
```

**Response (202 Accepted):**
```json
{
  "success": true,
  "data": {
    "transaction": {
      "id": "vault_txn_id",
      "vaultId": "vault_id",
      "userId": "user_id",
      "type": "DEPOSIT",
      "status": "PENDING",
      "amount": "200.00",
      "fee": "0",
      "description": "Monthly savings",
      "reference": "dep-...",
      "requestedAt": "2024-01-01T00:00:00.000Z"
    }
  }
}
```

**Important:** Deposits are recorded as `PENDING` and do **not** change `currentAmount` until the corresponding on-chain event is `CONFIRMED`. This prevents the backend from falsely claiming a balance before the Stellar network confirms the transaction.

---

#### Withdraw from Vault

**Endpoint:** `POST /api/v1/vaults/:vaultId/withdraw`

**Request Body:**
```json
{
  "amount": "100.00",
  "description": "Emergency withdrawal",
  "idempotencyKey": "idem-wth-1"
}
```

**Response (202 Accepted):**
```json
{
  "success": true,
  "data": {
    "transaction": { /* pending transaction object */ }
  }
}
```

Withdrawals are blocked when:
- The vault is closed
- The vault is locked and `unlocksAt` has not passed
- The requested amount exceeds the **confirmed** `currentAmount`

---

#### Lock Vault

**Endpoint:** `POST /api/v1/vaults/:vaultId/lock`

**Request Body:**
```json
{
  "lockPeriod": 30
}
```

**Response (200):**
```json
{
  "success": true,
  "data": {
    "vault": {
      "id": "vault_id",
      "status": "LOCKED",
      "lockedAt": "2024-01-01T00:00:00.000Z",
      "unlocksAt": "2024-01-31T00:00:00.000Z"
      // ... other fields
    }
  }
}
```

---

#### Unlock Vault

**Endpoint:** `POST /api/v1/vaults/:vaultId/unlock`

Unlocks a vault whose lock period has expired.

**Response (200):**
```json
{
  "success": true,
  "data": {
    "vault": {
      "status": "ACTIVE"
      // ... other fields
    }
  }
}
```

---

#### Close Vault

**Endpoint:** `POST /api/v1/vaults/:vaultId/close`

Permanently closes a vault. The vault must have a zero balance.

**Response (200):**
```json
{
  "success": true,
  "data": { "vault": { "status": "CLOSED" } }
}
```

---

#### Get Vault Transaction History

**Endpoint:** `GET /api/v1/vaults/:vaultId/history`

**Query Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| page | integer | 1 | Page number |
| limit | integer | 20 | Items per page (max 100) |
| status | enum | optional | PENDING, CONFIRMED, FAILED, or CANCELLED |
| type | enum | optional | DEPOSIT, WITHDRAWAL, TRANSFER, or INTEREST |

**Response (200):**
```json
{
  "success": true,
  "transactions": [ /* transaction history items */ ],
  "pagination": {
    "total": 5,
    "page": 1,
    "limit": 20,
    "pages": 1
  }
}
```

---

### Vault Lifecycle and On-Chain Reconciliation

The backend intentionally keeps off-chain vault metadata separate from on-chain confirmation state. A vault's `currentAmount` only reflects amounts from `CONFIRMED` `VaultTransaction` records.

1. **Request:** Client calls deposit or withdraw. Backend creates a `VaultTransaction` with `status: PENDING` and a `VaultEvent` with `status: REQUESTED`.
2. **Submit:** A background job enqueues the vault action for on-chain processing.
3. **Reconcile:** The `vault-reconciliation` worker polls for the on-chain event. If confirmed, it updates the `VaultTransaction` to `CONFIRMED` and mutates `currentAmount`. If failed, it marks the transaction as `FAILED`.
4. **History:** The `/history` endpoint returns stable, paginated records ordered by `requestedAt` descending.

This ensures that no state or balance is declared complete before the Stellar network actually confirms it.

---

## Banking Service

Integrates with payment providers to support local bank transfers.

Responsibilities:

* Receive NGN deposits
* Verify payments
* Convert NGN to USDT
* Transfer assets to Stellar wallets
* Process withdrawals back to bank accounts

---

## Lending Service

Manages lending operations.

Features:

* Supply assets
* Track lending positions
* Interest calculations
* Loan monitoring

---

## Borrowing Service

Supports collateralized borrowing.

Responsibilities:

* Collateral verification
* Borrow limit calculation
* Loan issuance
* Repayment tracking

---

## Investment Service

Handles investment portfolios.

Features:

* Portfolio allocation
* Performance tracking
* Earnings calculation
* Investment history

---

## Rewards Service

Calculates user rewards.

Tracks:

* Saving streaks
* Achievements
* Financial Discipline Score
* Milestone completion

---

## Notification Service

Sends user notifications.

Examples:

* Deposit confirmation
* Streak reminders
* Goal milestones
* Loan due dates
* Reward unlocks

---

## Analytics Service

Provides reporting and platform insights.

Examples:

* User activity
* Savings growth
* Vault performance
* Transaction metrics
* Platform statistics

---

# Folder Structure

```text
backend/
│
├── src/
│   │
│   ├── config/
│   ├── controllers/
│   ├── services/
│   ├── routes/
│   ├── middleware/
│   ├── models/
│   ├── repositories/
│   ├── database/
│   │   ├── prisma/
│   │   └── migrations/
│   │
│   ├── blockchain/
│   │   ├── stellar/
│   │   ├── soroban/
│   │   └── contracts/
│   │
│   ├── integrations/
│   │   ├── banking/
│   │   ├── notifications/
│   │   └── payments/
│   │
│   ├── jobs/
│   ├── queues/
│   ├── utils/
│   ├── types/
│   ├── validators/
│   ├── constants/
│   ├── app.ts
│   └── server.ts
│
├── prisma/
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
├── .env.example
├── docker-compose.yml
├── Dockerfile
├── package.json
├── tsconfig.json
└── README.md
```

---

# API Modules

The backend is organized into independent modules.

* Authentication
* Users
* Wallets
* Savings Vaults
* Transactions
* Banking
* Lending
* Borrowing
* Investments
* Rewards
* Notifications
* Analytics

Each module follows a layered architecture with controllers, services, repositories, and validation.

---

# Deployment Modes

The backend now supports separate API and worker deployment modes.

## API mode

Use this when you want the Express server and health endpoint.

```bash
docker compose up app
# or locally
npm run dev
```

## Worker mode

Use this when you want BullMQ workers to process background jobs such as notifications, streak calculations, emails, and payments.

```bash
docker compose up worker
# or locally
npm run dev -- --worker
```

## Health and readiness

The health endpoint reports the status of Prisma, Redis, and worker bootstrapping:

```bash
curl http://localhost:3000/health
```

A healthy deployment should report `ready: true` and `checks.workers.status: "ok"` when workers have been bootstrapped successfully.

---

# Authentication API

Base URL: `http://localhost:3000/api/v1/auth`

## Register

Create a new user account.

**Endpoint:** `POST /api/v1/auth/register`

**Request Body:**
```json
{
  "email": "user@example.com",
  "password": "SecurePass123",
  "firstName": "John",
  "lastName": "Doe",
  "phoneNumber": "+2348012345678"
}
```

**Response (201):**
```json
{
  "success": true,
  "message": "Registration successful. Please check your email to verify your account.",
  "data": {
    "user": {
      "id": "clxxx",
      "email": "user@example.com",
      "firstName": "John",
      "lastName": "Doe",
      "phoneNumber": "+2348012345678",
      "isEmailVerified": false,
      "role": "USER",
      "createdAt": "2024-01-01T00:00:00.000Z"
    }
  }
}
```

**Validation:**
- Email must be valid and unique (stored trimmed + lowercased)
- Password must be at least 8 characters with uppercase, lowercase, and number
- Phone number is optional; when provided it must be a valid Nigerian number and is stored in E.164 (`+234XXXXXXXXXX`)
- Verification links are delivered by email. Raw verification secrets are never returned in API responses.

---

## Identity Canonicalization

Auth identity fields are normalized before lookup and persistence so case, whitespace, and phone-format variants cannot create duplicate accounts.

| Field | Canonical form | Accepted input examples | Stored as |
|-------|----------------|-------------------------|-----------|
| `email` | Trim whitespace, lowercase | `User@Example.com`, ` user@example.com ` | `user@example.com` |
| `phoneNumber` | Nigerian E.164 | `08012345678`, `2348012345678`, `+2348012345678` | `+2348012345678` |

Normalization runs in:

- Zod auth validators (request transform)
- `AuthService` (defense in depth on register/login/forgot/resend)
- `UserRepository` find/create helpers

### Existing data

Apply the data migration to backfill rows created before this change:

```bash
npx prisma db execute --file prisma/sql/normalize_identity_fields.sql
```

The script lowercases emails, rewrites Nigerian phones to E.164, keeps the oldest row on collisions, tags newer email duplicates with `+dup.<id>`, and clears newer duplicate phone values.

## Login

Authenticate a user and receive tokens.

**Endpoint:** `POST /api/v1/auth/login`

**Request Body:**
```json
{
  "email": "user@example.com",
  "password": "SecurePass123"
}
```

**Response (200):**
```json
{
  "success": true,
  "message": "Login successful",
  "data": {
    "user": {
      "id": "clxxx",
      "email": "user@example.com",
      "firstName": "John",
      "lastName": "Doe",
      "phoneNumber": "+2348012345678",
      "isEmailVerified": false,
      "role": "USER",
      "createdAt": "2024-01-01T00:00:00.000Z",
      "lastLoginAt": "2024-01-01T12:00:00.000Z"
    },
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

---

## Refresh Token

Obtain a new access token using a refresh token.

**Endpoint:** `POST /api/v1/auth/refresh-token`

**Request Body:**
```json
{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Response (200):**
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

---

## Forgot Password

Initiate password reset process.

**Endpoint:** `POST /api/v1/auth/forgot-password`

**Request Body:**
```json
{
  "email": "user@example.com"
}
```

**Response (200):**
```json
{
  "success": true,
  "message": "If the email exists, a reset link has been sent"
}
```

---

## Reset Password

Reset password using a reset token.

**Endpoint:** `POST /api/v1/auth/reset-password`

**Request Body:**
```json
{
  "token": "abc123...",
  "password": "NewSecurePass456"
}
```

**Response (200):**
```json
{
  "success": true,
  "message": "Password has been reset successfully"
}
```

---

## Verify Email

Verify user email using verification token.

**Endpoint:** `POST /api/v1/auth/verify-email`

**Request Body:**
```json
{
  "token": "abc123..."
}
```

**Response (200):**
```json
{
  "success": true,
  "message": "Email has been verified successfully"
}
```

---

## Resend Verification Email

Queue a new verification email for an unverified account.

**Endpoint:** `POST /api/v1/auth/resend-verification-email`

**Request Body:**
```json
{
  "email": "user@example.com"
}
```

**Response (200):**
```json
{
  "success": true,
  "message": "If the email exists and is unverified, a verification link has been sent"
}
```

Existing unused verification tokens are invalidated before a new email is queued.

---

## Get Profile

Get current user profile (requires authentication).

**Endpoint:** `GET /api/v1/auth/profile`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Response (200):**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "clxxx",
      "email": "user@example.com",
      "firstName": "John",
      "lastName": "Doe",
      "phoneNumber": "+2348012345678",
      "isEmailVerified": true,
      "emailVerifiedAt": "2024-01-01T12:00:00.000Z",
      "role": "USER",
      "createdAt": "2024-01-01T00:00:00.000Z",
      "updatedAt": "2024-01-01T12:00:00.000Z",
      "lastLoginAt": "2024-01-01T12:00:00.000Z"
    }
  }
}
```

---

## Error Responses

All endpoints return consistent error responses:

**400 Bad Request:**
```json
{
  "success": false,
  "message": "Validation failed"
}
```

**401 Unauthorized:**
```json
{
  "success": false,
  "message": "Invalid or expired token"
}
```

**404 Not Found:**
```json
{
  "success": false,
  "message": "User not found"
}
```

**409 Conflict:**
```json
{
  "success": false,
  "message": "User with this email already exists"
}
```

**500 Internal Server Error:**
```json
{
  "success": false,
  "message": "Internal server error"
}
```

---

---

# API Workflow

```text
Client
   │
   ▼
REST API
   │
   ▼
Controllers
   │
   ▼
Services
   │
   ├── PostgreSQL
   ├── Redis
   ├── Stellar SDK
   ├── Soroban Contracts
   └── Banking Providers
```

---

# Security

Security is built into every layer of the backend.

Measures include:

* JWT authentication
* Password hashing
* Input validation
* Rate limiting
* CORS protection
* Secure HTTP headers
* Role-based permissions
* Audit logging
* Environment-based secrets
* Request validation

---

# Database

PostgreSQL stores platform data including:

* Users
* Wallets
* Savings vaults
* Transactions
* Goals
* Rewards
* Loans
* Investments
* Notifications

Redis is used for:

* Session caching
* Background jobs
* Queue management
* Temporary data
* Rate limiting
* Outbox event publishing

---

# Transactional Outbox Pattern

The backend implements a transactional outbox pattern to ensure reliable delivery of financial and notification jobs. This pattern guarantees that database state changes and queue enqueuing are atomic, preventing lost emails, duplicate payments, stale streak calculations, or notifications referring to non-existent records.

## How It Works

1. **Write Phase**: Business services write domain changes and an `OutboxEvent` record inside the same Prisma `$transaction`. This ensures the event is persisted atomically with the domain data.
2. **Publish Phase**: A dedicated outbox processor (`outbox.processor.ts`) polls for pending events and publishes them to the correct BullMQ queue.
3. **Idempotency**: Each outbox event is keyed by `aggregateId` + `eventType`, preventing duplicate queue submissions on retries.
4. **Retry with Backoff**: Failed publish attempts are recorded with exponential backoff (`5s`, `10s`, `20s`, `40s`, `80s`), up to a maximum of 5 attempts.
5. **Dead Letter**: Events exceeding the retry limit are moved to a `DEAD_LETTER` terminal state for manual inspection and recovery.

## Event Types

| Event Type | Queue | Description |
|---|---|---|
| `EMAIL_VERIFICATION` | `email` | Sends a verification email on user registration |
| `PASSWORD_RESET` | `email` | Sends a password reset email |
| `EMAIL_RESEND` | `email` | Resends a verification email |
| `PAYMENT_INITIATED` | `payment-processing` | Triggers payment polling after deposit/withdrawal initiation |
| `PAYMENT_INSTRUCTIONS` | `payment-processing` | Triggers payment processing after provider instructions are requested |
| `PAYMENT_STATUS_UPDATE` | `payment-processing` | Triggers status update reconciliation |
| `VAULT_DEPOSIT` | `vault-reconciliation` | Triggers on-chain deposit reconciliation |
| `VAULT_WITHDRAWAL` | `vault-reconciliation` | Triggers on-chain withdrawal reconciliation |
| `VAULT_LOCK` | `vault-reconciliation` | Triggers vault lock reconciliation |
| `VAULT_UNLOCK` | `vault-reconciliation` | Triggers vault unlock reconciliation |
| `VAULT_CLOSE` | `vault-reconciliation` | Triggers vault close reconciliation |
| `NOTIFICATION` | `notifications` | Sends a user notification |
| `STREAK_UPDATE` | `streak-calculation` | Triggers streak recalculation |
| `RECONCILIATION` | `stellar-confirmation` | Triggers Stellar transaction reconciliation |

## Retry Behavior

- **Max attempts**: 5
- **Backoff strategy**: Exponential, starting at 5 seconds and doubling each attempt (capped at 300 seconds)
- **Retry scheduling**: Failed events have a `nextRetryAt` timestamp; the processor only picks up events whose retry time has arrived
- **Terminal failure**: After 5 failed attempts, the event is moved to `DEAD_LETTER` status with a `deadLetterReason`

## Operational Recovery

### Processor Startup

The outbox processor starts automatically when the server boots (in both API and worker modes). It runs as a continuous loop that:

1. Queries for pending events with `nextRetryAt <= now` and `attemptCount < 5`
2. Publishes each event to the appropriate BullMQ queue
3. Marks the event as `PUBLISHED` on success
4. Marks the event as `FAILED` with a retry timestamp on transient failure
5. Marks the event as `DEAD_LETTER` when `attemptCount >= 5`

### Processor Shutdown

On `SIGTERM` or `SIGINT`, the server gracefully shuts down the outbox processor before disconnecting Prisma and Redis. This prevents in-flight publishes from being lost.

### Manual Recovery

To recover stuck events:

```bash
# Reset failed events that are past their retry time back to PENDING
# (run via Prisma Studio or a one-off script)
npx prisma db execute --file prisma/sql/reset_outbox.sql
```

To inspect dead-letter events:

```sql
SELECT id, eventType, aggregateId, attemptCount, deadLetterReason, createdAt
FROM outbox_events
WHERE status = 'DEAD_LETTER'
ORDER BY createdAt DESC;
```

## Monitoring Expectations

- **Outbox lag**: Monitor the count of `PENDING` events with `nextRetryAt <= now`. A growing lag indicates the processor is falling behind.
- **Dead-letter queue**: Monitor the count of `DEAD_LETTER` events. Any non-zero count requires manual investigation.
- **Publish failure rate**: Track the ratio of `FAILED` events to total published events. A spike indicates a downstream queue or connectivity issue.
- **Event age**: Alert on `PENDING` events older than 5 minutes (indicating the processor may be stuck).

## Testing

Unit tests for the outbox repository are in `tests/unit/outbox.repository.test.ts`. Integration tests verifying transactional atomicity and retry behavior are in `tests/integration/outbox.integration.test.ts`.

---

# Running the Project

## Prerequisites

- Node.js 20 LTS
- PostgreSQL 15+
- Redis 7+
- Docker (optional, for containerized setup)

## Install dependencies

A `package-lock.json` is committed alongside `package.json`. Always use it for
reproducible installs:

```bash
# Reproducible install from lockfile (recommended — CI and Docker both use this)
npm ci

# Only run npm install if you are deliberately adding or updating a dependency,
# then commit the updated lockfile immediately.
npm install
```

## Configure environment

```bash
cp .env.example .env
```

Update the environment variables with your database, Stellar, Redis, and third-party integration credentials.

### Configuration requirements by environment

- Development: local defaults are allowed for convenience, but the app still warns if values are malformed. Use `NODE_ENV=development` and the built-in localhost defaults if you are just running the service locally.
- Testnet: set `NODE_ENV=development` or `test`, `STELLAR_NETWORK=testnet`, `STELLAR_HORIZON_URL=https://horizon-testnet.stellar.org`, and provide real values for `DATABASE_URL`, `REDIS_HOST`, `REDIS_PORT`, `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET`, and `CORS_ORIGIN`.
- Production: the app exits immediately if any required secret, URL, or infrastructure setting is missing, left at a localhost/default value, or set to a known insecure placeholder. `JWT_ACCESS_SECRET` and `JWT_REFRESH_SECRET` must be unique, non-placeholder values that are at least 32 characters long. `REDIS_PASSWORD` is required, `CORS_ORIGIN` must be explicit and cannot be `*`, and `STELLAR_NETWORK`/`STELLAR_HORIZON_URL` must match the intended network.

**Required environment variables:**
- `DATABASE_URL` - PostgreSQL connection string
- `REDIS_HOST` - Redis server host
- `REDIS_PORT` - Redis server port
- `REDIS_PASSWORD` - Redis authentication password (required in production)
- `JWT_ACCESS_SECRET` - Secret for access token signing
- `JWT_REFRESH_SECRET` - Secret for refresh token signing
- `CORS_ORIGIN` - Allowed CORS origins (comma-separated list; do not use `*` in production)
- `STELLAR_NETWORK` - Stellar network (testnet/mainnet)
- `STELLAR_HORIZON_URL` - Stellar Horizon server URL

## Database Setup

Generate Prisma client:

```bash
npm run prisma:generate
```

Run database migrations:

```bash
npm run prisma:migrate
```

To open Prisma Studio (database GUI):

```bash
npm run prisma:studio
```

## Run the development server

```bash
npm run dev
```

The server will start on `http://localhost:3000`

## Build the project

```bash
npm run build
```

## Start production

```bash
npm start
```

## Docker Setup

Using Docker Compose (recommended for development):

```bash
docker-compose up -d
```

The Dockerfile uses `npm ci` in both build and production stages, which requires
`package-lock.json` to be present and in sync with `package.json`. The lockfile
is committed to the repository — never delete or gitignore it.

This will start:
- The backend application on port 3000
- PostgreSQL on port 5432
- Redis on port 6379

To view logs:

```bash
docker-compose logs -f app
```

To stop services:

```bash
docker-compose down
```

---

# Testing

Run all tests:

```bash
npm test
```

Run integration tests:

```bash
npm run test:integration
```

Run end-to-end tests:

```bash
npm run test:e2e
```

---

# Development Guidelines

* Use TypeScript throughout the project.
* Keep business logic inside services.
* Keep controllers lightweight.
* Validate all incoming requests.
* Write tests for new features.
* Follow RESTful API conventions.
* Document public endpoints.
* Handle errors consistently.
* Avoid hardcoded values; use configuration and environment variables.

---

# API Error Envelope

All error responses from the Vaulty Backend follow a standard JSON envelope format:

```json
{
  "success": false,
  "message": "Error description message"
}
```

In development environments (`NODE_ENV=development`), the response also includes the error stack trace:

```json
{
  "success": false,
  "message": "Error description message",
  "stack": "Error stack trace..."
}
```

## Validation Errors

When a request payload fails validation, the response HTTP status code is `400 Bad Request` and includes a structured array of validation issues under the `errors` key:

```json
{
  "success": false,
  "message": "Validation failed",
  "errors": [
    {
      "path": ["email"],
      "message": "Invalid email address",
      "code": "invalid_format"
    },
    {
      "path": ["password"],
      "message": "Password must be at least 8 characters",
      "code": "too_small"
    }
  ]
}
```

Each validation error entry contains:
- `path`: An array of strings/numbers indicating the path to the invalid field (e.g., `["email"]`).
- `message`: A human-readable description of the validation failure.
- `code`: A stable, standardized error code representing the validation failure type (e.g., `invalid_type`, `invalid_format`, `too_small`, `too_big`).

---

# Roadmap

### Phase 1

* Authentication
* Wallet management
* Savings vault APIs
* Transaction history
* Stellar integration

### Phase 2

* Nigerian bank integration
* Automated savings
* Rewards engine
* Notification service

### Phase 3

* Lending
* Borrowing
* Investment portfolios
* Analytics dashboard

### Phase 4

* Multi-country banking support
* AI-powered financial insights
* Multi-asset support
* Advanced reporting
* Open developer APIs

---

# Vision

The Vaulty backend is the operational core of the platform, connecting users, banking systems, and the Stellar blockchain into a single, secure ecosystem. Designed with scalability, modularity, and reliability in mind, it enables millions of users to save, invest, lend, and grow their wealth through a seamless financial experience.
