# Observability & Operations

The operating question: **"how would I know this broke, and how fast?"** — asked per flow.

## What's wired

- **Sentry** in all three processes ([backend main.ts L1-L2](../../../apps/backend/src/main.ts#L1-L2), [orchestrator main.ts L1-L2](../../../apps/orchestrator/src/main.ts#L1-L2), frontend sentry configs); custom metrics like `Sentry.metrics.count('post_created', 1)` ([posts.service L800](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L800)).
- **Temporal Web UI** (`:8080`) — per-workflow history, activity failures with stack traces, searchable by `postId`/`organizationId` ([search attributes](../../../libraries/nestjs-libraries/src/temporal/temporal.search.attribute.ts)). The richest operational surface; treat it as your primary dashboard.
- **`Errors` table + admin UI** — publish failures persist message/platform/body ([changeState L413-L425](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L393-L428)); viewable at `/admin/errors`.
- **Health endpoints** — orchestrator serves a health check ([orchestrator main.ts L14-L18](../../../apps/orchestrator/src/main.ts#L13-L22), [health.controller.ts](../../../apps/orchestrator/src/health.controller.ts)); Temporal/DB/Redis liveness via their own tooling (pgadmin `:8081`, redisinsight `:5540`).
- **Startup config checker** — warns on common misconfigs at boot ([main.ts L83-L97](../../../apps/backend/src/main.ts#L83-L97)).
- **Exception filters** — [HttpExceptionFilter + Sentry filter](../../../apps/backend/src/main.ts#L64-L65) shape error responses and capture.

## Per-flow "how would I know?"

| Flow | Failure signal today | Gap |
| --- | --- | --- |
| Schedule → publish (Flow 1) | Temporal history; `state=ERROR`; `Errors` row; user notification | workflow-start failures are **swallowed** ([posts.service L747, L797](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L747)) — no metric; you'd only notice via the sweep or a user complaint (ticket J13 adds a metric) |
| Channel connect (Flow 2) | `{ err: true }` return; provider errors logged | silent-ish; no metric on connect-failure rate |
| Token refresh (Flow 4/scenario 4) | `refreshNeeded` flag; notification | no alert on refresh-failure spikes (e.g. a provider API change breaking all X refreshes at once) |
| Outgoing webhooks (Flow 4) | **none** — errors swallowed per-hook ([activity L331-L334](../../../apps/orchestrator/src/activities/post.activity.ts#L321-L335)) | no delivery log, no success rate, no dead-letter (R9) |
| Missed-post sweep (Flow 5) | runs hourly *if* `RUN_CRON` set | no metric on how many it repairs — a rising repair count is a leading indicator something upstream is broken, and nobody's watching it |
| Stripe (Flow 6) | Sentry on throw; Stripe dashboard | verify redelivery/idempotency (investigate) |

## The observability gaps that matter most

1. **Swallowed workflow-start failures** — the one place the DB and scheduler diverge has no signal. Fix: metric + Sentry breadcrumb (J13).
2. **Webhook delivery is a black hole** — add a delivery record + success-rate metric (R9). "We fired it" ≠ "they got it."
3. **No sweep-repair metric** — instrument `searchForMissingThreeHoursPosts` to emit "repaired N" so a spike alerts you to upstream breakage.
4. **Refresh-failure rate** — a provider changing its API breaks refresh *en masse*; per-provider failure counters would catch it before users do.

Good instinct to name in interviews: **the most important metric is often a leading indicator of a hidden failure, not the failure itself** — here, "posts repaired by the sweep" and "webhook delivery success rate" tell you about problems before the error count does.

## Operations: deploy & rollback

- Prod runs under **pm2** (`pnpm run pm2`: delete-all → `prisma-db-push` → parallel starts → logs — [package.json L16-L18](../../../package.json#L16-L18)). Two things to flag:
  1. **Schema pushed on every deploy** with `--accept-data-loss` — a bad schema edit is a data-loss event with no migration to review or revert. Rollback = restore from backup, not "deploy previous version" (the schema already changed). This is the single scariest operational property (R4).
  2. **Three independently-deployable processes** sharing one schema/dep universe — a schema change must be backward-compatible during the rolling window, or the not-yet-updated process breaks.
- **Temporal workflow versioning**: you can't hot-swap running workflows; new code = new workflow version (v1.0.1…v1.0.5). Deploys must keep old versions runnable until their executions drain.

## Rollback playbook (what you'd write)

1. Code-only change → redeploy previous build under pm2.
2. Schema change → **no clean rollback** currently; adopt `prisma migrate` with reversible migrations first (see [data model doc](../03-architecture-and-patterns/02-data-model-and-persistence.md)).
3. Workflow change → ensure the prior version is still bundled; drain or terminate stuck executions via Temporal UI.
4. Bad provider deploy → disable the channel/provider via the disabled flag; the sweep won't touch `disabled` integrations.

Interview angle:
1. "How would you know if publishing broke for one provider?" → per-provider error/refresh metrics; Temporal history; `Errors` table filtered by platform.
2. "What's a leading vs lagging indicator?" → sweep-repair count vs error count.
3. "How do you do zero-downtime deploys with a schema change?" → expand-contract + migrations + version-aware workflows; explain why this repo's auto-push makes it hard.

Drill: pick the webhook flow and design its observability from scratch — what three metrics, what one alert threshold, what dashboard panel. Then diff against what exists (nothing). That gap is a real, mergeable improvement (senior project S4 territory).
