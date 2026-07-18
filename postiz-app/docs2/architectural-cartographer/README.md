# Architectural Cartographer

This suite is a top-down onboarding guide for Postiz. It teaches the system as a map: where the apps live, where requests enter, where state changes, where database persistence happens, and where a reviewer should slow down.

Use it in this order:

1. `JOURNAL.md` to see the inspection trail and the reasoning behind the teaching anchors.
2. `00-reading-map.md` to get the mental model and the first files to open.
3. `01-junior-engineer.md` to learn setup, folders, entry points, beginner TypeScript, components, and controllers.
4. `02-mid-level-engineer.md` to trace contracts, state, API calls, and one full feature.
5. `03-senior-engineer.md` to practice critique, risk finding, performance, security, and test design.
6. `04-reference-suite.md` when you are actively building, reviewing, debugging, or explaining the repo.

Every codebase claim in this suite points to exact source files and line ranges. Treat those references like a map legend: open the file, read the cited lines, then come back to the explanation. For checkpoints, answer out loud or in a notebook, then compare against the included self-grade section. Do not aim for perfect first-pass answers. Aim to notice what evidence your answer depends on.

This suite was written into `docs2/` so it does not modify the existing documentation tree.
