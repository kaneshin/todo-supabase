# Phase 5: Integration & Testing

## Objective

Wire all components together, verify end-to-end flows, and validate the security model through targeted tests.

## Dependencies

- All previous phases (1–4) complete

## Key Wiring

### 1. Hono RPC Types

The type bridge between API and frontend:

```
apps/api/src/index.ts  →  export type AppType = typeof app;
apps/web/app/utils/api.server.ts  →  import type { AppType } from "@todo-supabase/api";
```

- `@todo-supabase/api` is listed as `devDependencies` in `apps/web/package.json` with `workspace:*`.
- Only the **type** crosses the boundary — no runtime code from the API ships to the frontend.
- Turborepo's `dependsOn: ["^build"]` ensures the API package builds first.

### 2. Auth Flow

```
Browser → Remix (cookie-based session via @supabase/ssr)
       → Remix loader/action extracts session.access_token
       → Sends as Authorization: Bearer <token> to Hono API
       → Hono auth middleware calls supabase.auth.getUser(token)
       → Sets userId in Hono context
       → Routes use userId for DB queries
```

- Cookies are HTTP-only; the browser never sees the raw access token.
- Token refresh is handled by `@supabase/ssr` automatically on each request.

### 3. DB Auth Context Flow

```
Hono route handler
  → createDb(DATABASE_URL)
  → setRequestContext(db, jwtClaims)   // sets request.jwt.claims for RLS
  → db.select().from(todos).where(eq(todos.userId, userId))
  → RLS evaluates auth.uid() from JWT claims (defense in depth)
```

- `setRequestContext` must be called **before** any user-scoped query in the same request.
- If per-request context is unreliable with connection pooling, fall back to Supabase PostgREST.

### 4. CORS Configuration

In `apps/api/src/index.ts`:

```typescript
cors({
  origin: ["http://localhost:5173"],  // dev
  credentials: true,
  allowHeaders: ["Content-Type", "Authorization"],
})
```

- Production: origin should be the deployed Cloudflare Pages URL.
- `credentials: true` is required for cookie-based requests (though in this architecture, cookies stay in Remix and Bearer tokens are sent to the API).

## Test Plan

### Automated Tests

#### Pre-requisites

```bash
pnpm install    # Must succeed with no errors
pnpm build      # Turbo builds: typescript-config → database → api → web
pnpm test       # All unit tests pass
```

#### API Unit Tests (`apps/api/src/__tests__/todos.test.ts`)

| # | Test | Expected |
|---|------|----------|
| 1 | GET /todos without auth | 401 Unauthorized |
| 2 | POST /todos with valid auth | 201 + todo object |
| 3 | GET /todos after create | Array containing created todo |
| 4 | PATCH /todos/:id toggle | Updated todo with flipped completed |
| 5 | DELETE /todos/:id | 200 + success |
| 6 | GET /todos/:id of other user | 404 Not Found |

### Manual E2E Testing

#### Setup

1. Create a Supabase project (or use existing)
2. Copy `.dev.vars.example` to `.dev.vars` in both `apps/api/` and `apps/web/`
3. Fill in `DATABASE_URL`, `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `API_URL`
4. Run `pnpm db:push` to apply schema + RLS to Supabase

#### User Flow Test

1. `pnpm dev` — both apps start via Turborepo
2. Visit `http://localhost:5173`
3. Click "Sign up" → enter email + password → submit
4. Log in with the created account
5. Create a todo → verify it appears in the list
6. Toggle the todo's completed checkbox → verify state persists on reload
7. Delete the todo → verify it disappears
8. Sign out → verify redirect to login

#### Multi-User Isolation Test

1. Sign up as **User A** → create 3 todos
2. Sign out
3. Sign up as **User B** → verify todo list is empty
4. Create 2 todos as User B
5. Sign out, sign in as User A → verify only User A's 3 todos are visible

### Security Tests (Required)

#### Test 1: API-Level Isolation

```bash
# As User B, try to access User A's todo by ID
curl -X GET http://localhost:8787/todos/USER_A_TODO_ID \
  -H "Authorization: Bearer USER_B_TOKEN"
# Expected: 404 Not Found (not 403 — avoids leaking existence)
```

```bash
# As User B, try to update User A's todo
curl -X PATCH http://localhost:8787/todos/USER_A_TODO_ID \
  -H "Authorization: Bearer USER_B_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"completed": true}'
# Expected: 404 Not Found
```

```bash
# As User B, try to delete User A's todo
curl -X DELETE http://localhost:8787/todos/USER_A_TODO_ID \
  -H "Authorization: Bearer USER_B_TOKEN"
# Expected: 404 Not Found
```

#### Test 2: Database-Level RLS

Using Supabase SQL editor or psql:

```sql
-- Set JWT claims for User B
SELECT set_config('request.jwt.claims', '{"sub": "USER_B_UUID"}', true);

-- Try to read User A's todo
SELECT * FROM todos WHERE id = 'USER_A_TODO_ID';
-- Expected: 0 rows (RLS blocks access)
```

#### Test 3: RLS Regression Guard

Temporarily create a test route (or modify existing) that **removes** the app-layer `userId` filter:

```typescript
// Dangerous: queries ALL todos without userId filter
const result = await db.select().from(todos);
```

Even without the app-layer filter, RLS should ensure only the authenticated user's todos are returned. This validates that the defense-in-depth model works.

**Important**: Remove this test route after verification. Do not ship it.

## Verification Checklist

- [ ] `pnpm install` — clean install, no warnings
- [ ] `pnpm build` — all packages and apps build successfully
- [ ] `pnpm test` — all tests pass
- [ ] User flow — sign up, login, CRUD todos, sign out
- [ ] Multi-user isolation — users see only their own data
- [ ] Security: API rejects cross-user access with 404
- [ ] Security: RLS blocks direct DB cross-user queries
- [ ] Security: RLS protects even without app-layer userId filter
