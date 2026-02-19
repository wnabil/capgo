# Capgo AI Coding Agent Instructions

For comprehensive guidance, see **[AGENTS.md](../AGENTS.md)** - this file highlights critical patterns only.

## Project Overview

Capgo is a live update platform for Capacitor apps:

- **Frontend**: Vue 3 SPA (Vite, Tailwind, DaisyUI)
- **Backend**: Multi-platform edge functions (Cloudflare Workers 99%, Supabase Functions fallback)
- **Database**: PostgreSQL via Supabase with read replicas (5 continents)
- **Mobile**: Capacitor iOS/Android with OTA updates

## Critical Architecture Patterns

### Multi-Platform Backend: Shared Code Pattern

Backend **source code lives in `supabase/functions/_backend/`** and deploys to three platforms:

1. **Cloudflare Workers** (API: 8787, Plugin: 8788, Files: 8789)
2. **Supabase Functions** (internal tasks, CRON jobs)

All code uses **Hono framework** with middleware context `c.get('requestId')`, `c.get('auth')`, `c.get('apikey')`.
**CRITICAL**: Never hard-code platform-specific logic; all logic must work identically across platforms.

### Cache System for Plugin Endpoints (⚠️ CRITICAL)

Plugin endpoints (`/updates`, `/stats`, `/channel_self`) rely on **layered caching** with specific response codes:

- **On-prem cache**: `429 + { error: 'on_premise_app' }` detected by Cloudflare snippet (`cloudflare_workers/snippet/index.js`)
- **Plan-upgrade cache**: `429 + { error: 'need_plan_upgrade' }`
- **App status cache**: `supabase/functions/_backend/utils/appStatus.ts` (60s in-memory)

**Implication**: Do NOT change `429` error payloads or cache logic without updating edge snippet logic.

### Database Layer: Postgres

- Active migration from Supabase Postgres to Cloudflare D1
- Schema defined in `utils/postgress_schema.ts` with Drizzle ORM
- Use `getPgClient()`, `getDrizzleClient()` via `pg.ts`

**Migration rules**:

- **Never edit committed migrations** in `supabase/migrations/`
- Create new migrations with `supabase migration new <feature_slug>`
- Edit the **single migration file** until feature ships
- Run `bun types` after schema changes to regenerate TypeScript types

### Request Context & HTTP Response Patterns

All endpoints receive Hono `Context` with `c.get('requestId')`, `c.get('auth')`, `c.get('apikey')`.
Use `cloudlog({ requestId: c.get('requestId'), message: '...' })` for structured logging.

**HTTP Response Patterns**:

- **Success with data**: `return c.json(data)` or `return c.json(data, 200)`
- **Success without data**: `return c.json(BRES)` where `BRES = { status: 'ok' }`
- **Errors**: Use `return simpleError()` or `return quickError(status, ...)`
- **Never use**: `c.body(null, 204)` for success responses

**API Backward Compatibility (⚠️ CRITICAL)**:

