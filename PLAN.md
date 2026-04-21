# Telegram Groupchat Trading Build Plan

## Overview
- Build a Telegram-first, Solana-first social trading system that turns Telegram groups into structured trading workspaces with role controls, immutable signal history, follower copy execution, and trader scorecards.
- Ship the MVP as a non-custodial, copy-trading-first product using consented trade execution from personal wallets. Shared vaults, pooled treasury custody, and multichain support stay out of scope for the first release.
- Use a polyrepo architecture with four repos: `groupchat-api`, `groupchat-bot`, `groupchat-admin`, and `groupchat-shared`.
- Enforce strict service boundaries: the bot and admin never touch Prisma, BullMQ, Redis, Solana RPC, quote routing, or risk engines directly. They only call the API over HTTP.
- Keep `groupchat-shared` intentionally thin as a published npm package, `@groupchat/shared`, containing only Zod schemas, TypeScript types, and shared constants.
- Treat compliance, execution safety, and scorecard integrity as first-class workstreams, not post-launch hardening.

## Tech Stack
- Runtime: `Node.js 22 LTS`, `TypeScript`
- Repo strategy: polyrepo + thin shared npm package
- `groupchat-api`: `Express`, `Zod`, `Prisma`, `PostgreSQL`, `Redis`, `BullMQ`, `Pino`
- `groupchat-bot`: `grammY`, Telegram Bot API, HTTP client to API
- `groupchat-admin`: `React`, `Vite`, `Tailwind CSS`, HTTP client to API
- `groupchat-shared`: `Zod`, `TypeScript`, package build via `tsup` or `tsc`
- Solana execution inside API only: `@solana/web3.js`, Jupiter primary routing, fallback quote providers, dedicated RPC providers
- Market and risk data inside API only: `Pyth` and/or `Birdeye`, `RugCheck`
- Testing: `Vitest`, `Supertest`, `Playwright`
- Infra: `Docker Compose` in `groupchat-api`, managed Postgres/Redis, dedicated Solana RPC

## Dependency Notes
- `groupchat-api` owns all business logic, persistence, queues, Solana adapters, token screening, compliance logic, and scorecard computation.
- `groupchat-bot` and `groupchat-admin` consume the API through versioned HTTP endpoints and import DTOs from `@groupchat/shared`.
- `@groupchat/shared` exports only request/response schemas, UI-safe types, and shared constants. It must not contain business rules, Prisma models, env/config code, Solana clients, logging, or queue logic.
- Prisma schema, migrations, BullMQ workers, `RpcProvider`, `QuoteProvider`, and risk-screening integrations all live only inside `groupchat-api`.
- Wallet import of private keys or seed phrases is removed from MVP; supported flows are deep-link signing, wallet-signature auth, and read-only wallet linking.
- Dedicated RPC is required from day one; public Solana RPC must never be used on execution paths.
- Version `@groupchat/shared` explicitly and release it before API, bot, or admin changes that depend on new contracts. Pin exact versions until release cadence stabilizes.

## Project Structure
```text
groupchat-api/          # Express API + workers + Prisma + BullMQ jobs
  src/
  prisma/
  infra/
  docs/
  tests/

groupchat-bot/          # grammY bot only, calls API over HTTP
  src/
  tests/

groupchat-admin/        # React + Vite admin app, calls API over HTTP
  src/
  tests/

groupchat-shared/       # published npm package (@groupchat/shared)
  src/
    schemas/            # Zod DTOs only
    types/              # shared TypeScript types
    constants/          # shared constants
  tests/
```

- Put compliance and architecture docs in `groupchat-api/docs/` because the API is the system of record for policy, execution, and launch gates.
- If custom vault contracts are revisited after MVP, create a separate `groupchat-programs/` repo instead of extending these four repos.

## Core Features
- Telegram DM onboarding with internal profile creation, ToS acceptance, geo access checks, and wallet linking.
- Group registration, owner/admin roles, approved trader roles, and immutable audit logs for all privileged actions.
- Structured signals with asset, side, entry, stop, targets, thesis, confidence, posted market price snapshot, and versioned edit history.
- Follower opt-in copy settings with sizing rules, slippage limits, token safety checks, delay buffer, instant pause, and per-trade preview cards.
- One-tap trade confirmation from Telegram after a risk-screened preview rather than blind background execution.
- Leader scorecards with realized PnL, unrealized drawdown, max open drawdown, copied volume, signal sample minimums, and leaderboard eligibility rules.
- Token screening, block/allow policies, moderation controls, fee records, and emergency pause tools.

## Compliance & Safety Controls
- Add `groupchat-api/docs/legal-risk-matrix.md` in the first milestone with jurisdictional risk notes, launch gates, and high-risk geo handling.
- Add mandatory ToS acceptance during onboarding and store acceptance in `AuditEvent` with timestamp and terms version.
- Implement geo-block middleware for restricted jurisdictions using IP geolocation and phone prefix where available.
- Frame all user-facing signal language as trade ideas and execution tools, never as guaranteed advice or managed portfolios.
- Add `groupchat-api/docs/wallet-threat-model.md` and remove private-key or seed import from scope.
- Add token risk screening before follower execution and show risk flags directly in the Telegram UX.
- Require an external crypto-specialized legal review before any public launch.

