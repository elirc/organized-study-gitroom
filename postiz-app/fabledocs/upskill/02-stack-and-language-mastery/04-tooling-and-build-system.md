# Tooling & Build System

## pnpm workspace mechanics

One lockfile, one root dependency list ([package.json L42-L256](../../../package.json#L42-L256)), apps hold scripts only. `pnpm run --filter ./apps/backend dev` scopes execution; `--parallel` runs the dev fleet ([L14](../../../package.json#L14)). `pnpm.overrides` pins next/react versions repo-wide ([L322-L329](../../../package.json#L322-L333)); `onlyBuiltDependencies: [bcrypt]` allowlists native postinstall builds — a supply-chain guard worth knowing for interviews (pnpm blocks arbitrary install scripts by default).

Trade-off to articulate: single dependency universe = zero version skew between apps, but every dependency bump risks all four apps at once, and the frontend "sees" server deps (only imports actually bundle, but nothing *prevents* a bad import — see [module boundaries, Q7](../08-interview-prep/01-js-ts-node-deep-dive.md)).

## Path aliases

`@gitroom/backend/*`, `@gitroom/nestjs-libraries/*`, `@gitroom/helpers/*`, `@gitroom/frontend/*`, `@gitroom/react/*` map to source dirs via [tsconfig.base.json](../../../tsconfig.base.json). Consequences: imports are stable under file moves within a package; builds must each resolve the aliases (Nest via tsconfig paths, Next via its webpack config, tsup for SDK).

## Builds

| App | Builder | Output |
| --- | --- | --- |
| backend / orchestrator | `nest build` (tsc under the hood) | `dist/`, run with `node --experimental-require-module` ([apps/backend/package.json](../../../apps/backend/package.json)) — ESM/CJS straddling, worth a question card |
| frontend | `next build` | `.next/` |
| extension | Vite + `@crxjs/vite-plugin` | `dist/` |
| sdk | tsup | publishable package |

Root `pnpm run build` forces `--workspace-concurrency=1` ([L20](../../../package.json#L20)) — build order dependency (or just RAM discipline). Prod runs under **pm2** (`pnpm run pm2` → delete-all, prisma push, parallel pm2 starts — [L16-L18](../../../package.json#L16-L18)); note the prod start *pushes schema changes automatically* — `db push --accept-data-loss` on boot is a deployment decision you should be able to critique.

## Lint / format / hooks

ESLint flat config at root ([eslint.config.mjs](../../../eslint.config.mjs)); "linting runs only from root" per [CLAUDE.md L60](../../../CLAUDE.md#L60). Prettier 2.x. No husky/lint-staged, no CI config in-repo for tests (a [Jenkins](../../../Jenkins) folder and [sonar-project.properties](../../../sonar-project.properties) exist — external CI). `inferred`: lint command shape not verified.

## The broken test toolchain (and why you should care)

Root [jest.config.ts](../../../jest.config.ts) calls `getJestProjects()` from `@nx/jest`, but there is no `nx.json` and `@nx/jest` isn't in dependencies — **`pnpm test` cannot work as configured** (`inferred`; not executed). Zero spec files exist. This is leftover scaffolding from an Nx past — repo archaeology you can cite when asked "what do you look at first in an unfamiliar repo?" (answer: tooling that references missing infrastructure tells you about the project's history and its real test culture).

Interview angle:
1. "Monorepo tooling — what did you use, what were the trade-offs?" → pnpm workspaces without Nx/Turbo: simple, no caching/affected-graph; single dep universe pros/cons.
2. "How do you prevent one package importing another's internals?" → nothing here does; name eslint boundaries / exports maps.
3. "Native modules in Node?" → bcrypt + sharp + canvas here; pnpm's build allowlist; Docker implications (musl vs glibc — see [Dockerfile.dev](../../../Dockerfile.dev)).

Drill: run `pnpm run build:backend` mentally: list the three things that must be true beforehand (prisma client generated; tsconfig paths resolvable; env not needed — builds don't read `.env`... verify that last claim by checking whether any imported module reads env at module top-level. Counter-example hunt: [redis.service.ts](../../../libraries/nestjs-libraries/src/redis/redis.service.ts) likely constructs a client at import time — does the *build* import it? No — builds don't execute; only type-check. Say why that distinction matters.)
