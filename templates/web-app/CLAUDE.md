# [YOUR_PROJECT_NAME] — Next.js Web Application

<!-- WHY THIS EXISTS: Claude needs project identity upfront to make contextually accurate decisions.
     Replace all [BRACKETED] values with your actual project details. -->

[YOUR_PROJECT_NAME] is a [brief description — e.g., "B2B SaaS dashboard for inventory management"].

## Tech Stack

- **Framework**: Next.js 15 (App Router)
- **Language**: TypeScript 5.x (strict mode)
- **Styling**: Tailwind CSS 4.x + [shadcn/ui | Radix | your component library]
- **Database**: PostgreSQL via [Prisma | Drizzle] ORM
- **Auth**: [NextAuth.js v5 | Clerk | Lucia]
- **State**: React 19 Server Components by default; [Zustand | Jotai] for client state
- **Validation**: Zod (shared between client + server)

## Route Structure

<!-- WHY: Claude generates files in the wrong place without explicit route conventions. -->

```
app/
  (marketing)/        # Public pages — landing, pricing, blog
  (auth)/             # Login, register, forgot-password
  (dashboard)/        # Authenticated app — uses layout with sidebar
    settings/
    [resource]/       # e.g., /projects, /teams
      [id]/
        page.tsx      # Detail view
  api/
    v1/               # Versioned API routes
      [resource]/
        route.ts      # GET, POST, PATCH, DELETE handlers
```

- **Page components**: Server Components by default. Add `"use client"` only when you need interactivity.
- **Data fetching**: Server Components call DB/ORM directly. No API routes for internal data.
- **API routes** (`route.ts`): Only for external consumers and webhooks.
- **Server Actions**: Use for mutations (forms, buttons). Defined in `app/_actions/[domain].ts`.

## Component Patterns

<!-- WHY: The server/client boundary is Claude's most common Next.js mistake. -->

| Pattern | Convention |
|---------|-----------|
| Server Component | Default. Fetches data, renders HTML. No hooks, no event handlers. |
| Client Component | `"use client"` directive. For interactivity, browser APIs, hooks. |
| Shared UI | `components/ui/` — pure presentational, no data fetching |
| Feature components | `components/[feature]/` — can be server or client |
| Layouts | `app/**/layout.tsx` — server components, handle auth checks |

**Key rule**: Pass server data DOWN to client components as props. Never import a Server Component into a Client Component.

## Environment Variables

<!-- WHY: Claude must know what exists without seeing .env values. NEVER commit secrets. -->

```bash
# .env.local (NEVER committed — listed in .gitignore)
DATABASE_URL="postgresql://..."        # Prisma/Drizzle connection
NEXTAUTH_SECRET="..."                  # Auth signing key
NEXTAUTH_URL="http://localhost:3000"
[PROVIDER]_CLIENT_ID="..."             # OAuth provider
[PROVIDER]_CLIENT_SECRET="..."
STRIPE_SECRET_KEY="..."                # Payment processing
RESEND_API_KEY="..."                   # Transactional email

# Public (safe to expose — prefixed with NEXT_PUBLIC_)
NEXT_PUBLIC_APP_URL="http://localhost:3000"
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY="..."
NEXT_PUBLIC_POSTHOG_KEY="..."          # Analytics
```

## Database Schema Highlights

<!-- WHY: Claude needs to understand data relationships to write correct queries. -->

Key models: `User`, `Organization`, `[YOUR_CORE_RESOURCE]`, `[YOUR_SECONDARY_RESOURCE]`

- Users belong to Organizations (many-to-many via `Membership` with role)
- [YOUR_CORE_RESOURCE] belongs to Organization, has [describe key relationships]
- Soft deletes: `deletedAt` timestamp on [list models that use soft delete]
- Migrations: `npx prisma migrate dev` (dev) / `npx prisma migrate deploy` (prod)

## Quick Commands

```bash
npm run dev              # Start dev server (port 3000)
npm run build            # Production build
npm run db:push          # Push schema changes (dev only)
npm run db:migrate       # Create + apply migration
npm run db:seed          # Seed development data
npm run db:studio        # Open Prisma Studio / Drizzle Studio
npm run lint             # ESLint + TypeScript checks
npm run test             # Vitest unit + integration tests
npm run test:e2e         # Playwright browser tests
```

## Testing

<!-- WHY: Claude writes better tests when it knows the exact stack and conventions. -->

- **Unit/Integration**: Vitest — test files colocated as `[name].test.ts`
- **E2E**: Playwright — tests in `e2e/` directory, one spec per user flow
- **Test DB**: Uses separate `DATABASE_URL_TEST` connection, reset between suites
- **Mocking**: `vi.mock()` for external services; MSW for API mocking in integration tests

## Deployment

- **Platform**: Vercel (auto-deploys from `main` branch)
- **Preview**: Every PR gets a preview deployment
- **Env vars**: Set in Vercel dashboard (never in code)
- **Edge**: Middleware runs at edge; API routes run in Node.js runtime by default

## Performance Targets

<!-- WHY: Concrete numbers prevent Claude from ignoring performance during implementation. -->

| Metric | Target | Measured Via |
|--------|--------|-------------|
| LCP | < 2.5s | Vercel Analytics / Lighthouse |
| FID | < 100ms | Vercel Analytics |
| CLS | < 0.1 | Vercel Analytics |
| Bundle size (JS) | < 200KB first load | `next build` output |
| API response (p95) | < 300ms | Application monitoring |

## Common Gotchas

<!-- WHY: These are real issues that burn hours. Claude seeing them upfront saves significant time. -->

1. **Hydration mismatch**: Never use `Date.now()`, `Math.random()`, or browser-only APIs in Server Components. Wrap dynamic client-only content in `<Suspense>`.
2. **Server/Client boundary**: Importing a module with `"use client"` makes the ENTIRE import tree client-side. Keep client components as leaf nodes.
3. **Middleware auth**: `middleware.ts` runs on EVERY request (including static assets). Use matchers: `export const config = { matcher: ['/((?!_next/static|favicon.ico).*)'] }`.
4. **Server Actions + revalidation**: Always call `revalidatePath()` or `revalidateTag()` after mutations, or the UI shows stale data.
5. **Prisma in dev**: Hot reload creates multiple Prisma clients. Use the global singleton pattern in `lib/db.ts`.
6. **`fetch` caching**: Next.js 15 does NOT cache `fetch()` by default (changed from 14). Add `{ cache: 'force-cache' }` or use `unstable_cache` explicitly when needed.
7. **Image optimization**: Always use `next/image` with explicit `width`/`height` or `fill`. Remote images require domain allowlist in `next.config.ts`.
8. **Route handler methods**: Export named functions (`GET`, `POST`), not default exports. Wrong casing silently fails.
