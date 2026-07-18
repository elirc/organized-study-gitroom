# Language & Runtime Model

## The event loop, through this repo's eyes

Model: one JS thread per process; I/O returns via callbacks queued as macrotasks; promise continuations run as **microtasks** (all of them drain before the next macrotask). `await` = "suspend me, resume as a microtask when settled."

Where it matters here:
- **Serial when order is state** — the thread-building upsert must run one-at-a-time because each row `connect`s to the previous row's id ([posts.repository.ts L500-L561](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L500-L561)). Serialization is the *correctness mechanism*, not laziness.
- **Parallel when independent** — media resolution fans out with `Promise.all` ([posts.service.ts L343-L424](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L340-L437)); webhook fan-out too ([post.activity.ts L321-L335](../../../apps/orchestrator/src/activities/post.activity.ts#L310-L336)).
- **Unbounded parallel is a footgun** — `getIntegrationList` maps every channel through async provider metadata ([integrations.controller.ts L88-L124](../../../apps/backend/src/api/routes/integrations.controller.ts#L88-L124)); fine at 10 channels, thundering at 500. `p-limit` sits unused-for-this in the deps.
- **Blocking the loop** — `sharp` image conversion ([updateMedia L387-L389](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L373-L423)) does native work off-thread (good), but `JSON.stringify` of a whole post array for regex replacement ([updateTags L553-L575](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L553-L575)) is synchronous CPU on the activity worker.

Pitfall checklist: forgotten `await` (fire-and-forget *by accident* — contrast the deliberate one at [createPost L791-L798](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L790-L798) which at least has `.catch`); `await` inside `Array.map` (returns promises, not values — this repo always wraps in `Promise.all`, verify one: [getPostsByGroup L493-L502](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L486-L507)); async recursion without termination (see [findFreeDateTimeRecursive](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L995-L1018) — Q9 in the interview deck).

## Process model

Three cooperating Node processes (backend/orchestrator/frontend) sharing code but not memory — all cross-process coordination goes through Postgres, Redis, or Temporal. Nothing in-process is durable: that's why OAuth state lives in Redis, not a Map ([integrations.controller.ts L229-L248](../../../apps/backend/src/api/routes/integrations.controller.ts#L229-L248)), and why scheduling lives in Temporal, not `setTimeout`.

## The fourth runtime: Temporal's deterministic sandbox

Workflow code ([post.workflow.v1.0.5.ts](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts)) is **replayed** to rebuild state after worker restarts. Rules: no I/O (delegate to activities), no ambient nondeterminism (Temporal shims `Date`/`Math.random`/timers inside the sandbox), same code must produce the same decision sequence. Consequences you can see:
- `sleep()` from `@temporalio/workflow` is a durable timer, not `setTimeout` ([L96-L102](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L96-L102)).
- Activity results become workflow history — hence [slimPost](../../../apps/orchestrator/src/activities/post.activity.ts#L27-L52) trimming payloads.
- Changing workflow code under running instances breaks replay — hence the v1.0.1…v1.0.5 file series.

Interview angle:
1. "Explain the event loop / microtasks vs macrotasks" → answer with the serial-upsert vs Promise.all contrast above (Q1).
2. "How do you run something at a future time reliably?" → setTimeout dies with the process; queue delays vs durable timers (Q-scheduler in [04-system-design](../08-interview-prep/04-system-design-from-this-repo.md) Step 4).
3. "What is determinism and when does it matter?" → Temporal replay, plus test flakiness (same root concept).
4. "When would you *not* use Promise.all?" → unbounded fan-out, partial-failure semantics (`allSettled`), rate limits.

Drill: find every `await timer(…)` call site (grep `timer(` in `libraries/nestjs-libraries/src/integrations`) and classify each as: rate-limit courtesy, API eventual-consistency wait, or retry backoff. Which would you replace with Temporal sleeps, and why can't the ones inside providers be replaced? (They run in activities, where plain timers are fine but non-durable.)
Self-grade — Basic: classified them. Solid: articulated durable vs non-durable waiting. Strong: explained why a 10s `refreshWait` ([posts.service.ts L116-L118](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L96-L123)) blocks an HTTP request thread and what you'd do instead.
