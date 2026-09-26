# 0001. Stack and architecture for the client feedback app

**Date**: 2026-09-26
**Status**: Proposed

## Summary

The app is one TypeScript codebase: a Next.js web app that serves the dashboard, the public review pages, and the landing page, plus a small React widget (the pin overlay) built separately. Supabase supplies the database, sign in, and file storage; Vercel hosts it; Resend sends email. Review pages live on their own separate domain, so a client's website shown inside a review can never touch your users' sign in cookies. The scaffold for feature 1 builds exactly this, and every later slice builds on it.

## Decision

**Chosen option**: Option 1: Next.js on Vercel with Supabase (one app, one overlay package)

Build a single Next.js 16 app in an npm workspace with a separate React overlay package, backed by Supabase (Postgres, Auth, Storage) through Drizzle, deployed to Vercel as two projects (app surface and review surface) on two separate domains.

**Implementation skills**: `supabase` (`supabase/agent-skills`, `.claude/skills/supabase/`) · `supabase-postgres-best-practices` (`supabase/agent-skills`, `.claude/skills/supabase-postgres-best-practices/`) · `vercel-react-best-practices` (`vercel-labs/agent-skills`, `.claude/skills/vercel-react-best-practices/`) · `vercel-optimize` (`vercel-labs/agent-skills`, `.claude/skills/vercel-optimize/`) · `shadcn` (`shadcn-ui/ui`, `.claude/skills/shadcn/`) · `turborepo` (`vercel/turborepo`, `.claude/skills/turborepo/`) · `playwright-cli` (`microsoft/playwright-cli`, `.claude/skills/playwright-cli/`)

## Rationale

Reasoning, options compared, and sources: see [rationale.md](rationale.md).

## Proposed stack

| Layer | Choice | Reason |
|---|---|---|
| Architecture pattern | Modular monolith: one Next.js app plus an overlay package, in an npm workspace | Solo builder, about 1k users: one deployable is the simplest to build and run; the overlay is split out only because it is a different build target. |
| Language & runtime | TypeScript (strict) everywhere, Node.js 24 LTS | The overlay must be JS anyway, so one language shares types end to end; Node 24 is the current Active LTS. |
| Package manager & tasks | npm workspaces (the npm bundled with Node 24) + Turborepo | Ships with Node, so there is no extra tool to install locally, in CI, or on Vercel; Turborepo orders builds (overlay before web) and caches them. |
| Framework | Next.js 16 (App Router), React 19 | Server rendering for the SEO landing page, server components for the dashboard, largest ecosystem. |
| Server interface | Server Actions for dashboard mutations; Route Handlers under `/api/public/v1/*` for the overlay | Actions need no API layer; the overlay runs on another origin and needs plain, versioned HTTP with CORS. |
| Validation | Zod 4, schemas shared from `packages/shared` | One schema validates forms, actions, the public API, and env vars. |
| Primary DB | Supabase Postgres, region `us-east-1` | Relational data (users, projects, comments, attachments) with constraints; one platform also covering auth and storage. |
| Data access & migrations | Drizzle ORM over `postgres` (postgres.js) through the Supabase transaction pooler; tables in a private `app` schema; `drizzle-kit generate` writes SQL into `supabase/migrations/` | Type safe SQL you can take to any Postgres; one migration folder applied by the Supabase CLI locally, in CI, and in production. |
| Tenant isolation | Row Level Security (RLS, database rules that filter rows per user) on every table, plus explicit owner checks in server code | Two layers, so one missed check in code cannot leak another user's data. |
| Auth | Supabase Auth, email and password, sessions in cookies via `@supabase/ssr` | Already in the platform, and `auth.uid()` plugs straight into RLS. Flows are decided in the sign up & sign in spec. |
| File storage | Supabase Storage, private buckets | Object storage with policies tied to the same user identity; files never go in the database. |
| Email | Resend: custom SMTP for Supabase Auth mail; `resend` SDK + React Email for app mail later | Supabase's built in mailer is too rate limited for real password resets. |
| UI | Tailwind CSS v4 + shadcn/ui (components copied into the repo) | Accessible Radix based components you own; the design system spec decides the look. |
| Forms | React Hook Form + Zod resolver; the same schema validated again in the Server Action | Instant field errors on the client, and the server never trusts the client. |
| Overlay widget | React 19 rendered inside a Shadow DOM root, built by Vite library mode to one IIFE file | Engineer's pick: same component model as the dashboard; Shadow DOM keeps the client site's CSS out and ours in. |
| Hosting | Vercel, two projects from one repo (`app` and `review` surfaces), functions in `iad1` next to the database; Supabase production plus a staging project for previews | Smoothest Next.js hosting; two projects give each surface its own domain and its own preview URLs; previews never touch production data. |
| Domains | App domain for dashboard and landing; a separate registrable domain for review pages and overlay assets | A client site's scripts running on the review domain can never read app domain cookies. |
| Background jobs | None yet | No slice needs one; when email notifications land, start with Vercel Cron or a database backed queue. |
| Observability | `pino` JSON logs with a request id, read in Vercel's log view | Zero vendors now, and the logs can be forwarded to an error tracker when error monitoring is picked up. |
| Local dev | Supabase CLI local stack in Docker (Postgres, Auth, Storage, Mailpit email inbox) | Free, offline, and migrations are tested locally before production. |
| Testing | Vitest + React Testing Library (unit, integration); Playwright (end to end, multiple viewports) | Vitest shares Vite config with the overlay; Playwright handles iframes, Shadow DOM, and screen widths for pin tests. |
| CI & delivery | GitHub Actions: checks on every pull request; on merge to `main`, migrate production then trigger the Vercel production deploy | Checks catch regressions; ordering migrate before deploy means code never ships ahead of its schema. |
| Abuse protection | Deferred (known risk, see Consequences) | Engineer's pick: only input validation and size limits at launch. |

