# Boundaries & Layers

A **boundary** is where one component's responsibility ends and its promises to others begin. This repo's declared layering ([CLAUDE.md L31-L37](../../../CLAUDE.md#L31-L37)): Controller → (Manager →) Service → Repository, with logic in `libraries/`.

## What each layer owns — and must not own

| Layer | Owns | Must not own | Good example | Leak |
| --- | --- | --- | --- | --- |
| Controller | HTTP shape, authz decorators, param extraction | queries, business rules, external calls | [posts.controller.ts](../../../apps/backend/src/api/routes/posts.controller.ts#L165-L179) — one-liners delegating | [deleteChannel](../../../apps/backend/src/api/routes/integrations.controller.ts#L402-L418) loops posts + orchestrates deletes; [connectSocialMedia](../../../apps/backend/src/api/routes/no.auth.integrations.controller.ts#L42-L307) is 260 lines of business flow in a controller; [setNickname](../../../apps/backend/src/api/routes/integrations.controller.ts#L138-L176) calls providers directly |
| Service | rules, orchestration across repos/providers/Temporal | raw HTTP concerns, SQL | [posts.service.ts createPost](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L750-L808) | services reading `process.env` scattered (billing flags) — config as ambient state |
| Repository | Prisma queries, where-clause discipline | rules, external calls | [posts.repository.ts changeDate](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L439-L464) | `createOrUpdatePost` embeds the *business* rule "tags attach to first post only" ([L563-L599](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L563-L599)) |
| Provider | one platform's API dialect, error mapping | Prisma, org logic | [x.provider.ts](../../../libraries/nestjs-libraries/src/integrations/social/x.provider.ts#L26-L45) | a few providers read env for behavior toggles (`STRIP_LINKS_FROM_X_POSTS` L34) — acceptable, but note it |
| Workflow | time, ordering, retry *decisions* | I/O | [post.workflow.v1.0.5.ts](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L52-L102) | clean — the strongest boundary in the repo, enforced by Temporal's sandbox rather than discipline (boundaries enforced by *tools* outlive boundaries enforced by *rules*) |
| Activity | I/O on behalf of workflows | long-lived state | [post.activity.ts](../../../apps/orchestrator/src/activities/post.activity.ts#L54-L66) | — |

## Boundary inventory (the ones that matter)

1. **Auth boundary** — membership in [authenticatedController](../../../apps/backend/src/api/api.module.ts#L48-L68). Fail-open: forget to list a controller and it ships unauthenticated. Senior fix: default-deny (apply middleware to `*`, opt out explicitly).
2. **Tenant boundary** — `orgId` in every repository where-clause. By convention; the [editTag lapse](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L804-L814) proves conventions leak.
3. **Determinism boundary** — workflow/activity split; tool-enforced.
4. **Provider boundary** — the typed-failure taxonomy ([social.abstract.ts](../../../libraries/nestjs-libraries/src/integrations/social.abstract.ts#L5-L27)); strings quarantined per provider.
5. **Process boundary** — backend/orchestrator share code but communicate only via Postgres/Redis/Temporal. Hidden coupling: both may refresh the same token concurrently (see [debugging scenario 4](../05-quality-engineering/03-systematic-debugging.md)).
6. **Ownership boundary in data** — `libraries/nestjs-libraries/database` modules pair service+repository per aggregate (posts/, integrations/, organizations/…): a proto-"domain module" layout worth naming in interviews.

## How to *use* this in review

Three questions per PR: Which boundary does this change cross? Does data crossing it get validated/scoped on arrival? Who else depends on the promise being changed? Practice on [review katas](../04-code-reading-gym/04-review-katas.md).

Interview angle: "How do you structure a backend so juniors can't break tenant isolation?" — the honest answer this repo teaches: layering alone doesn't; you need tool-enforced boundaries (RLS, query middleware, or generated scoped clients) for the invariants that matter most. Saying *that* — with the editTag example — is a senior answer.

Drill: pick [webhooks.controller.ts](../../../apps/backend/src/api/routes/webhooks.controller.ts) (unread by this curriculum). Predict its layering compliance from what you now know, then read it and grade your prediction.
