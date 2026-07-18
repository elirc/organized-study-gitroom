# Debugging & Code-Review Rounds (Timed)

Simulations of the live rounds. Set a timer, think aloud (interviewers grade your *narration*, not just the answer), then check against the rubric.

---

## Debugging Round 1 (15 min): "Scheduled posts stopped publishing an hour ago."

Setup: A batch of posts are stuck in QUEUE. Some published fine earlier today. No error emails.
Your job: narrate reproduce → narrow → hypothesize → verify → fix.
Interviewer will drip clues if you ask the right question:
- "Are new posts affected or only old ones?" → *all new ones since ~1h ago.*
- "Anything deploy around then?" → *orchestrator was redeployed.*
- "What does the Temporal UI show?" → *workflows exist, all failing the same activity: `postSocial` → connection refused to DB.*
Expected path: not a scheduling bug — the *worker* can't reach Postgres after the deploy (bad env / connection pool). You'd check the orchestrator's DB env, connection, and the activity error stack.
Rubric — Strong: you separated "scheduler didn't fire" from "activity failed," used the Temporal UI as the oracle, and reached "worker infra, not app logic" without guessing. See [debugging scenario 1](../05-quality-engineering/03-systematic-debugging.md).
Interviewer follow-ups: "How would you have known within 1 minute instead of 15?" (alert on activity failure rate) — ties to [observability](../05-quality-engineering/06-observability-and-operations.md).

## Debugging Round 2 (15 min): "A user got the same tweet posted twice."

Setup: one scheduled post, two identical tweets on the platform.
Clues on request:
- "How many activity attempts in Temporal?" → *one activity, two attempts.*
- "What happened on attempt 1?" → *it threw after 10 min (startToCloseTimeout).*
- "Did attempt 1 actually post?" → *yes — the tweet exists with an earlier timestamp.*
Expected: attempt 1 posted then timed out before recording `releaseId`; Temporal retried → duplicate. Root cause = non-idempotent posting; fix = persist `releaseId` immediately + check on retry entry ([workflow L164-L200](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L164-L200)).
Rubric — Strong: you named the two retry layers, located the success-then-crash window, and proposed the idempotency fix (project M6). See [debugging scenario 2](../05-quality-engineering/03-systematic-debugging.md).
Follow-up: "Which platforms would this *not* duplicate on?" (X rejects exact dupes — [handleErrors 'duplicate-rules'](../../../libraries/nestjs-libraries/src/integrations/social/x.provider.ts#L47-L108)).

## Debugging Round 3 (12 min): "Self-hoster: 403 on everything right after login."

Clues: cookie present? → yes. JWT decodes? → yes, `activated: true`. Only membership is `disabled`.
Expected: empty filtered org list → `organization[0]` undefined → `.apiKey` TypeError → caught → generic 403 ([middleware L79-L104](../../../apps/backend/src/services/auth/auth.middleware.ts#L79-L104)). The catch-all hides the real cause.
Rubric — Strong: elimination order (transport → token → claims → org state → CORS), and you spotted that a broad try/catch converts a crash into "forbidden." See [debugging scenario 5](../05-quality-engineering/03-systematic-debugging.md).

---

## Code-Review Round 1 (15 min): the IDOR PR

Present yourself [review kata 1](../04-code-reading-gym/04-review-katas.md) (tag rename without org scope). Produce: the blocking finding, the fix, a kind comment, and the test you'd require.
Interviewer follow-ups: "How did this get past review originally?" (convention-based scoping, no test) → "How do you prevent the whole class?" (project M2 — structural enforcement). This is where you show layer-3/senior thinking.
Rubric — Strong: caught the IDOR, referenced the correct sibling pattern, required a cross-org test, and proposed the structural fix when pushed.

## Code-Review Round 2 (15 min): the "helpful" retry PR

Present [review kata 5](../04-code-reading-gym/04-review-katas.md) (manual retry loop added inside an already-retried webhook activity). Produce findings ranked blocking/important/optional.
Follow-ups: "Is retrying wrong?" (no — but here it multiplies attempts and still isn't idempotent/signed) → "What would you do instead?" (sign + delivery id first, then a single retry policy — project S4).
Rubric — Strong: you recognized the double-retry, the idempotency gap, and sequenced the fix (sign/id before retry).

## Code-Review Round 3 (12 min): the schema PR

Present [review kata 8](../04-code-reading-gym/04-review-katas.md) (add `position` column alongside `parentPostId`, via `db push`, no backfill).
Rubric — Strong: caught two-orderings drift, the missing backfill / need for a real migration, and the still-missing transaction; proposed expand-contract.

---

## How to practice these

1. Record yourself (audio) doing one round; listen back for: did you state a hypothesis before testing it? Did you ask before assuming? Did you name the tool you'd use?
2. The debugging rounds reward *method visible in narration* — say "I'd check X because it would distinguish hypothesis A from B" out loud, every time.
3. The review rounds reward *catching the blocking issue* + *kind, specific language* + *the structural follow-up when pushed*.

These convert directly from [systematic debugging](../05-quality-engineering/03-systematic-debugging.md) and [review katas](../04-code-reading-gym/04-review-katas.md) — do those first, then time yourself here.