### Repository layout

```
apps/web/                 Next.js app (both surfaces)
  app/(app)/              dashboard + landing + auth pages   (served when APP_SURFACE=app)
  app/(review)/           public review pages                (served when APP_SURFACE=review)
  app/api/public/v1/      overlay HTTP API                   (review surface only)
  lib/db/                 Drizzle schema, clients, RLS helper
  lib/supabase/           @supabase/ssr server and browser clients
packages/overlay/         React + Shadow DOM widget, Vite library build -> overlay.js
packages/shared/          Zod schemas, shared types, design tokens (CSS variables)
supabase/                 config.toml, migrations/ (written by drizzle-kit, meta/ committed), seed.sql
.github/workflows/        ci.yml (pull requests), deploy.yml (merge to main)
```

### How the two surfaces are served

- One Next.js codebase, deployed twice. The request proxy (`proxy.ts`, the Next.js 16 name for middleware) resolves the surface per request: `APP_SURFACE` when set (each Vercel project pins it, so preview URLs work), otherwise by matching the request host against the hosts of `NEXT_PUBLIC_APP_URL` and `NEXT_PUBLIC_REVIEW_URL` (local dev: one server answers as both). It returns 404 for any route group that does not belong to the resolved surface.
- **App surface** (app domain): landing, auth pages, dashboard. Supabase session cookies are set here as host only (no `Domain` attribute), `Secure`, `HttpOnly`, `SameSite=Lax`. Response header `frame-ancestors 'none'`.
- **Review surface** (review domain): review pages, `/api/public/v1/*`, and the built `overlay.js`. It never reads or sets auth cookies. Every response sends `X-Robots-Tag: noindex, nofollow`, and `robots.txt` disallows all. CORS on `/api/public/v1/*` allows only the review origin, plus whatever origins spec #7 (live preview & pinning) requires.
- The dashboard links to review pages by absolute URL built from `NEXT_PUBLIC_REVIEW_URL`.
- How the client's live site is actually shown (iframe, proxy, or embedded script) is decided in spec #7. This stack supports all three: a proxy would run as a Route Handler on the review surface.

### Data access rules

