# TODO

Implementation checklist extracted from [PLAN.md](./PLAN.md).

## Phase 1: Monorepo Foundation

- [ ] Create root `package.json` (private, pnpm, scripts: dev/build/test/db:generate/db:migrate/db:push)
- [ ] Create `pnpm-workspace.yaml` (apps/*, packages/*)
- [ ] Create `turbo.json` (build, dev, test, lint tasks)
- [ ] Create root `tsconfig.json` (strict, skipLibCheck, isolatedModules)
- [ ] Update `.gitignore` (node_modules, dist, build, .turbo, .wrangler, .dev.vars, .env)
- [ ] Create `.npmrc` (auto-install-peers=true)
- [ ] Create `packages/typescript-config/package.json`
- [ ] Create TypeScript presets: base.json, remix.json, worker.json, library.json
- [ ] Verify: `pnpm install && pnpm turbo build` succeeds

## Phase 2: Database Package (Drizzle + Supabase)

- [ ] Create `packages/database/package.json` (drizzle-orm, postgres, drizzle-kit)
- [ ] Create `packages/database/tsconfig.json` (extends library preset)
- [ ] Create `packages/database/drizzle.config.ts`
- [ ] Create `packages/database/src/schema.ts` (todos table with id, title, completed, userId, timestamps)
- [ ] Create `packages/database/src/index.ts` (createDb factory, re-export schema)
- [ ] Create SQL migration with RLS policies (SELECT/INSERT/UPDATE/DELETE scoped to auth.uid())
- [ ] Add updated_at trigger function and trigger definition
- [ ] Create `packages/database/src/security.ts` (per-request JWT claim context helper)
- [ ] Verify: `pnpm --filter @todo-supabase/database build` succeeds
- [ ] Verify: `pnpm db:generate && pnpm db:migrate` succeeds
- [ ] Verify: RLS is enabled and policies exist

## Phase 3: Hono API (Cloudflare Workers)

- [ ] Create `apps/api/package.json` (hono, supabase-js, drizzle-orm, zod, etc.)
- [ ] Create `apps/api/tsconfig.json` (extends worker preset)
- [ ] Create `apps/api/wrangler.toml` (main: src/index.ts, nodejs_compat)
- [ ] Create `apps/api/.dev.vars.example`
- [ ] Create `apps/api/src/index.ts` (Hono app with CORS, mount /todos, export AppType)
- [ ] Create `apps/api/src/middleware/auth.ts` (Bearer token verification via Supabase)
- [ ] Create `apps/api/src/routes/todos.ts` (GET, POST, PATCH, DELETE endpoints)
- [ ] Create `apps/api/vitest.config.ts` (@cloudflare/vitest-pool-workers)
- [ ] Write tests: 401 guard test first, then CRUD tests (TDD)
- [ ] Verify: `pnpm --filter @todo-supabase/api build` succeeds
- [ ] Verify: `pnpm --filter @todo-supabase/api test` passes
- [ ] Verify: `curl http://localhost:8787/todos` returns 401

## Phase 4: Remix Frontend (Cloudflare Pages)

- [ ] Create `apps/web/package.json` (remix, supabase/ssr, hono, tailwindcss, shadcn/ui deps)
- [ ] Create `apps/web/tsconfig.json` (extends remix preset, ~/* path alias)
- [ ] Create `apps/web/vite.config.ts` (cloudflareDevProxy, tailwindcss, remix plugins)
- [ ] Create `apps/web/wrangler.toml` (pages_build_output_dir)
- [ ] Create `apps/web/.dev.vars.example`
- [ ] Create `apps/web/env.d.ts` (Env interface, AppLoadContext)
- [ ] Create `apps/web/components.json` (shadcn/ui config)
- [ ] Create `app/tailwind.css` (Tailwind v4 + shadcn CSS variable theme)
- [ ] Create `app/lib/utils.ts` (cn() helper)
- [ ] Add shadcn/ui components: button, card, input, label, checkbox
- [ ] Create `app/root.tsx` (Links, Meta, Scripts, ScrollRestoration, Outlet)
- [ ] Create `app/entry.server.tsx`
- [ ] Create `app/utils/supabase.server.ts` (cookie-based auth via @supabase/ssr)
- [ ] Create `app/utils/api.server.ts` (Hono RPC client with AppType)
- [ ] Create route: `_index.tsx` (redirect to /todos or /login)
- [ ] Create route: `login.tsx` (email/password form)
- [ ] Create route: `signup.tsx` (email/password form)
- [ ] Create route: `logout.tsx` (action-only, signOut)
- [ ] Create route: `todos.tsx` (loader + action for CRUD)
- [ ] Create route: `auth.callback.tsx` (exchange auth code for session)
- [ ] Create `apps/web/vitest.config.ts`
- [ ] Verify: `pnpm --filter @todo-supabase/web build` succeeds
- [ ] Verify: `pnpm dev` starts both apps, visit http://localhost:5173

## Phase 5: Integration & Testing

- [ ] Wire Hono RPC types (api exports AppType, web imports as type-only)
- [ ] Wire auth flow (Remix cookies → Bearer token → Hono → Supabase verify)
- [ ] Wire DB auth context (per-request JWT claim context for auth.uid())
- [ ] Configure CORS (Hono allows Remix origin with credentials + Authorization)
- [ ] Verify: `pnpm install` succeeds
- [ ] Verify: `pnpm build` succeeds (turbo builds packages → api → web)
- [ ] Verify: `pnpm test` passes all unit tests
- [ ] Manual E2E: set up Supabase project, fill .dev.vars
- [ ] Manual E2E: `pnpm db:push` pushes schema
- [ ] Manual E2E: sign up → login → create todos → toggle → delete → sign out
- [ ] Manual E2E: verify different users see only their own todos
- [ ] Security test: user B cannot access user A's todo via API
- [ ] Security test: RLS blocks cross-user access at DB level
- [ ] Security test: RLS prevents access even without app-layer userId filter

## Phase 6: Documentation

- [ ] Create `CLAUDE.md` (project-specific dev instructions, monorepo structure, commands)
- [ ] Create `README.md` (setup guide, prerequisites, dev/deploy commands)
