# groupchat-admin

`groupchat-admin` is the internal operations and moderation dashboard for Telegram Groupchat Trading.

This repo exists for staff and trusted operators, not end users. It should fetch all data from `groupchat-api` over HTTP and render moderation, compliance, execution, and analytics workflows without any direct database access.

## What This Repo Needs

- A `React + Vite` application for internal tooling
- Typed API client wrappers for `groupchat-api`
- Pages for:
  - groups
  - traders
  - executions
  - fees
  - signal history
  - scorecards
  - token policy management
  - geo-block decisions
  - ToS records
  - moderation and dispute review
- Operational views for:
  - quote-provider health
  - RPC latency and error rate
  - stuck executions
  - reconciliation backlog
  - suspicious activity review
- Tests for UI rendering, API integration boundaries, and critical moderation flows

## Expected Structure

```text
groupchat-admin/
  src/
    app/              # app shell and routing
    pages/            # internal screens
    components/       # UI components
    api/              # typed HTTP client to groupchat-api
    features/         # domain-oriented UI logic
    config/           # env parsing and app config
  tests/
```

## Boundaries

This repo is allowed to own:

- Internal UI and workflows
- API orchestration for moderation and operations
- Frontend state and filtering
- Presentation of scorecards, audits, and execution status

This repo must not own:

- Database access
- Prisma schema or migrations
- Queue logic
- Solana clients
- Quote-provider logic
- Risk-screening rules
- Canonical scorecard calculations

## Delivery Checklist

- Admin actions route through the API and are auditable
- Operators can inspect signals, posted-price baselines, and execution state transitions
- Token block/allow rules are managed through API endpoints, not local state
- Geo-block and ToS data are visible to support legal and compliance workflows
- Emergency pause actions are available without bypassing backend permissions

## What Should Stay Out

- Hidden backend logic in frontend helpers
- Direct access to Postgres, Redis, or Solana
- Reimplementation of validation or decision logic that should remain canonical in `groupchat-api`
