# Testing Strategy

## The situation

There are **zero test files** in this repo (no `*.spec.ts`, no `*.test.ts`). The root [jest.config.ts](../../../jest.config.ts) delegates to `@nx/jest`'s `getJestProjects()`, but there is no `nx.json` and `@nx/jest` isn't installed — so `pnpm test` can't even discover a project (`inferred`; not executed while authoring). Testing dependencies *are* present (jest, ts-jest, @testing-library/react, jest-mock-extended, vitest), so the intent existed; the wiring rotted.

This is common in fast-moving commercial OSS and it's a genuine teaching gift: you get to decide the strategy from scratch, which is exactly the "how would you introduce testing here?" interview question.

## The pyramid this codebase wants

| Layer | What belongs | Seams here | Effort |
| --- | --- | --- | --- |
| **Unit (pure)** | logic with no I/O | [minify/expand codec](../../../libraries/helpers/src/utils/posts.list.minify.ts), [safeStringify](../../../libraries/nestjs-libraries/src/integrations/social.abstract.ts#L35-L47), [checkScopes](../../../libraries/nestjs-libraries/src/integrations/social.abstract.ts#L175-L193), [arrangePostsByGroup](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L509-L525), [checkInternalPlug key-parsing](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L577-L617), each provider's `handleErrors` | trivial, zero mocks — **start here** |
| **Unit (service w/ mocked repo)** | rules that orchestrate | [PermissionsService.check](../../../apps/backend/src/services/auth/permissions/permissions.service.ts#L39-L172) with a mocked subscription/integration service; `mapTypeToPost` validation | `jest-mock-extended` (already a dep) shines |
| **Integration (DB)** | repository queries, org scoping | [editTag/deleteTag](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L804-L826) cross-org tests; `createOrUpdatePost` thread chaining | needs a test Postgres (Testcontainers or the compose DB) |
| **Workflow (Temporal test env)** | orchestration logic | `postWorkflowV105` happy path + refresh-retry, with mocked activities via `@temporalio/testing` | higher setup cost; high value |
| **E2E** | one golden path | schedule → publish (against a fake provider) | last; brittle |

## What to test — and what not to

- **Test the invariants that have no other guard**: tenant scoping (there's a live IDOR), thread all-or-nothing, state-transition legality, the minify/expand round-trip. These are exactly the places [the risk register](../09-reference/risk-register.md) flags — tests would have caught them.
- **Don't test**: framework behavior (Nest DI, Prisma itself), the 30 providers' live APIs (mock at the `fetch` boundary — [SocialAbstract.fetch](../../../libraries/nestjs-libraries/src/integrations/social.abstract.ts#L101-L173) is the seam), or trivial pass-through services ([posts.service delegation methods L1020-L1047](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L1020-L1047)).

## Fixtures, isolation, determinism

- **Time**: code uses `dayjs` and `new Date()` extensively ([findFreeDateTime](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L974-L1018), workflow sleeps). Inject a clock or use jest fake timers; never assert on `dayjs()` directly.
- **Randomness**: `makeId`, `Math.random` date-picking ([generatePostsDraft findTime L892-L910](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L892-L910)) — seed or stub.
- **DB isolation**: transaction-per-test rollback, or a fresh schema via the compose Postgres; the `prisma-reset` script ([package.json L36](../../../package.json#L36)) force-resets — usable for a test DB.
- **External I/O**: mock `fetch`/provider methods; never hit real social APIs in tests.
- **Flake sources to pre-empt**: unbounded `Promise.all`, timezone (tests must set `TZ=UTC` like prod — [main.ts L10](../../../apps/backend/src/main.ts#L10)), order-dependence in `arrangePostsByGroup`.

## Introduction plan (this is the interview answer)

1. Ship a working `jest.config.ts` for `libraries/nestjs-libraries` + `helpers` (ticket J17).
2. Cover the pure cores (10 tests, one afternoon, zero mocks) — instant regression value on the codec and error mappers.
3. Add the IDOR integration test *with* the editTag fix (ticket J2) — a test that fails before, passes after: the canonical way to introduce testing (fix a real bug, leave a guard).
4. Add one workflow test for the refresh-retry path — the highest-logic, least-covered code.
5. Wire CI to run them; only then start requiring tests on PRs.

Interview angle: "You join a team with no tests — what do you do?" → *not* "mandate 80% coverage." → this five-step plan: working config, zero-mock cores first, bug-fix-with-test, highest-risk logic next, CI, then policy. Pragmatism + risk-prioritization is the senior signal.

Drill: pick `posts.list.minify.ts`, read it, and write (on paper) the round-trip property test `expand(minify(x))` preserves every field the app reads. Then write one `handleErrors` table test for [x.provider](../../../libraries/nestjs-libraries/src/integrations/social/x.provider.ts#L47-L108). Estimate total time — that's your "first PR of tests."
