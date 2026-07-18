# Writing PRs & RFCs

## PR description template (use for every ticket in module 06)

```
## What
One sentence: the change.

## Why
The problem it solves. Link the issue/risk (e.g. R3 in the risk register).

## How
Approach in 2–4 bullets. Note the pattern you followed (e.g. "mirrors deleteTag's org scoping").

## Testing
What you ran/added. If the repo has no test for this area, say so and add one (or explain why not).
For this repo, always include a manual repro: "two orgs, tag from A, PUT from B → 403".

## Risk & blast radius
What could break; who's affected; is it behind a flag?

## Rollback
How to undo (revert build? / feature flag off? / migration down?).

## Follow-ups
Deliberately out of scope.
```

Why each field: maintainers merge on *confidence*, and confidence comes from you having thought about risk and rollback, not from the diff being clever. The "Testing" + "Rollback" fields are what distinguish a mid-level PR.

## Commit messages

- Imperative subject ≤ 50 chars: `Scope editTag to organization`.
- Body: *why*, not what (the diff shows what). Reference the risk/ticket.
- One logical change per commit; don't mix the fix and a drive-by rename.

## When to RFC instead of just PR

RFC when the change: touches a public contract (webhook payload, SDK, workflow signature), crosses many files (M2 tenant isolation, S3 outbox), changes the data model non-trivially, or has more than one reasonable approach. A 30-line bug fix does not need an RFC; a new eventing pattern does.

## RFC template (tailored to this repo)

```
# RFC: <title>

## Problem
What's broken/missing, with evidence (file:line anchors, the risk id).

## Invariant at stake
The property we must preserve or establish (e.g. "a QUEUE'd post eventually
publishes or becomes a visible ERROR"; "no cross-org data access").

## Options
For each: sketch, pros, cons, blast radius.
  Option A: <e.g. Prisma client extension>
  Option B: <e.g. Postgres RLS>
  Option C: <do nothing / minimal>

## Recommendation
Which and why. Address the trade-off you're accepting.

## Migration plan
Expand-contract steps across the 3 processes; how running Temporal workflows
are handled (versioning); the backfill.

## Test plan
What proves it works (the recipe/framework); the metric that confirms success
in prod (e.g. "sweep repair count → 0").

## Rollout & rollback
Flag name; staged rollout; exact rollback steps.

## Open questions
The things you genuinely don't know yet.
```

## This repo's specific gotchas to always address in an RFC

- **No migrations / auto-push on deploy** ([package.json L16-L18](../../../package.json#L16-L18)): any schema change must spell out the migration strategy and rollback — you can't just "deploy the old version" once the schema changed.
- **Three processes, one schema**: backward-compatibility during the rolling window.
- **Temporal versioning**: workflow changes need a new version; say how old executions drain.
- **Self-host vs cloud**: does your change assume Stripe/env vars present? State the self-host behavior.

Interview angle: "How do you propose a big change?" → the RFC structure above, emphasizing invariant + migration + rollback. Show the artifact from a module-06 project and you're demonstrating, not claiming.

Drill: write the full RFC for project S1 (encrypt tokens) or S3 (outbox) using this template. It's reusable as a real PR *and* as a system-design talking point *and* as a STAR story — one artifact, three interview surfaces.