## Interfaces & Data Model Changes
- `User` uses an internal UUID as primary identity; Telegram account links are secondary and re-linkable.
- Add `UserTelegramAccount` to support multiple Telegram links and account recovery.
- Add `TosAcceptance`, `GeoAccessDecision`, `TokenPolicy`, and `TokenRiskCheck` tables.
- Extend `Signal` with `postedPrice`, `priceSource`, `validUntil`, `statusReason`, and immutable revision metadata.
- Extend `TradeExecution` with lifecycle states: `PREPARED`, `SIGNED`, `SUBMITTED`, `CONFIRMED`, `RECONCILED`, plus `failedReason`, `quoteProvider`, and `rpcEndpoint`.
- Keep Prisma models private to `groupchat-api`; do not expose DB models through `@groupchat/shared`.
- Export only API-facing DTOs through `@groupchat/shared`, such as onboarding payloads, signal creation payloads, scorecard responses, execution preview responses, and token risk summaries.

## Implementation
- Build `groupchat-api` around domain modules: `auth`, `users`, `wallets`, `groups`, `roles`, `signals`, `copy`, `executions`, `analytics`, `fees`, `moderation`, `audit`, and `compliance`.
- Persist the full execution saga in Postgres; BullMQ workers only advance state after checking current DB state.
- Keep all Solana integrations behind API-internal `RpcProvider` and `QuoteProvider` adapters so routing and RPC failover never leak into the bot or admin repos.
- Build `groupchat-bot` as a pure Telegram client that orchestrates DM flows, group actions, and callback handling through API requests plus `@groupchat/shared` DTO validation.
- Build `groupchat-admin` as a pure internal dashboard that fetches data from the same API and never uses direct DB access.
- Keep `groupchat-shared` small and stable so versioning stays manageable; if logic starts accumulating there, move it back into `groupchat-api`.
- Snapshot market price at signal creation time and use it as the baseline for integrity checks and scorecard calculations.
- Auto-expire stale signals, enforce minimum-sample leaderboard eligibility, and track both realized and unrealized performance.

## Development Phases

### Phase 1: Shared Contracts and Backend Foundation
- 1.1 Create `groupchat-shared` as a standalone npm package with initial DTOs for onboarding, groups, signals, copy settings, execution previews, scorecards, moderation actions, and error payloads.
- 1.2 Set up package build, tests, versioning, changelog, and publish workflow for `@groupchat/shared`.
- 1.3 Create `groupchat-api` with Express base routing, validation, error handling, health checks, structured logs, and request correlation IDs.
- 1.4 Add `groupchat-api/docs/legal-risk-matrix.md`, `wallet-threat-model.md`, `execution-state-machine.md`, and `disclosures-and-ux.md`.
- 1.5 Define the Prisma schema for users, Telegram account links, groups, members, signals, copy relationships, executions, fees, audit events, ToS acceptance, geo decisions, and token policy or risk checks.
- 1.6 Implement identity and recovery: internal UUID user model, nullable Telegram links, multi-account linking, wallet-signature recovery, and owner/admin permissions.
- 1.7 Implement compliance controls: ToS acceptance capture, terms versioning, geo-block middleware, disclosure copy management, and onboarding gating.
- 1.8 Implement signals and scorecards: price-at-post snapshot, signal validity window, immutable revisions, minimum signal count, realized PnL, unrealized drawdown, and max open drawdown.
- 1.9 Implement the durable execution lifecycle in Postgres with saga states, idempotent transitions, dead-letter queues, and a reconciliation cron for stuck executions.

### Phase 2: Solana Execution Inside `groupchat-api`
- 2.1 Build `RpcProvider` with ordered dedicated RPC endpoints, regional fallback, timeout rules, and automatic failover. Use Helius or Triton as primary providers and never use public mainnet RPC for execution.
- 2.2 Build `QuoteProvider` with Jupiter as primary, then fallback providers such as OKX DEX API and direct Raydium or Orca pool queries.
- 2.3 Build wallet interaction flows for deep-link signing, wallet-signature auth, and read-only wallet linking. Do not store or transmit plaintext private keys.
- 2.4 Add transaction simulation, slippage enforcement, duplicate prevention, and quote staleness checks before every user confirmation.
- 2.5 Integrate token risk screening with RugCheck and/or Birdeye. Block execution for failed safety rules and record the reason in `TokenRiskCheck`.
- 2.6 Keep custom smart contracts out of MVP. If vaults or delegated on-chain permissions are introduced later, create a separate `groupchat-programs` repo with its own audit and deployment workflow.

