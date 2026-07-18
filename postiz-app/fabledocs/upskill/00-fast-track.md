# Fast Track: One Weekend

Goal: a running system, one end-to-end trace you can explain aloud, one safe change, and the vocabulary to keep going. Everything deeper is deferred — the "what this skips" list is at the bottom.

## Saturday morning — run it

All commands `inferred` from [package.json](../../package.json#L13-L41) and [docker-compose.dev.yaml](../../docker-compose.dev.yaml) (not executed while authoring — see [verification log](09-reference/verification-log.md)). Node `>=22.12.0 <23` and pnpm 10 are pinned in [package.json](../../package.json#L9-L12).

```bash
cp .env.example .env               # then fill DATABASE_URL, REDIS_URL, JWT_SECRET, FRONTEND_URL, NEXT_PUBLIC_BACKEND_URL, BACKEND_INTERNAL_URL
pnpm install                       # postinstall runs prisma generate
pnpm run dev:docker                # Postgres :5432, Redis :6379, Temporal :7233 + UI :8080, pgadmin :8081
pnpm run prisma-db-push            # push schema (no migration files exist — see note below)
pnpm run dev                       # backend :3000, frontend :4200, orchestrator :3002, extension — in parallel
```

Useful sub-targets: `pnpm run dev-backend` (backend+frontend only). The Temporal Web UI at `http://localhost:8080` is your window into every scheduled post — open it now, you'll need it Sunday.

**Pause and predict**: before opening the code — where do you think "publish this post at 5pm Tuesday" is implemented? A cron loop? A `setTimeout`? A DB poller? Write your guess down.

## Saturday afternoon — first ten files, in order

| # | File | What to look for |
| --- | --- | --- |
| 1 | [CLAUDE.md](../../CLAUDE.md) | The repo's own rules: Controller → Service → Repository, logic lives in `libraries/`, pnpm only. Also note what's stale (it says Vite; the frontend is Next.js — [apps/frontend/package.json](../../apps/frontend/package.json)) |
| 2 | [package.json](../../package.json#L13-L41) | Scripts = the repo's verbs. One root dependency list for the whole monorepo |
| 3 | [apps/backend/src/main.ts](../../apps/backend/src/main.ts#L22-L67) | Bootstrap: CORS allowlist, global `ValidationPipe`, 50 MB body limit only for `/posts` and `/copilot` |
| 4 | [apps/backend/src/api/api.module.ts](../../apps/backend/src/api/api.module.ts#L48-L110) | The `authenticatedController` array + `configure()` — this is the **auth boundary**, one list to rule them all |
| 5 | [auth.middleware.ts](../../apps/backend/src/services/auth/auth.middleware.ts#L33-L104) | JWT from cookie/header, org selection via `showorg`, super-admin impersonation |
| 6 | [schema.prisma](../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L393-L446) | The `Post` model: `state`, `group`, `parentPostId` (threads are a linked list), `settings`/`image` are **JSON-in-String** columns |
| 7 | [posts.controller.ts](../../apps/backend/src/api/routes/posts.controller.ts#L170-L179) | Thin controller; note `@Body() rawBody: any` on the most important endpoint |
| 8 | [posts.service.ts](../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L750-L808) | `createPost`: shortlinks, repository write, then hands off to Temporal |
| 9 | [post.workflow.v1.0.5.ts](../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L52-L102) | The answer to your prediction: a **Temporal workflow** `sleep()`s until publish time — durable, survives restarts |
| 10 | [x.provider.ts](../../libraries/nestjs-libraries/src/integrations/social/x.provider.ts#L26-L45) | One of 30+ providers implementing one interface; `handleErrors` maps X's error strings to a shared taxonomy |

## Sunday — trace, change, test, teach

**Trace 1 (scheduling)**: UI composer ([add.edit.modal.tsx](../../apps/frontend/src/components/new-launch/add.edit.modal.tsx#L36-L60)) → `POST /posts` → [mapTypeToPost](../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L243-L292) (manual DTO validation with a per-provider discriminator) → [createOrUpdatePost](../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L488-L631) → [startWorkflow](../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L690-L748) → Temporal sleeps → [postSocial activity](../../apps/orchestrator/src/activities/post.activity.ts#L203-L264) → provider `post()`. Full version with a step table: [01-codebase-cartography/05-key-flows.md](01-codebase-cartography/05-key-flows.md).

**Trace 2 (channel connect)**: "Add channel" → [getIntegrationUrl](../../apps/backend/src/api/routes/integrations.controller.ts#L193-L254) writes OAuth state to Redis → user consents at the provider → callback hits [connectSocialMedia](../../apps/backend/src/api/routes/no.auth.integrations.controller.ts#L42-L307), which redeems the Redis state to find the org. Notice: the *state token* is the credential on the way back.

**One small safe change**: in [posts.controller.ts](../../apps/backend/src/api/routes/posts.controller.ts#L176) delete the `console.log(JSON.stringify(rawBody, null, 2))` line (it logs every user's full post content to stdout). Run the app, schedule a post, confirm nothing else changed. This is ticket J1 in [06-contribution-practice/01-good-first-tickets.md](06-contribution-practice/01-good-first-tickets.md).

**One test to run**: `pnpm test` (`inferred`). Expect it to find **nothing** — there are zero `*.spec.ts` files in the repo and the jest config references `@nx/jest` with no `nx.json` present. That's not a dead end, it's the headline of [05-quality-engineering/01-testing-strategy.md](05-quality-engineering/01-testing-strategy.md).

**Teach-back (do this aloud, 3 minutes, as if an interviewer asked "walk me through a system you know")**: explain how a scheduled post gets published — the API writes rows and starts a workflow, the workflow durably sleeps, activities call provider APIs with retry/refresh semantics, failures mark the post `ERROR` and notify the user. If you can't say why Temporal instead of cron, read [08-interview-prep/04-system-design-from-this-repo.md](08-interview-prep/04-system-design-from-this-repo.md) next.

## What the fast track skips

Billing/CASL plan limits, the marketplace models (Orders/Messages), plugs (post-publish automations), the AI agent stack (Mastra/LangGraph/CopilotKit), the Chrome extension and public API, uploads, the missing-posts recovery cron, and all security analysis. They're covered in modules 01, 03, and 05.

Schema-change warning before you experiment: `prisma-db-push` (used by this repo — [package.json](../../package.json#L34)) has **no migration files**; `db push --accept-data-loss` will happily drop columns. See [03-architecture-and-patterns/02-data-model-and-persistence.md](03-architecture-and-patterns/02-data-model-and-persistence.md).
