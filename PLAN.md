# TODO App Implementation Plan

## Context

Build a full-stack TODO web app in a fresh repository. The app lets users sign up, sign in, and manage their own TODO items (create, toggle completion, delete). Each user's data is isolated via Supabase Row Level Security.

**Tech stack**: TypeScript, Remix (Cloudflare Pages), Hono (Cloudflare Workers), Vite, Vitest, pnpm, Turborepo, Supabase (PostgreSQL + Auth), Drizzle ORM, Tailwind CSS v4, shadcn/ui.

## Architecture

```
todo-supabase/
├── apps/
│   ├── web/              # Remix + Vite → Cloudflare Pages
│   │   ├── app/
│   │   │   ├── components/
│   │   │   │   └── ui/   # shadcn/ui components (owned source)
│   │   │   ├── lib/
│   │   │   │   └── utils.ts  # cn() helper (clsx + tailwind-merge)
│   │   │   └── tailwind.css  # Tailwind globals + CSS variable theme
│   │   └── components.json   # shadcn/ui config
│   └── api/              # Hono → Cloudflare Workers
├── packages/
│   ├── database/         # Drizzle schema, migrations, db factory
│   └── typescript-config/ # Shared tsconfig presets
├── turbo.json
├── pnpm-workspace.yaml
└── package.json
```

**Data flow**: Browser → Remix (SSR, cookie-based auth via `@supabase/ssr`) → Hono API (Bearer token auth, type-safe RPC via `hono/client`) → Drizzle ORM → Supabase PostgreSQL (RLS enabled).

## Security Model (Required)

The API uses direct PostgreSQL access (Drizzle + `postgres`), so user isolation must be enforced in **both** application logic and database policy.

1. **Application layer**: Hono verifies Bearer JWT with Supabase Auth and extracts `userId`.
2. **Database layer**: each request must run TODO queries in a single transaction that sets JWT claim context (`request.jwt.claims`) and `SET LOCAL ROLE authenticated` before queries so RLS can evaluate `auth.uid()` correctly.
3. **Defense in depth**: API queries always filter by `userId`, and RLS independently denies cross-user access if API filtering is wrong or missing.

If per-request claim context cannot be reliably set with the chosen connection strategy, use Supabase PostgREST for user-scoped data access instead of direct SQL for TODO CRUD.

## Phase 1: Monorepo Foundation

Set up root configs and shared TypeScript presets.

### Files to create
- `package.json` — root, private, `packageManager: pnpm`, scripts: `dev/build/test/db:generate/db:migrate/db:push`
- `pnpm-workspace.yaml` — `apps/*`, `packages/*`
- `turbo.json` — tasks: `build` (dependsOn ^build, outputs dist/build), `dev` (persistent, no cache), `test`, `lint`
- `tsconfig.json` — root base config (strict, skipLibCheck, isolatedModules)
- `.gitignore` — node_modules, dist, build, .turbo, .wrangler, .dev.vars, .env
- `.npmrc` — `auto-install-peers=true`
- `packages/typescript-config/package.json` + 4 presets:
  - `base.json` — strict, ES2022, bundler moduleResolution
  - `remix.json` — extends base, jsx: react-jsx, DOM libs
  - `worker.json` — extends base, @cloudflare/workers-types
  - `library.json` — extends base, declaration: true

### Verification
```bash
pnpm install && pnpm turbo build
```

## Phase 2: Database Package (Drizzle + Supabase)

### Files to create in `packages/database/`
- `package.json` — deps: `drizzle-orm`, `postgres`; devDeps: `drizzle-kit`; exports `./` and `./schema`
- `tsconfig.json` — extends library preset
- `drizzle.config.ts` — dialect: postgresql, schema: `./src/schema.ts`, out: `./drizzle`
- `src/schema.ts` — `todos` table:
  - `id` (uuid, defaultRandom, PK)
  - `title` (text, notNull)
  - `completed` (boolean, default false)
  - `userId` (uuid, default `auth.uid()`)
  - `createdAt` / `updatedAt` (timestamptz)
  - RLS policies: authenticated users can CRUD only their own rows (enforced by SQL migration, not only TypeScript schema)
