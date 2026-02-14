# StellarRemit

**Cross-Border Remittance Platform for Africa, Powered by Stellar & Soroban**

StellarRemit is an open-source remittance platform that leverages the Stellar blockchain and Soroban smart contracts to enable fast, low-cost, and transparent cross-border money transfers across African corridors. By combining on-chain escrow logic with fiat on/off-ramp orchestration, StellarRemit makes international payments accessible to anyone with a mobile phone.

---

## The Problem

Cross-border remittances in Africa are broken. Traditional corridors between countries like Nigeria, Kenya, Ghana, and South Africa are plagued by high fees (often 7–12%), slow settlement times (1–5 business days), opaque exchange rates, and limited accessibility in rural areas. Billions of dollars flow through these corridors annually, yet the infrastructure serving them hasn't meaningfully improved for decades.

## The Solution

StellarRemit replaces the legacy correspondent banking chain with Stellar's global payment network. Senders initiate transfers through a clean web interface, funds are locked in a Soroban smart contract escrow, converted via on-chain liquidity pools (USDC, NGNX, KESC, and other Stellar-native stablecoins), and delivered to recipients through local payout partners — all within seconds and at a fraction of the traditional cost.

---

## Architecture Overview

StellarRemit follows a monorepo architecture with three core layers:

```
stellar-remit/
├── contracts/          # Soroban smart contracts (Rust)
│   ├── escrow/         # Transfer escrow contract
│   ├── fee-manager/    # Dynamic fee calculation contract
│   ├── compliance/     # On-chain compliance & limits contract
│   └── liquidity/      # Liquidity pool interaction contract
├── apps/
│   ├── web/            # Frontend application (Next.js)
│   └── api/            # Backend API service (NestJS)
├── packages/
│   ├── shared/         # Shared types, constants, utilities
│   ├── stellar-sdk/    # Stellar SDK wrapper & helpers
│   └── config/         # Shared ESLint, TypeScript configs
├── docs/               # Documentation & ADRs
├── scripts/            # Deployment & setup scripts
├── docker-compose.yml
├── turbo.json
└── README.md
```

### Smart Contracts Layer (Rust / Soroban)

The on-chain layer is written in Rust using the Soroban SDK and compiled to WebAssembly (Wasm) for deployment on the Stellar network. It consists of four contracts:

**Escrow Contract** — The core transfer primitive. When a sender initiates a remittance, funds (in USDC or a supported stablecoin) are locked in the escrow contract. The contract enforces time-locked release conditions: funds are released to the recipient's designated payout channel once delivery is confirmed, or automatically refunded to the sender after a configurable timeout period. The escrow supports partial releases and multi-party authorization for high-value transfers.

**Fee Manager Contract** — Calculates and collects transfer fees on-chain. Fees are dynamic, based on the corridor, transfer amount, and current network conditions. The contract distributes collected fees to the platform treasury and liquidity providers according to configurable split ratios. All fee logic is transparent and auditable on-chain.

**Compliance Contract** — Enforces per-user and per-corridor transfer limits, velocity checks (daily/weekly/monthly caps), and sanctions screening flags. This contract works in tandem with the backend's KYC system — a user's compliance tier (verified via the API) determines their on-chain transfer limits. The contract can freeze or flag transfers that exceed thresholds.

**Liquidity Contract** — Interfaces with Stellar DEX and Soroban-based AMMs (like Soroswap) to perform on-chain currency conversion. When a transfer requires conversion between stablecoins (e.g., USDC → NGNX), this contract routes through the best available liquidity path, handling slippage protection and minimum output enforcement.

### Backend API Layer (NestJS / PostgreSQL / TypeORM)

The backend is a NestJS application that serves as the orchestration layer between the frontend, the Stellar network, and external services.

**Core Modules:**

