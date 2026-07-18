# User Story Build Path

Use these stories as real tickets. Read the story, inspect every cited file, write your own plan in a scratch note, then implement in a branch. Do not ask an assistant to write the solution first. Ask it to question your reasoning.

Difficulty progression:

- Stories 1-3: Easy. Small UI or display changes in one or two files.
- Stories 4-5: Medium-Easy. UI features that read existing data and follow established hooks/components.
- Stories 6-7: Medium. New API or state behavior across frontend and backend.
- Stories 8-9: Hard. Full-stack changes touching UI, backend, and database.
- Story 10: Expert. Architectural feature with caching/performance/security implications.

Done means the acceptance criteria pass manually, the code follows nearby patterns, the user-visible behavior is clear, and you can explain the full path from UI to data or state. For backend changes, done also means the endpoint is registered, org-scoped, validated, and reflected in the frontend state update path.

Use this prompt when stuck:

```text
I'm working on Story X. I'm stuck on Y. Here is what I've tried: Z.
Don't give me the solution. Ask me questions that help me figure it out.
Focus on the existing Postiz code patterns and ask me to cite files or line ranges.
```