- Adding optional fields is safe; removing fields breaks old clients
- Never change meaning of existing fields or HTTP status codes
- Use plugin version detection (if needed) to support old clients
- See [AGENTS.md#api-backward-compatibility](../AGENTS.md) for examples

## Development Workflows

### Local Development Setup

```bash
# Start Supabase (required for all development)
supabase start

# Seed database with fresh test data
supabase db reset

# Start frontend (localhost:5173)
bun serve:local  # Uses local env
bun serve:dev    # Uses development branch env

# Start backend edge functions
bun backend  # Supabase functions on :54321

# Start Cloudflare Workers (optional, for testing CF deployment)
./scripts/start-cloudflare-workers.sh
```

Test accounts (after `supabase db reset`):

- `test@capgo.app` / `testtest` (demo user with data)
- `admin@capgo.app` / `adminadmin` (admin user)

### Testing Strategy

**Backend tests** (`tests/` directory, Vitest):

```bash
bun test:all          # All tests against Supabase
bun test:backend      # Exclude CLI tests
bun test:cli          # Only CLI tests (requires LOCAL_CLI_PATH=true)

# Cloudflare Workers testing
bun test:cloudflare:all      # All tests against CF Workers
bun test:cloudflare:backend  # Backend tests on CF Workers
```

Tests use `tests/test-utils.ts` helpers:

- `getEndpointUrl(path)` routes to correct worker based on endpoint
- `USE_CLOUDFLARE_WORKERS=true` env var switches backend target
- Tests modify local database; always reset before test runs

**⚠️ CRITICAL: Test Isolation for Parallel Execution**:

- **ALL TEST FILES RUN IN PARALLEL** - design tests with this in mind
- Use `it.concurrent()` instead of `it()` to run tests in parallel within same file
- **Create dedicated seed data** if your test modifies shared resources
- See [AGENTS.md#test-isolation](../AGENTS.md) for detailed patterns and examples

**Frontend tests** (Playwright):

```bash
bun test:front  # E2E tests in playwright/e2e/
```

### Code Quality Commands

```bash
bun lint          # ESLint for frontend (src/**/*.{vue,ts,js})
bun lint:fix      # Auto-fix linting issues
bun lint:backend  # ESLint for backend (supabase/**/*.{ts,js})
bun typecheck     # Vue + TypeScript type checking
bun types         # Generate Supabase types (after migrations)
```

**Never commit without running `bun lint:fix`** before validation.

## Database Conventions

### RLS Policies (⚠️ CRITICAL)

Use `get_identity_org_appid()` when table has `app_id`:

```sql
public.get_identity_org_appid(
    '{read,upload,write,all}'::public.key_mode[],
    owner_org,  -- or org_id
    app_id
)
```

Use `get_identity_org_allowed()` only as **last resort** when no `app_id` exists and cannot be joined.

**NEVER use `get_identity()` directly.** See [AGENTS.md#database-rls-policies](../AGENTS.md) for full patterns.

### PostgreSQL Function Security

**ALWAYS set empty search_path** in all functions:

```sql
CREATE OR REPLACE FUNCTION "public"."my_function"()
RETURNS void
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = ''
```

Use fully qualified names for all table references.

### Migrations Workflow

1. Create migration: `supabase migration new <feature_slug>`
2. Edit the **single migration file** until feature ships
3. Test locally: `supabase db reset`
4. Update `supabase/seed.sql` for new test fixtures
5. Push to cloud: `supabase db push --linked` (prod only)

## Frontend Conventions

- **Framework**: Vue 3 with `<script setup>` syntax, Composition API
- **Build**: Vite with file-based routing (`src/pages/` → auto-generated routes)
- **Styling**: Tailwind utilities + DaisyUI components (`d-btn`, `d-input`, etc.)
- **Safe areas**: Use Konsta components **only** for safe area helpers (top/bottom insets)
- **Colors**: Primary `#119eff` (azure-500), secondary `#515271` (primary-500) from `src/styles/style.css`
- **State**: Pinia stores in `src/stores/`, API clients in `src/services/`

Avoid custom CSS; prefer utility composition and DaisyUI theming.

## Deployment & CI/CD

**Do not manually deploy.** CI/CD handles:

- Version bumping and `CHANGELOG.md` generation (semantic-release)
- Auto-deployment to Cloudflare/Supabase after merge to `main`

**PR Requirements**:

- Summary, Motivation, Business Impact, Test Plan sections required
- **⚠️ CRITICAL: Mark all AI-generated sections with "(AI generated)"** - transparency is mandatory
- See [AGENTS.md#pull-request-guidelines](../AGENTS.md) for examples

## Key Files Reference

| Path                                             | Purpose                                          |
| ------------------------------------------------ | ------------------------------------------------ |
| `supabase/functions/_backend/`                   | Shared backend code for all platforms            |
| `cloudflare_workers/{api,plugin,files}/index.ts` | Platform-specific entry points                   |
| `supabase/functions/_backend/utils/hono.ts`      | Hono app factory, middleware, error handling     |
| `supabase/functions/_backend/utils/appStatus.ts` | 60s in-memory app status cache                   |
| `tests/test-utils.ts`                            | Test helpers, endpoint routing, seeding          |
| `src/services/supabase.ts`                       | Frontend Supabase client setup                   |
| `supabase/migrations/`                           | Database migrations (never edit committed files) |

## Common Pitfalls

1. **Changing cache response codes**: Don't modify `429` error payloads without updating `cloudflare_workers/snippet/index.js`
2. **Missing RLS app_id check**: Use `get_identity_org_appid()` not `get_identity()` directly
3. **Parallel test conflicts**: Create dedicated seed data for tests that modify shared resources
4. **Editing old migrations**: Always create new migration files
5. **Missing search_path in functions**: ALL PostgreSQL functions need `SET search_path = ''`
6. **Breaking API changes**: Adding optional fields is safe; removing fields breaks old plugins
7. **Forgetting lint before commit**: Run `bun lint:fix` before validation

For detailed guidance, see **[AGENTS.md](../AGENTS.md)**.