### Phase 3: Telegram Bot Repo (`groupchat-bot`)
- 3.1 Create `groupchat-bot` with grammY, config loading, structured logging, API client wrappers, and `@groupchat/shared` contract validation.
- 3.2 Implement DM onboarding with ToS acceptance, geo gating, wallet connect, profile creation, and default risk settings through API calls.
- 3.3 Implement group setup flows: add bot, verify owner/admin, register group, assign roles, approve traders, and log every admin action through the API.
- 3.4 Build the structured signal composer with validation, preview, posted price snapshot, expiry rules, and visible token risk badges driven by API responses.
- 3.5 Build follower setup UX: scorecard review, follow settings, delay buffer, asset limits, slippage caps, and pause/resume controls.
- 3.6 Build trade preview cards that show token, size, estimated slippage, fees, risk flags, and explicit `Confirm` action before any copy execution.
- 3.7 Add receipts, status updates, searchable history, relink/recovery flows, and resilient callback handling for duplicate Telegram events.

### Phase 4: Admin Repo (`groupchat-admin`)
- 4.1 Create `groupchat-admin` with React, Vite, Tailwind CSS, typed API client wrappers, and `@groupchat/shared` DTO validation.
- 4.2 Add pages for groups, traders, executions, fees, signal history, scorecards, geo-block decisions, ToS records, and compliance review.
- 4.3 Add moderation actions: trader suspension, group pause, token block/allow, delayed execution override, and dead-letter queue review through API endpoints.
- 4.4 Add operational views for quote-provider health, RPC latency/error rate, stuck executions, reconciliation backlog, and suspicious activity review.
- 4.5 Add dispute tooling so operators can inspect signal timestamps, posted price baselines, execution state transitions, and user consent records.

### Phase 5: Cross-Repo Testing, Infrastructure, and Release Readiness
- 5.1 Add Docker Compose for Postgres and Redis inside `groupchat-api/infra`, plus seed scripts and local bootstrap commands.
- 5.2 Add CI per repo: contract tests for `groupchat-shared`, API tests for `groupchat-api`, Telegram flow tests for `groupchat-bot`, and UI tests for `groupchat-admin`.
- 5.3 Add cross-repo integration checks so bot and admin are tested against the published or packed version of `@groupchat/shared` they actually consume.
- 5.4 Add failure-path tests for queue restarts, RPC failover, quote-provider fallback, stale quotes, token-risk blocks, duplicate callback delivery, and shared-contract mismatches.
- 5.5 Add observability for request IDs, job IDs, saga state transitions, quote latency, RPC latency, execution failure rates, and geo-block metrics.
- 5.6 Run staging end-to-end pilots with invite-only groups and verify onboarding, signal creation, follower confirmation, chain reconciliation, and scorecard accuracy.
- 5.7 Make legal review, disclosure review, and restricted-geo enforcement explicit launch gates before opening public access.

### Phase 6: Post-MVP Expansion
- 6.1 Add richer analytics, trader monetization, competition mode, and token-gated groups after the core trust loop is stable.
- 6.2 Add delegated automation only after consent, state-machine reliability, and risk controls are proven in production.
- 6.3 Revisit shared vaults and treasury smart contracts only after legal sign-off, contract audit planning, and operational readiness, using a separate `groupchat-programs` repo.
- 6.4 Evaluate multichain support only after Solana execution, retention, and moderation loops are consistently healthy.

## Necessary Commands

### `groupchat-shared`
- `npm install`
- `npm run build`
- `npm run test`
- `npm pack`
- `npm publish`

### `groupchat-api`
- `npm install`
- `docker compose -f infra/docker/docker-compose.yml up -d`
- `npm run prisma:migrate`
- `npm run prisma:seed`
- `npm run dev`
- `npm run worker`
- `npm run test`
- `npm run test:integration`

### `groupchat-bot`
- `npm install`
- `npm run dev`
- `npm run test`

### `groupchat-admin`
- `npm install`
- `npm run dev`
- `npm run build`
- `npm run test`

## Test Checklist
- Onboarding requires ToS acceptance and correctly blocks restricted geographies.
- Telegram account loss or re-linking does not orphan the internal user profile.
- Private-key or seed import is impossible in the MVP flow.
- Signals record a posted-price baseline, expire correctly, and preserve immutable history.
- Leaders cannot gain leaderboard credit from stale, edited, or under-sampled signals.
- Followers always see a risk-screened preview card before confirming execution.
- Token-risk failures block execution and surface readable warnings in Telegram.
- Execution saga transitions remain idempotent through retries, queue restarts, and worker crashes.
- RPC failover and quote-provider fallback work without corrupting execution state.
- Reconciliation recovers stuck `SUBMITTED` executions and updates scorecards correctly.
- Scorecards show realized PnL, current unrealized drawdown, and max open drawdown.
- Bot and admin function only through API endpoints and shared DTOs; they have no direct DB or Solana dependency surface.
- Shared-contract changes are caught by cross-repo tests before release.
- Emergency pause stops new executions while preserving history, auditability, and admin review.
