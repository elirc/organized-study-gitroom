# Data Model & Persistence

Schema: [schema.prisma](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma), ~40 models, Postgres via Prisma 6.

## Core entities and relationships

```
Organization ─┬─< UserOrganization >─ User          role: USER|ADMIN|SUPERADMIN
              ├─< Integration (channel)  unique(orgId, internalId)
              ├─< Post                    state, group, parentPostId (thread chain)
              ├─< Media / Tags / Signatures / Webhooks / Sets / AutoPost / Customer
              ├── Subscription (1:1)      Stripe mirror
              └─< Credits                 AI usage
Post ─< Comments (team feedback) ; Post ─< Errors (publish failures) ; Post ─< TagsPosts >─ Tags
Marketplace: MessagesGroup ─< Messages ; Orders ─< OrderItems ; PayoutProblems  (dormant subsystem)
mastra_* : AI agent memory/traces (Prisma-managed but externally shaped)
```

Read the full ERD reasoning in [01-codebase-cartography/03-domain-glossary.md](../01-codebase-cartography/03-domain-glossary.md).

## Design decisions worth studying

- **Threads as adjacency list** (`parentPostId` self-relation, [schema L404, L425-L426](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L393-L446)) built via upsert-chaining ([repo L500-L561](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L500-L561)). Alternative: a `position int` column — simpler reads, no recursion. Trade-off card: [pattern 10](05-pattern-catalog.md).
- **Recurrence as `intervalInDays`, expanded at read** ([repo L199-L219](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L199-L219)) — no occurrence rows, so no per-occurrence state and an **unbounded expansion loop** for old recurring posts (a real DoS-shaped risk; [risk register](../09-reference/risk-register.md)).
- **JSON-in-String columns** (`Post.settings`, `Post.image`, `Integration.postingTimes/additionalSettings`) — schemaless flexibility inside a relational DB; Prisma's `Json` type was available but unused. Integrity lives in the DTO at write-time only.
- **Soft delete everywhere** (`deletedAt?` + index) — recoverability; requires disciplined read filters and `deletedAt` in unique keys (`Customer @@unique([orgId, name, deletedAt])` [L311](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L301-L312)).
- **Encryption asymmetry**: third-party apiKeys and custom-instance details are `fixedEncryption`-wrapped, but **social OAuth tokens are plaintext** (`Integration.token/refreshToken`, [L322-L325](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L314-L356); no encryption call in [integration.repository.ts](../../../libraries/nestjs-libraries/src/database/prisma/integrations/integration.repository.ts)). The highest-value secrets are the least protected — top of the risk register.

## Indexes vs query shapes

The schema is index-rich (nearly every FK and filter column, [Post L431-L445](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L431-L445)) but **single-column**. The hottest query filters `organizationId + publishDate + deletedAt + parentPostId` ([getPosts L134-L172](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L129-L172)); Postgres can bitmap-combine single indexes but a composite `(organizationId, publishDate)` would serve the calendar better (ticket J16, with EXPLAIN evidence). Interview gold: "how do you choose indexes?" → from the `WHERE`+`ORDER BY` of your top queries, not per-column reflex.

## Transactions — the notable absence

`createOrUpdatePost` ([L488-L631](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L488-L631)) performs, sequentially and **without `$transaction`**: N post upserts, a tags delete, a tags create, and an old-group soft-delete. A crash between steps leaves a partial thread or a group with mixed delete state. `changeState` + `Errors` insert are similarly two independent writes ([L393-L428](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L393-L428)). Where the repo *does* use atomicity-ish patterns: `Promise.all([findMany, count])` for pagination ([L254-L287](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L254-L287)) — parallel, not transactional (count can race a concurrent insert; usually fine).
Senior framing: transactions protect **invariants that span rows**. "A thread is all-or-nothing" is such an invariant; it's currently unprotected. The fix is `prisma.$transaction(async tx => …)` around the loop — small blast radius, high value (senior project S2).

## Changing the schema safely

The scary part: this repo uses `prisma db push` ([package.json L33-L36](../../../package.json#L33-L36)), and prod `pm2` **pushes on every deploy** ([L16-L18](../../../package.json#L16-L18)). There are **no migration files** — no history, no reviewable diff, and `--accept-data-loss` will drop columns silently.

Safe-change playbook (what you'd propose):
1. Adopt `prisma migrate dev`/`deploy`; commit migrations; stop auto-push in prod.
2. Expand-contract for renames/removals: add new → backfill → dual-write → switch reads → drop old, across deploys (because three processes deploy independently).
3. Never widen a NOT NULL without a default; never drop a column still read by an un-deployed process.
4. Test the migration against a seeded copy; rehearse rollback.

Interview angle:
1. "Walk me through a zero-downtime column rename" → expand-contract, with this repo's multi-process deploy as the reason you can't do it in one step.
2. "Soft vs hard delete?" → recoverability vs GDPR vs unique-constraint complications, all present here.
3. "When do you reach for a transaction?" → the thread-creation invariant.

Drill: write the `$transaction` wrapper for `createOrUpdatePost` and identify the one operation that *can't* go inside it (hint: the workflow start — it's not a DB op and must happen *after* commit, which is exactly why it's in the *service*, not the repository — [posts.service.ts L777-L798](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L777-L798)). Explain why that ordering is correct.
Self-grade — Strong: you noted that starting the workflow before commit would let the worker read a not-yet-committed post (the transactional-outbox motivation).