- `src/index.ts` — `createDb(url)` factory (postgres.js with `prepare: false`), re-exports schema
- `drizzle/<timestamp>_init.sql` (or equivalent migration) must include:
  - `ALTER TABLE todos ENABLE ROW LEVEL SECURITY;`
  - Explicit `SELECT`/`INSERT`/`UPDATE`/`DELETE` policies scoped to `auth.uid() = user_id`
  - `updated_at` trigger function + trigger definition
  - Any required grants for authenticated role
- `src/security.ts` (or equivalent) — helpers to run transaction-scoped user queries with per-request claim context and local DB role

### Verification
```bash
pnpm --filter @todo-supabase/database build  # tsc --noEmit
pnpm db:generate && pnpm db:migrate
# Verify RLS is enabled and policies exist in SQL (psql or Supabase SQL editor)
```

## Phase 3: Hono API (Cloudflare Workers)

### Files to create in `apps/api/`
- `package.json` — deps: `hono`, `@supabase/supabase-js`, `drizzle-orm`, `postgres`, `zod`, `@hono/zod-validator`, `@todo-supabase/database`
- `tsconfig.json` — extends worker preset
- `wrangler.toml` — main: `src/index.ts`, compatibility_flags: `nodejs_compat`
- `.dev.vars.example` — template for DATABASE_URL, SUPABASE_URL, SUPABASE_ANON_KEY
- `src/index.ts` — Hono app with CORS, mounts `/todos` route, exports `AppType`
- `src/middleware/auth.ts` — extracts Bearer token, verifies with `supabase.auth.getUser()`, sets `userId`
- `src/routes/todos.ts` — CRUD endpoints:
  - `GET /todos` — list user's todos
  - `GET /todos/:id` — get one todo (404 when not found or not owned)
  - `POST /todos` — create (zod validates `{ title: string }`)
  - `PATCH /todos/:id` — update title/completed
  - `DELETE /todos/:id` — delete
- `vitest.config.ts` — uses `@cloudflare/vitest-pool-workers`
- `src/__tests__/todos.test.ts` — TDD: 401 guard test first, then CRUD tests

### Verification
```bash
pnpm --filter @todo-supabase/api build
pnpm --filter @todo-supabase/api test
pnpm --filter @todo-supabase/api dev  # → localhost:8787
curl http://localhost:8787/todos      # → 401
```

## Phase 4: Remix Frontend (Cloudflare Pages)

### Files to create in `apps/web/`
- `package.json` — deps: `@remix-run/cloudflare`, `@remix-run/react`, `@supabase/ssr`, `@supabase/supabase-js`, `hono`, `react`, `react-dom`, `tailwindcss`, `@tailwindcss/vite`, `clsx`, `tailwind-merge`, `class-variance-authority`, `lucide-react`; devDeps: `@remix-run/dev`, `vite`, `wrangler`, `@todo-supabase/api` (type-only)
- `tsconfig.json` — extends remix preset, paths: `~/*` → `./app/*`
- `vite.config.ts` — plugins in order: `cloudflareDevProxyVitePlugin()`, `tailwindcss()`, `remix()` (order matters)
- `wrangler.toml` — pages_build_output_dir: `./build/client`
- `.dev.vars.example` — SUPABASE_URL, SUPABASE_ANON_KEY, API_URL
- `env.d.ts` — Env interface, AppLoadContext augmentation
- `components.json` — shadcn/ui config (style: "new-york", rsc: false, aliases: `~/components`, `~/lib`)
- `app/tailwind.css` — `@import "tailwindcss"` + shadcn CSS variable theme (`@theme inline`)
- `app/lib/utils.ts` — `cn()` utility combining `clsx` + `tailwind-merge`
- `app/components/ui/` — shadcn components: `button.tsx`, `card.tsx`, `input.tsx`, `label.tsx`, `checkbox.tsx`
- `app/root.tsx` — Links, Meta, Scripts, ScrollRestoration, Outlet; imports `./tailwind.css`
- `app/entry.server.tsx` — standard Remix entry
- `app/utils/supabase.server.ts` — `createSupabaseClient(request, env)` using `@supabase/ssr` cookie helpers
- `app/utils/api.server.ts` — `createApiClient(apiUrl, accessToken)` using `hono/client` with `AppType`
- Routes:
  - `app/routes/_index.tsx` — redirect to `/todos` or `/login`
  - `app/routes/login.tsx` — email/password form using Card, CardHeader, CardContent, Input, Label, Button
  - `app/routes/signup.tsx` — email/password form using Card, CardHeader, CardContent, Input, Label, Button
  - `app/routes/logout.tsx` — action-only, `signOut`, redirect
  - `app/routes/todos.tsx` — loader fetches via RPC, action handles create/toggle/delete; uses Input, Button, Checkbox
  - `app/routes/auth.callback.tsx` — exchanges auth code for session
