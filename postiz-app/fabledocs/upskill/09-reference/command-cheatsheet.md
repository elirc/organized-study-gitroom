# Command Cheatsheet

`verified` = actually run while authoring (read-only inspection only). `inferred` = read from [package.json](../../../package.json)/[docker-compose.dev.yaml](../../../docker-compose.dev.yaml)/app package.jsons but **not executed**. No build/dev/test command was run against this repo — see [verification-log.md](verification-log.md).

## Setup
| Command | What | Label |
| --- | --- | --- |
| `cp .env.example .env` | seed env; fill DATABASE_URL, REDIS_URL, JWT_SECRET, FRONTEND_URL, NEXT_PUBLIC_BACKEND_URL, BACKEND_INTERNAL_URL, STORAGE_PROVIDER | inferred |
| `pnpm install` | install; `postinstall` runs prisma generate ([L39](../../../package.json#L39)) | inferred |
| `pnpm run dev:docker` | Postgres:5432, Redis:6379, Temporal:7233 (+UI:8080), pgadmin:8081, redisinsight:5540 | inferred |

## Dev
| Command | What | Label |
| --- | --- | --- |
| `pnpm run dev` | extension+orchestrator+backend+frontend parallel ([L14](../../../package.json#L14)) | inferred |
| `pnpm run dev-backend` | backend+frontend only ([L15](../../../package.json#L15)) | inferred |
| `pnpm run dev:backend` / `dev:frontend` / `dev:orchestrator` | single app, clears dist first ([L25-L27](../../../package.json#L25)) | inferred |
| `pnpm run dev:stripe` | dev + stripe listener ([L19](../../../package.json#L19)) | inferred |

Ports: backend 3000, frontend 4200, orchestrator health 3002 (from app package.jsons + [orchestrator main.ts L16](../../../apps/orchestrator/src/main.ts#L16)).

## Prisma / DB
| Command | What | Label |
| --- | --- | --- |
| `pnpm run prisma-generate` | generate client from [schema](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma) ([L33](../../../package.json#L33)) | inferred |
| `pnpm run prisma-db-push` | push schema, `--accept-data-loss` — **no migrations** ([L34](../../../package.json#L34)) | inferred |
| `pnpm run prisma-db-pull` | introspect DB → schema ([L35](../../../package.json#L35)) | inferred |
| `pnpm run prisma-reset` | force-reset schema (usable for a test DB) ([L36](../../../package.json#L36)) | inferred |

⚠️ There is no `prisma migrate` in scripts and no migrations directory. Prod `pm2` pushes schema on every deploy ([L16-L18](../../../package.json#L16-L18)).

## Build / prod
| Command | What | Label |
| --- | --- | --- |
| `pnpm run build` | frontend→backend→orchestrator, concurrency 1 ([L20](../../../package.json#L20)) | inferred |
| `pnpm run build:backend` / `:frontend` / `:orchestrator` / `:extension` | single app | inferred |
| `pnpm run pm2` | prod: delete-all → db-push → parallel pm2 → logs ([L16](../../../package.json#L16)) | inferred |
| `pnpm run start:prod:backend` / `:frontend` / `:orchestrator` | prod start single app | inferred |

## Test / lint
| Command | What | Label |
| --- | --- | --- |
| `pnpm test` | jest via `@nx/jest` — **broken**: no nx.json, no spec files ([jest.config.ts](../../../jest.config.ts)) | inferred (would fail) |
| lint | "run only from root" per [CLAUDE.md L60](../../../CLAUDE.md#L60); flat config [eslint.config.mjs](../../../eslint.config.mjs) — exact command not stated | inferred |

To make tests real: see [ticket J17](../06-contribution-practice/01-good-first-tickets.md) + [testing recipes](../05-quality-engineering/02-writing-tests-here.md).

## SDK / commands / extension
| Command | What | Label |
| --- | --- | --- |
| `pnpm run publish-sdk` | publish `apps/sdk` ([L17](../../../package.json#L17)) | inferred |
| `pnpm run commands:build:development` | build `apps/commands` CLI ([L32](../../../package.json#L32)) | inferred |

## Docker images (prod)
| Command | What | Label |
| --- | --- | --- |
| `pnpm run docker-build` | [var/docker/docker-build.sh](../../../var/docker/docker-build.sh) ([L37](../../../package.json#L37)) | inferred |
| `pnpm run docker-create` | [var/docker/docker-create.sh](../../../var/docker/docker-create.sh) ([L38](../../../package.json#L38)) | inferred |

## Read-only commands actually run while authoring (verified)
| Command | Purpose |
| --- | --- |
| `ls` / `find` over `apps`, `libraries` | file inventory |
| `grep` for `fixedEncryption`, throttle config, anchors | fact-checking |
| `wc -l` on key files | size before reading |
| `find -name "*.spec.ts"` | confirmed **zero tests** |

Node `>=22.12 <23`, pnpm 10.6.1 ([package.json L9-L12](../../../package.json#L9-L12)).
