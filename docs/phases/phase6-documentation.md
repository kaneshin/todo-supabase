# Phase 6: Documentation

## Objective

Create project documentation that enables developers to onboard quickly and understand architectural decisions.

## Dependencies

- All previous phases (1–5) complete and verified

## Files to Create

### `CLAUDE.md`

Project-specific instructions for Claude Code sessions. Should include:

#### Structure

```markdown
# CLAUDE.md

## Project Overview
- TODO app: TypeScript monorepo, Remix + Hono + Supabase
- Deployment: Cloudflare Pages (web) + Workers (api)

## Monorepo Structure
- apps/web — Remix frontend (Cloudflare Pages)
- apps/api — Hono API (Cloudflare Workers)
- packages/database — Drizzle schema, migrations, db factory
- packages/typescript-config — Shared tsconfig presets

## Commands
- pnpm dev — start all apps
- pnpm build — build all
- pnpm test — run all tests
- pnpm db:generate — generate Drizzle migrations
- pnpm db:migrate — apply migrations
- pnpm db:push — push schema directly (dev only)

## Architecture Decisions
- Hono RPC for type-safe API communication
- @supabase/ssr for cookie-based auth in Remix
- RLS + app-layer userId filtering (defense in depth)
- prepare: false for Supabase connection pooling
- shadcn/ui components owned as source in apps/web

## Conventions
- snake_case for DB columns, camelCase in TypeScript
- One concern per commit (structural vs behavioral)
- Tests required before merging
```

### `README.md`

Public-facing documentation for the repository.

#### Structure

```markdown
# TODO Supabase

A full-stack TODO app built with TypeScript, Remix, Hono, and Supabase.

## Tech Stack
- **Frontend**: Remix (Cloudflare Pages), Tailwind CSS v4, shadcn/ui
- **API**: Hono (Cloudflare Workers)
- **Database**: Supabase (PostgreSQL + Auth), Drizzle ORM
- **Build**: pnpm, Turborepo, Vite, Vitest

## Prerequisites
- Node.js >= 20
- pnpm >= 9
- Supabase account (free tier works)

## Setup

1. Clone the repository
2. Install dependencies:
   ```bash
   pnpm install
   ```
3. Create a Supabase project at https://supabase.com
4. Copy environment files:
   ```bash
   cp apps/api/.dev.vars.example apps/api/.dev.vars
   cp apps/web/.dev.vars.example apps/web/.dev.vars
   ```
5. Fill in your Supabase credentials in both `.dev.vars` files
6. Push the database schema:
   ```bash
   pnpm db:push
   ```
7. Start development:
   ```bash
   pnpm dev
   ```
8. Visit http://localhost:5173

## Project Structure
(monorepo diagram)

## Scripts
(table of available commands)

## Deployment
(Cloudflare Pages + Workers deployment instructions)
```

## Key Decisions

| Decision | Rationale |
|----------|-----------|
| CLAUDE.md separate from README | CLAUDE.md is for AI dev tooling; README is for humans |
| Concise CLAUDE.md | Stays under 200 lines for context window efficiency |
| README includes full setup | Enables zero-to-running without external docs |

## Verification

- [ ] CLAUDE.md accurately reflects the implemented architecture
- [ ] README setup instructions work from a clean clone
- [ ] All commands listed in README are functional
- [ ] No stale references to unimplemented features
