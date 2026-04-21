# groupchat-shared

`groupchat-shared` is the thin shared contract package for Telegram Groupchat Trading.

It should be published as `@groupchat/shared` and consumed by `groupchat-api`, `groupchat-bot`, and `groupchat-admin`. Its job is to define stable API-facing contracts, not business logic.

## What This Repo Needs

- `Zod` schemas for API request and response payloads
- Shared TypeScript types derived from or aligned with those schemas
- Shared constants that are safe for both server and client use
- Tests that catch contract drift before publishing
- A versioning and publish process that makes API contract changes explicit

## Expected Structure

```text
groupchat-shared/
  src/
    schemas/          # Zod DTOs only
    types/            # shared TS types
    constants/        # shared constants
    index.ts          # package exports
  tests/
```

## What Belongs Here

- Onboarding request and response schemas
- Signal creation payload schemas
- Copy settings DTOs
- Execution preview and confirmation DTOs
- Scorecard response shapes
- Moderation action request shapes
- Public enums and constants shared across repos

## What Must Stay Out

- Prisma models
- Database query helpers
- Environment parsing
- Logger setup
- Solana clients
- Queue logic
- Compliance rules
- Token screening logic
- Scorecard calculations
- Any business logic that could drift between services

## Delivery Checklist

- Contracts are small, explicit, and versioned
- Types are safe for both Node and browser consumers
- Schemas are the source of truth for external API payloads
- New shared exports are reviewed for boundary creep before release

## Design Rule

If a new file in this repo feels like logic instead of a contract, it probably belongs back in `groupchat-api`.
