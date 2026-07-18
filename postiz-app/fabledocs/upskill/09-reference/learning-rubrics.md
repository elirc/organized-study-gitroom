# Learning Rubrics

Observable behaviors, not vague traits. Rate yourself per skill: Junior / Mid / Senior, plus the **Interview-ready** bar (the answer that gets a mid-level hire). Self-assess honestly — "I could probably…" is a No.

## Codebase comprehension
| | Behavior |
| --- | --- |
| Junior | Can locate a feature's files with grep; can run the app. |
| Mid | Traces a flow across layers unaided ([Flow 1](../01-codebase-cartography/05-key-flows.md)); explains why Temporal not cron. |
| Senior | Identifies the two-sources-of-truth risk and the compensating sweep; predicts where a new bug class would live. |
| Interview-ready | Narrates schedule→publish in 3 min with anchors and one failure mode. |

## Async / event loop
| Junior | Knows `await` pauses; can read a promise chain. |
| Mid | Distinguishes serial-for-order ([thread upsert](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L500-L561)) from parallel ([Promise.all](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L343-L424)); spots unbounded fan-out. |
| Senior | Reasons about determinism (Temporal replay), backpressure, and non-termination in async recursion. |
| Interview-ready | Answers "event loop / microtasks" with the repo's serial-vs-parallel example and a failure mode. |

## TypeScript
| Junior | Adds annotations; reads generics. |
| Mid | Explains `any` contagion vs `unknown`; recognizes the DTO discriminator ([create.post.dto L71-L80](../../../libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts#L71-L80)) as a runtime union. |
| Senior | Fixes seams structurally (Express augmentation, `Prisma.*GetPayload`); knows where types are theater. |
| Interview-ready | Turns "types at runtime?" into class-validator/zod + the discriminator, with the `any`-endpoint cautionary tale. |

## React / frontend
| Junior | Builds components; uses `useState`/`useEffect`. |
| Mid | Splits server state (SWR) from client state (zustand); explains the rules-of-hooks-shaped composer. |
| Senior | Reasons about render cost (selectors), stale closures, RSC-vs-client trade-offs for this app. |
| Interview-ready | "SWR vs zustand vs useState" with this repo's clean three-way split + one failure of conflating them. |

## API & data modeling
| Junior | Writes CRUD endpoints and a basic schema. |
| Mid | Applies layering; org-scopes queries; picks offset vs cursor with reasons. |
| Senior | Designs for idempotency, transactions, migrations, tenant isolation as invariants; spots the IDOR. |
| Interview-ready | Designs the schema, defends adjacency-list vs position column, and names what the model can't express. |

## Reliability / distributed systems
| Junior | Adds a retry. |
| Mid | Explains at-least-once + idempotency; identifies fire-and-forget + compensation. |
| Senior | Designs a transactional outbox; reasons about exactly-once impossibility, dedup, versioning. |
| Interview-ready | "Exactly-once?" → honest at-least-once+idempotency answer with the workflowId scheme. |

## Debugging
| Junior | Adds logs, tries fixes. |
| Mid | Reproduces, narrows by layer, forms one hypothesis at a time. |
| Senior | Uses the right oracle (Temporal history), distinguishes symptom from cause, adds a regression guard. |
| Interview-ready | Narrates method before answer in a [timed round](../08-interview-prep/05-debugging-and-code-review-rounds.md); asks before assuming. |

## Code review
| Junior | Catches "does it work." |
| Mid | Catches correctness + fit; writes specific comments. |
| Senior | Catches "will it stay correct"; catches security (IDOR); comments are kind + offer a path. |
| Interview-ready | On a [kata](../04-code-reading-gym/04-review-katas.md): catches every blocking issue, phrases kindly, proposes the structural fix when pushed. |

## Testing
| Junior | Writes a happy-path test when asked. |
| Mid | Chooses the right layer; isolates time/randomness/DB; mocks at boundaries. |
| Senior | Introduces testing pragmatically to an untested repo; tests invariants that have no other guard. |
| Interview-ready | "Team with no tests — what do you do?" → the five-step plan, not "mandate 80%". |

## Security
| Junior | Knows to hash passwords. |
| Mid | Checks tenant scope, input validation, auth boundary per change. |
| Senior | Reasons about IDOR classes, secrets-at-rest, webhook signing, SSRF; defense in depth. |
| Interview-ready | "Secure a multi-tenant SaaS" → the [checklist](../05-quality-engineering/05-security-checklist.md) with this repo's concrete gaps. |

## Communication / collaboration
| Junior | Asks questions; takes review feedback. |
| Mid | Writes clear PRs (what/why/test/risk/rollback); disagrees on facts, yields on taste. |
| Senior | Writes RFCs; influences without authority; separates fact from preference; disagree-and-commit. |
| Interview-ready | Two crisp STAR stories under 2 min each, ending on impact. |

---

## Self-assessment checklist (tick honestly)

- [ ] I can draw the system topology from memory.
- [ ] I can trace schedule→publish and name its biggest risk.
- [ ] I can give example+trade-off+failure on any card in modules 01–03 of interview-prep, cold.
- [ ] I can run a 40-min system design with anchors.
- [ ] I can narrate a debugging round (method first).
- [ ] I can review a kata catching every blocking issue, kindly.
- [ ] I have two rehearsed STAR stories under 2 minutes.
- [ ] I can explain the IDOR, the token-plaintext risk, and the no-tests situation and what I'd do about each.

Eight ticks = interview-ready for mid-level fullstack. Fewer? The unticked line points you at the module to revisit.
