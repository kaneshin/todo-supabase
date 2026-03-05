# Phase 3: Hono API (Cloudflare Workers)

## Objective

Build the Hono-based REST API deployed to Cloudflare Workers. It provides authenticated CRUD endpoints for todos, validates input with Zod, applies transaction-scoped DB auth context for RLS, and exports `AppType` for type-safe RPC from the Remix frontend.

## Dependencies

- Phase 1 complete (monorepo infrastructure)
- Phase 2 complete (database package with schema and security helper)

## Files to Create

### Package Setup

#### `apps/api/package.json`

```jsonc
{
  "name": "@todo-supabase/api",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "wrangler dev",
    "build": "tsc --noEmit",
    "test": "vitest run",
    "deploy": "wrangler deploy"
  },
  "dependencies": {
    "hono": "^4.x",
    "@hono/zod-validator": "^0.x",
    "@supabase/supabase-js": "^2.x",
    "drizzle-orm": "^0.38.x",
    "postgres": "^3.x",
    "zod": "^3.x",
    "@todo-supabase/database": "workspace:*"
  },
  "devDependencies": {
    "@cloudflare/vitest-pool-workers": "^0.x",
    "@cloudflare/workers-types": "^4.x",
    "@todo-supabase/typescript-config": "workspace:*",
    "typescript": "^5.x",
    "vitest": "^2.x",
    "wrangler": "^3.x"
  }
}
```

#### `apps/api/tsconfig.json`

```jsonc
{
  "extends": "@todo-supabase/typescript-config/worker.json",
  "compilerOptions": {
    "rootDir": "./src",
    "outDir": "./dist"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

#### `apps/api/wrangler.toml`

```toml
name = "todo-supabase-api"
main = "src/index.ts"
compatibility_date = "2024-12-01"
compatibility_flags = ["nodejs_compat"]

[vars]
# Non-secret values can go here; secrets go in .dev.vars
```

#### `apps/api/.dev.vars.example`

```
DATABASE_URL=postgresql://app_user:password@db.xxx.supabase.co:5432/postgres
SUPABASE_URL=https://xxx.supabase.co
SUPABASE_ANON_KEY=eyJ...
```

Use a runtime DB user that cannot bypass RLS.

### Application Code

#### `apps/api/src/index.ts`

```typescript
import { Hono } from "hono";
import { cors } from "hono/cors";
import { todosRoute } from "./routes/todos";

type Bindings = {
  DATABASE_URL: string;
  SUPABASE_URL: string;
  SUPABASE_ANON_KEY: string;
};

const app = new Hono<{ Bindings: Bindings }>();

app.use("/*", cors({
  origin: ["http://localhost:5173"],  // Remix dev server
  credentials: true,
  allowHeaders: ["Content-Type", "Authorization"],
}));

app.route("/todos", todosRoute);

export type AppType = typeof app;
export default app;
```

- `AppType` is the key export — the Remix frontend imports this type for Hono RPC client.
- CORS origin should be configurable via env for production.

#### `apps/api/src/middleware/auth.ts`

```typescript
import { createMiddleware } from "hono/factory";
import { createClient } from "@supabase/supabase-js";

type Env = {
  Bindings: {
    SUPABASE_URL: string;
    SUPABASE_ANON_KEY: string;
  };
  Variables: {
    userId: string;
    accessToken: string;
  };
};

export const authMiddleware = createMiddleware<Env>(async (c, next) => {
  const authorization = c.req.header("Authorization");
  if (!authorization?.startsWith("Bearer ")) {
    return c.json({ error: "Unauthorized" }, 401);
  }

  const token = authorization.slice(7);
  const supabase = createClient(c.env.SUPABASE_URL, c.env.SUPABASE_ANON_KEY);
  const { data: { user }, error } = await supabase.auth.getUser(token);

  if (error || !user) {
    return c.json({ error: "Unauthorized" }, 401);
  }

  c.set("userId", user.id);
  c.set("accessToken", token);
  await next();
});
```

- Extracts Bearer token from Authorization header.
- Verifies token with Supabase Auth — this is server-side verification, not just JWT decode.
- Sets `userId` and `accessToken` on the Hono context for downstream use.

#### `apps/api/src/routes/todos.ts`

```typescript
import { Hono } from "hono";
import { zValidator } from "@hono/zod-validator";
import { z } from "zod";
import { eq, and } from "drizzle-orm";
import { createDb, todos } from "@todo-supabase/database";
import { runAsAuthenticated } from "@todo-supabase/database/security";
import { authMiddleware } from "../middleware/auth";

type Bindings = {
  DATABASE_URL: string;
  SUPABASE_URL: string;
  SUPABASE_ANON_KEY: string;
};

type Variables = {
  userId: string;
  accessToken: string;
};

const route = new Hono<{ Bindings: Bindings; Variables: Variables }>();

