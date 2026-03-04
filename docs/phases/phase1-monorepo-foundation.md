# Phase 1: Monorepo Foundation

## Objective

Set up the pnpm + Turborepo monorepo with shared TypeScript configuration presets. This phase produces zero application code — only infrastructure that all subsequent phases depend on.

## Dependencies

- Node.js >= 20
- pnpm >= 9

## Files to Create

### Root Configuration

#### `package.json`

```jsonc
{
  "name": "todo-supabase",
  "private": true,
  "packageManager": "pnpm@9.x.x",
  "scripts": {
    "dev": "turbo dev",
    "build": "turbo build",
    "test": "turbo test",
    "lint": "turbo lint",
    "db:generate": "pnpm --filter @todo-supabase/database db:generate",
    "db:migrate": "pnpm --filter @todo-supabase/database db:migrate",
    "db:push": "pnpm --filter @todo-supabase/database db:push"
  }
}
```

- `private: true` prevents accidental npm publish.
- `db:*` scripts delegate to the database package via `--filter`.

#### `pnpm-workspace.yaml`

```yaml
packages:
  - "apps/*"
  - "packages/*"
```

#### `turbo.json`

```jsonc
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", "build/**"]
    },
    "dev": {
      "persistent": true,
      "cache": false
    },
    "test": {},
    "lint": {}
  }
}
```

- `build.dependsOn: ["^build"]` ensures packages build before apps that consume them.
- `dev.persistent: true` keeps dev servers alive; `cache: false` since dev output is ephemeral.

#### `tsconfig.json` (root)

```jsonc
{
  "compilerOptions": {
    "strict": true,
    "skipLibCheck": true,
    "isolatedModules": true
  }
}
```

Minimal root config — each package/app extends a preset from `packages/typescript-config`.

#### `.gitignore`

Ensure these entries exist (some may already be present):

```
node_modules
dist
build
.turbo
.wrangler
.dev.vars
.env
```

#### `.npmrc`

```
auto-install-peers=true
```

Prevents pnpm from prompting about peer dependencies.

### TypeScript Config Package

#### `packages/typescript-config/package.json`

```jsonc
{
  "name": "@todo-supabase/typescript-config",
  "version": "0.0.0",
  "private": true,
  "files": ["*.json"]
}
```

#### `packages/typescript-config/base.json`

```jsonc
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "compilerOptions": {
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "isolatedModules": true,
    "module": "ES2022",
    "moduleResolution": "bundler",
    "target": "ES2022",
    "resolveJsonModule": true,
    "forceConsistentCasingInFileNames": true
  }
}
```

#### `packages/typescript-config/remix.json`

```jsonc
{
  "extends": "./base.json",
  "compilerOptions": {
    "jsx": "react-jsx",
    "lib": ["ES2022", "DOM", "DOM.Iterable"]
  }
}
```

#### `packages/typescript-config/worker.json`

```jsonc
{
  "extends": "./base.json",
  "compilerOptions": {
    "types": ["@cloudflare/workers-types"]
  }
}
```

#### `packages/typescript-config/library.json`

```jsonc
{
  "extends": "./base.json",
  "compilerOptions": {
    "declaration": true
  }
}
```

## Key Decisions

| Decision | Rationale |
|----------|-----------|
| pnpm over npm/yarn | Strict dependency resolution, disk-efficient, workspace protocol support |
| Turborepo | Task orchestration with dependency-aware build ordering, caching |
| 4 TypeScript presets | Each deployment target (Remix/Workers/library) needs different compiler settings |
| Root tsconfig is minimal | Apps and packages extend presets — root is only for IDE fallback |

## Verification

```bash
pnpm install        # Lockfile is created, no errors
pnpm turbo build    # No build tasks yet, but turbo graph resolves without errors
```

Both commands must exit 0. At this point there are no buildable packages, so turbo will simply report no tasks found — that's expected.
