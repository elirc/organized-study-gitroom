# Two-Week Cram Plan

Fourteen days, ~2–3 hrs/day, assuming you can run the app. Front-loads comprehension, back-loads performance. Checkpoints at day 7 and day 14. Every item links to where the material lives.

## Week 1 — comprehension + core mechanisms

**Day 1 — Get it running + first trace.** [00-fast-track.md](../00-fast-track.md) start to finish: app up, ten files, trace scheduling. Teach-back aloud. ✅ when you can explain schedule→publish without notes.

**Day 2 — Map the system.** [01-system-map](../01-codebase-cartography/01-system-map.md) + [glossary](../01-codebase-cartography/03-domain-glossary.md). Draw the topology from memory. Learn channel/integration/provider cold.

**Day 3 — The seven flows.** [05-key-flows](../01-codebase-cartography/05-key-flows.md). Do the drill on Flow 1 (find the workflow in Temporal UI, kill+restart the worker). Fill trace tables 1–2 from [02-trace-tables](../04-code-reading-gym/02-trace-tables.md).

**Day 4 — Language/runtime.** [01-language-runtime-model](../02-stack-and-language-mastery/01-language-runtime-model.md) + [02-framework-mental-models](../02-stack-and-language-mastery/02-framework-mental-models.md). Then JS/TS cards [Q1–Q7](01-js-ts-node-deep-dive.md) aloud.

**Day 5 — Architecture + patterns.** [pattern catalog](../03-architecture-and-patterns/05-pattern-catalog.md) (all 16, name each from memory) + [boundaries](../03-architecture-and-patterns/01-boundaries-and-layers.md). JS/TS cards Q8–Q14.

**Day 6 — Data + auth.** [data model](../03-architecture-and-patterns/02-data-model-and-persistence.md) + [validation-auth](../03-architecture-and-patterns/03-validation-auth-and-permissions.md). Do the IDOR audit drill. API cards [Q20–Q24](03-api-and-data-modeling-questions.md).

**Day 7 — CHECKPOINT + first mock.** Self-test: draw the system, list 5 patterns with anchors, name 3 confirmed risks, do one JS + one API card cold. Then a **45-min system-design mock** ([04](04-system-design-from-this-repo.md)) — record it.
Day-7 bar: you can trace Flow 1, name the Temporal bet + its two-sources-of-truth risk, and cite the IDOR. If not, repeat days 3+6 before continuing.

## Week 2 — performance under time pressure

**Day 8 — Async reliability + remaining API cards.** [side-effects doc](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md); API cards [Q25–Q34](03-api-and-data-modeling-questions.md). Focus: idempotency, transactions, webhooks, migrations.

**Day 9 — Frontend.** [framework cards Q1–Q12](02-frontend-framework-questions.md) + skim [calendar/composer](../01-codebase-cartography/05-key-flows.md). Two timed answers on state management.

**Day 10 — Debugging round 1 (timed).** [debugging scenarios](../05-quality-engineering/03-systematic-debugging.md) → [timed round 1 + 2](05-debugging-and-code-review-rounds.md). Narrate aloud; record; grade with the rubric.

**Day 11 — Code review round (timed).** [review katas](../04-code-reading-gym/04-review-katas.md) → [timed review rounds](05-debugging-and-code-review-rounds.md). Catch every blocking issue; practice kind phrasing.

**Day 12 — System design, second pass.** Re-do the [walkthrough](04-system-design-from-this-repo.md) plus two variation prompts (multi-tenancy scaling, exactly-once). 40 min, cold.

**Day 13 — Behavioral.** Fill 6 of the 10 [STAR worksheets](06-behavioral-star-stories.md) in your own words; rehearse each under 2 min; record two.

**Day 14 — CHECKPOINT + full loop simulation.** Simulate a loop: 1 JS deep-dive card + 1 API card + 1 system-design (30 min) + 1 debugging round + 2 STAR stories, back to back, recorded.
Day-14 bar (interview-ready): example+trade-off+failure on any card without notes; a coherent 40-min design with anchors; a debugging round where you narrate method before answer; two crisp STAR stories ending on impact.

## Daily habits (all 14 days)

- 10 min flashcards: pick 3 random cards, answer aloud, self-grade example/trade-off/failure.
- Keep a "one thing I couldn't explain" note; resolve it next morning.
- Every answer ends with a failure mode or trade-off — drill the habit until automatic.

## If you have less time

- **1 week**: days 1, 3, 5, 6, 7, 10, 12 (comprehension + one mock each of design/debug).
- **1 weekend**: [00-fast-track](../00-fast-track.md) + [05-key-flows](../01-codebase-cartography/05-key-flows.md) + [04-system-design](04-system-design-from-this-repo.md) + the JS/TS cards. You'll have one strong system to talk about — enough to be dangerous.

## Self-assessment rubric

Use [09-reference/learning-rubrics.md](../09-reference/learning-rubrics.md) — the "interview-ready" column is your target for each skill. You're ready when you'd bet on yourself to give a mid-level answer to a *cold* question in each of: event loop/async, React state, API/auth design, schema/migrations, a system design, and a debugging narration.
