# Local Dev Guide

This guide is a cleaned-up path from clone to running app.

## Prerequisites

Install:

- Node.js `>=18.x`
- Yarn 4 through Corepack or the repo-pinned Yarn binary
- Git
- Docker and Docker Compose for the easiest setup path
- PostgreSQL `>=13.x` if you are not using Docker

On Windows, clone with symlinks enabled from an elevated Git Bash:

```bash
git clone -c core.symlinks=true https://github.com/calcom/cal.diy.git
```

The current workspace is already cloned, so you usually start from:

```powershell
cd C:\Users\Owner\Desktop\mastercal\cal.diy
```

## Install dependencies

```powershell
yarn
```

The repo is pinned to Yarn `4.12.0`.

## Create environment file

Copy the example env:

```powershell
Copy-Item .env.example .env
```

Set at least:

```text
DATABASE_URL=...
DATABASE_DIRECT_URL=...
NEXTAUTH_SECRET=...
CALENDSO_ENCRYPTION_KEY=...
NEXTAUTH_URL=http://localhost:3000/api/auth
NEXT_PUBLIC_WEBAPP_URL=http://localhost:3000
NEXT_PUBLIC_WEBSITE_URL=http://localhost:3000
```

Generate secrets:

```bash
openssl rand -base64 32
openssl rand -base64 24
```

Use the first for `NEXTAUTH_SECRET` and the second for `CALENDSO_ENCRYPTION_KEY`.

Windows note from the README: if `packages/prisma/.env` is a symlink and does not work on your machine, replace it with a real copy of the root `.env`.

## Fastest local start: `yarn dx`

The easiest local flow is:

```powershell
yarn dx
```

What it does:

- Starts local infrastructure through Docker.
- Prepares the database.
- Seeds development users.
- Runs the web app.

Open:

```text
http://localhost:3000
```

Seed users documented by the repo:

```text
free@example.com / free
pro@example.com / pro
trial@example.com / trial
admin@example.com / ADMINadmin2022!
onboarding@example.com / onboarding
```

Open Prisma Studio:

```powershell
yarn db-studio
```

Then visit:

```text
http://localhost:5555
```

## Manual database setup

Use this path if you want to run Postgres yourself instead of the Docker quick start.

1. Create a local PostgreSQL database.
2. Set `DATABASE_URL` and `DATABASE_DIRECT_URL` in `.env`.
3. Install dependencies:

```powershell
yarn
```

4. Run migrations:

```powershell
yarn workspace @calcom/prisma db-migrate
```

For production-like deploy migration:

```powershell
yarn db-deploy
```

5. Start the web app:

```powershell
yarn dev
```

## Important scripts

Run from the repo root unless noted.

### `yarn dev`

Starts the monorepo development tasks through Turborepo.

Use it when:

- You already have local infrastructure ready.
- You want normal app development.

### `yarn dx`

Runs the batteries-included local developer experience.

Use it when:

- You want Docker-backed database setup and seeded local data.

### `yarn build`

Builds packages/apps through Turborepo.

Use it when:

- You need to verify production build behavior.

### `yarn type-check`

Runs TypeScript checks across the monorepo.

Use it when:

- You changed types, API contracts, schemas, or shared package exports.

### `yarn lint`

Runs linting.

Use it when:

- You changed code style-sensitive files or want pre-PR cleanup.

### `yarn format`

Formats files using repo tooling.

Use it when:

- Formatting drift appears after edits.

### `yarn test`

Runs Vitest tests through the root test setup.

Use it when:

- You changed package-level logic or UI components with unit/integration coverage.

### `yarn test-e2e`

Runs Playwright E2E tests.

Use it when:

- You changed a full browser workflow such as booking, auth, or event type management.

### `yarn e2e`

Related E2E helper script. Check root `package.json` for exact current behavior.

### `yarn db-studio`

Starts Prisma Studio.

Use it when:

- You need to inspect or edit local data.

### `yarn db-seed`

Runs the Prisma seed script.

Use it when:

- You need to repopulate local seed users/data.

### `yarn db-deploy`

Runs Prisma deploy migrations.

Use it when:

- Applying migrations in a deploy-like environment.

## Running the web app only

From root:

```powershell
yarn workspace @calcom/web dev
```

The web app's dev script runs:

- app-store static copy
- Next.js dev with Turbopack

Open:

```text
http://localhost:3000
```

## Running API v2

API v2 lives under:

- `apps/api/v2`

Use workspace scripts from its `package.json`, for example:

```powershell
yarn workspace @calcom/api-v2 dev
```

Check the package script names if this changes:

```powershell
yarn workspace @calcom/api-v2 run
```

## Running tests

### All root tests

```powershell
yarn test
```

### Find relevant tests first

```powershell
rg --files | rg "\.(test|spec)\.(ts|tsx)$"
```

### Run E2E tests

```powershell
yarn test-e2e
```

If Playwright browsers are missing:

```powershell
npx playwright install
```

Show the Playwright HTML report:

```powershell
yarn playwright show-report test-results/reports/playwright-html-report
```

### API v2 tests

API v2 uses Jest through Nest tooling. Run from workspace scripts:

```powershell
yarn workspace @calcom/api-v2 test
```

## Adding a new environment variable

1. Add it to `.env.example` with a clear default or comment.
2. Add it to local `.env`.
3. If it affects build output or Turborepo cache correctness, update `turbo.json`.
4. If there is an existing config/env helper in `packages/lib`, use or update that helper instead of scattering raw `process.env` usage.
5. If the variable is public in the browser, it must use the `NEXT_PUBLIC_` prefix.
6. Document any required setup in the relevant README/docs.

## Common errors and fixes

### `CLIENT_FETCH_ERROR`

Likely `NEXTAUTH_URL` mismatch.

Use:

```text
NEXTAUTH_URL=http://localhost:3000/api/auth
```

### Out-of-memory during install/build

Increase Node memory:

```powershell
$env:NODE_OPTIONS="--max-old-space-size=16384"
```

Then rerun the failing command.

### Missing VAPID public key

If you see an error like no key set for `vapidDetails.publicKey`, add the relevant VAPID env vars documented in `.env.example`.

### Need more logs

Set:

```text
NEXT_PUBLIC_LOGGER_LEVEL=3
```

### SSL edge termination in local trusted environments

The README notes `NODE_TLS_REJECT_UNAUTHORIZED=0` for trusted SSL edge termination debugging. Use this carefully. It disables TLS verification for Node requests.

### Prisma user `metadata` is empty

The README notes that empty user `metadata` should be `{}`.

### E2E mail testing

If E2E flows require email, run Mailhog. The README suggests exposing:

- web UI: `8025`
- SMTP: `1025`

## First local debugging loop

For booking work:

1. Run `yarn dx`.
2. Log in with `free@example.com / free`.
3. Open an event type.
4. Open its public booking page.
5. Submit a booking.
6. Inspect `Booking`, `Attendee`, and `BookingReference` in Prisma Studio.
7. Trace server logs around `apps/web/pages/api/book/event.ts` and `RegularBookingService`.

This loop teaches the repo faster than reading files in isolation.
