# RemitX

> Decentralized cross-border remittance and anchor platform built on the Stellar network and Soroban smart contracts.

[![CI](https://img.shields.io/badge/CI-passing-brightgreen.svg)](https://github.com/PawTrust/RemitX/actions/workflows/ci.yml)
[![Tests](https://img.shields.io/badge/tests-passing-brightgreen.svg)](https://github.com/PawTrust/RemitX)
[![Stellar SEP](https://img.shields.io/badge/Stellar-SEP--1%20%7C%20SEP--10%20%7C%20SEP--12%20%7C%20SEP--31-blue.svg)](https://stellar.org)
[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)
[![Soroban SDK](https://img.shields.io/badge/Soroban_SDK-v21.0.0-orange.svg)](https://soroban.stellar.org)

---

## Project Overview

**RemitX** is an institutional-grade, decentralized cross-border remittance and Stellar Anchor platform. It is engineered to solve the high fees (5–10%), multi-day settlement delays, and counterparty risks inherent in traditional remittance corridors to Africa and emerging markets (such as USD/USDC to NGN, GHS, and KES).

The platform is built on:
- **Stellar Network & Soroban Smart Contracts (Rust):** Provides trustless, non-custodial escrow fund locking, automated timelocked sender refunds, and cryptographic delivery attestations with sub-minute settlement finality using USDC.
- **Node.js & TypeScript API Backend:** Implements standardized Stellar Ecosystem Proposals (**SEP-1**, **SEP-10**, **SEP-12**, and **SEP-31**) alongside REST and Server-Sent Events (SSE) streaming endpoints.
- **React & Vite Frontend (PWA):** Progressive Web App featuring offline transaction queueing via Service Workers and real-time escrow timeline tracking.
- **Rust AML/CFT Engine:** Dedicated high-performance compliance microservice enforcing transaction monitoring rules, structuring detection, and automated **NFIU Suspicious Activity Report (SAR)** schema exports.
- **PostgreSQL 16 & Redis 7:** Event-sourced immutable audit logging, distributed sequence locking (`remitx:seq:<account>`), and job queue execution.
- **HashiCorp Vault & pgBackRest:** Dynamic database credential rotation, AES-256-GCM envelope encryption at rest, and point-in-time disaster recovery archiving.

RemitX is intended for diaspora senders, off-ramp payout agents, liquidity providers, and fintech anchors requiring compliant, non-custodial cross-border payment rails.

---

## Table of Contents

- [Key Features](#key-features)
- [Built With / Technology Stack](#built-with--technology-stack)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Installation & Quick Start](#installation--quick-start)
- [Environment Variables](#environment-variables)
- [Running the Project](#running-the-project)
- [Testing](#testing)
- [API Reference](#api-reference)
- [Smart Contracts](#smart-contracts)
- [Database & Data Storage](#database--data-storage)
- [Security Model](#security-model)
- [Background Services & Workers](#background-services--workers)
- [Deployment & Operations](#deployment--operations)
- [CI/CD Workflows](#cicd-workflows)
- [Contributing](#contributing)
- [License](#license)
- [Support & Links](#support--links)

---

## Key Features

### 🛡️ Smart Contract Escrow
- **Non-Custodial Escrow:** Senders deposit USDC into Soroban contracts without custodial intermediary risk.
- **Arbitration & Dispute Resolution:** Built-in dispute mechanism where an assigned arbitrator can step in to resolve conflicts between sender and beneficiary.
- **Deterministic Refund Timelocks:** Senders can autonomously claim 100% refunds if the off-ramp agent does not confirm delivery before a pre-set expiration timeout.
- **Corridor Fee Engine:** Calculates fees and corridor exchange rates transparently on-chain.

### 🌐 Stellar Anchor Protocols (SEPs)
- **SEP-1 (Discovery):** Dynamic, verified `stellar.toml` metadata host resolution with live origin rewriting.
- **SEP-10 (Web Authentication):** Challenge transaction generation, cryptographic verification, and short-lived JWT session issuance.
- **SEP-12 (KYC Customer API):** Sender and recipient KYC profile lifecycle management, Bank Verification Number (BVN) caching, and Smile Identity verification integration.
- **SEP-31 (Direct Remittances):** Cross-border payment initiation, quote generation, and transaction state queries.

### ⚡ Infrastructure & Observability
- **Horizon SSE Listener:** Persistent stream subscriber with durable PostgreSQL cursor checkpointing to eliminate event loss or duplicate processing.
- **Continuous Reconciliation:** Periodic background worker reconciling on-chain contract states against internal PostgreSQL ledger snapshots.
- **Redis Sequence Locking:** Mutex lock serialization for Stellar transaction submissions per account (`remitx:seq:<account>`), preventing sequence collisions (`txBAD_SEQ`).
- **Real-Time Settlement Streaming:** Server-Sent Events (SSE) endpoint providing live visual timeline updates to the client dashboard.
- **Full-Stack Metrics:** Prometheus metric instrumentation across the API, event listeners, and reconciler, with preconfigured Grafana monitoring dashboards.

---

## Built With / Technology Stack

### Blockchain & Smart Contracts
- **Stellar Network:** Base settlement layer utilizing Horizon REST APIs and Soroban RPC.
- **Soroban SDK (`v21.0.0`):** Smart contract framework for WASM execution on Stellar.
- **Rust (2021 Edition):** Systems programming language used for smart contracts and the AML engine.
- **Stellar CLI (`v28.1.0`):** Contract compilation and testnet deployment toolchain.

### Backend & Microservices
- **Node.js (v20/v22 LTS):** JavaScript runtime.
- **TypeScript (5.x):** Strongly-typed application code.
- **Express.js (4.19+):** REST API and middleware server framework.
- **`@stellar/stellar-sdk` (v12.x/v13.x):** Stellar transaction building, SEP-10 challenge signing, and Horizon client.
- **Zod & JSONSchema:** Request validation and schema parsing.
- **Winston & Prom-Client:** Structured logging and Prometheus metrics generation.

### Frontend
- **React 18 & TypeScript:** UI component architecture.
- **Vite 5:** Frontend build tool and development server.
- **Service Worker Outbox API:** Offline payment queueing and sync.
- **Lucide Icons:** Modern interface iconography.

### Databases, Caching & Security
- **PostgreSQL 16 (Alpine):** Relational database storing checkpoints, idempotency locks, consent records, and event logs.
- **Redis 7 (Alpine) & BullMQ:** Distributed key-value cache, transaction sequence mutexes, and job queue.
- **HashiCorp Vault:** Dynamic PostgreSQL credential leases and secrets management.
- **pgBackRest:** WAL archiving and point-in-time recovery (PITR).

---

## Architecture

```mermaid
flowchart TD
    subgraph Clients["Clients & Presentation"]
        WebApp["PWA / Web Dashboard (React + Vite)"]
        AgentUI["Off-Ramp Agent"]
    end

    subgraph AnchorAPI["RemitX Anchor API (Express / TypeScript)"]
        SEP10["SEP-10 Web Auth (/auth)"]
        SEP12["SEP-12 KYC (/kyc/customer)"]
        SEP31["SEP-31 Remittances (/sep31)"]
        EscrowRoutes["Escrow Management (/api/v1/escrow)"]
        SSEStream["SSE Event Stream (/events)"]
        Webhooks["Off-Ramp Webhooks (/webhooks)"]
    end

    subgraph DataStore["Databases & Secrets"]
        Postgres[(PostgreSQL 16)]
        Redis[(Redis 7)]
        Vault["HashiCorp Vault"]
    end

    subgraph Workers["Background Services"]
        Listener["Horizon Event Listener"]
        Recon["Ledger Reconciliation"]
        Oracle["FX Oracle & Delivery Signer"]
        Relayer["Tx Relayer & Sequence Manager"]
        AML["Rust AML/CFT Engine"]
    end

    subgraph StellarChain["Stellar & Soroban Blockchain"]
        Horizon["Stellar Horizon RPC"]
        Escrow["Soroban Escrow Contract"]
        Remittance["Remittance Fee Contract"]
    end

    WebApp -->|"1. Authenticate"| SEP10
    WebApp -->|"2. Submit KYC"| SEP12
    WebApp -->|"3. Initiate Remittance"| SEP31
    SEP31 -->|"Screen Risk"| AML
    SEP31 -->|"Submit Deposit"| Relayer
    Relayer -->|"Sequence Lock"| Redis
    Relayer -->|"Lock USDC"| Escrow

    AgentUI -->|"4. Payout Fiat & Sign Proof"| Oracle
    Oracle -->|"5. Release to Agent"| Escrow

    Escrow -->|"Emit Contract Events"| Horizon
    Listener -->|"Ingest Events"| Horizon
    Listener -->|"Save Checkpoint & State"| Postgres
    Recon -->|"Compare On-Chain vs DB"| Postgres
    Recon -->|"Query State"| Escrow

    WebApp -.->|"Live Timeline Stream"| SSEStream
    Webhooks -->|"Disbursement Webhook"| AnchorAPI
```

---

## Project Structure

```
remitx/
├── api/                           # RemitX Anchor API service (Node.js/TypeScript)
│   ├── app.ts                     # Express app setup and middleware chain
│   ├── server.ts                  # Server entrypoint and lifecycle handlers
│   ├── config.ts                  # Typed configuration schema and environment validation
│   ├── openapi.yaml               # OpenAPI 3.0 specification
│   ├── routes/                    # Route handlers: sep10, sep12, sep31, escrow, privacy
│   ├── middleware/                # SEP-10 auth guard, rate limiter, prometheus metrics
│   ├── services/                  # Crypto, KYC providers, event stores, audit logging
│   └── __tests__/                 # Jest test suites (unit and adversarial)
├── app/                           # Progressive Web App (React + Vite)
│   ├── src/                       # UI components, payment hooks, views
│   ├── sw/                        # Service Worker outbox & offline background sync
│   └── components/                # Timeline monitoring & offline components
├── contracts/                     # Soroban Smart Contracts (Rust)
│   ├── escrow/                    # Escrow Contract (soroban-sdk v21.0.0)
│   │   ├── src/lib.rs             # Contract entrypoint, state machine, storage keys
│   │   ├── src/migration.rs       # Contract upgrade and migration handlers
│   │   ├── src/reentrancy_tests.rs# Reentrancy and race condition test suite
│   │   └── INVARIANTS.md          # Contract invariant documentation
│   └── remittance/                # Remittance fee engine and corridor calculation
├── services/                      # Background microservices
│   ├── aml/                       # Rule-based AML/CFT engine (Rust)
│   │   ├── src/lib.rs             # Rule evaluators (structuring, velocity, sanctions)
│   │   └── src/sar.rs             # NFIU SAR JSON and CSV export serializers
│   ├── listener/                  # Horizon SSE event listener with cursor tracking
│   ├── oracle/                    # Multi-source FX rate aggregator & delivery signer
│   ├── reconciliation/            # Chain-vs-DB ledger reconciliation engine
│   ├── relayer/                   # Stellar transaction relayer & sequence manager
│   ├── db/                        # Vault credential manager for dynamic Postgres auth
│   └── metrics/                   # Shared Prometheus metric registries
├── db/migrations/                 # Core PostgreSQL migrations
│   ├── 001_checkpoint_store.sql   # Cursor storage for horizon event listeners
│   ├── 002_escrow_events.sql      # Event-sourced immutable escrow state tables
│   ├── 003_ndpa_privacy.sql       # Consent, NDPA audit log, and encryption records
│   └── 004_distributed_idempotency.sql # Distributed idempotency keys and locks
├── infrastructure/                # Production and DR infrastructure configuration
│   ├── backup/                    # pgBackRest and PostgreSQL WAL archiving configs
│   ├── grafana/dashboards/        # Grafana dashboard JSON (remitx-overview.json)
│   ├── prometheus/                # Prometheus alert rules (alerts.yml)
│   └── vault/                     # HashiCorp Vault initialization scripts and policies
├── public/.well-known/            # Public discovery documents
│   └── stellar.toml               # SEP-1 Stellar Anchor Discovery Document
├── scripts/                       # Automation and operational scripts
│   ├── dev-setup.sh               # One-command development bootstrapping script
│   ├── validate-stellar-toml.mjs  # Automated SEP-1 specification validator
│   ├── check-unchecked-arithmetic.sh # Smart contract static analysis tool
│   └── dr/restore-postgres.sh     # Point-in-time PostgreSQL recovery runner
├── load-tests/                    # k6 performance, smoke, and stress testing suites
├── e2e/                           # Playwright end-to-end integration tests
├── docker-compose.dev.yml         # Local development container orchestration
├── Cargo.toml                     # Root Rust workspace manifest
└── README.md                      # Project documentation
```

---

## Prerequisites

To run the complete RemitX stack locally via Docker:

| Requirement | Minimum Version | Installation |
|---|---|---|
| **Docker Engine** | 24.0+ (Compose v2.24+) | [docs.docker.com/get-docker](https://docs.docker.com/get-docker/) |
| **Git** | 2.x | [git-scm.com](https://git-scm.com/) |

For native host-level development (optional):
- **Node.js:** `>= 20.10.0` (Node 22 LTS recommended)
- **Rust Toolchain:** `>= 1.79.0` with `wasm32-unknown-unknown` target
- **Stellar CLI:** `v28.1.0`

---

## Installation & Quick Start

### 1. Clone the Repository
```bash
git clone https://github.com/PawTrust/RemitX.git
cd RemitX
```

### 2. Bootstrap Local Environment
Run the automated bootstrap script:
```bash
bash scripts/dev-setup.sh
```

The script will:
1. Verify Docker and Docker Compose prerequisites.
2. Generate a valid `.env` file from development defaults.
3. Build container images in parallel.
4. Start PostgreSQL 16 and Redis 7 and apply all SQL migrations.
5. Fund a testnet Stellar signing keypair via Friendbot.
6. Start the Anchor API and verify `/health`.

When complete, services are accessible at:
- **Anchor API:** `http://localhost:8000`
- **Health Check:** `http://localhost:8000/health`
- **Stellar TOML (SEP-1):** `http://localhost:8000/.well-known/stellar.toml`
- **API Swagger Docs:** `http://localhost:8000/api/docs`
- **Metrics:** `http://localhost:8000/metrics`
- **PostgreSQL:** `localhost:5432` (user: `remitx`, database: `remitx`)
- **Redis:** `localhost:6379`

### 3. Running Background Services
To start optional background microservices (Oracle, Listener, Reconciliation):
```bash
bash scripts/dev-setup.sh --full
```
- **Reconciliation Admin API:** `http://localhost:8001`
- **Oracle / FX Feed:** `http://localhost:8002`

---

## Environment Variables

Configure environment variables in `.env` (automatically generated on first run):

```env
# ── Core Server & Network ─────────────────────────────────────
PORT=8000
NODE_ENV=development
HOME_DOMAIN=localhost:8000
WEB_AUTH_DOMAIN=localhost:8000
NETWORK_PASSPHRASE="Test SDF Network ; September 2015"
HORIZON_URL=https://horizon-testnet.stellar.org
SOROBAN_RPC_URL=https://soroban-testnet.stellar.org

# ── Stellar Accounts & Contracts ──────────────────────────────
# 56-character Ed25519 secret key starting with 'S'
SEP10_SIGNING_SEED=SDDEVTESTSEEDEXAMPLEFORLOCALDEVELOPMENTONLYNOTFORPROD12345
CONTRACT_ID=CAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

# ── Cryptography & Authentication ────────────────────────────
# 32-byte base64-encoded key for AES-256-GCM encryption
MASTER_ENCRYPTION_KEY=AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=
JWT_SECRET=dev-jwt-secret-replace-in-production-min-32-chars-length
JWT_EXPIRY_SECONDS=86400
CHALLENGE_TIMEOUT_SECONDS=300

# ── Databases & Cache ─────────────────────────────────────────
DATABASE_URL=postgresql://remitx:remitx_dev_password@localhost:5432/remitx
REDIS_URL=redis://localhost:6379

# ── Webhook Secrets (Optional) ────────────────────────────────
PAYSTACK_SECRET_KEY=sk_test_example_secret_key
FLUTTERWAVE_SECRET_HASH=flw_secret_webhook_hash
```

---

## Running the Project

### Docker Operations
```bash
# Start core API, PostgreSQL, and Redis
docker compose -f docker-compose.dev.yml up -d

# Start all microservices (listener, reconciler, oracle)
docker compose -f docker-compose.dev.yml --profile services up -d

# Follow container logs
docker compose -f docker-compose.dev.yml logs -f

# Stop containers (preserves database volumes)
bash scripts/dev-setup.sh --down

# Full reset (stops containers and wipes all volumes)
bash scripts/dev-setup.sh --reset
```

### Native Host Development
```bash
# Run Anchor API in development mode
npm run dev --prefix api

# Run React Frontend
npm run dev --prefix app

# Run AML Compliance Service
cargo run --manifest-path services/aml/Cargo.toml
```

---

## Testing

The codebase includes test suites spanning smart contracts, API routes, compliance rules, static analysis, and performance:

```bash
# 1. Anchor API Unit & Adversarial Tests (Jest)
npm test --prefix api

# 2. Smart Contract Tests (Escrow & Invariants)
docker compose -f docker-compose.dev.yml --profile contracts \
  run --rm contracts bash -c \
  "cd /contracts/src/escrow && cargo test -- --nocapture"

# 3. AML Service Rule & SAR Export Tests (Rust)
cargo test --manifest-path services/aml/Cargo.toml

# 4. SEP-1 Discovery Document Validation
node scripts/validate-stellar-toml.mjs

# 5. Smart Contract Arithmetic Safety Analysis
bash scripts/check-unchecked-arithmetic.sh

# 6. End-to-End Test Suite (Playwright)
npm test --prefix e2e

# 7. Performance & Load Smoke Tests (k6)
k6 run --env BASE_URL=http://localhost:8000 load-tests/scenarios/smoke.js
```

---

## API Reference

The Anchor API adheres to the **OpenAPI 3.0** specification located at `api/openapi.yaml`.

### 1. Discovery & System
- `GET /health` — Check server status, version, and Horizon connectivity.
- `GET /.well-known/stellar.toml` — SEP-1 Anchor discovery configuration.
- `GET /metrics` — Prometheus metrics scrape endpoint.
- `GET /api/docs` — Swagger UI documentation.

### 2. SEP-10 Web Authentication
- `GET /auth?account={stellarAddress}` — Request an unsigned SEP-10 challenge transaction.
- `POST /auth` — Submit signed challenge transaction to receive a signed JWT session token.

### 3. SEP-12 KYC Management
- `GET /kyc/customer?account={stellarAddress}` — Retrieve customer onboarding status and required fields.
- `PUT /kyc/customer` — Register/update customer KYC data (Name, Email, BVN, Address).
- `DELETE /kyc/customer/{id}` — Delete customer record (NDPA compliance).

### 4. SEP-31 Cross-Border Remittances
- `GET /sep31/info` — List supported currency corridors, fees, and required KYC fields.
- `POST /sep31/transactions` — Initiate a cross-border payment transfer.
  ```json
  {
    "amount": "250.00",
    "asset_code": "USDC",
    "sender_id": "cust_sender_01",
    "receiver_id": "cust_receiver_02",
    "fields": {
      "dest": "0123456789",
      "dest_extra": "058"
    }
  }
  ```
- `GET /sep31/transactions/{id}` — Query the current status of a payment transfer.

### 5. Escrow & Event Streaming
- `GET /api/v1/escrow` — Query active escrows filtered by status.
- `POST /api/v1/escrow` — Register new escrow transfer metadata.
- `GET /api/v1/escrow/{id}/events` — **Server-Sent Events (SSE)** real-time state stream.
- `POST /api/v1/escrow/{id}/attest` — Submit oracle delivery attestation for fund release.
- `POST /api/v1/escrow/{id}/refund` — Execute timelocked refund.

---

## Smart Contracts

Smart contracts are implemented in Rust targeting `wasm32-unknown-unknown` with the Soroban SDK.

### Primary Escrow Contract (`contracts/escrow/src/lib.rs`)

```rust
#[contractimpl]
impl EscrowContract {
    /// Create a new escrow
    pub fn create_escrow(
        env: Env, id: String, sender: Address, beneficiary: Address,
        arbitrator: Address, amount: i128, asset: String, timelock: u64,
    ) -> String;

    /// Fund the escrow (Pending -> Funded)
    pub fn fund_escrow(env: Env, id: String, sender: Address);

    /// Release funds to beneficiary (Funded -> Released)
    pub fn release_escrow(env: Env, id: String, beneficiary: Address);

    /// Refund funds to sender (Funded -> Refunded)
    pub fn refund_escrow(env: Env, id: String, sender: Address);

    /// Dispute the escrow (Funded -> Disputed)
    pub fn dispute_escrow(env: Env, id: String, caller: Address);

    /// Resolve dispute (Disputed -> Resolved)
    pub fn resolve_dispute(env: Env, id: String, arbitrator: Address, _release_to_beneficiary: bool);

    /// Get escrow data
    pub fn get_escrow(env: Env, id: String) -> Escrow;

    /// Initialise the contract and set the admin address
    pub fn initialize(env: Env, admin: Address);
    
    /// Apply pending schema migrations after a WASM upgrade
    pub fn migrate(env: Env, admin: Address) -> Result<(), EscrowMigrationError>;
}
```

### Building and Deploying Contracts
```bash
# Build Escrow WASM binary
docker compose -f docker-compose.dev.yml --profile contracts \
  run --rm contracts stellar contract build \
  --manifest-path /contracts/src/escrow/Cargo.toml

# Deploy to Stellar Testnet
docker compose -f docker-compose.dev.yml --profile contracts \
  run --rm contracts stellar contract deploy \
  --wasm /contracts/wasm/escrow.wasm \
  --source "$DEPLOYER_SECRET" \
  --network testnet
```

---

## Database & Data Storage

Database migrations are located in `db/migrations/`:

| Migration | Purpose | Tables |
|---|---|---|
| `001_checkpoint_store.sql` | Cursor tracking for Horizon SSE event listener | `checkpoint_store` |
| `002_escrow_events.sql` | Event-sourced immutable state transition log | `escrow_events`, `escrow_snapshots` |
| `003_ndpa_privacy.sql` | Data privacy log and user consent records | `privacy_audit_log`, `consent_records` |
| `004_distributed_idempotency.sql` | Distributed idempotency locks & payload hashes | `idempotency_keys` |

---

## Security Model

### Implemented Controls
- **Constant-Time Verification:** Key comparison and signature validation use constant-time operations to prevent timing side-channel attacks (`api/routes/transaction.ts`).
- **AES-256-GCM Envelope Encryption:** Sensitive recipient credentials and BVNs are encrypted at rest with encryption keys managed through HashiCorp Vault.
- **Webhook Replay Protection:** Off-ramp webhooks enforce HMAC-SHA512 signature verification with a strict timestamp tolerance window (≤ 300s).
- **Sequence Mutex Locking:** Redis mutex locks serialize Stellar transaction submissions per account, eliminating transaction sequence collisions.

### AML/CFT Compliance Engine (`services/aml`)
- **Structuring Detection:** Identifies multiple transactions structured just below the statutory ₦5,000,000 reporting threshold within rolling time windows.
- **Velocity Screening:** Flags accounts exceeding frequency limits.
- **NFIU SAR Serialization:** Exports Suspicious Activity Reports conforming to the Nigerian Financial Intelligence Unit JSON schema (`docs/compliance/nfiu-sar-schema.json`).

---

## Background Services & Workers

- **Horizon Event Listener (`services/listener`):** Consumes real-time contract events from Stellar Horizon, writing durable state updates to PostgreSQL.
- **Reconciliation Engine (`services/reconciliation`):** Performs continuous invariant checking between on-chain contract state and database ledger records.
- **Oracle Service (`services/oracle`):** Aggregates multi-source exchange rates and signs delivery attestation messages (`REMITX_ATTESTATION|...`).
- **Relayer Service (`services/relayer`):** Submits signed transactions on behalf of users, sponsoring network fees.

---

## Deployment & Operations

### Blue-Green Staging & Production
- **Staging Deployments (`deploy-staging.yml`):** Automatically triggered upon push to `main`, building Docker images, applying PostgreSQL migrations, and running k6 smoke tests.
- **Production Blue-Green Deployment (`deploy-prod.yml`):** Manual promotion workflow swapping Kubernetes service label selectors to eliminate downtime.

### Disaster Recovery & Backups
- **PostgreSQL Archiving:** Configured with `pgBackRest` WAL archiving to S3-compatible buckets (`infrastructure/backup/pgbackrest.conf`).
- **Point-in-Time Recovery:** Automated database restoration script via `bash scripts/dr/restore-postgres.sh`.

---

## CI/CD Workflows

Automated GitHub Actions workflows defined in `.github/workflows/`:

| Workflow | File | Trigger | Description |
|---|---|---|---|
| **CI** | [`ci.yml`](.github/workflows/ci.yml) | Push / PR to `main`, `develop` | Runs linting, type checks, unit tests, and contract compilation in parallel. |
| **Escrow Tests** | [`escrow-tests.yml`](.github/workflows/escrow-tests.yml) | Push / PR touching `contracts/**` | Runs Cargo test suite on Soroban smart contracts. |
| **AML Tests** | [`aml-tests.yml`](.github/workflows/aml-tests.yml) | Push / PR touching `services/aml/**` | Runs Rust unit and SAR schema validation tests. |
| **API Contract** | [`api-contract.yml`](.github/workflows/api-contract.yml) | Push / PR touching `api/**` | Validates OpenAPI specification using Dredd. |
| **Security Scan** | [`security-scan.yml`](.github/workflows/security-scan.yml) | Push / PR / Weekly schedule | Runs Gitleaks secret scanning across git history. |
| **SEP Compliance** | [`sep-compliance.yml`](.github/workflows/sep-compliance.yml) | Push / PR | Validates `stellar.toml` metadata structure. |
| **Load Tests** | [`load-tests.yml`](.github/workflows/load-tests.yml) | Push / PR touching `api/**` | Executes k6 smoke tests against containerized API. |
| **PWA Offline** | [`pwa-offline.yml`](.github/workflows/pwa-offline.yml) | Push / PR touching `app/**` | Runs Playwright tests validating service worker outbox queueing. |
| **DR Drill** | [`dr-drill.yml`](.github/workflows/dr-drill.yml) | Monthly / Manual | Runs automated database disaster recovery restoration drill. |
| **Deploy Staging**| [`deploy-staging.yml`](.github/workflows/deploy-staging.yml) | Push to `main` | Deploys containers to staging and executes smoke verification. |
| **Deploy Prod** | [`deploy-prod.yml`](.github/workflows/deploy-prod.yml) | Manual dispatch | Executes zero-downtime blue-green production deployment. |

---

## Contributing

Contributions are welcome! Please follow our established development standards:

1. Fork the repository and create a feature branch (`git checkout -b feat/corridor-support`).
2. Verify all test suites pass: `npm test --prefix api` and `cargo test --manifest-path services/aml/Cargo.toml`.
3. Check contract arithmetic safety: `bash scripts/check-unchecked-arithmetic.sh`.
4. Validate Stellar TOML discovery: `node scripts/validate-stellar-toml.mjs`.
5. Open a Pull Request adhering to conventional commits (`feat:`, `fix:`, `chore:`).

See [CONTRIBUTING.md](CONTRIBUTING.md) and [SECURITY.md](SECURITY.md) for full contribution guidelines and vulnerability disclosure policies.

---

## License

RemitX is open-source software licensed under the **Apache License 2.0**. See the [LICENSE](LICENSE) file and [CONTRIBUTING.md](CONTRIBUTING.md#license) for details.

---

## Support & Links

- **Repository:** [https://github.com/PawTrust/RemitX](https://github.com/PawTrust/RemitX)
- **Issue Tracker:** [https://github.com/PawTrust/RemitX/issues](https://github.com/PawTrust/RemitX/issues)
- **Stellar Development Foundation:** [https://stellar.org](https://stellar.org)
- **Soroban Documentation:** [https://soroban.stellar.org](https://soroban.stellar.org)
- **Contact:** `dev@remitx.io`
