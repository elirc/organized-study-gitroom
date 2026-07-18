# Runtime & Tooling Map

## Package manager and scripts

pnpm only (enforced culturally via [CLAUDE.md L19](../../../CLAUDE.md#L19)). All meaningful verbs live in the root [package.json](../../../package.json#L13-L41):

| Verb | Script | Notes |
| --- | --- | --- |
| Dev (all) | `pnpm run dev` | extension+orchestrator+backend+frontend in parallel |
| Dev (web only) | `pnpm run dev-backend` | backend+frontend |
| Infra | `pnpm run dev:docker` | compose file above |
| Build | `pnpm run build` | workspace-concurrency=1, frontend→backend→orchestrator |
| Schema | `pnpm run prisma-generate` / `prisma-db-push` | `db push --accept-data-loss` — **no migrations** |
| Prod | `pnpm run pm2` | pm2-managed multi-process |
| Test | `pnpm test` | jest via `@nx/jest` projects — **broken/vestigial**: no nx.json, no spec files |

`postinstall` runs prisma generate — a fresh `pnpm install` needs no extra codegen step but *does* need the schema readable.

## Runtime boundaries (where code executes)

| Boundary | Code | Gotchas |
| --- | --- | --- |
| **Browser** | `apps/frontend/src/components/**` (`'use client'` everywhere) | Cookie parsing in [custom.fetch.func.ts L22-L44](../../../libraries/helpers/src/utils/custom.fetch.func.ts#L16-L44) guards `typeof document` |
| **Next server (SSR)** | `apps/frontend/src/app/**` server components ([(site)/layout.tsx](../../../apps/frontend/src/app/(app)/(site)/layout.tsx)), `/api/uploads` proxy route | Same helper files run in both runtimes — isomorphic discipline required |
| **Node API** | `apps/backend` | `TZ=UTC` forced ([main.ts L10](../../../apps/backend/src/main.ts#L10)); `--experimental-require-module` in prod start |
| **Temporal workflow sandbox** | `apps/orchestrator/src/workflows/**` | Deterministic subset of JS — no I/O, no ambient Date/random (Temporal patches them); code here is *replayed* |
| **Temporal activity (plain Node)** | `apps/orchestrator/src/activities/**` | Full Nest DI, Prisma, fetch |
| **Chrome extension** | `apps/extension` | webextension-polyfill; cookies harvested for extension-type providers |

The same `libraries/` file may be imported by three of these — check its imports before assuming it's safe (pure helpers like [timer.ts](../../../libraries/helpers/src/utils/timer.ts) yes; anything with Prisma no).

## Frameworks

- **NestJS 11** (backend + orchestrator): DI container, decorators, guards/pipes/filters. Middleware applied per-controller-list, not per-route ([api.module.ts L107-L109](../../../apps/backend/src/api/api.module.ts#L106-L110)).
- **Temporal** via `nestjs-temporal-core`: worker auto-discovers `@Activity()` classes; workflows bundled separately (see [temporal.module.ts](../../../libraries/nestjs-libraries/src/temporal/temporal.module.ts)).
- **Next.js 16 App Router** (react 19): route groups `(app)/(site)`; heavy `'use client'` usage — this is mostly a client-rendered app wearing Next clothes; data via SWR not server components.
- **Prisma 6**: client generated to default location; repository classes wrap a thin [PrismaRepository](../../../libraries/nestjs-libraries/src/database/prisma/prisma.service.ts) generic.
- **Tailwind**: root deps pin tailwind 3.4.17 *and* include `@tailwindcss/postcss` 4.x + `@tailwindcss/vite` ([package.json L94, L236, L266](../../../package.json#L94)) — version straddle; CLAUDE.md says "tailwind 3" and deprecates `--color-custom*` vars.

## Env vars that shape behavior (high level, no secrets)

| Var | Effect | Anchor |
| --- | --- | --- |
| `STRIPE_PUBLISHABLE_KEY` / `STRIPE_SECRET_KEY` | Billing on; without them: PRO-for-free, no subscription gates | [permissions.service.ts L26-L28](../../../apps/backend/src/services/auth/permissions/permissions.service.ts#L21-L37), [post.activity.ts L115](../../../apps/orchestrator/src/activities/post.activity.ts#L113-L124) |
| `RUN_CRON` | Enables the missed-post sweep (exactly one instance should set it) | [infinite.workflow.register.ts L9](../../../libraries/nestjs-libraries/src/temporal/infinite.workflow.register.ts#L8-L19) |
| `NOT_SECURED` | Disables secure/httpOnly cookies + exposes auth headers (http self-host) | [auth.middleware.ts L14-L20](../../../apps/backend/src/services/auth/auth.middleware.ts#L11-L25), [main.ts L26-L40](../../../apps/backend/src/main.ts#L22-L48) |
| `STORAGE_PROVIDER` (`local`/`cloudflare`) | Upload backend | [upload.factory.ts](../../../libraries/nestjs-libraries/src/upload/upload.factory.ts#L5-L20) |
| `API_LIMIT` | Requests/hour throttle (default 90) | [app.module.ts L35-L43](../../../apps/backend/src/app.module.ts#L35-L43) |
| `DISABLE_REGISTRATION`, `DISALLOW_PLUS` | Signup policy | [auth.service.ts L24-L48](../../../apps/backend/src/services/auth/auth.service.ts#L24-L48) |
| `FRONTEND_URL`, `MAIN_URL`, `NEXT_PUBLIC_BACKEND_URL`, `BACKEND_INTERNAL_URL` | CORS origin, cookie domain, link generation — several features break subtly if these disagree | [main.ts L42-L46](../../../apps/backend/src/main.ts#L22-L48), media URL assembly in [updateMedia](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L356-L371) |

A startup config checker warns about common misconfigurations ([main.ts L83-L97](../../../apps/backend/src/main.ts#L83-L97), [configuration.checker.ts](../../../libraries/helpers/src/configuration/configuration.checker.ts)) — read it before writing your own env docs.

Full command list with verified/inferred labels: [09-reference/command-cheatsheet.md](../09-reference/command-cheatsheet.md).
