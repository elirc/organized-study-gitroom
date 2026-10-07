# Verification Log

Running log of what was inspected while authoring this curriculum. Every anchor in the curriculum was written against files opened during this session. Nothing in this repo was executed except read-only shell commands (`ls`, `find`, `grep`, `wc`, `cat`, `sed`); **no install, build, dev server, or test run was performed** — all commands in the curriculum are marked `verified` (ran) or `inferred` (read from scripts/configs but not executed) accordingly.

Date: 2026-07-10. Environment: Windows 11, repo at `03 Master Projects\masterpostiz\postiz-app`, no `.git` directory of its own (clone without history).

## Files fully read

| File | Why |
| --- | --- |
| `CLAUDE.md` | Repo's own contributor instructions (notes drift: calls the frontend "Vite ReactJS"; `apps/frontend/package.json` runs `next dev`) |
| `package.json` (root) | Scripts, dependency inventory, engines, pnpm config |
| `apps/backend/src/main.ts` | Bootstrap, CORS, global pipes/filters |
| `apps/backend/src/app.module.ts` | Global guards (throttler, policies), module graph |
| `apps/backend/src/api/api.module.ts` | Auth middleware wiring, authenticated controller list |
| `apps/backend/src/services/auth/auth.middleware.ts` | Cookie/JWT auth, impersonation, org selection |
| `apps/backend/src/services/auth/auth.service.ts` | Register/login/forgot/activate flows |
| `apps/backend/src/services/auth/permissions/permissions.service.ts` | CASL ability construction, plan limits |
| `apps/backend/src/services/auth/permissions/permissions.guard.ts` | Global guard, path bypasses |
| `apps/backend/src/services/auth/permissions/permissions.ability.ts` | `CheckPolicies` decorator |
| `apps/backend/src/api/routes/posts.controller.ts` | Posts API surface |
| `apps/backend/src/api/routes/integrations.controller.ts` | Channel management + OAuth URL generation |
| `apps/backend/src/api/routes/no.auth.integrations.controller.ts` | OAuth callback / social-connect |
| `apps/backend/src/api/routes/stripe.controller.ts` | Stripe webhook |
| `apps/backend/src/api/routes/media.controller.ts` | Upload endpoints |
| `libraries/helpers/src/auth/auth.service.ts` | JWT sign/verify, bcrypt, legacy AES helpers |
| `libraries/helpers/src/utils/custom.fetch.tsx` + `custom.fetch.func.ts` | Frontend fetch layer |
| `libraries/nestjs-libraries/src/database/prisma/schema.prisma` | Full schema (970 lines) |
| `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts` | Full 1048 lines |
| `libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts` | Full 898 lines |
| `libraries/nestjs-libraries/src/integrations/integration.manager.ts` | Provider registry |
| `libraries/nestjs-libraries/src/integrations/social.abstract.ts` | Shared fetch/retry/error taxonomy |
| `libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts` | Validation contract |
| `apps/orchestrator/src/main.ts` | Worker bootstrap |
| `apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts` | Full posting workflow |
| `apps/orchestrator/src/activities/post.activity.ts` | Full activity implementations |
| `apps/frontend/src/components/new-launch/add.edit.modal.tsx` | Post composer entry |
| `docker-compose.dev.yaml` | Dev infrastructure |

## Files partially read (targeted excerpts)

| File | What was read |
| --- | --- |
| `libraries/nestjs-libraries/src/integrations/social/x.provider.ts` | Lines 1–130 + method map via grep (`post` L427, `comment` L511, `analytics` L615, `postAnalytics` L705) |
| `libraries/nestjs-libraries/src/database/prisma/subscriptions/pricing.ts` | Lines 1–50 (interface + FREE/STANDARD tiers) |
| `apps/backend/src/public-api/routes/v1/public.integrations.controller.ts` | Lines 1–80 (MIME allowlist, SSRF-safe dispatcher import) |
| `apps/frontend/src/components/new-launch/store.ts` | Lines 1–60 (zustand store shape) |
| `apps/frontend/src/components/launches/calendar.context.tsx` | Lines 1–50 (SWR context) |
| `apps/frontend/src/components/launches/calendar.tsx` | Export map via grep (DayView L263, WeekView L344, MonthView L416, Calendar L567, CalendarColumn L583) |
| `apps/frontend/src/components/launches/launches.component.tsx` | grep for `useFetch`/`mutate` usage (L353, L364) |
| `apps/frontend/src/app/(app)/(site)/layout.tsx` | Full (9 lines) |
| `jest.config.ts` (root) | Full — references `@nx/jest`, but no `nx.json` exists |

## Commands run (all read-only, all succeeded unless noted)

