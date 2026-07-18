# Postiz Upskill Curriculum

A training lab built from this repository for one learner: a junior fullstack JS engineer (React/Node/TS CRUD experience) who wants to reach mid-level fast, build senior judgment, and pass interviews for mid-level fullstack roles. Every page teaches two things at once: **where things live in this repo** and **the transferable pattern you'll meet in every other repo — and in every interview**.

## What this repo is

Postiz is an open-source social-media scheduling platform (AGPL-3.0): you connect channels (X, LinkedIn, Instagram, TikTok, Mastodon, Telegram — 30+ providers), compose posts in a calendar UI, and the system publishes them at the right time, retries on token expiry, fans out webhooks, and runs "plugs" (automations like auto-repost). It is a pnpm monorepo: a NestJS API (`apps/backend`), a **Temporal**-based background worker (`apps/orchestrator`) that owns all time-delayed publishing, a Next.js 16 frontend (`apps/frontend` — note [CLAUDE.md](../../CLAUDE.md#L16) still calls it "Vite ReactJS"; trust the code, not the docs), a Chrome extension and SDK, and shared libraries (`libraries/nestjs-libraries` holds nearly all business logic; controllers are thin). Persistence is Postgres via Prisma, cache/state handoff is Redis, uploads go to local disk or Cloudflare R2, billing is Stripe, and multi-tenancy is organization-based with CASL abilities enforcing plan limits. There are **no automated tests in the repo** — which is itself one of the most instructive things about it.

## How to use this

- **One weekend** → [00-fast-track.md](00-fast-track.md).
- **Two weeks, interview coming** → [08-interview-prep/07-two-week-cram-plan.md](08-interview-prep/07-two-week-cram-plan.md). Everything else feeds it.
- **Eight weeks, deep skill building** → modules in order 01 → 05, doing every drill; then 06 (build things) with 07 (collaborate) alongside; 08 throughout.
- **Ongoing contribution** → live in [06-contribution-practice/](06-contribution-practice/README.md) and [09-reference/risk-register.md](09-reference/risk-register.md).

Recommended paths by profile:

| You are | Start | Then | Skip for now |
| --- | --- | --- | --- |
| Brand-new junior | 00-fast-track | 01 all, 02 all, 04 drills | 03-.../06 critique, 06-.../03 projects |
| Junior who knows React/Nest | 01-.../05-key-flows | 03 all, 04, 05 | 02-.../01 (skim) |
| Mid-level, new to this repo | 01-.../01-system-map + 05-key-flows | 03-.../05-pattern-catalog, 03-.../06-critique | 02 |
| Senior doing architecture review | 03-.../06-architecture-critique | 09-reference/risk-register | drills |
| **Interview in two weeks** | 08-.../07-two-week-cram-plan | as the plan directs | everything it doesn't cite |

## Conventions

- **File anchors**: every claim about this codebase links like [posts.controller.ts](../../apps/backend/src/api/routes/posts.controller.ts#L170-L179) (path + line range, valid at authoring time). If a link drifts after upstream changes, the symbol name in the text will still find it.
- **Fake code is labeled**: any snippet not from this repo starts with `// Illustrative fake code: not from this repo`. Unlabeled excerpts are real.
- **Drills and self-grading**: exercises end with Basic / Solid / Strong criteria. Grade yourself honestly; "Strong" is the mid-level interview bar.
- **Verification labels**: commands are `verified` (actually run while authoring) or `inferred` (read from scripts, not run — see [09-reference/verification-log.md](09-reference/verification-log.md)). Suspected-but-unproven problems say "investigate", confirmed ones cite exact lines.
- **Interview angle**: sections flagged this way tell you how the material shows up in interviews and what a mid-level answer sounds like.
- **Senior vocabulary** (also interview vocabulary) is used and defined in context: *invariant* (a condition that must always hold), *boundary* (where one component's responsibility ends), *contract* (the promises an interface makes), *idempotency* (safe to run twice), *isolation* (one tenant/user can't touch another's data), *blast radius* (how much breaks if this goes wrong), and friends.

## The mindset ladder

- **Junior asks**: "How do I make it work?"
- **Mid-level asks**: "Is this the right pattern? What breaks it?"
- **Senior asks**: "What does this commit us to, who pays the cost, and how do we reduce the risk?"

Interviews for mid-level roles are testing exactly the second and third questions. This codebase is unusually good practice because it's full of deliberate trade-offs (Temporal for scheduling, provider abstraction over 30 APIs, org-scoped repositories) *and* real shortcuts (no tests, `any` bodies, missing org filters) — you get to practice both admiring and critiquing, with evidence.

## Map of the curriculum

| Module | What it gives you |
| --- | --- |
| [00-fast-track.md](00-fast-track.md) | Running system + first mental model in a weekend |
| [01-codebase-cartography/](01-codebase-cartography/README.md) | System map, reading order, glossary, 7 traced flows |
| [02-stack-and-language-mastery/](02-stack-and-language-mastery/README.md) | Node/TS/Nest/Next/Temporal mental models anchored here |
| [03-architecture-and-patterns/](03-architecture-and-patterns/README.md) | Layers, data model, authz, async reliability, 16 pattern cards, critique |
| [04-code-reading-gym/](04-code-reading-gym/README.md) | Annotation drills, trace tables, bad-vs-better contrasts, review katas |
| [05-quality-engineering/](05-quality-engineering/README.md) | Testing strategy for an untested repo, debugging, performance, security, ops |
| [06-contribution-practice/](06-contribution-practice/README.md) | 18 junior tickets, 12 mid tickets, 6 senior projects, refactor katas |
| [07-career-and-collaboration/](07-career-and-collaboration/README.md) | Review mindset, PRs/RFCs, maintainer communication |
| [08-interview-prep/](08-interview-prep/README.md) | 50+ question cards (mostly repo-anchored), system design walkthrough, debugging rounds, STAR stories, cram plan |
| [09-reference/](09-reference/command-cheatsheet.md) | Commands, risk register, rubrics, verification log |
