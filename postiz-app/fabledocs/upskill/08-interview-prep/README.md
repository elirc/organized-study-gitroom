# 08 — Interview Prep

First-class module. Target: **mid-level fullstack JS interviews**. 60%+ of questions are anchored to real code here, so you practice answering with concrete evidence instead of memorized definitions.

## How mid-level fullstack loops are structured

1. **Recruiter/technical screen** — background, one or two concept questions.
2. **Technical deep-dive** — JS/TS/Node + framework; "explain X, then critique it."
3. **Practical coding** — implement/fix something, often small and real.
4. **System design** — design a product; trade-offs.
5. **Behavioral** — collaboration, conflict, ambiguity, mistakes.

## Use this repo as your portfolio of talking points

You studied a real production system. That means for almost any question you can say "I saw exactly this in a codebase — here's what they did, here's the trade-off, here's what I'd change." That beats textbook answers every time.

## The golden rule

Answer with a **concrete example + a trade-off + a failure mode**, never just a definition. "JWT is a signed token" is a screen-fail; "This app puts the whole user in a JWT with no expiry — stateless and simple, but no revocation and stale claims; I'd use short-lived access tokens + refresh" is a hire.

## Files

- [01-js-ts-node-deep-dive.md](01-js-ts-node-deep-dive.md) — 14 cards (event loop, promises, TS, Node).
- [02-frontend-framework-questions.md](02-frontend-framework-questions.md) — 12 cards (React/Next/SWR/zustand).
- [03-api-and-data-modeling-questions.md](03-api-and-data-modeling-questions.md) — 14 cards (REST, auth, schema, migrations, caching).
- [04-system-design-from-this-repo.md](04-system-design-from-this-repo.md) — the full "design a scheduler" walkthrough.
- [05-debugging-and-code-review-rounds.md](05-debugging-and-code-review-rounds.md) — timed simulations.
- [06-behavioral-star-stories.md](06-behavioral-star-stories.md) — 10 STAR worksheets sourced from module-06 work.
- [07-two-week-cram-plan.md](07-two-week-cram-plan.md) — day-by-day.

Count: 40+ question cards across 01–03 plus the design walkthrough and debugging rounds. Start with the [cram plan](07-two-week-cram-plan.md) if time is short.
