# Code Review Mindset

## The five review layers (apply in order)

1. **Does it work?** — does it do what the PR says? (run it if you can)
2. **Is it correct?** — edge cases, nulls, concurrency, the failure paths.
3. **Will it stay correct?** — tests, invariants, what a future edit could break.
4. **Does it fit?** — the repo's layering ([CLAUDE.md](../../../CLAUDE.md#L31-L37)), naming, existing patterns.
5. **Is it kind to future maintainers?** — readable, documented where non-obvious, no surprises.

Most juniors stop at layer 1–2. Mid-level reviewers live at 3–4. The senior signal is catching a layer-3 problem ("this works now but the next person who adds a provider will forget X") kindly.

## Repo-specific review checklist

Run this mentally on every PR (it's the [security checklist](../05-quality-engineering/05-security-checklist.md) plus fit):

- **Tenant scope**: new query has `organizationId` in `where`? (the live IDOR at [editTag](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L804-L814) is why this is #1)
- **Auth boundary**: new controller in [authenticatedController](../../../apps/backend/src/api/api.module.ts#L48-L68)?
- **Layering**: business logic in the service, not the controller or repository?
- **`any` bodies**: typed DTO instead?
- **Side effects in retried units**: idempotent/signed? ([webhook pattern](../../../apps/orchestrator/src/activities/post.activity.ts#L310-L336))
- **Swallowed errors**: is `catch {}` hiding something that needs a signal?
- **Temporal**: workflow code change → new version, not a mutation? Activity rename → breaks running workflows?
- **Transactions**: multi-row invariant without `$transaction`?
- **Migrations**: schema change → is it backward-compatible for a rolling deploy? (there are no migration files — extra scrutiny)
- **Tests**: given there are none, does this at least add one for the risky part?

## Example comments (specific + kind)

Weak: "This is wrong." Better:
> Blocking: `where: { id }` here lets another org edit this row (IDOR) — `deleteTag` a few lines down shows the org-scoped pattern. Mind mirroring it and adding a two-org test?

Weak: "Use a transaction." Better:
> This does four writes that should be all-or-nothing (a thread shouldn't half-exist). Wrapping the loop in `prisma.$transaction` fixes it — the workflow start stays outside since it must run after commit. Happy to pair.

Praise matters too:
> Nice — sniffing magic bytes instead of trusting the mimetype is exactly right here.

## Reviewing *your own* PR first

Before requesting review, read your diff as if it's someone else's: does the description explain *why*? Did you leave a `console.log`? Is there a test? A self-review comment ("I left X unhandled because Y — thoughts?") preempts a round-trip.

Interview angle: "How do you review code?" → the five layers, then a concrete catch (the IDOR) and how you'd phrase it kindly. "Tell me about a time you disagreed in review" → [behavioral stories](../08-interview-prep/06-behavioral-star-stories.md). Interviewers want to see you catch *correctness* and communicate *kindly* — both, not either.

Drill: take any [review kata](../04-code-reading-gym/04-review-katas.md), write your comments, then rewrite each to be more specific and kinder. The rewrite is the skill.
