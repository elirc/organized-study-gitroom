# Maintainer Communication

Contributing to an active open-source project (Postiz is AGPL, fast-moving, commercially backed) is a specific skill: get help without outsourcing your thinking, and disagree without friction.

## Asking for help (without making them do your job)

Bad: "How does posting work?" Good: "I traced scheduling from `posts.controller` → `startWorkflow` → `postWorkflowV105`; I understand the durable sleep, but I can't see where a *repeat* post re-enters the pipeline — is [L417-L436](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L417-L436)'s child workflow the whole story?" — shows work, asks one precise thing, cites a line.

The formula: **what I'm trying to do · what I tried · what I observed · the specific gap · my current hypothesis.**

## Filing a bug (a reproducible one)

Use the repo's own debug capsule: superadmin `GET /posts/group/:group/debug-export` ([posts.controller L148-L158](../../../apps/backend/src/api/routes/posts.controller.ts#L148-L158)) produces a re-importable JSON of the failing post group + its errors — the ideal minimal repro attachment. Otherwise: environment (self-host vs cloud, which providers), exact steps, expected vs actual, the `Errors` row / Temporal workflow id, and the smallest input that triggers it.

## Proposing a feature

Lead with the problem and the user, not the solution: "trial users can lose scheduled posts silently when workflow start fails; here's evidence (R7); I'd like to add a transactional outbox — RFC attached." Link to the risk register / your RFC. Ask early whether the direction is welcome *before* writing 2000 lines.

## Respectful disagreement in review

- Separate facts from preferences: "this is an IDOR (fact)" vs "I'd name this differently (preference)" — hold the line on facts, yield on preferences.
- Assume the author had a reason; ask before asserting: "Is the missing org filter intentional here — is this only ever called internally?"
- Offer a path, not just a verdict: "...and here's the two-line fix / happy to pair."
- Disagree-and-commit: once the maintainer decides, support the decision (or escalate once, calmly, with new evidence).

## Templates

Question:
> Working on <ticket>. I traced <path> and understand <X>. I'm stuck on <precise gap> — my hypothesis is <Y>. Am I reading <file:line> right?

Feature proposal:
> Problem: <user-facing pain + evidence>. I'd like to <approach>. Before I build it — is this direction something you'd accept? RFC: <link>.

Bug report:
> <expected> vs <actual>. Repro: <steps / debug-export attached>. Env: <self-host/cloud, providers>. Signal: <Errors row / workflow id>. Smallest trigger: <input>.

Responding to review:
> Good catch on <X> — fixed in <sha>. On <Y>: I chose <Z> because <reason>; happy to change if you'd prefer <alternative>.

## What makes a maintainer reject (avoid these)

Drive-by refactors bundled with a fix; large PRs with no prior discussion; changes that break self-host (assuming Stripe/env); no tests where the risk is real; ignoring the layering; casual public-contract changes (webhook shape, workflow signature). Every module-06 ticket lists its own rejection risk — read it before you start.

Interview angle: open-source contribution stories are gold for behavioral rounds ("tell me about working with people you don't control"). The [STAR worksheets](../08-interview-prep/06-behavioral-star-stories.md) turn a real contribution here into a rehearsed answer for conflict/ambiguity/collaboration prompts.
