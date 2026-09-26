# 0001. Stack and architecture: rationale

Decision record for [index.md](index.md). `/develop` does not need this file.

## Context

A solo developer is building a feedback tool for freelancers: add a client's website, share a link, the client pins comments on the live site without an account, and the freelancer works through them on a dashboard board. The first year target is about 1k users. Hosting should be managed, starting on free tiers. The engineer works in TypeScript. (basis: `docs/scope/scope.md`, feature 1 and project intro)

The product has three surfaces with very different trust levels: an authenticated dashboard holding private data; a public, anonymous review page that displays **someone else's website** and accepts writes from anyone with the link; and public marketing pages that must rank in search. The pin overlay is a separate build target that runs inside or on top of a foreign page, so its size and style isolation matter. (basis: scope features 5, 7, 13)

The live preview approach (spec #7) is not decided yet. If it ends up serving a client's HTML through our own server, that site's scripts execute on whatever origin serves it. On the dashboard's origin they could read or ride session cookies. The stack must not rule out that approach, and must stay safe if it is chosen. (basis: same origin policy; OWASP session management guidance)

The core data is relational (users own projects, projects own comments, comments own attachments) and every row must stay private to its owner. Scope features 3 and 5 require that isolation from day one. Deciding the stack late means every slice would start without real structure, which the Tracer Bullet approach forbids. (basis: scope build approach; org isolation as afterthought failure pattern)

## Options considered

### Option 1: Next.js on Vercel with Supabase (chosen)

Next.js 16 App Router in an npm workspace with an overlay package; Supabase for Postgres, Auth, and Storage; Drizzle for schema and queries; Vercel hosting; Resend for mail.

**Pros**: one vendor covers database, auth, storage, and RLS; the smoothest Next.js hosting; the largest ecosystem and the most AI tooling familiarity.
**Cons**: Vercel Hobby forbids commercial use, so Pro is needed before charging; proxied bandwidth is metered; Supabase free projects pause when idle.

### Option 2: Next.js on Cloudflare Workers with Neon, Better Auth, and R2

Same framework via the OpenNext adapter; Neon serverless Postgres; Better Auth (actively maintained, the successor to Auth.js) with tables in your own schema; R2 for files; Resend for mail.

**Pros**: the free tier allows commercial use; generous bandwidth and `HTMLRewriter` suit a proxy based preview; every piece is portable.
**Cons**: four vendors instead of one; RLS does not know the user without extra wiring; OpenNext has edge cases on Workers; Neon's direction after its Databricks acquisition is still settling.

### Option 3: SvelteKit on Cloudflare with Supabase

SvelteKit with Svelte 5 and its first class Cloudflare adapter, backed by Supabase.

**Pros**: lean output, less boilerplate, native Workers support; the overlay could be Svelte, which compiles very small.
**Cons**: smaller component ecosystem (no shadcn equivalent of the same depth); fewer examples and less AI tooling fluency for a solo builder.

### Option 4: Separate SPA plus API service on a container host

A client rendered React dashboard, a standalone Hono or Fastify API, and Postgres on Render or Railway.

**Pros**: a long running server makes a proxy trivial; clear API boundary for future clients (extension, mobile).
**Cons**: two deploys and a hand built API for a solo developer; the landing page needs separate SSR for SEO; free container tiers sleep or are credit based.

## Rationale

The dominant forces are a solo builder, three surfaces with different trust levels, and relational data that must be isolated per user. Option 1 wins on the first and third: one platform (Supabase) gives Postgres, auth, storage, and RLS keyed on the same user id, which is the simplest way to make per user isolation a database guarantee rather than a coding habit. Drizzle keeps the schema in TypeScript and portable. The `withUserRls` transaction helper makes RLS apply even to Drizzle queries, which would otherwise bypass it. (basis: defense in depth; ORM for CRUD, SQL for complexity)

The trust split is handled by architecture, not vendor: a separate registrable domain for the review surface means a client site's scripts, even if proxied, can never reach app cookies. Deploying the same app as two Vercel projects, instead of routing two domains to one project, keeps preview deploys working for both surfaces, because Vercel preview URLs cannot stand in for a second domain. (basis: same origin policy; cookie scoping)

**Hosting: the engineer chose Vercel; Cloudflare Workers was recommended.** Cloudflare was recommended because its free tier allows commercial use and because a proxy based preview would push every client page's bandwidth through the host. Vercel works: it is the smoothest Next.js host and nothing here depends on Cloudflare. The engineer consciously accepts two costs: Vercel Pro before any commercial use, and a bandwidth cost check if spec #7 chooses a proxy. (basis: Vercel Hobby fair use terms)

**Overlay: the engineer chose React; Preact was recommended.** React adds about 45 KB gzipped to every reviewed page, against about 4 KB for Preact. It is acceptable because review pages are opened deliberately by a client, not on the critical path of a site's own visitors. The `preact/compat` alias remains a one line escape hatch. Radix portals do not render inside a shadow root, so shadcn components are not reused in the overlay; only tokens and schemas are shared.

**Abuse protection: the engineer chose to defer it; Upstash rate limiting was recommended.** Share tokens are hard to guess, which limits exposure while links are shared privately. That protection ends when a link leaks or is posted publicly, so rate limiting is enrolled as a follow up rather than dropped.

**Package manager: the engineer chose npm; pnpm was the first pick.** pnpm's strict layout catches undeclared dependencies. npm ships with Node, so there is nothing extra to install or pin in CI and on Vercel. At two or three packages npm workspaces are plenty, and Turborepo works identically on both. The cost is recorded in the spec's Consequences, with a lint rule to guard against phantom dependencies.

Smaller calls made during writing (pick, then runner up):
- Surface routing: by `APP_SURFACE` env var, falling back to the request host. Runner up: host only routing in one Vercel project, which breaks review surface previews.
- Migrations: `drizzle-kit generate` into `supabase/migrations/` applied by the Supabase CLI. Runner up: `drizzle-kit migrate` directly, which splits tooling between local reset and production.
- Env validation: `@t3-oss/env-nextjs`. Runner up: a hand written Zod parse in `env.ts`.
- Logger: `pino`. Runner up: a tiny JSON `console` wrapper.
- Runtime: Node.js functions (Fluid compute), not Edge. Runner up: Edge for the public comment endpoint, rejected because postgres.js and pino need Node APIs.
- Client data: server components plus Server Actions with `revalidatePath`; no client cache library. Runner up: TanStack Query, to add when live dashboard updates land.
- Local review host: `127.0.0.1` beside `localhost`. Runner up: `*.localhost` subdomains, which share more browser site scope and model production isolation less faithfully.

Resolved after an independent cross check (a different model reviewed the draft; the engineer approved each fix):
- Production migrations run in CI before a hook triggered deploy, rather than trusting Vercel's deploy on push, so code never runs against an older schema.
- Previews use a staging Supabase project rather than paid branching or production data.
- `withUserRls` passes the verified `getClaims()` output unchanged, so policies behave exactly as they would behind Supabase's own API, instead of a hand built minimal claim set that later policies could outgrow.
- Tables live in a private `app` schema that the Data API does not expose, so a forgotten policy cannot leak a table through Supabase's public API; RLS remains the second wall. (basis: installed `supabase` skill, RLS in exposed schemas)
- Supabase publishable and secret keys replace the legacy anon and service role keys. (basis: installed `supabase` skill, API key guidance)

## References

**Project sources**:
- `docs/scope/scope.md`: product intro, build approach (Tracer Bullet), workflow (Beta), features 1, 3, 5, 7, 13, and the Deferred list
- Installed `supabase` skill (`supabase/agent-skills`, `.claude/skills/supabase/`): security checklist, exposed schemas, key types, scoped CI tokens

**Practices & standards**:
- Monolith first for small teams
- Defense in depth: database RLS plus application checks
- Same origin policy and host only cookie scoping (OWASP session management guidance)
- ORM for CRUD, SQL for complexity
- Boring technology: relational database as the default

**Links** (verified by the landscape check on 2026-09-26):
- Supabase pricing (free tier limits, 7 day pause): https://supabase.com/pricing
- Vercel pricing (Hobby is non commercial): https://vercel.com/pricing
- Neon pricing: https://neon.tech/pricing
- Resend pricing (3,000 emails per month free): https://resend.com/pricing
- Auth.js joins Better Auth (Auth.js in maintenance mode): https://better-auth.com/blog/authjs-joins-better-auth
- Drizzle ORM releases: https://orm.drizzle.team/docs/latest-releases
- Supabase MCP server: https://supabase.com/blog/mcp-server
- Vercel MCP: https://vercel.com/docs/agent-resources/vercel-mcp
- Resend MCP server: https://resend.com/docs/mcp-server
- Playwright MCP: https://playwright.dev/docs/getting-started-mcp

### Landscape check, 2026-09-26 (summary)

- Frameworks: Next.js 16.x stable (App Router); SvelteKit with Svelte 5 stable; TanStack Start still at v1 release candidate.
- Postgres: Supabase free = 2 projects, 500 MB database, 1 GB storage, 50k auth MAU, pauses after 1 idle week. Neon free = 0.5 GB per project; acquired by Databricks.
- Auth: Better Auth v1.6 actively developed; Auth.js security patches only; Supabase Auth 50k MAU free.
- ORM: Drizzle 1.0 status and Prisma 8 status were not confirmed from official sources; check the current Drizzle release when scaffolding.
- Hosting: Vercel Hobby non commercial; Cloudflare Workers free tier allows commercial use with `HTMLRewriter`; Fly.io no longer offers a free tier to new accounts.
- Email and storage: Resend 3,000 per month free (100 per day); UploadThing 2 GB free; Supabase Storage 1 GB free.
