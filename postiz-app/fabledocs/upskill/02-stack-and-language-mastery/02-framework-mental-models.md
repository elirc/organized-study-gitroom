# Framework Mental Models

## NestJS: the request lifecycle as a pipeline

Mental model: `middleware → guards → interceptors → pipes → handler → filters`. Map it onto a real request, `POST /posts`:

1. **Middleware**: [AuthMiddleware](../../../apps/backend/src/services/auth/auth.middleware.ts#L33-L104) attaches `req.user`/`req.org` (only for controllers in the [authenticatedController list](../../../apps/backend/src/api/api.module.ts#L48-L68)).
2. **Guards** (global): [ThrottlerBehindProxyGuard](../../../apps/backend/src/app.module.ts#L47-L51) then [PoliciesGuard](../../../apps/backend/src/services/auth/permissions/permissions.guard.ts#L22-L64) reading `@CheckPolicies` metadata.
3. **Pipes**: global `ValidationPipe({transform: true})` ([main.ts L52-L56](../../../apps/backend/src/main.ts#L52-L56)) — inert here because the param is `any`; compare [SetsController or others with typed DTOs] where it actually validates.
4. **Handler** → service → repository ([CLAUDE.md's mandated layering](../../../CLAUDE.md#L31-L37)).
5. **Filters**: [SubscriptionExceptionFilter + HttpExceptionFilter](../../../apps/backend/src/main.ts#L64-L65) map exceptions to responses (`SubscriptionException` → upgrade prompt).

DI mental model: providers are singletons per module graph; `@Global()` on [AppModule](../../../apps/backend/src/app.module.ts#L21) makes the shared services visible everywhere. The same service classes are re-instantiated in the *orchestrator's* container — same code, two containers, two processes.

Sharp edges seen here: param decorators hide typing (`@GetOrgFromRequest()` returns `any`-ish — [org.from.request.ts](../../../libraries/nestjs-libraries/src/user/org.from.request.ts)); middleware ordering is by controller list membership, so adding a controller and forgetting the list = **unauthenticated endpoint** (fail-open design — the single most important review check in this repo).

## Next.js App Router, as actually used

This app uses Next 16 mostly as a **shell for a client app**: route groups organize layouts ([(app)/(site)](../../../apps/frontend/src/app/(app)/(site))), nearly every component is `'use client'`, and data flows through SWR against the separate NestJS API — not through server components/actions. SSR still matters for: cookies/headers at layout level, the uploads proxy route (`(app)/api/uploads`), and first-paint.

Consequences: no server-side data caching layers to reason about; hydration boundaries are where `typeof window` checks appear ([custom.fetch.func.ts L17-L44](../../../libraries/helpers/src/utils/custom.fetch.func.ts#L16-L44)). If an interviewer asks "RSC vs client components," you can honestly describe this architecture and *why it's legitimate*: a highly interactive, auth-gated dashboard gains little from RSC; the cost is bundle size.

## React state: the three-store split

- **Server state** → SWR: calendar posts, integrations list ([calendar.context.tsx](../../../apps/frontend/src/components/launches/calendar.context.tsx#L1-L50)); invalidation via `mutate` after writes ([launches.component.tsx L364-L394](../../../apps/frontend/src/components/launches/launches.component.tsx#L353-L394)). One hook per resource is house law ([CLAUDE.md L45-L58](../../../CLAUDE.md#L43-L58)) — because conditional hook calls break the rules of hooks.
- **Complex client state** → zustand: the composer ([store.ts](../../../apps/frontend/src/components/new-launch/store.ts#L28-L60)), selected via `useShallow` to keep re-renders scoped ([add.edit.modal.tsx L37-L47](../../../apps/frontend/src/components/new-launch/add.edit.modal.tsx#L36-L47)).
- **Local state** → `useState` in leaf components.

Render mental model checkpoints: a zustand selector re-renders only when its slice changes (not on every store write); the staged-mount pattern in the composer ([three nested components, each gating on store readiness](../../../apps/frontend/src/components/new-launch/add.edit.modal.tsx#L36-L107)) exists because hooks can't run conditionally — **the rules of hooks shaped the component tree**. Empty-dep `useEffect`s treat props as init-only; stale-closure risk is the classic follow-up.

## Temporal-as-framework

Think "NestJS for time": workflows are controllers for *durations*; activities are services; signals are webhooks into a running execution ([poke](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L48-L78)); task queues are routing ([per-provider queues](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L63-L73)); `workflowId` + conflict policy are your idempotency primitives ([signalWithStart usage](../../../apps/orchestrator/src/activities/post.activity.ts#L73-L106)).

Interview angle:
1. "Walk me through a request in your framework" → the 5-stage Nest pipeline with the `/posts` example (they're testing whether you know *where* validation/authz live, not Nest trivia).
2. "How do you decide between Redux/zustand/React Query?" → the three-store split above, with this repo as the worked example.
3. "What breaks if you call a hook conditionally?" → the composer's staged mounting is your concrete story.
4. "Server components — when not to bother?" → this dashboard's answer.

Drill: `GET /posts` — write the full pipeline it traverses (which middleware/guards actually fire, which are skipped and why). Then do the same for `POST /stripe`. The differences are the lesson.
Self-grade — Strong: you noted that `/stripe` skips AuthMiddleware (not in the list) but still passes through both global guards, and that `PoliciesGuard` allows it because it has no `@CheckPolicies` metadata ([guard L39-L41](../../../apps/backend/src/services/auth/permissions/permissions.guard.ts#L33-L41)).