- `vitest.config.ts` — standard Vitest, alias `~` → `./app`

### Verification
```bash
pnpm --filter @todo-supabase/web build
pnpm dev  # Both apps start via turbo
# Visit http://localhost:5173
```

## Phase 5: Integration & Testing

### Key wiring
1. **Hono RPC types**: `apps/api` exports `AppType`, `apps/web` imports it as type-only via `workspace:*`
2. **Auth flow**: Remix manages cookies → extracts `session.access_token` → sends as Bearer to Hono → Hono verifies with Supabase
3. **DB auth context flow**: Hono runs each request's TODO query inside a transaction that sets `request.jwt.claims` and `SET LOCAL ROLE authenticated` before SQL, so `auth.uid()` resolves to the authenticated user under RLS
4. **CORS**: Hono allows Remix origin with credentials + Authorization header

### Test plan
1. `pnpm install` succeeds
2. `pnpm build` succeeds (turbo builds packages → api → web)
3. `pnpm test` passes all unit tests
4. Manual E2E:
   - Set up Supabase project, fill `.dev.vars`
   - `pnpm db:push` pushes schema
   - `pnpm dev` starts both apps
   - Sign up → login → create todos → toggle → delete → sign out
   - Verify different users see only their own todos
5. Security tests (required):
   - API-level: user B cannot read/update/delete user A's todo even if they know the ID
   - DB-level: run a direct query path with user B claim context and verify RLS blocks access to user A row
   - Regression guard: temporarily remove app-layer `userId` filter in a test-only route/query and verify RLS still prevents cross-user data access

## Phase 6: Documentation

- `CLAUDE.md` — project-specific dev instructions (monorepo structure, commands, architecture decisions)
- `README.md` — setup guide, prerequisites, dev/deploy commands

## Key Decisions

| Decision | Rationale |
|----------|-----------|
| Hono RPC over raw fetch | Compile-time type safety between frontend ↔ backend |
| `@supabase/ssr` (not auth-helpers) | auth-helpers is deprecated |
| Cookie auth in Remix, Bearer in Hono | SSR-standard cookies for browser; stateless tokens for API |
| RLS + application-level filtering | Defense-in-depth: API filters by userId AND DB enforces via RLS with per-request DB auth claim context |
| `prepare: false` on postgres.js | Required for Supabase connection pooling (Transaction mode) |
| shadcn/ui over component libraries (MUI, Chakra) | Zero runtime dependency — components copied as source. Full ownership, tree-shakeable, Tailwind-native. No SSR hydration issues on Cloudflare Pages. |
| Tailwind CSS v4 via `@tailwindcss/vite` | Simpler setup — no PostCSS/autoprefixer config needed. Native CSS cascade layers. |
| No shared UI package | YAGNI — shadcn components live inside `apps/web`, not in a shared package |
| No state management library | YAGNI — Remix loader/action pattern suffices |
