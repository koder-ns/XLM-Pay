# XLMPay 💫

### Non-Custodial Recurring & Pull Payments on Stellar

Soroban Smart Contracts • Recurring Payments • Subscriptions • Payroll Streaming • Non-Custodial Infrastructure

> ⚠️ **This project has not undergone a third-party security audit.** Contract test coverage does not substitute for one. Do not use with real funds in production until an independent audit has been completed. See [Threat Model & Limitations](#️-threat-model--limitations).

XLMPay is an open-source protocol for **recurring and pull-based payments** on the **Stellar** network, built on **Soroban** smart contracts.

It lets a payer authorize a recipient to pull funds on a schedule — a subscription, a salary, rent, an installment plan — **without** giving up custody of their funds and **without** having to manually sign a transaction every single payment cycle.

---

# 🚀 The Problem

Sending a one-time crypto payment is easy: you sign a transaction, it settles, done.

**Recurring** payments are not. On most blockchains, including Stellar today, there is no native way for a merchant or employer to "charge" a wallet the way a credit card network can auto-charge a card. This leaves people with two bad options:

1. **Manually sign every single payment, every cycle.** Works in theory, fails in practice — people forget, get busy, or simply stop. This is a major reason on-chain subscriptions and recurring payroll barely exist today.
2. **Hand custody of funds to a third party** who pays on your behalf automatically. This defeats the entire point of holding your own wallet — you're back to trusting a middleman with your money.

There is currently no good middle option: a way to pre-authorize a *bounded, revocable, automatic* pull of funds while keeping full self-custody.

---

# ⚡ What XLMPay Does

XLMPay introduces a third option: a smart-contract-enforced **payment authorization** that sits between the payer and the recipient.

1. **The payer signs one authorization**, not a payment every cycle. That authorization defines hard limits:
   - maximum amount per pull
   - minimum interval between pulls (e.g., no more than once every 30 days)
   - an expiry date or maximum number of pulls
   - which asset (XLM, USDC, or any Stellar asset)
2. **The authorization is stored in a Soroban contract** — not with XLMPay, not with the recipient. The payer's funds never leave their wallet until a pull actually executes.
3. **`pull()` is executed each billing cycle.** The contract checks the authorization's limits before releasing any funds. If the caller tries to pull more than allowed, more often than allowed, or after expiry/revocation, the contract simply rejects it.
4. **The payer can revoke at any time**, unilaterally, on-chain. No cooperation from the recipient is required, and no further pulls succeed after revocation.
5. **Funds stay in the payer's wallet the entire time** — this is not a custodial pool, an escrow deposit, or a pre-funded balance. The contract holds a *permission*, not money.

In short: **autopay for self-custodial wallets, where the limits are enforced by code instead of trust.**

### Who can call `pull()`?

This matters more than it sounds like it should, so we're stating it plainly: **`pull()` is permissionless.** Any address can submit the call once the contract-enforced interval has elapsed — not just the recipient, and not just XLMPay's backend. The contract independently verifies the authorization's limits (amount, frequency, expiry, revocation status) regardless of who sends the transaction, and funds always route directly **payer → recipient**, never through the caller.

This is a deliberate design choice: it means no single relayer's uptime is a precondition for a recipient getting paid. XLMPay's backend (below) exists to make triggering pulls convenient — tracking billing cycles, submitting the transaction, sending webhooks — but it is not the only way a valid pull can happen. Recipients who want stronger liveness guarantees than "trust XLMPay's server" can run their own relayer, use a third-party keeper network, or call `pull()` directly themselves.

---

# 🏗 How It Works (Architecture)

```text
┌──────────────┐                                   ┌──────────────────┐
│    Payer      │  1. signs ONE authorization        │    Recipient      │
│  (wallet)     │ ───────────────────────────────►   │ (merchant/employer)│
└──────┬───────┘                                    └─────────┬─────────┘
       │                                                       │
       │                                                       │ 2. anyone calls
       │ funds remain in                                       │    pull() once the
       │ payer's wallet                                        │    interval elapses
       │                                                        ▼
       │                                          ┌─────────────────────────────┐
       │                                          │   Soroban Smart Contracts     │
       │                                          │  • Authorization Registry     │
       │                                          │  • Pull Execution Engine      │
       │                                          │  • Limit Enforcement          │
       │                                          │    (amount/frequency/expiry)  │
       │                                          │  • Revocation Registry        │
       └─────────────── 4. revoke any time ───────►                               │
                                                  └─────────────────────────────┘
                                                                 │
                                                  3. on each valid pull, contract
                                                     releases funds directly
                                                     payer → recipient
                                                                 ▼
                                                       Funds settle on Stellar
```

**The contract never custodies funds.** It custodies a *rule*. On every `pull()` call, the contract checks the rule, and if it passes, the transfer happens directly from payer to recipient in the same operation — there's no intermediate holding account, and no dependency on who submitted the call.

---

# 🧑‍🤝‍🧑 Who This Is For

* **On-chain subscription services** — SaaS products, content platforms, or any app that wants "subscribe and forget" instead of manual monthly payments
* **DAOs** — automated contributor salaries / payroll without a treasury admin manually sending funds every cycle
* **Rent and installment payments** — landlords, lenders, or BNPL-style products that need predictable recurring transfers
* **Remittances** — recurring family support payments sent automatically on a schedule
* **Any recurring B2B or B2C payment** where both sides want automation without a custodian in the middle

---

# ⚡ Core Features

## 🔁 One-Time Authorization, Recurring Payments

Sign once, pay automatically on schedule — no manual re-signing every cycle.

## 🛡 Hard On-Chain Limits

Every authorization is bounded by amount, frequency, and expiry, enforced by the contract — not by trusting the recipient's good behavior.

## 🚫 Instant, Unilateral Revocation

The payer can cancel an authorization at any time, without needing the recipient's cooperation or approval. Revocation is checked at the moment `pull()` executes: once a revocation transaction is confirmed on-chain, no subsequent pull can succeed under that authorization, regardless of any pull transaction that may be simultaneously in flight.

## 💰 Fully Non-Custodial

Funds never leave the payer's wallet until a valid pull executes. There is no pooled balance, escrow, or custodian holding funds in between payments.

## 🌐 Permissionless Execution

`pull()` can be called by any address once conditions are met — not only the recipient or XLMPay's own infrastructure. This removes a single relayer's uptime as a point of failure for whether a recipient gets paid on schedule.

## 📊 Streaming Mode (Roadmap)

In addition to discrete periodic pulls, a continuous streaming mode is planned for payroll- and rent-style use cases where funds accrue gradually rather than in lump-sum intervals.

## 🔄 Multi-Asset Support

Authorizations can be denominated in XLM or any Stellar-issued asset (e.g., USDC), since Soroban contracts can interact with the Stellar asset layer directly. See the trustline note in the Threat Model below.

## 🔌 Simple Integration API

Recipients (merchants, payroll systems) integrate via a backend API and SDK without needing to write Soroban contract calls themselves — while retaining the option to call the contract directly or run their own relayer if they prefer not to depend on XLMPay's infrastructure.

---

# ⚠️ Threat Model & Limitations

Being upfront about what can go wrong matters more for payment infrastructure than for most software.

| Risk | Mitigation | Status |
|---|---|---|
| Recipient (or any caller) attempts to pull more than authorized | Contract enforces hard amount/frequency caps at execution time, independent of who submits the call | Core contract responsibility |
| A pull is attempted before the allowed interval has passed | Contract checks last-pull timestamp before releasing funds | Core contract responsibility |
| Payer's balance is insufficient at pull time | Pull fails; no partial-pull or debt is created. **The elapsed interval is still consumed** — a failed pull does not extend a retry window before the next interval opens under current logic | Documented behavior; retry logic without double-consuming the interval is a roadmap item |
| Non-XLM asset authorization exists but payer's trustline is missing or removed before a pull | Pull fails at execution since the transfer cannot settle; no funds move and no debt accrues | Documented behavior; contract does not restore or manage trustlines |
| Authorization continues after payer intends to cancel but forgets/delays | Revocation is instant once submitted, but payer must actively revoke — there's no "pause and resume" yet (tracked in Roadmap, below) | Roadmap item |
| Revocation submitted while a pull transaction is already in flight | Contract checks revocation status atomically at execution time; a pull either fully succeeds under the still-valid authorization or is rejected outright — there is no partial or inconsistent state | Core contract responsibility |
| XLMPay's backend relayer is offline or discontinued | Because `pull()` is permissionless, recipients are not dependent on XLMPay's infrastructure — they can call the contract directly or run/use an alternative relayer | Mitigated by design; self-hosted relayer tooling is a roadmap item |
| Malicious or buggy recipient contract integration | Authorization Registry only grants pull *rights* defined by the payer; it cannot be widened by the recipient or any other caller | Needs third-party audit before production use |
| Smart contract bugs in limit enforcement logic | Test suite covers core paths (see Testing) | **Not yet audited** — do not use with real funds before an independent audit |
| Recipient identity confusion (wrong recipient pulls funds) | Authorizations are tied to a specific recipient address at creation time; funds only ever route to that address regardless of who calls `pull()` | Core contract responsibility |

### Disclaimer

XLMPay is payment **infrastructure**, not a managed payment service. Integrators are responsible for their own security review, key management practices, and any regulatory obligations relevant to their use case (e.g., payroll compliance, consumer protection rules for subscription billing). This project has not undergone a third-party security audit — treat it as early-stage software.

---

# 📁 Monorepo Structure

```text
xlmpay/
│
├── frontend/                  # Payer-facing web app: connect wallet, create/manage authorizations
│
├── backend/                   # Optional relayer/API for recipients: trigger pulls, track billing cycles, webhooks
│
├── contract/                  # Soroban smart contract: authorization, pull execution, revocation
│
├── docs/                      # Architecture, integration guides, contract reference
│
├── .github/                   # CI/CD workflows
│
├── package.json               # Root workspace configuration
├── turbo.json                  # Turborepo config (optional)
├── README.md
└── LICENSE
```

---

# 🖥 Frontend

The payer-facing interface for creating, viewing, and managing payment authorizations.

### Responsibilities

* connect Stellar wallet
* create a new authorization (set amount, frequency, expiry, recipient, asset)
* view active authorizations and their pull history
* revoke an authorization
* view upcoming/expected pulls

### Stack

* Next.js
* React
* TypeScript
* Tailwind CSS
* Stellar SDK

---

# ⚙️ Backend

An **optional convenience layer** used by recipients (merchants, payroll systems, landlords) who don't want to run their own relayer or write Soroban calls directly. Because `pull()` is permissionless on-chain, this service is not a required dependency for the protocol to function — it exists to make integration easier.

### Responsibilities

* track billing cycles and submit `pull()` calls at the right time
* expose a simple REST/GraphQL API so recipients don't need to write Soroban calls directly
* webhook notifications on successful pulls, failed pulls, and revocations
* authorization status lookups (active, expired, revoked)
* basic recipient dashboard data (upcoming pulls, payment history)

### Stack

* Node.js
* Express / NestJS
* PostgreSQL
* Stellar SDK

---

# ⛓ Contract

The trust-minimized core of the system. All enforcement of limits happens here, not in the frontend or backend, and it accepts calls from any address — see [Who can call `pull()`?](#who-can-call-pull) above.

### Responsibilities

* store payment authorizations (payer, recipient, asset, amount cap, frequency, expiry)
* execute `pull()`: validate limits regardless of caller, then transfer funds payer → recipient
* handle revocation, marking an authorization as permanently inactive, checked atomically against any in-flight pull
* emit events for successful pulls, failed pull attempts, and revocations (for backend/frontend to index)

### Stack

* Rust
* Soroban SDK

---

# 📦 Installation & Setup

## Prerequisites

* Node.js 18+
* Rust
* Soroban CLI
* npm / yarn / pnpm

## Clone Repository

```bash
git clone https://github.com/xlmpay/xlmpay.git
cd xlmpay
```

## Install Dependencies

```bash
npm install
```

## Run Frontend

```bash
cd frontend
npm run dev
```

## Run Backend

```bash
cd backend
npm run dev
```

## Build Smart Contract

```bash
cd contract
soroban contract build
```

---

# 🧪 Testing

## Frontend Tests

```bash
cd frontend
npm test
```

## Backend Tests

```bash
cd backend
npm test
```

## Contract Tests

```bash
cd contract
cargo test
```

> Contract test coverage does not constitute a security audit. See [Threat Model & Limitations](#️-threat-model--limitations) before using with real funds.

---

# 🌍 Use Cases

* on-chain SaaS subscriptions
* DAO contributor payroll
* recurring remittances
* rent payments
* installment / BNPL-style loan repayments
* recurring DAO membership dues

---

# 🔄 Future Roadmap

* continuous streaming payments (payroll/rent-style accrual, not just discrete pulls)
* pause/resume for authorizations (instead of only revoke-and-recreate — see the related Threat Model row above)
* retry logic for failed pulls due to insufficient balance, without double-consuming the elapsed interval
* self-hosted relayer tooling / keeper network integrations for recipients who want independence from XLMPay's backend
* multi-recipient / split-payment authorizations
* mobile wallet integration
* developer SDK for faster recipient-side integration
* third-party smart contract security audit

---

# 🤝 Contributing

We welcome contributions from:

* frontend developers
* backend engineers
* Rust / Soroban developers
* security researchers
* technical writers

### Contribution Steps

1. Fork repository
2. Create feature branch
3. Commit changes
4. Open Pull Request

---

# 📚 Documentation

Planned documentation includes:

* contract reference (function signatures, authorization schema)
* recipient integration guide
* frontend wallet-connection guide
* security model and threat model
* API reference for backend endpoints

---

# 📜 License

MIT License
