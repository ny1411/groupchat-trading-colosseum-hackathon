# groupchat-api

`groupchat-api` is the system-of-record backend for Telegram Groupchat Trading.

This repo owns the entire trusted server-side execution path. It is the only repo allowed to talk to the database, Redis, BullMQ, Solana RPC providers, quote providers, and risk-screening services.

## What This Repo Needs

- An `Express` API that serves versioned HTTP endpoints for the bot and admin app
- Domain modules for:
  - `auth`
  - `users`
  - `wallets`
  - `groups`
  - `roles`
  - `signals`
  - `copy`
  - `executions`
  - `analytics`
  - `fees`
  - `moderation`
  - `audit`
  - `compliance`
- BullMQ workers for execution lifecycle progression, reconciliation, price polling, and analytics refresh jobs
- A `Prisma` schema, migrations, and seeds for all backend-owned entities
- Solana integration code for RPC failover, quote routing, simulation, execution, and reconciliation
- Risk and compliance logic for geo blocking, token screening, scorecard integrity, and audit capture
- Local development infra for Postgres and Redis
- Internal docs for legal risk, wallet threat modeling, execution state machine rules, and disclosure copy

## Expected Structure

```text
groupchat-api/
  src/
    modules/          # domain logic grouped by feature
    workers/          # BullMQ handlers and worker entrypoints
    routes/           # route composition only
    middleware/       # HTTP middleware
    lib/              # infrastructure clients and app wiring
  prisma/             # schema, migrations, seeds
  packages/
    solana/           # RpcProvider, QuoteProvider, simulation
    risk/             # token screening, geo rules, scorecard helpers
    config/           # env parsing, logger
  infra/
    docker/           # Postgres + Redis for local dev
  docs/               # legal, security, and architecture notes
  tests/              # unit and integration tests
```

## Boundaries

This repo is allowed to own:

- Business logic
- Database access
- Queue definitions and worker logic
- Solana execution infrastructure
- Risk-scoring and compliance enforcement
- Scorecard calculations
- Audit and moderation systems

This repo must not depend on:

- Telegram bot runtime code from `groupchat-bot`
- Admin UI code from `groupchat-admin`
- UI-specific presentation logic

## Why `src/lib/` Exists

One structural improvement over the original folder layout is `src/lib/`.

It should hold shared backend infrastructure such as:

- Prisma client bootstrap
- Redis and BullMQ bootstrap
- API client wrappers for internal services if needed later
- shared backend-only helpers for app and worker startup

This keeps feature modules focused on domain rules instead of constructing infrastructure clients directly.

## Delivery Checklist

- API routes are versioned and validated
- Execution lifecycle is persisted in Postgres, not just in BullMQ jobs
- Token screening and geo gating happen before follower execution
- Signal integrity rules include posted-price snapshots, expiry windows, and immutable revisions
- Scorecards include realized PnL, unrealized drawdown, and max open drawdown
- Audit events capture ToS acceptance, permissions changes, moderation actions, and execution state changes

## What Should Stay Out

- Direct Telegram bot handlers
- React components or admin pages
- Reusable DTOs that belong in `@groupchat/shared`
- Any assumption that clients can bypass the API and talk to DB or Solana directly
