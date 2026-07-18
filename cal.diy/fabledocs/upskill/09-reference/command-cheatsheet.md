# Command Cheatsheet

All commands from root `package.json`, `agents/commands.md`, and `packages/prisma/package.json`. Marked `inferred` — none were executed while writing (see [verification-log.md](verification-log.md)). Package manager: **yarn 4.12.0** (Corepack); Node 22+.

## Install & setup

| Command | Does | Status |
| --- | --- | --- |
| `yarn` | Install workspace deps | inferred |
| `cp .env.example .env` | Base env (fill secrets) | inferred |
| `cp .env.appStore.example .env.appStore` | App-store dev env | inferred |
| `yarn dx` | Full dev setup incl. DB (`turbo run dx` → prisma `db-setup` = db-up + db-deploy + db-seed) | inferred |
| `docker compose up -d database redis` | Postgres + Redis (`docker-compose.yml`) | inferred |

## Dev servers

| Command | Does | Status |
| --- | --- | --- |
| `yarn dev` | Web app only (`--filter=@calcom/web`) | inferred |
| `yarn dev:all` | web + website + console | inferred |
| `yarn dev:api` | web + api proxy | inferred |

## Build / lint / types

| Command | Does | Status |
| --- | --- | --- |
| `yarn build` | Build web and deps (`turbo run build --filter=@calcom/web...`) | inferred |
| `yarn lint` / `yarn lint:fix` | Biome lint (`biome.json`) | inferred |
| `yarn format` | `biome format --write .` | inferred |
| `yarn type-check` | `turbo run type-check` | inferred |
| `yarn type-check:ci --force` | CI type-check; run before blaming CI (per `AGENTS.md`) | inferred |

## Tests

| Command | Does | Status |
| --- | --- | --- |
| `yarn test` | Unit tests, `TZ=UTC vitest run` | inferred |
| `yarn test <file>` | One test file | inferred |
| `yarn test <file> -t "<name>"` | One test by name | inferred |
| `yarn tdd` | Watch mode | inferred |
| `VITEST_MODE=integration yarn test` | Integration tests (`*.integration-test.ts`) | inferred |
| `TZ=<zone> VITEST_MODE=timezone yarn test` | Timezone-gated tests (TZ required) | inferred |
| `yarn e2e` | Playwright (`NEXT_PUBLIC_IS_E2E=1`, project `@calcom/web`) | inferred |
| `yarn e2e <file> --grep "<name>"` | One E2E test | inferred |
| `yarn test-e2e` | `db-seed` then `e2e` | inferred |

## Database

| Command | Does | Status |
| --- | --- | --- |
| `yarn db-seed` | Seed (`prisma db seed`) | inferred |
| `yarn db-deploy` | Apply migrations (`turbo run db-deploy`) | inferred |
| `yarn db-migrate` (prisma ws) | `prisma migrate dev` (new migration) | inferred |
| `yarn db-studio` | Prisma Studio GUI | inferred |
| `yarn db-reset` (prisma ws) | Nuke + setup | inferred |
| `yarn prisma <args>` | Prisma CLI passthrough | inferred |

## App store

| Command | Does | Status |
| --- | --- | --- |
| `yarn create-app` / `edit-app` / `delete-app` | App-store integration scaffolding | inferred |

## Notes

- The runner forces `TZ=UTC` for unit tests; timezone-sensitive tests are a separate mode requiring `TZ`.
- Vitest uses `pool: "forks"` (`vitest.workspace.ts`) to avoid RPC-during-fetch errors.
- Direct DB: `psql "postgresql://postgres:@localhost:5432/calendso"` (per `agents/commands.md`; note `docker-compose.yml` uses `unicorn_user`/`magical_password` — reconcile with your `.env`).
- Never `--no-verify` or skip type-checks before pushing (per `AGENTS.md`).