- **Auth Module** — JWT-based authentication with refresh token rotation. Supports email/password, phone/OTP, and social OAuth (Google). Implements role-based access control (RBAC) for users, agents, and admins.
- **User Module** — User profile management, KYC document uploads (integrated with a third-party KYC provider), and compliance tier assignment. KYC status changes trigger on-chain compliance tier updates via the Compliance Contract.
- **Transfer Module** — The central business logic module. Handles transfer creation, validation, exchange rate locking, Soroban contract invocation (escrow creation), payout partner dispatch, status tracking, and webhook processing for delivery confirmation.
- **Wallet Module** — Manages Stellar wallet creation and keypair encryption for custodial users. Also supports non-custodial flows where users connect their own Freighter or LOBSTR wallet. Handles trustline management for supported Stellar assets.
- **Corridor Module** — Manages supported remittance corridors (e.g., NG→KE, GH→ZA), including supported currencies, payout methods (bank transfer, mobile money, cash pickup), fee structures, and partner availability per corridor.
- **Payout Module** — Integrates with local payout partners via their APIs to deliver funds in the recipient's local currency. Handles partner selection, delivery tracking, failure retries, and reconciliation.
- **Notification Module** — Real-time transfer status updates via WebSocket (Socket.IO) and push notifications. Also handles email and SMS notifications for critical events (transfer sent, delivered, failed).
- **Admin Module** — Internal dashboard API for monitoring transfers, managing corridors, reviewing flagged transactions, adjusting fee parameters, and viewing platform analytics.

**Database Schema (PostgreSQL / TypeORM):**

The database tracks users, KYC records, transfers, wallets, corridors, payout partners, fee configurations, audit logs, and notification preferences. All sensitive data (wallet keys, KYC documents) is encrypted at rest. The schema supports soft deletes and maintains a complete audit trail for regulatory compliance.

### Frontend Layer (Next.js / TypeScript / Tailwind CSS)

The frontend is a Next.js 14+ application using the App Router, server components, and server actions where appropriate.

**Key Pages & Features:**

- **Landing Page** — Product overview, supported corridors, fee calculator, and trust indicators.
- **Dashboard** — Transfer history, active transfers with real-time status tracking, and quick-send shortcuts for frequent recipients.
- **Send Flow** — Multi-step transfer creation: select corridor → enter amount (with live exchange rate and fee preview) → add recipient details → review and confirm → sign transaction (custodial or via wallet extension).
- **Recipients** — Saved recipient management with support for bank accounts, mobile money numbers, and cash pickup locations.
- **Wallet** — View Stellar wallet balance across supported assets, fund wallet via on-ramp partners, and manage trustlines.
- **KYC Verification** — Guided document upload flow with real-time status tracking.
- **Admin Dashboard** — Transfer monitoring, corridor management, compliance review queue, and analytics charts.

**Tech Details:**

- State management via React Query (TanStack Query) for server state and Zustand for client state.
- Real-time updates via Socket.IO client for transfer status changes.
- Stellar wallet integration using `@stellar/stellar-sdk` and Freighter API for non-custodial signing.
- Form handling with React Hook Form + Zod validation.
- Responsive design with Tailwind CSS, optimized for mobile-first usage.
- Internationalization (i18n) support for English, French, and Portuguese (covering major African language corridors).

---

## Tech Stack

| Layer | Technology |
|---|---|
| Smart Contracts | Rust, Soroban SDK, WebAssembly (Wasm) |
| Backend API | NestJS, TypeScript, TypeORM, PostgreSQL |
| Frontend | Next.js 14+, React, TypeScript, Tailwind CSS |
| Blockchain | Stellar Network, Soroban, Stellar SDK |
| Auth | JWT, Passport.js, OAuth 2.0 |
| Real-time | Socket.IO (WebSocket) |
| File Storage | Cloudinary (KYC documents) |
| Caching | Redis |
| Testing | Jest (API & Frontend), Soroban SDK testutils (Contracts) |
| CI/CD | GitHub Actions |
| Deployment | Vercel (Frontend), Render / Railway (Backend), Stellar Testnet → Mainnet (Contracts) |

---

## Getting Started

### Prerequisites

- **Node.js** >= 18.x
- **pnpm** >= 8.x (package manager)
- **Rust** >= 1.74 (with `wasm32-unknown-unknown` target)
- **Stellar CLI** (latest)
- **PostgreSQL** >= 15
- **Redis** >= 7
- **Docker** (optional, for containerized development)

### 1. Clone the Repository