// All routes require authentication
route.use("/*", authMiddleware);

// GET /todos — list user's todos
route.get("/", async (c) => {
  const db = createDb(c.env.DATABASE_URL);
  const userId = c.get("userId");
  const claims = { sub: userId, role: "authenticated" as const };
  const result = await runAsAuthenticated(db, claims, async (tx) =>
    tx.select().from(todos).where(eq(todos.userId, userId)).orderBy(todos.createdAt)
  );
  return c.json(result);
});

// GET /todos/:id — single todo
route.get("/:id", async (c) => {
  const db = createDb(c.env.DATABASE_URL);
  const userId = c.get("userId");
  const id = c.req.param("id");
  const claims = { sub: userId, role: "authenticated" as const };
  const [todo] = await runAsAuthenticated(db, claims, async (tx) =>
    tx.select().from(todos).where(and(eq(todos.id, id), eq(todos.userId, userId)))
  );
  if (!todo) {
    return c.json({ error: "Not found" }, 404);
  }
  return c.json(todo);
});

// POST /todos — create a new todo
const createSchema = z.object({ title: z.string().min(1) });

route.post("/", zValidator("json", createSchema), async (c) => {
  const db = createDb(c.env.DATABASE_URL);
  const userId = c.get("userId");
  const { title } = c.req.valid("json");
  const claims = { sub: userId, role: "authenticated" as const };
  const [todo] = await runAsAuthenticated(db, claims, async (tx) =>
    tx.insert(todos).values({ title, userId }).returning()
  );
  return c.json(todo, 201);
});

// PATCH /todos/:id — update title or completed
const updateSchema = z.object({
  title: z.string().min(1).optional(),
  completed: z.boolean().optional(),
});

route.patch("/:id", zValidator("json", updateSchema), async (c) => {
  const db = createDb(c.env.DATABASE_URL);
  const userId = c.get("userId");
  const id = c.req.param("id");
  const data = c.req.valid("json");
  const claims = { sub: userId, role: "authenticated" as const };
  const [todo] = await runAsAuthenticated(db, claims, async (tx) =>
    tx
      .update(todos)
      .set(data)
      .where(and(eq(todos.id, id), eq(todos.userId, userId)))
      .returning()
  );
  if (!todo) {
    return c.json({ error: "Not found" }, 404);
  }
  return c.json(todo);
});

// DELETE /todos/:id
route.delete("/:id", async (c) => {
  const db = createDb(c.env.DATABASE_URL);
  const userId = c.get("userId");
  const id = c.req.param("id");
  const claims = { sub: userId, role: "authenticated" as const };
  const [deleted] = await runAsAuthenticated(db, claims, async (tx) =>
    tx
      .delete(todos)
      .where(and(eq(todos.id, id), eq(todos.userId, userId)))
      .returning()
  );
  if (!deleted) {
    return c.json({ error: "Not found" }, 404);
  }
  return c.json({ success: true });
});

export { route as todosRoute };
```

- Every query filters by `userId` — this is the application-layer defense.
- RLS provides the database-layer defense (defense in depth).
- `createDb` is called per-request. For connection pooling optimization, this may be refactored later.

### Testing

#### `apps/api/vitest.config.ts`

```typescript
import { defineWorkersConfig } from "@cloudflare/vitest-pool-workers/config";

export default defineWorkersConfig({
  test: {
    poolOptions: {
      workers: {
        wrangler: { configPath: "./wrangler.toml" },
      },
    },
  },
});
```

#### `apps/api/src/__tests__/todos.test.ts`

TDD order:

1. **401 guard** — `GET /todos` without Authorization header returns 401.
2. **Create** — `POST /todos` with valid token and body returns 201.
3. **List** — `GET /todos` returns the created todo.
4. **Get by id** — `GET /todos/:id` returns own todo, 404 otherwise.
5. **Update** — `PATCH /todos/:id` toggles completed.
6. **Delete** — `DELETE /todos/:id` removes the todo.
7. **Isolation** — User B cannot access User A's todo (returns 404, not 403).

Note: Tests may need to mock Supabase Auth or use a test-specific Supabase project.

## Key Decisions

| Decision | Rationale |
|----------|-----------|
| `getUser()` over JWT decode | Server-side verification is more secure; handles revoked tokens |
| userId in both query filter AND RLS | Defense in depth — neither layer alone is sufficient |
| Zod validation via `@hono/zod-validator` | Type-safe request validation with automatic error responses |
| 404 (not 403) for other user's todo | Prevents information leakage about resource existence |
| `AppType` export | Enables end-to-end type safety with `hono/client` in Remix |

## Verification

```bash
# Type-check
pnpm --filter @todo-supabase/api build

# Run tests
pnpm --filter @todo-supabase/api test

# Start dev server
pnpm --filter @todo-supabase/api dev

# Manual smoke test (should return 401)
curl -i http://localhost:8787/todos
```
