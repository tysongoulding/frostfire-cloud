# Frostfire Architecture & Design Decision Ledger (`questions.md`)

This document preserves the complete record of all `/grill-me` design interviews conducted throughout the Frostfire project. It logs every architectural question asked, all options evaluated, the recommended path, the user's final decision, and the resulting implementation impact.

---

## Table of Contents
1. [Session 1: Production Billing, User Identity & Stripe Token Metering](#session-1-production-billing-user-identity--stripe-token-metering)
2. [Session 2: Local Desktop Permissions (Terminal, Network, Files & Secret Shield)](#session-2-local-desktop-permissions-terminal-network-files--secret-shield)
3. [Session 3: Three-Area System Architecture & Distributed Runtime](#session-3-three-area-system-architecture--distributed-runtime)

---

## Session 1: Production Billing, User Identity & Stripe Token Metering

### Q1.1: Initial User Signup & Stripe Onboarding Flow
* **Question**: How should the initial user signup and Stripe onboarding flow work to establish their unique UUID?
* **Options Presented**:
  1. `(Recommended)` **Web Portal + Stripe Checkout with Magic Link/License Key activation for Tauri**: User signs up on web, pays via Stripe Checkout, and activates the Tauri desktop app via a deep-link (`frostfire://activate?token=...`) or license token tied to their UUID.
  2. **In-App Native Tauri Signup with Browser Checkout**: User registers email/password or OAuth directly inside Tauri, which opens Stripe Checkout in the browser and waits for webhook confirmation to activate.
  3. **Device-First Keypair Genesis**: Tauri generates a unique hardware-backed UUID and keypair on initial launch, then prompts the user to attach a Stripe subscription via a web billing portal.
* **Agent Recommendation**: Option 1 (Cleanest SaaS flow, keeps payment and webhook processing on the server, returns a signed activation token).
* **User Decision**: **Option 1: Web Portal + Stripe Checkout with Magic Link/License Key activation for Tauri**.
* **Implementation Impact**: Added `POST /v1/auth/signup-checkout`, `GET /v1/auth/success` deep-link landing page, and Tauri protocol handler for `frostfire://activate`.

---

### Q1.2: Stripe Billing Structure for License & Token Metering
* **Question**: How should the Stripe billing structure handle the combination of the platform license and LLM token usage?
* **Options Presented**:
  1. `(Recommended)` **Base Monthly Subscription + Metered Usage with Spend Guardrails**: Fixed monthly license fee billed upfront, with token consumption tracked in real-time and billed in arrears via Stripe Metered Billing, backed by configurable hard spend caps to avoid bill shock.
  2. **Base Subscription + Prepaid Token Wallet**: Fixed monthly license fee for desktop access, with tokens purchased in prepaid credit packs like $10 or $25; zero balance triggers an instant hard-lock until replenished.
  3. **Tiered All-Inclusive Subscriptions**: Fixed pricing tiers (e.g. Starter $20/mo with 5M tokens, Pro $50/mo with 20M tokens); usage halts or throttles when the monthly token quota is reached.
  4. **Pure Metered Pay-As-You-Go**: No fixed license fee; users are billed strictly at the end of the month based on exact microVM hours and LLM tokens consumed.
* **Agent Recommendation**: Option 1 (Standard SaaS engineering model; upfront platform license with metered overage protection).
* **User Decision**: **Option 1: Base Monthly Subscription + Metered Usage with Spend Guardrails**.
* **Implementation Impact**: Configured Gateway Stripe integration with a base recurring subscription plus metered price reporting under event name `frostfire_compute_tokens`.

---

### Q1.3: User UUID Authentication & Credential Format
* **Question**: What authentication and credential format should represent the user's unique UUID across Tauri, Gateway, and LLM APIs?
* **Options Presented**:
  1. `(Recommended)` **Cryptographically Signed License JWT**: Web portal issues an Ed25519-signed JWT embedding `user_uuid`, `stripe_customer_id`, and subscription tier; Tauri securely stores it via OS Keystore/DPAPI and passes it as a Bearer token across Gateway, gRPC tunnels, and LLM proxy calls.
  2. **Hashed Gateway API Key**: `ff_live_<token>` generated upon Stripe payment, stored securely in Tauri, and validated against Gateway Redis/Postgres on every cloud call to map back to `user_uuid`.
  3. **Native OAuth2 / OIDC with PKCE**: Desktop app opens browser to login, receives auth code on a local loopback server, and receives access/refresh tokens containing `user_uuid`.
* **Agent Recommendation**: Option 1 (Cryptographic signature enables zero-lookup offline validation while strictly tying all network/LLM requests to the canonical `user_uuid`).
* **User Decision**: **Option 1: Cryptographically Signed License JWT**.
* **Implementation Impact**: Implemented `LicenseAuthority` in `frostfire-gateway` using Ed25519 (`ring::signature`) with claims: `sub`, `email`, `stripe_customer_id`, `tier`, `iat`, `exp`.

---

### Q1.4: License Validity, Offline Usage, and Stripe Lifecycle Events
* **Question**: How should Tauri and the Gateway enforce license validity, offline usage, and Stripe subscription lifecycle events (e.g., cancellations or payment failures)?
* **Options Presented**:
  1. `(Recommended)` **Dual-Tier Grace Policy**: Local desktop features work offline for up to 7-14 days with cached JWT; Cloud Gateway, MicroVM, and LLM proxy require live verification and hard-lock immediately upon Stripe cancellation or payment failure.
  2. **Soft-Degrade to Bring-Your-Own-Key**: If subscription lapses or card fails, cloud microVM and managed proxy lock, but desktop app gracefully degrades to a BYOK mode where the user can enter their own OpenAI/Anthropic keys.
  3. **Strict Online-Only Heartbeat**: Tauri must re-authenticate with the Gateway on every launch; if offline > 24 hours or if Stripe flags delinquency, app locks out until reconnected.
* **Agent Recommendation**: Option 1.
* **User Decision**: **User Write-In**: *"If it's offline it can't make LLM requests, not sure we need this"*.
* **Implementation Impact**: Pragmatic YAGNI simplification. Omitted complex local offline cache countdown timers. Cloud Gateway enforces live online validity and credit balance checks on every LLM/MicroVM request in real-time.

---

### Q1.5: Token Allowances & Monthly Spend Caps
* **Question**: How should token allowances and monthly spend caps be structured for new accounts?
* **Options Presented**:
  1. `(Recommended)` **Included Starter Allowance + Configurable Safety Cap**: Base subscription fee includes a starter token pool (e.g. $10/mo); usage beyond that is billed in arrears up to a user-customizable hard cap (e.g. $50/mo), which immediately halts agent LLM calls when reached.
  2. **Zero Included Allowance + Default Safety Cap**: Base subscription covers platform seat only; all token usage is billed 100% in arrears from $0.00 up to a default $50/mo safety cap.
  3. **Pre-authorized Tier Caps selected at Stripe Checkout**: User picks their max monthly token budget at signup (e.g. $25, $50, $100 cap); Gateway enforces that exact limit each billing cycle.
* **Agent Recommendation**: Option 1 (Delights users by including initial compute out of the box, while protecting against runaway bills).
* **User Decision**: **Option 1: Included Starter Allowance + Configurable Safety Cap**.
* **Implementation Impact**: Webhook provisioning seeds $10.00 in Redis credit storage on account creation and subscription renewal; spend cap enforced via `SpendCapExceeded` errors in Gateway.

---

### Q1.6: Tauri UI Activation State & Live Billing Telemetry
* **Question**: How should Tauri display the activation state, live token usage, and billing controls to the user?
* **Options Presented**:
  1. `(Recommended)` **Dedicated Billing & License Panel in Settings with Header Quota Indicator**: Displays user UUID, active plan, current cycle token spend vs. included allowance/cap, link to Stripe Customer Portal for cards/invoices, and a slider to adjust spend cap; shows activation screen if unauthenticated.
  2. **Minimalist Activation Modal with External Stripe Portal Only**: Tauri only handles activation and shows remaining credit balance; all cap adjustments, payment methods, and invoice history are handled exclusively on the hosted Stripe Customer Portal.
  3. **Per-Turn Cost Breakdown in Chat Stream**: Displays token consumption and dollar cost inline beneath each agent turn, plus a top-level balance counter in the status bar.
* **Agent Recommendation**: Option 1 (Balances high-level visibility with full self-service control inside the app, without cluttering conversational stream).
* **User Decision**: **Option 1: Dedicated Billing & License Panel in Settings with Header Quota Indicator**.
* **Implementation Impact**: Created `BillingSettingsPanel.tsx`, `ActivationModal.tsx`, and wired live usage telemetry into `userStore.ts`.

---

### Q1.7: Hosting of Web Signup, Stripe Webhooks & Billing Endpoints
* **Question**: Where should the Web Signup, Stripe Webhooks, and Billing Portal endpoints be hosted and executed?
* **Options Presented**:
  1. `(Recommended)` **Native Endpoints in frostfire-gateway**: Single unified Rust backend: Gateway hosts `/v1/auth/signup-checkout`, `/v1/webhooks/stripe`, `/v1/billing/portal`, and `/v1/billing/usage` directly alongside Redis metering and the LLM router.
  2. **Serverless AWS Lambda / API Gateway**: Stripe webhooks and signup endpoints run as decoupled Python or TypeScript Lambdas fronted by API Gateway, writing to shared Redis/DynamoDB.
  3. **Dedicated Lightweight Next.js / Node.js Web Portal**: Separate web service deployed on Vercel or AWS ECS for marketing, user registration, and Stripe webhook handling, communicating with the Gateway over internal RPC.
* **Agent Recommendation**: Option 1 (Zero unnecessary infrastructure; keeps metering, router, and auth co-located in high-performance Rust).
* **User Decision**: **Option 1: Native Endpoints in frostfire-gateway**.
* **Implementation Impact**: Built `StripeWebhookHandler` directly into `crates/frostfire-gateway` handling lifecycle events (`checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_succeeded`, `invoice.payment_failed`).

---

### Q1.8: Gateway Account & Webhook Audit Storage
* **Question**: What storage engine should the Gateway use for durable user accounts, Stripe customer mappings, and audit history?
* **Options Presented**:
  1. `(Recommended)` **Redis with AOF + Embedded SQLite Audit Store**: Redis provides sub-millisecond atomic token metering and session lookups; embedded SQLite stores durable user accounts, license keys, and an immutable ledger of all Stripe webhook events.
  2. **Pure Redis with AOF on EBS**: Simplest architecture with zero extra database engines; all account metadata, Stripe customer IDs, spend caps, and token balances live in Redis with fsync disk persistence.
  3. **External Managed AWS Database**: DynamoDB or RDS Aurora Serverless PostgreSQL for user records and billing state, decoupled from the Gateway EC2 instance.
* **Agent Recommendation**: Option 1 (Fast atomic in-memory ops for hot token metering + ACID relational durability for billing events and accounts).
* **User Decision**: **Option 1: Redis with AOF + Embedded SQLite Audit Store**.
* **Implementation Impact**: Implemented `AccountDatabase` using `rusqlite` with tables `accounts`, `stripe_events`, and `license_audit`.

---

### Q1.9: Multi-Device Access & Seat Limits per user_uuid
* **Question**: How should multi-device access and seat limits be handled for an individual user_uuid?
* **Options Presented**:
  1. `(Recommended)` **Shared Account UUID with Multi-Device Allowance**: Users can activate on up to 3 developer workstations (e.g. desktop + laptop); all devices share the canonical `user_uuid`, starter token pool, and monthly spend cap.
  2. **Unlimited Devices per Account**: No device count restrictions; any Tauri installation activated with the user's License JWT operates under their `user_uuid` and draws from the shared token budget.
  3. **Single-Device Hardware Lock**: Each license token is locked to a single machine's hardware fingerprint; activating on a new machine revokes the previous session.
* **Agent Recommendation**: Option 1 (3-device allowance).
* **User Decision**: **Option 2: Unlimited Devices per Account (No device count restrictions; any Tauri installation activated with the user's License JWT operates under their user_uuid and draws from the shared token budget)**.
* **Implementation Impact**: Removed machine-fingerprint lockouts. Clients authenticate with the signed JWT; all instances draw concurrently from the tenant's shared token allocation.

---

### Q1.10: Default Pricing Parameters
* **Question**: What are your preferred default pricing numbers for the base subscription license, included token allowance, and default safety cap?
* **Options Presented**:
  1. `(Recommended)` **Configurable via Environment Variables with $20/mo Base & $10 Included Defaults**: Base fee $20/mo, $10/mo included token allowance, default $50/mo spend cap; fully configurable via `GATEWAY_BASE_PRICE_CENTS`, `GATEWAY_INCLUDED_CREDITS_CENTS`, and `GATEWAY_DEFAULT_SPEND_CAP_CENTS`.
  2. **$29/mo Base License with $15/mo Included Tokens**: Includes $15/mo compute credits; metered overages at provider pass-through + 15% margin, default $100/mo spend cap.
  3. **$49/mo Pro Tier with $25/mo Included Tokens**: Geared towards power engineers with higher default compute budgets and $150/mo default spend cap.
* **Agent Recommendation**: Option 1 (Sensible developer-friendly entry point with dynamic environment variable tuning).
* **User Decision**: **Option 1: Configurable via Environment Variables with $20/mo Base & $10 Included Defaults**.
* **Implementation Impact**: Codified defaults (`DEFAULT_BASE_PRICE_CENTS = 2000`, `DEFAULT_INCLUDED_CREDITS_MICRO_CENTS = 1_000_000_000`, `DEFAULT_SPEND_CAP_MICRO_CENTS = 5_000_000_000`) with environment variable overrides.

---

## Session 2: Local Desktop Permissions (Terminal, Network, Files & Secret Shield)

### Q2.1: Local Host Protection & Granularity Rails
* **Question**: What are other gaps we need to solve? We need permissions for local access (desktop machine) to network, terminal and file access.
* **Architecture Evaluated**:
  - Fine-grained host protection rails guarding the user's local workstation against untrusted agent execution or indirect prompt injection.
* **User Decision**: **Hybrid Scoped Policy with Inviolable Secret Shield**.
* **Implementation Decisions & Rules**:
  1. **Terminal Rails**:
     - *Tier 1 (Auto-Allowed)*: Routine development commands (`git status/diff/log`, `cargo check/test/build/clippy`, `npm test/build/lint`) when executing within the active workspace root.
     - *Tier 2 (HITL Approval Required)*: Arbitrary shell execution and unclassified scripts with a strict **60-second fail-closed timeout**.
     - *Tier 3 (Permanently Blocked)*: Destructive commands (`rm -rf /`, `rmdir /s /q c:\`, `mkfs`, fork bombs).
  2. **Network Governance**:
     - *Tier 1 (Auto-Allowed)*: Loopback (`127.0.0.1`, `localhost`) and trusted developer registries (`github.com`, `crates.io`, `npmjs.org`).
     - *Tier 2 (HITL Approval Required)*: External unlisted web domains, third-party APIs, and webhooks.
     - *Tier 3 (Permanently Blocked)*: Raw unbound sockets (`RawSocketConnect`).
  3. **File Access & The Inviolable Secret Shield**:
     - *Tier 1 (Auto-Allowed)*: Reads/writes inside workspace root.
     - *Tier 2 (HITL Approval Required)*: Operations outside workspace root.
     - *Tier 3 (Permanently Blocked)*: System operating system paths (`/etc/shadow`, `/dev`, Windows `System32`, `/usr/bin`).
     - *Secret Shield*: Any read/mutation touching `.env*`, `credentials.json`, `token.json`, `*.pem`, `id_rsa*`, or `*.key` **always** triggers modal approval (`secret_access`), even if located inside the workspace.
  4. **Session Grants**:
     - Thread-safe session grant table supporting temporary approvals, `grant_session`, and dynamic revocation.
* **Implementation Impact**: Implemented in `application/src-tauri/src/permissions.rs` with 8 dedicated tests in `local_permissions_tests.rs`.

---

## Session 3: Three-Area System Architecture & Distributed Runtime

### Q3.1: Compaction Engine Boundary
* **Question**: Where should the Compaction Engine execution boundary live between Cloud (microVM) and Cloud (services)?
* **Options Presented**:
  1. `(Recommended)` **Hybrid**: MicroVM tracks local token count, but requests compaction from Cloud Gateway when exceeding 75% limit.
  2. **MicroVM-Autonomous**: MicroVM runs its own local compaction logic and calls Fast-Tier model directly through the gateway.
  3. **Centralized**: MicroVM sends raw turn history every turn; Cloud Gateway transparently compresses and manages the context buffer.
* **Agent Recommendation**: Option 1.
* **User Decision**: **Option 1: Hybrid (MicroVM tracks local token count, requests compaction from Cloud Gateway at 75% watermark)**.
* **Implementation Impact**: Built in `crates/frostfire-engine/src/compaction.rs` and `agent_loop.rs` with sliding window fallback.

---

### Q3.2: Desktop Client to MicroVM Transport Boundary
* **Question**: How should the Desktop Client (Area 3) establish its transport connection to the User MicroVM (Area 1)?
* **Options Presented**:
  1. `(Recommended)` **Gateway Reverse Tunnel**: MicroVM initiates an outbound TLS 1.3 tunnel to Cloud Gateway; Desktop client connects to Gateway with auth token (Strict Outbound-Only Ingress).
  2. **Direct WireGuard / Mesh VPN**: Desktop and MicroVM join a private overlay network for direct P2P gRPC and VNC traffic.
  3. **Dual Path**: Cloud Gateway brokers control/RPC commands; Desktop attaches directly to MicroVM's public IP on authenticated ports with IP allowlisting.
* **Agent Recommendation**: Option 1.
* **User Decision**: **Option 1: Gateway Reverse Tunnel (Strict Outbound-Only Ingress)**.
* **Implementation Impact**: Codified in `crates/frostfire-tunnel` and `crates/frostfire-gateway/src/broker.rs` with constant-time token verification (`timingSafeEqual`).

---

### Q3.3: Sub-Agent Workspace & Execution Isolation
* **Question**: How should parallel Sub-Agents (Area 1) isolate their working file state and command execution within the MicroVM?
* **Options Presented**:
  1. `(Recommended)` **Git Worktree + Process Jail**: Sub-agents modify code in ephemeral git worktrees with strict cgroup/namespace sandboxing, merging only on verifier pass.
  2. **File-First Blackboard Only**: Sub-agents operate in a shared repo root but write all intermediate deliverables to isolated `/artifacts/runs/<id>/` directories.
  3. **Full Container Isolation**: Each sub-agent runs inside an isolated rootless container inside the microVM.
* **Agent Recommendation**: Option 1.
* **User Decision**: **Option 1: Git Worktree + Process Jail**.
* **Implementation Impact**: Codified in sub-agent orchestration protocols and workspace management.

---

### Q3.4: WARM Memory Engine inside MicroVM
* **Question**: Which embedded engine should power the WARM Memory subsystem (Area 1) inside the MicroVM?
* **Options Presented**:
  1. `(Recommended)` **SQLite + sqlite-vec + FTS5**: Single embedded `.db` file for relational metadata, hybrid keyword/lexical search, and vector embeddings with zero external processes.
  2. **LanceDB (Embedded Arrow)**: High-performance embedded columnar vector store.
  3. **In-Memory HNSW with JSON Spills**: Transient in-memory vector index persisted to structured JSON/binary snapshots.
* **Agent Recommendation**: Option 1.
* **User Decision**: **Option 1: SQLite + sqlite-vec + FTS5**.
* **Implementation Impact**: Specified in `docs/ARCHITECTURE_SPEC.md` section 1.3 for WARM memory relational schema.

---

### Q3.5: Cloud Gateway Token Metering Enforcement
* **Question**: How should token metering and usage deduction be enforced in the Cloud Gateway (Area 2)?
* **Options Presented**:
  1. `(Recommended)` **Asynchronous Batched Metering**: In-memory/Redis aggregation flushed to Stripe Metered Billing every 60s with local credit cache to enforce zero-latency hard stops.
  2. **Real-time Synchronous Ledger**: Verify and deduct credit balance synchronously before issuing each upstream model call (~100ms latency penalty).
  3. **Post-paid Aggregated Billing**: Stream token events to data warehouse and push daily/monthly rollups to Stripe invoices without real-time enforcement.
* **Agent Recommendation**: Option 1.
* **User Decision**: **Option 1: Asynchronous Batched Metering**.
* **Implementation Impact**: Implemented in `crates/frostfire-gateway/src/metering.rs` and `stripe.rs` with Redis atomic Lua deductions and 60-second Stripe flush loops.

---

### Q3.6: Dual-Stream Display Protocol
* **Question**: How should the Dual-Stream Display Protocol deliver the desktop screen from the MicroVM (Area 1) to the Desktop Client (Area 3)?
* **Options Presented**:
  1. `(Recommended)` **Adaptive Hybrid**: Low-framerate noVNC stream (1-5 FPS) for live background visibility + on-demand high-res lossless PNG snapshots triggered on agent action turns.
  2. **Continuous WebRTC 60 FPS**: High-framerate video stream.
  3. **Pure Snapshot-On-Action**: No live video stream; client requests screen captures only when an agent executes a mouse/keyboard action.
* **Agent Recommendation**: Option 1.
* **User Decision**: **Option 1: Adaptive Hybrid (Low-FPS noVNC + high-res PNG snapshots)**.
* **Implementation Impact**: Implemented dual-stream display viewer in `ScreenView.tsx` with high-res scrot IPC capture.

---

### Q3.7: COLD Memory & Persistent Workspace Sync
* **Question**: How should COLD Memory and persistent workspace state synchronize between MicroVM (Area 1) and Cloud Storage (Area 2)?
* **Options Presented**:
  1. `(Recommended)` **Session Boundary Snapshot**: Pull encrypted bundle on boot; run fast local ext4/SQLite during session; snapshot and push to S3 on commit or idle-shutdown.
  2. **Continuous Event Sync**: Stream modified files and SQLite WAL frames to S3/DynamoDB in near real-time via inotify.
  3. **Persistent EBS Volumes**: Keep persistent EBS volumes attached to tenants instead of cold S3 packaging.
* **Agent Recommendation**: Option 1.
* **User Decision**: **Option 1: Session Boundary Snapshot**.
* **Implementation Impact**: Incorporated into microVM lifecycle and EC2 auto-idle shutdown script (`check-idle-shutdown.sh`).

---

### Q3.8: Desktop Host Security Rails
* **Question**: How should the Desktop Client (Area 3) enforce security guardrails and user approval for host-level actions?
* **Options Presented**:
  1. `(Recommended)` **Tiered Scope & Path Whitelisting**: Auto-permit reads within project workspace; require explicit native prompt for clipboard, outside directories, or host commands with 'Always allow' rules.
  2. **Mode-Based Toggle**: Global switch between 'Supervised' and 'Autonomous'.
  3. **Air-Gapped Sandbox**: Agent has zero direct host access.
* **Agent Recommendation**: Option 1.
* **User Decision**: **Option 1: Tiered Scope & Path Whitelisting**.
* **Implementation Impact**: Expanded in Session 2 into the complete 3-Tier engine with Secret Shield.

---

### Q3.9: Recursive Training & Trajectory Harvesting Privacy
* **Question**: How should the Recursive Training & Trajectory Harvesting Pipeline (Area 2) handle telemetry collection and privacy?
* **Options Presented**:
  1. `(Recommended)` **Opt-in Verifier-Gated Streaming**: Tenant opt-in + client-side secret scrubbing + hard test gate pass; outputs paired (chosen/rejected) DPO datasets in S3 Parquet.
  2. **Local-Only Sovereign Evals**: Trajectories and evaluation datasets remain strictly inside tenant boundary for enterprise private fine-tuning.
  3. **Synthetic Benchmark Pipeline**: Do not harvest user trajectories; generate synthetic tasks and harness mutations offline for fine-tuning open models.
* **Agent Recommendation**: Option 1.
* **User Decision**: **Option 1: Opt-in Verifier-Gated Streaming**.
* **Implementation Impact**: Codified in `docs/ARCHITECTURE_SPEC.md` section 2.5 with Parquet schema and regex secret scrubbers.