| Command | Result |
| --- | --- |
| `ls` / `find` over `apps/*/src`, `libraries/*/src` | File inventory used throughout |
| `find . -name "*.spec.ts(x)" -not -path node_modules` | **Zero test files found in the entire repo** |
| `git rev-parse --is-inside-work-tree` inside `postiz-app` | `true` but **no `.git` in postiz-app itself** — it sits inside an unrelated outer repo; treat as history-less clone |
| `grep fixedEncryption` across repo | Used by oauth service, org repository, third-party repo, 3 providers — **not** by `integration.repository.ts` (social access tokens stored unencrypted) |
| `grep -n ThrottlerModule` in `app.module.ts` | L35–39: 90 requests/hour default |
| `wc -l` on ~20 key files | Sizes recorded before reading |

## Uncertainties and things NOT verified

- **No command execution**: `pnpm install`, `pnpm run dev`, `pnpm test`, `prisma-db-push`, Docker startup were **not run**. All such commands in `09-reference/command-cheatsheet.md` are marked `inferred`.
- **Behavioral claims about third-party APIs** (X rate limits, LinkedIn quirks) come from code comments and error-handler strings, not from live calls.
- `apps/extension`, `apps/sdk`, `apps/commands`, `libraries/react-shared-libraries`, `libraries/nestjs-libraries/src/chat|agent|videos|3rdparties` were inventoried but not read in depth; claims about them are kept minimal and labeled.
- The `docs2/` directory contains earlier, unrelated learning material; it was not used as a source.
- Frontend rendering behavior (SWR revalidation timing, calendar drag-and-drop) is described from code reading only.

## Confirmed-by-reading defects (used as teaching seams; each verified against the exact lines cited)

1. `editTag` updates `where: { id }` without `orgId` — cross-org write possible. `posts.repository.ts` L804–814 (contrast `deleteTag` L816–826 which scopes correctly).
2. `createPopularPosts` inserts the literal strings `'category'`, `'topic'`, `'content'`, `'hook'` instead of the argument values. `posts.repository.ts` L728–742.
3. `createOrUpdatePost` performs upserts, tag rewrites, and group soft-deletes with **no transaction**. `posts.repository.ts` L488–631.
4. `PostsController.createPost` takes `@Body() rawBody: any` and `console.log`s the full body. `posts.controller.ts` L170–179.
5. `AuthService.signJWT` signs with no `expiresIn`; session JWTs never expire. `libraries/helpers/src/auth/auth.service.ts` L42–44.
6. `PoliciesGuard` has a duplicated `/auth` path condition (copy-paste). `permissions.guard.ts` L25–26.
7. `PostsService.deletePost` always returns `{ error: true }` regardless of outcome. `posts.service.ts` L679.
8. Social `Integration.token` / `refreshToken` stored without encryption (schema L322, L325; no `fixedEncryption` call in `integration.repository.ts`).
9. CLAUDE.md says frontend is "Vite ReactJS"; it is Next.js 16 (`apps/frontend/package.json` dev script `next dev -p 4200`).
10. Password-reset JWT (`forgot`, `auth.service.ts` L217–233) has a custom `expires` claim checked manually and is not single-use.

## Re-check on 2026-10-07 (study copy `elirc/organized-study-gitroom`)

Static re-check of `fabledocs/upskill/` and `docs2/`; nothing was installed or run.

- **Defects 1–7 and 10 re-confirmed at the same lines.** `editTag` is at L804 and `deleteTag` at L816. `createPopularPosts` writes literal `'category'` at L736. Both `/auth` conditions sit at `permissions.guard.ts` L25–26. `deletePost` returns `{ error: true }` at L679. `rawBody` + `console.log` are at `posts.controller.ts` L174–177. `signJWT` at L42–44 calls `sign` with no options.
- **Anchors:** every `#L` link and `path:N` citation lands inside its file, with one exception: `permissions.ability.ts#L1-L8` pointed one line past a 7-line file and now reads `#L1-L7`. A content spot-check of the posts service/repository, guard, calendar and settings anchors found them on the named code. Many `docs2/` anchors deliberately point at the *insertion site* for a feature that does not exist yet (e.g. `privateNote`), so the symbol is absent by design.
- **Path fix:** `libraries/nestjs-libraries/database` → `libraries/nestjs-libraries/src/database/prisma/*` in `01-boundaries-and-layers.md`.
- **Test evidence:** still zero `*.spec.ts`/`*.test.ts` outside `node_modules`. The committed `reports/junit.xml` (11 `PermissionsService` cases, timestamp 2025-03-24) is a stale artifact of a spec that is not in this tree. R2 now says so.
- **New review contrast (R3):** `setNickname` scopes its read by org (`integrations.controller.ts` L144–150) before an id-only write (`integration.repository.ts` L365–375). It is safe today, and it is a useful comparison with `editTag`'s unscoped write.

Labeled "investigate" (not confirmed): behavior of `changeDate` when `getPostById` returns null (`posts.service.ts` L849–859 dereferences `.state` unguarded); `IntegrationManager.getSocialIntegration` non-null assertion on unknown identifier (`integration.manager.ts` L170–172); `functionIntegration` dispatching arbitrary `body.name` methods on providers (`integrations.controller.ts` L341–352).