```bash
git clone https://github.com/your-org/stellar-remit.git
cd stellar-remit
```

### 2. Install Dependencies

```bash
pnpm install
```

### 3. Environment Setup

Copy the example environment files and fill in your values:

```bash
cp apps/api/.env.example apps/api/.env
cp apps/web/.env.example apps/web/.env
```

**Required environment variables for the API:**

```env
# Database
DATABASE_URL=postgresql://user:password@localhost:5432/stellar_remit

# Redis
REDIS_URL=redis://localhost:6379

# JWT
JWT_SECRET=your-jwt-secret
JWT_REFRESH_SECRET=your-refresh-secret

# Stellar
STELLAR_NETWORK=testnet
STELLAR_RPC_URL=https://soroban-testnet.stellar.org
STELLAR_HORIZON_URL=https://horizon-testnet.stellar.org
STELLAR_NETWORK_PASSPHRASE=Test SDF Network ; September 2015

# Contract IDs (after deployment)
ESCROW_CONTRACT_ID=
FEE_MANAGER_CONTRACT_ID=
COMPLIANCE_CONTRACT_ID=
LIQUIDITY_CONTRACT_ID=

# Platform Stellar Account
PLATFORM_SECRET_KEY=
PLATFORM_PUBLIC_KEY=

# Cloudinary
CLOUDINARY_URL=cloudinary://...

# KYC Provider
KYC_PROVIDER_API_KEY=
KYC_PROVIDER_WEBHOOK_SECRET=
```

**Required environment variables for the Frontend:**

```env
NEXT_PUBLIC_API_URL=http://localhost:4000
NEXT_PUBLIC_STELLAR_NETWORK=testnet
NEXT_PUBLIC_WS_URL=ws://localhost:4000
```

### 4. Database Setup

```bash
# Run migrations
pnpm --filter api migration:run

# Seed initial data (corridors, fee configs, etc.)
pnpm --filter api seed
```

### 5. Build & Deploy Smart Contracts

```bash
# Build all contracts
cd contracts
cargo build --target wasm32-unknown-unknown --release

# Generate a testnet identity
stellar keys generate --global deployer --network testnet

# Deploy contracts to testnet
stellar contract deploy \
  --wasm target/wasm32-unknown-unknown/release/escrow.wasm \
  --source deployer \
  --network testnet \
  --alias escrow_contract

stellar contract deploy \
  --wasm target/wasm32-unknown-unknown/release/fee_manager.wasm \
  --source deployer \
  --network testnet \
  --alias fee_manager_contract

stellar contract deploy \
  --wasm target/wasm32-unknown-unknown/release/compliance.wasm \
  --source deployer \
  --network testnet \
  --alias compliance_contract

stellar contract deploy \
  --wasm target/wasm32-unknown-unknown/release/liquidity.wasm \
  --source deployer \
  --network testnet \
  --alias liquidity_contract
```

After deployment, copy the contract IDs into your API `.env` file.

### 6. Start Development Servers

```bash
# Start everything (API + Frontend + Redis via Docker)
pnpm dev

# Or individually:
pnpm --filter api dev        # Backend on http://localhost:4000
pnpm --filter web dev        # Frontend on http://localhost:3000
```

---

## Smart Contract Details

### Escrow Contract

The escrow contract is the backbone of every transfer. Here's a simplified overview of its interface:

```rust
#![no_std]
use soroban_sdk::{contract, contractimpl, token, Address, Env, Symbol};

#[contract]
pub struct EscrowContract;

#[contractimpl]
impl EscrowContract {
    /// Create a new escrow for a remittance transfer
    pub fn create_escrow(
        env: Env,
        sender: Address,
        recipient: Address,
        token: Address,          // USDC or supported stablecoin
        amount: i128,
        timeout_ledger: u32,     // Auto-refund after this ledger
        transfer_id: Symbol,     // Off-chain transfer reference
    ) -> Symbol;

    /// Release escrowed funds to the recipient (called by platform)
    pub fn release(
        env: Env,
        transfer_id: Symbol,
        platform: Address,       // Must be authorized platform account
    );

    /// Refund escrowed funds to the sender (after timeout or cancellation)
    pub fn refund(
        env: Env,
        transfer_id: Symbol,
    );

    /// Query escrow status
    pub fn get_escrow(env: Env, transfer_id: Symbol) -> EscrowData;
}
```

