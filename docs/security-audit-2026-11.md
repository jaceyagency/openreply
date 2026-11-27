# Dependency security audit — 2026-11-27

Result: `npm audit` reports **0 vulnerabilities** (before: critical in `@auth/core`, high in `nodemailer`, `brace-expansion`, `browserslist`, `deepmerge-ts`, `effect`; moderate in `@hono/node-server`, `@vitest/mocker`, others).

## Changes
- `next` 16.2.6 -> 16.4.6
- `next-auth` 5.0.0-beta.31 -> 5.0.0-beta.32 (`@auth/core` 0.41.2 -> 0.41.3: fixes email homoglyph bypass, `getToken()` crash, OAuth state/PKCE cookie provider binding)
- `@auth/prisma-adapter` 2.11.2 -> 2.11.3
- `nodemailer` 7.0.13 -> 10.6.1 (all advisories affect <=10.1.3)
- Transitive fixes via `npm audit fix` (vitest, babel, browserslist, hono, brace-expansion, ...)
- `overrides.effect = ^3.22.5` (Prisma CLI's `@prisma/config` pinned a vulnerable `effect`; `npm audit fix --force` would have downgraded Prisma to 6.12, so it was not used)
- `overrides.nodemailer = $nodemailer` forces a single nodemailer copy

## Verification
Tests (455/455 incl. the 38 DB/Redis tests on local PostgreSQL + Redis), lint, `tsc --noEmit` and `next build` all pass.
Run the DB/Redis tests with `TEST_DATABASE_URL` and `TEST_REDIS_URL` set (migrations applied first).

## Remaining risks
- **nodemailer 10 is outside Auth.js's declared peer range** (`^7.0.7 || ^8.0.5`) and is forced via an override. Auth.js only uses `createTransport`/`sendMail`; no regression was seen in tests, but SMTP magic-link sending was **not tested against a real SMTP server**. Test it before relying on email login, or revert to nodemailer 8.x (accepting its open advisories).
- `next-auth` is still a v5 **beta** (no stable v5 release exists).
- The `effect` override sits on a Prisma CLI/dev dependency (build/migrate time only) and is not exercised at runtime; revisit when Prisma ships a fixed `@prisma/config`.
- `npm audit` only covers known advisories; it does not replace review of the app's own code.
- No Instagram connection, deploy, merge or new paid service was made; ACUARELA stays disabled and only the official Meta API is used (no Zernio).