- **User context (dashboard):** queries run through a `withUserRls(claims, fn)` helper. `claims` is the verified claims object from `supabase.auth.getClaims()` on the server, passed through unchanged. The helper opens a Drizzle transaction, runs `select set_config('request.jwt.claims', <claims json>, true)` and `set local role authenticated`, then runs `fn`. `auth.uid()` and `auth.jwt()` inside policies therefore behave exactly as they do behind Supabase's own API. Server code still filters by owner explicitly.
- **Public context (review surface):** a separate `systemDb` client on the same pooled `DATABASE_URL`, with no role switch (it runs as the connection's owner role and bypasses RLS). It may only be used inside `/api/public/v1/*` and review page loaders. Every query must first resolve the project from the share token, then scope all reads and writes to that project id.
- **Private schema:** all app tables live in a Postgres schema named `app` (Drizzle `pgSchema('app')`), which is **not** in Supabase's Data API exposed schemas. The `authenticated` role gets `usage` on `app` plus table grants only as policies need them; `anon` gets nothing. RLS stays enabled on every table as the second wall.
- `@supabase/ssr` is used for auth (session read and refresh) and for Storage, never for table queries.
- Policies are declared in the Drizzle schema (`pgPolicy`) so schema and policies migrate together. Follow the installed `supabase` skill's security checklist (`to authenticated` plus an ownership predicate, `using` and `with check` on updates, no `security definer` in exposed schemas).
- Driver: `postgres` with `prepare: false` (the transaction pooler does not support prepared statements).
- `drizzle-kit generate` is the only Drizzle command that touches migrations. Migrations are applied only by the Supabase CLI; `drizzle-kit migrate` and `drizzle-kit push` are never run against a shared database. Drizzle's `meta/` snapshot folder is committed beside the SQL files.
- `supabase/seed.sql` holds local test data only; roles, grants, and helper functions belong in migrations.

### Delivery pipeline

- **Environments:** local (Supabase CLI in Docker) · staging (a second Supabase project, used by all Vercel preview deploys) · production (Supabase production project, Vercel production).
- **Pull request CI** (GitHub Actions): install, typecheck, lint, unit tests, build, then start the local Supabase stack, `supabase db reset`, and run Playwright. If the PR adds migrations, a job applies them to staging with `supabase db push` so the PR's preview deploy works.
- **Merge to `main`:** a `deploy` workflow runs `supabase db push` against production, then calls the Vercel deploy hook for each project (app, review). Vercel's automatic production deploys from Git are turned off; preview deploys stay automatic.
- **Migration rule:** every migration is backward compatible with the code currently in production (expand, then contract in a later PR), because the schema changes moments before the new code goes live.
- CI credentials: a scoped Supabase personal access token (not a classic full account token) plus the database password for each project, stored as GitHub Actions secrets.

### Overlay delivery

- `packages/overlay` builds with Vite library mode to `dist/overlay.js` (one IIFE, React bundled in). Its styles come from its own Tailwind v4 build, compiled to a CSS string and attached to the shadow root. Document level selectors (`html`, `body`, `:root`) in the reset are rewritten to `:host`, because they match nothing inside a shadow root.
- Turborepo: `web#build` and `web#dev` depend on `overlay#build`; a `copy-overlay` script copies `dist/overlay.js` to `apps/web/public/overlay/v1.js`.
- The review surface serves it at the stable path `/overlay/v1.js` with `Cache-Control: public, max-age=300, stale-while-revalidate=86400`. A breaking overlay change ships as `/overlay/v2.js`; old paths keep working until retired.

### Key invariants

- Each surface serves only its own route group; no auth cookie is ever set on or sent to the review domain.
- Every table lives in the `app` schema with RLS enabled; `systemDb` never appears outside the review surface's public handlers.
- The Supabase secret key and database URLs are server only; nothing but `NEXT_PUBLIC_*` values reach a browser bundle.
- The overlay never imports from `apps/web`; shared code goes through `packages/shared`.
- The overlay ships as a single self contained file, with React bundled in (not loaded from the host page).
- All migrations are generated by `drizzle-kit` into `supabase/migrations/` and applied only by the Supabase CLI; nobody edits a shared schema in the Supabase dashboard.
- Production is migrated before production code deploys, and every migration is backward compatible.
- Every input crossing a trust boundary (form, action, public API) is parsed by a Zod schema with explicit length and size limits.

### Configuration required

Validated at startup by one shared `@t3-oss/env-nextjs` + Zod schema for both surfaces; the app refuses to boot on a missing or malformed value. Both Vercel projects get the identical full set (only `APP_SURFACE` differs), and each Vercel environment points at its own Supabase project (Preview → staging, Production → production).

- `APP_SURFACE`: optional, `app` or `review`; set on each Vercel project to pin the surface, unset locally (surface then comes from the host)
- `NEXT_PUBLIC_APP_URL`: absolute app surface URL (local: `http://localhost:3000`)
- `NEXT_PUBLIC_REVIEW_URL`: absolute review surface URL (local: `http://127.0.0.1:3000`, the same dev server, but a different host, so `localhost` cookies are never sent to it)
- `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY`: Supabase project URL and publishable key (the current replacement for the legacy anon key)
- `SUPABASE_SECRET_KEY`: server only, for admin Storage and Auth operations (the current replacement for the legacy service role key)
- `DATABASE_URL`: pooler connection string (transaction mode, port 6543) for Drizzle at runtime
- `DIRECT_DATABASE_URL`: direct connection for `drizzle-kit` and the Supabase CLI
- `RESEND_API_KEY`: server only; the same key is entered as the SMTP password in Supabase Auth settings
- `LOG_LEVEL`: `pino` level, default `info`

CI secrets (GitHub Actions, not app env): `SUPABASE_ACCESS_TOKEN` (scoped), `SUPABASE_STAGING_PROJECT_REF`, `SUPABASE_STAGING_DB_PASSWORD`, `SUPABASE_PROD_PROJECT_REF`, `SUPABASE_PROD_DB_PASSWORD`, `VERCEL_DEPLOY_HOOK_APP`, `VERCEL_DEPLOY_HOOK_REVIEW`.

Platform settings, not env vars: Supabase Auth custom SMTP pointed at Resend; `app` schema left out of the Data API exposed schemas; Supabase and Vercel both in US East; Vercel automatic production deploys off; Node 24 pinned via `.nvmrc`, `engines`, and the `packageManager` field.

### Scaffold done when (feature 1 scope)

- `npm run dev` boots one dev server against the local Supabase stack: `localhost:3000` serves the app surface placeholder and `127.0.0.1:3000` serves the review surface placeholder, each returning 404 for the other surface's routes.
- `npm run build`, `npm run typecheck`, `npm test`, and one Playwright smoke test pass locally and in GitHub Actions.
- The overlay builds to one IIFE file that the review surface serves at `/overlay/v1.js`.
- A baseline migration (creates the `app` schema and its grants only, no domain tables; the data model spec owns those) applies cleanly with `supabase db reset`.
- The `proxy.ts` file convention is confirmed against the installed Next.js version (fall back to `middleware.ts` if the installed version still uses it).

## Consequences

**Positive**:
- One language, one repo, one database vendor: the whole system fits in one head.
- RLS plus server checks makes cross user data leaks a two failure event, not a one line bug.
- A separate review domain closes the worst preview risk (a client site stealing sessions) before spec #7 picks its approach.
- Supabase CLI plus Docker gives a faithful local copy of production, including the auth email inbox.

**Negative / tradeoffs**:
- **Vercel Hobby forbids commercial use.** Move both projects to Vercel Pro (about $20 per member per month) before charging or promoting the product commercially.
- **Proxied bandwidth is metered on Vercel.** If spec #7 chooses a server proxy of client sites, all of that traffic counts against Vercel limits; re-check the cost then.
- **Supabase free projects pause after 7 idle days** and cap at 2 projects. Upgrade production to Supabase Pro at launch, or accept the pause risk while there are no users.
- **React in the overlay** adds roughly 45 KB (gzipped) to every reviewed page. Escape hatch: alias `react` to `preact/compat` in the overlay's Vite config if load time hurts.
- **No abuse protection** on the anonymous comment endpoint: anyone with a share link can flood a project with comments until rate limiting lands.
- npm workspaces hoist dependencies into one root `node_modules`, so a package can import a dependency it never declared ("phantom" dependency) and still work locally. The overlay is most at risk: an undeclared import can quietly bloat its bundle. Each package must declare every dependency it imports; turn on an import lint rule (such as `import/no-extraneous-dependencies`) when `/audit` sets up tooling.
- Two Vercel projects double build minutes and mean env vars are kept in two places.
- Staging uses the second free Supabase project, so there is no free slot left, and all open PRs share one staging database (fine for one developer, messy for a team).
- Deploying through hooks after `supabase db push` is more pipeline to own than Vercel's default "deploy on push", and every migration must be written backward compatible.
- Drizzle plus the RLS transaction helper is a custom pattern; every contributor (human or AI) must use the helper, not a bare `db` client, in user context.
- Two domains to buy, renew, and configure.

**Neutral**:
- Next.js 16 renamed middleware to the request proxy (`proxy.ts`); older guides use the old name.
- Server functions run on the Node.js runtime (Vercel Fluid compute), not the Edge runtime, so postgres.js and pino work unchanged.
- Local dev needs Docker Desktop running.

## Follow-up

- [ ] Choose and register the app domain and the separate review domain before the first production deploy.
- [ ] Add rate limiting to `/api/public/v1/*` (per share token and per IP) before share links are used publicly; enrolled in the scope's Deferred list as "Abuse protection on public review links".
- [ ] Before charging users: move to Vercel Pro and Supabase Pro.
- [ ] Spec #7 (live preview & pinning) must confirm its approach fits the review surface and CORS rules here, and re-check Vercel bandwidth cost if it proxies.
- [ ] Spec #3 (data model) writes the first domain tables with RLS policies declared in the Drizzle schema, using the `withUserRls` / `systemDb` split.
- [ ] `/audit` (feature 2) should capture this stack, the data access rules, and the surface split in root `AGENTS.md`.
- [ ] Installed skills are not yet in any `AGENTS.md`: `supabase`, `supabase-postgres-best-practices`, `vercel-react-best-practices`, `vercel-optimize`, `shadcn`, and `turborepo` apply project wide (root `AGENTS.md` `## Agent skills`); `playwright-cli` belongs with the test area. Record the chosen MCP servers (Supabase, Vercel, Resend, Playwright) on the `MCP servers:` line.
- [ ] Connect the MCP servers you picked (your own config step). Give the Supabase server a scoped personal access token, never a classic full account token.
- [ ] Create the Supabase staging and production projects in `us-east-1`, and create both Vercel projects with their deploy hooks, before the first deploy.