### Contract Testing

Each contract includes comprehensive unit tests using Soroban's built-in test utilities:

```bash
cd contracts
cargo test
```

Integration tests that simulate full transfer flows across all four contracts are in `contracts/tests/integration/`.

---

## API Documentation

The NestJS API exposes a RESTful interface documented with Swagger/OpenAPI. Once the API is running, visit:

```
http://localhost:4000/api/docs
```

### Key Endpoints

**Auth:**
- `POST /auth/register` — Register a new user
- `POST /auth/login` — Login and receive JWT tokens
- `POST /auth/refresh` — Refresh access token

**Transfers:**
- `POST /transfers` — Create a new transfer
- `GET /transfers` — List user's transfers (with pagination & filters)
- `GET /transfers/:id` — Get transfer details with real-time status
- `POST /transfers/:id/cancel` — Cancel a pending transfer

**Recipients:**
- `POST /recipients` — Save a new recipient
- `GET /recipients` — List saved recipients
- `PUT /recipients/:id` — Update recipient details

**Wallet:**
- `GET /wallet/balance` — Get wallet balances across supported assets
- `POST /wallet/fund` — Initiate wallet funding via on-ramp

**Corridors:**
- `GET /corridors` — List supported corridors with current rates and fees
- `GET /corridors/:id/quote` — Get a transfer quote (amount, fees, exchange rate, delivery estimate)

---

## Supported Corridors (Initial Launch)

| Corridor | Send Currency | Receive Currency | Payout Methods |
|---|---|---|---|
| Nigeria → Kenya | NGN | KES | Bank Transfer, M-Pesa |
| Nigeria → Ghana | NGN | GHS | Bank Transfer, Mobile Money |
| Nigeria → South Africa | NGN | ZAR | Bank Transfer |
| Kenya → Nigeria | KES | NGN | Bank Transfer |
| Ghana → Nigeria | GHS | NGN | Bank Transfer, Mobile Money |
| South Africa → Nigeria | ZAR | NGN | Bank Transfer |

Additional corridors will be added based on demand and payout partner availability.

---

## Contributing

We welcome contributions from the community! StellarRemit is designed to be contributor-friendly, especially for the Stellar Drips Wave program.

### How to Contribute

1. **Browse open issues** — Look for issues labeled `Stellar Wave`, `good first issue`, or `help wanted`.
2. **Fork the repository** and create a feature branch from `develop`.
3. **Follow the coding standards** outlined below.
4. **Write tests** for any new functionality.
5. **Submit a pull request** against the `develop` branch with a clear description of your changes.

### Coding Standards

- **Contracts (Rust):** Follow standard Rust conventions. Run `cargo fmt` and `cargo clippy` before committing. All public functions must have doc comments.
- **Backend (TypeScript/NestJS):** Follow the existing module structure. Use DTOs with class-validator for all inputs. Write unit tests for services and e2e tests for controllers.
- **Frontend (TypeScript/Next.js):** Use functional components with hooks. Follow the existing Tailwind utility patterns. All new components need Storybook stories (when applicable).

### Issue Labels & Complexity

Issues in this repo follow the Drips Wave complexity system:

- **Trivial (100 pts)** — Typo fixes, copy changes, minor bug fixes, documentation updates
- **Medium (150 pts)** — Standard features, involved bug fixes, component implementations
- **High (200 pts)** — Complex features, new contract functions, refactors, new integrations

### Development Workflow

```bash
# Create a feature branch
git checkout -b feat/your-feature-name

# Make your changes, then run checks
pnpm lint                    # Lint all packages
pnpm test                    # Run all tests
cd contracts && cargo test   # Run contract tests

# Commit using conventional commits
git commit -m "feat(transfers): add corridor-based fee calculation"

# Push and open a PR
git push origin feat/your-feature-name
```

---

## License

This project is licensed under the **MIT License**. See the [LICENSE](./LICENSE) file for details.
