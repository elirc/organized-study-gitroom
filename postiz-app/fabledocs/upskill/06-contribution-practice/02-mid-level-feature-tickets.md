# Mid-Level Feature Tickets

Twelve cross-layer tickets (schema/API/service/UI/tests). Each **requires a design note before code** and has explicit risk + rollback. These are the "can you own a feature across the stack?" bar.

Format per ticket: Story · Design-note prompts · Layers touched · Risk · Rollback · Interview story.

---

## M1: Signed outgoing webhooks
Story: webhook receivers must verify the payload came from us.
Design note: HMAC-SHA256 over the raw body with a per-webhook secret; header scheme (`X-Postiz-Signature`, `X-Postiz-Timestamp`); replay window; where the secret is stored (add `Webhooks.secret`, encrypted like third-party keys).
Layers: schema (`Webhooks.secret`), [sendWebhooks activity](../../../apps/orchestrator/src/activities/post.activity.ts#L310-L336), webhooks controller/service, docs, test.
Risk: existing receivers break if verification is mandatory on their side — but signing is additive (they opt in). Rollback: feature-flag the header emission.
Interview story: "designed webhook signing for a multi-tenant system; additive rollout, no consumer breakage."

## M2: Structural tenant-scope enforcement
Story: make forgetting `organizationId` in a query impossible (fixes the class of bug, not one instance).
Design note: options — Prisma client extension that injects `organizationId` from an async context; vs Postgres RLS; vs a lint rule. Trade-offs (RLS = DB-enforced but harder to test locally; extension = app-enforced, needs request-scoped context). Pick one, justify.
Layers: [prisma.service.ts](../../../libraries/nestjs-libraries/src/database/prisma/prisma.service.ts), all repositories (audit), tests proving cross-org rejection.
Risk: high blast radius (touches every query) — do after M-tests exist; some queries are *intentionally* cross-org (workflow-internal) and must opt out.
Rollback: extension behind a flag; keep the old path until audited.
Interview story: "eliminated an entire IDOR class structurally instead of whack-a-mole."

## M3: Post analytics dashboard per channel
Story: show reach/likes/clicks over time per published post.
Design note: providers already expose `postAnalytics` ([x.provider L705](../../../libraries/nestjs-libraries/src/integrations/social/x.provider.ts#L705)); the service caches by date ([checkPostAnalytics](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L148-L229)). Design the aggregation + a chart (chart.js is a dep). Handle providers without analytics.
Layers: analytics controller/service, frontend analytics components, cache strategy, test.
Risk: rate limits on provider analytics endpoints — respect the existing 1h cache and per-provider concurrency.
Rollback: UI-only feature flag.
Interview story: "shipped analytics respecting third-party rate limits via existing cache primitives."

## M4: Draft → schedule promotion with full validation
Story: drafts skip settings validation ([@ValidateIf L71](../../../libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts#L71-L80)); scheduling one should re-validate against provider rules.
Design note: where to re-run the discriminated validation (`mapTypeToPost` already supports `replaceDraft` — [L246, L254](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L243-L292)); surface per-channel errors in the UI.
Layers: service, changePostStatus path, composer error UI, test.
Risk: a draft that was valid-as-draft may now fail to schedule — that's correct, but needs clear UX.
Rollback: keep old behavior behind a flag.
Interview story: "closed a data-quality gap where invalid drafts could become scheduled posts."

## M5: Cap recurrence expansion + occurrence model
Story: fix unbounded expansion ([getPosts L204-L216](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L199-L219)) and allow skipping an occurrence.
Design note: minimal fix (start iterating from the window, not `publishDate`) vs full model (materialize occurrence rows). Trade-offs. Migration if you add a table.
Layers: repository, possibly schema, calendar rendering, test.
Risk: changing calendar output — snapshot the expected occurrences first.
Rollback: minimal fix is low-risk; the model change is expand-contract.
Interview story: "found and fixed an unbounded query; weighed a quick fix vs a data-model change."

## M6: Idempotent provider posting
Story: retries must not duplicate posts.
Design note: persist `releaseId` the instant the provider returns; check it on retry entry ([workflow L164-L200](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L164-L200)); providers with native idempotency tokens use them.
Layers: workflow, activity, repository, test (the Temporal test env recipe 7).
Risk: subtle — a mid-flight change to the retry loop; test heavily.
Rollback: guard behind a workflow version (new v1.0.6, old still runs).
Interview story: "made at-least-once publishing safe with idempotency keys; the canonical distributed-systems fix."

## M7: Team member roles UI + enforcement
Story: ADMIN vs USER capabilities in the org (roles exist in schema + CASL but surface thinly).
Design note: which actions require ADMIN (delete channel? billing? invite?); enforce in guard/service; UI to manage members.
Layers: permissions service, controllers, settings UI, test.
Risk: over-restricting locks people out; under-restricting is a privilege bug.
Rollback: default everyone to current effective permissions.
Interview story: "designed RBAC surfacing for a team feature."

## M8: Webhook delivery log + retry
Story: record every delivery attempt, retry failures with backoff, expose in UI.
Design note: `WebhookDelivery` table (status, attempts, response code); a delivery activity with Temporal retry; dead-letter after N.
Layers: schema, activity, webhooks UI, test.
Risk: table growth — add retention. Pairs with M1 (sign then log).
Rollback: log-only first, retry later.
Interview story: "turned a fire-and-forget webhook into an observable, retriable delivery pipeline."

## M9: Per-org rate-limit tiers
Story: the flat 90/h ([app.module L38-L39](../../../apps/backend/src/app.module.ts#L35-L43)) should scale with plan.
Design note: throttler keyed by org+plan; where plan is read at guard time; cache.
Layers: throttler provider, permissions integration, test.
Risk: mis-keying = wrong limits; test the key derivation.
Rollback: revert to flat limit.
Interview story: "plan-aware rate limiting without a per-request DB hit."

## M10: Bulk reschedule (drag a day of posts)
Story: move all of a day's posts by an offset.
Design note: batch `changeDate` + workflow restart per post; transaction boundary; the calendar drag UX.
Layers: posts service/controller, calendar UI, test.
Risk: N workflow restarts — throttle; partial failure handling.
Rollback: UI feature flag.
Interview story: "batch operation touching persistence + durable scheduling + DnD UI."

## M11: Export a post group as debug capsule in the UI
Story: the superadmin-only `debug-export` ([posts.controller L148-L158](../../../apps/backend/src/api/routes/posts.controller.ts#L148-L158)) is powerful; give support staff a guarded button + import ([import-debug-post.modal](../../../apps/frontend/src/components/launches/import-debug-post.modal.tsx)).
Design note: authorization (who can export/import?); redaction of tokens; the round-trip contract.
Layers: controller authz, UI, test.
Risk: exports contain content — scope carefully; keep superadmin gate.
Interview story: "productized an internal debugging tool with the right guardrails."

## M12: Encrypt social tokens at rest (feature slice)
Story: implement R1 as a shippable feature (the senior version is project S1; this is the scoped ticket).
Design note: reuse `fixedEncryption` (or propose AEAD); encrypt at write, decrypt at every read (enumerate sites); idempotent backfill behind a flag.
Layers: integration repository, every token read site, backfill script, test.
Risk: a missed read site breaks a channel; double-encrypt on re-run.
Rollback: flag-gated; keep dual-read (try decrypt, fall back to plaintext) during migration.
Interview story: "encrypted the crown-jewel secrets with a safe, reversible migration."

---

Each ticket's design note should be ~1 page: problem, chosen approach + one rejected alternative, the invariant you're protecting, test plan, rollback. Write it *before* touching code — that discipline is the mid→senior differentiator. Template: [07-career-and-collaboration/02-writing-prs-and-rfcs.md](../07-career-and-collaboration/02-writing-prs-and-rfcs.md).
