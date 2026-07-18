# Performance Thinking

Rule zero: **measure first.** Every hotspot below is a *hypothesis* until an EXPLAIN, a flamegraph, or a timing says otherwise. Interviewers reward "I'd measure X with Y" over "I'd optimize Z."

## Performance domains in this app

| Domain | Where it lives | Likely hotspots |
| --- | --- | --- |
| DB query | Prisma repositories | recurrence expansion, per-request authz counting, N+1 recursion |
| Server CPU | activities | sharp transcodes, whole-array JSON stringify |
| Network fan-out | activities/controllers | uncapped `Promise.all` over channels/webhooks |
| Workflow history | Temporal | fat activity payloads |
| Frontend render | composer/calendar | month-view row counts, zustand selector scope |
| Bundle | Next build | single dep universe, heavy libs (polotno, tiptap, chart.js) |

## Confirmed / high-suspicion hotspots (with anchors)

1. **Unbounded recurrence expansion** — [getPosts while-loop L204-L216](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L199-L219): a daily recurring post viewed over a year expands to ~365 synthetic rows *in JS*, per request; a 5-year-old one, thousands. No cap. Measure: time `getPosts` with a seeded old daily post over a wide range. Fix: cap occurrences to the requested window's start (don't iterate from `publishDate` in the far past).
2. **Authz counting queries per gated request** — [permissions.service L83-L122](../../../apps/backend/src/services/auth/permissions/permissions.service.ts#L83-L122): CHANNEL check fetches the whole integrations list to `.length` it; POSTS_PER_MONTH runs a count. On every `POST /posts` and channel op. Measure: query log count per request. Fix: `count()` instead of fetch-then-length; cache per-request; or denormalize counts.
3. **N+1 thread reads** — [getPostsRecursively](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L294-L322): one query per thread item. Fine at length 3. Fix already half-present: `getPostsByGroup` + `arrangePostsByGroup` does it in one query — prefer it for long threads.
4. **`childrenPost: true` over-include** — [getPost L351-L363](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L339-L365) side-loads children on every recursion level. Combined with #3, quadratic-ish on deep threads.
5. **Whole-array JSON round-trip for link resolution** — [updateTags L553-L575](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L553-L575): `JSON.stringify(entire posts array)` → regex → `JSON.parse` → recurse. O(payload) per publish, on the activity worker. Fix: operate on `content` fields directly.
6. **Uncapped fan-out** — [getIntegrationList](../../../apps/backend/src/api/routes/integrations.controller.ts#L88-L124) (per-channel async metadata), [sendWebhooks](../../../apps/orchestrator/src/activities/post.activity.ts#L310-L336). Fix: `p-limit`.
7. **sharp transcode on the read path** — [updateMedia PNG→JPEG L373-L423](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L373-L423): CPU + a network re-upload triggered by *reading* a post group. Fix: transcode once at upload, cache the result.
8. **Missing composite indexes** — the calendar query's multi-column filter served by single-column indexes ([schema Post L431-L445](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L431-L445)). Measure with EXPLAIN; add `(organizationId, publishDate)` (ticket J16).

## How to find each class

| Symptom | Tool | Where |
| --- | --- | --- |
| N+1 | Prisma query logging (`log: ['query']`) | count queries per request |
| Serial async | timing spans / Sentry performance | activities, controllers |
| Expensive renders | React DevTools Profiler | composer keystrokes, calendar scroll |
| Oversized bundle | `next build` output / bundle analyzer | frontend |
| Missing index | `EXPLAIN ANALYZE` | pgadmin (`:8081`) against seeded data |
| Unbounded query/loop | read + adversarial input | recurrence, `findFreeDateTimeRecursive` |

## Frontend specifics

- Month view renders many post rows; ensure keys are stable and rows memoized (check [calendar.tsx MonthView L416](../../../apps/frontend/src/components/launches/calendar.tsx#L416)).
- Zustand `useShallow` selectors keep composer re-renders scoped ([add.edit.modal L37-L47](../../../apps/frontend/src/components/new-launch/add.edit.modal.tsx#L36-L47)) — a per-keystroke render of the whole modal would be the anti-pattern.
- Bundle: `polotno`, `tiptap`, `chart.js`, `@copilotkit/*` are heavy — candidates for dynamic import; the single root dependency list makes it easy to pull something huge in accidentally.

Interview angle:
1. "How do you find a slow endpoint?" → measure (query log/APM) → is it DB, CPU, or network? → this repo gives one of each.
2. "What's an N+1 and how do you fix it?" → `getPostsRecursively` vs `getPostsByGroup` — the repo contains both the problem and the fix.
3. "How do you choose an index?" → from the top query's WHERE+ORDER BY; the calendar query is your example.

Drill: pick hotspot #1 or #2, write the exact measurement you'd take (the query log line or EXPLAIN), predict the result, then the one-line fix and its blast radius. That triplet — measure/predict/fix — is the performance interview in miniature.
