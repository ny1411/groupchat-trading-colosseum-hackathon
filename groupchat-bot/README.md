# groupchat-bot

`groupchat-bot` is the Telegram client for Telegram Groupchat Trading.

This repo owns the user-facing Telegram experience in DMs and groups. It should be a thin client that talks to `groupchat-api` over HTTP and validates request and response contracts using `@groupchat/shared`.

## What This Repo Needs

- A `grammY` bot runtime with startup, shutdown, and error handling
- Command handlers and callback handlers for:
  - onboarding
  - ToS acceptance flow
  - geo-gated access messaging
  - wallet link handoff
  - group registration
  - role assignment actions
  - signal creation flow
  - follow and pause settings
  - trade preview and confirm flows
  - receipts and status updates
- A typed API client for calling `groupchat-api`
- Minimal bot-side session state only when needed for user interaction flow
- Tests for command handling, callback handling, and contract compatibility

## Expected Structure

```text
groupchat-bot/
  src/
    bot/              # bot bootstrap and grammY setup
    commands/         # slash command handlers
    callbacks/        # inline keyboard callback handlers
    api/              # typed HTTP client to groupchat-api
    formatters/       # Telegram message formatting helpers
    config/           # env parsing and bot config
  tests/
```

## Boundaries

This repo is allowed to own:

- Telegram command and callback handling
- Telegram-specific formatting and UX copy
- API request orchestration
- Retry-safe callback handling

This repo must not own:

- Database access
- Prisma models
- BullMQ workers
- Solana RPC access
- Quote routing
- Token risk engines
- Scorecard business logic

## Delivery Checklist

- All bot flows call the API instead of embedding business logic locally
- Followers always see a risk-screened preview before confirming a trade
- Group admin actions are routed through the API and become auditable
- Duplicate Telegram callbacks do not cause duplicate state transitions
- Wallet import of private keys or seed phrases is not implemented

## What Should Stay Out

- Direct SQL or Prisma usage
- Redis or BullMQ clients
- On-chain execution code
- Long-lived financial state or server-side orchestration logic
