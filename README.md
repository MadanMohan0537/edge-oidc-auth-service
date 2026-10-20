# Edge OIDC Auth Service

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](./LICENSE)

**An unaudited learning build of a self-hosted sign-in and token service on Cloudflare Workers.** It implements email and password, GitHub as a login provider, TOTP multi-factor auth, rotating refresh tokens with reuse detection, organizations with role checks, and per-user session revocation.

This is not an OIDC-compliant provider. The Worker issues RS256 access tokens and serves a JWKS document plus a discovery document. That discovery document lists grants and an ID token algorithm the routes do not implement. Details are under [Limitations](#limitations).

The repository was renamed from Pulse-Auth. In `wrangler.jsonc` the Worker name and the D1 database name stay `pulse-auth`, and the queue name stays `pulse-auth-events`. There is no automated CI. `npm run typecheck` currently fails with 3 TypeScript errors.

## Problem and who it's for

Hosted identity providers are often the right choice for a product, and they also hide how token lifetimes, session revocation, organization permissions, and MFA work. This repository is a learning build for developers and technical PMs who want to read those pieces on an edge runtime, where there is no long-lived server and per-user session state lives in a Durable Object.

## Approach

- **Access tokens:** RS256 JWTs from the `jose` library. The RSA key pair is generated on first use, stored in the D1 table `signing_keys`, and cached in the isolate. Public keys are published at `/.well-known/jwks.json`. The `aud` claim is the literal string `pulse`. Default access-token lifetime is 900 seconds (`ACCESS_TOKEN_TTL_SECONDS`).
- **Refresh rotation with reuse detection:** registration, password login, and GitHub login each insert a new refresh-token family. When the presented token is still active, `POST /v1/auth/refresh` revokes it and inserts a child in the same family. Reuse detection runs when a token that was already rotated (revoked, and already has a child) is presented again. The service then revokes every still-active token in that family, marks that user's Durable Object sessions revoked, and records a `token.reuse_detected` event. Default refresh lifetime is 2,592,000 seconds (30 days).
- **Passwords:** PBKDF2-SHA-256 with 100,000 iterations and a 16-byte salt, via Web Crypto. `validatePassword` requires at least 10 characters and both a letter and a number.
- **MFA:** TOTP enrollment, confirmation, and disabling (SHA-1, 6 digits, 30-second period, window 1). Confirming enrollment returns 8 recovery codes stored as SHA-256 hashes. Login accepts a TOTP code or one recovery code. Disabling MFA requires a current TOTP code.
- **Tenancy:** organizations with roles `owner`, `admin`, `member`, and `viewer`. Routes under `/v1/orgs/:orgId` load the caller's membership and check a permission from `src/lib/rbac.ts`. Listing and creating organizations require a bearer token and do not check a role permission.
- **Sessions:** one SQLite-backed `SessionManager` Durable Object per user (`idFromName(userId)`). Authenticated requests touch the session. The HTTP API lists sessions, revokes one, and revokes all.
- **Events:** `user.created`, `user.login`, `user.logout`, `user.mfa_enabled`, `session.revoked`, `token.reuse_detected`, `org.created`, and `org.member_added` are inserted into the D1 table `auth_events`. When the `AUTH_EVENTS` binding is present, the same payload is sent to that queue. A failed queue send is logged; the D1 row is already written. This repo has no queue consumer.

```text
client ── access JWT / refresh ──► Worker (Hono, src/index.ts)
                                     ├─ D1 binding DB (users, orgs, refresh families, signing keys, events)
                                     ├─ SessionManager Durable Object (one SQLite database per user)
                                     └─ AUTH_EVENTS queue producer (pulse-auth-events)
```

## What exists today

Checked by reading this repository and running `npm test` and `npm run typecheck` here.

Routes mounted in `src/index.ts`:

| Method | Path | Access |
| --- | --- | --- |
| `POST` | `/v1/auth/register` | Public. Creates a user, an organization, and an `owner` membership. |
| `POST` | `/v1/auth/login` | Public. Password, then TOTP or a recovery code when MFA is on. |
| `POST` | `/v1/auth/refresh` | Public. JSON body `{ "refresh_token" }`. Rotates that token. |
| `POST` | `/v1/auth/logout` | Bearer access token, or a JSON `refresh_token` plus `session_id`. |
| `GET` | `/v1/oauth/github/start` | Public. Redirects to GitHub when `GITHUB_CLIENT_ID` is set. |
| `GET` | `/v1/oauth/github/callback` | Public. Exchanges GitHub's code and redirects to `/app`. |
| `POST` | `/v1/mfa/enroll` | Bearer. Stores a TOTP secret and returns an `otpauth://` URL. |
| `POST` | `/v1/mfa/confirm` | Bearer. Enables MFA and returns recovery codes. |
| `POST` | `/v1/mfa/disable` | Bearer. Requires a valid TOTP code. |
| `GET`, `POST` | `/v1/orgs` | Bearer. List the caller's organizations, or create one as `owner`. |
| `GET` | `/v1/orgs/:orgId` | Bearer. Requires `org:read`. |
| `PATCH` | `/v1/orgs/:orgId` | Bearer. Requires `org:update`. Renames the organization. |
| `GET` | `/v1/orgs/:orgId/members` | Bearer. Requires `members:read`. |
| `POST` | `/v1/orgs/:orgId/members` | Bearer. Requires `members:invite`. Adds an existing user by email. |
| `PATCH` | `/v1/orgs/:orgId/members/:userId` | Bearer. Requires `members:role`. Cannot change the `owner` role. |
| `DELETE` | `/v1/orgs/:orgId/members/:userId` | Bearer. Requires `members:remove`. Cannot remove the owner. |
| `GET` | `/v1/sessions` | Bearer. Lists that user's Durable Object sessions. |
| `DELETE` | `/v1/sessions/:sessionId` | Bearer. Revokes one session and its refresh tokens. |
| `DELETE` | `/v1/sessions` | Bearer. Revokes every session and refresh token for the user. |
| `GET` | `/v1/me` | Bearer. Current user, session id, org, role, and scope. |
| `GET` | `/.well-known/openid-configuration` | Public discovery document. See [Limitations](#limitations). |
| `GET` | `/.well-known/jwks.json` | Public signing keys whose `active` flag is 1. |
| `GET` | `/health` | JSON `ok`, service name `pulse-auth`, and `ISSUER`. |

HTML pages: `/` (landing), `/login`, `/register`, and a session console at `/app`. Those pages still use the previous product name in their titles.

Schema: two D1 migrations, `migrations/0001_init.sql` and `migrations/0002_signing_keys.sql`. `wrangler.jsonc` also declares a Durable Object migration that creates the SQLite class `SessionManager`.

`npm test` runs 5 unit tests in `test/unit.test.ts`: role ordering and permissions, password strength and hashing, and TOTP verification. On this tree all 5 passed. They call helper functions only.

`npm run typecheck` (`tsc --noEmit`) currently fails with 3 errors, all in `src/routes/auth.ts` on the logout handler. After `.catch(() => ({}))`, the request body is typed as a union with `{}`, so `session_id` and `refresh_token` are rejected (`TS2339` at lines 300 and 301). Those errors are left as they are.

There is no CI workflow in this repository. This README does not claim the Worker is deployed.

## How success would be measured

[docs/PRD.md](docs/PRD.md) proposes four metrics: login success rate, MFA adoption rate, refresh success rate, and time to revoke a compromised session. **Not yet measured.** This repository records no measurements and no user counts.

## Limitations

- **Unaudited learning build.** Do not use it as production identity without a separate security review.
- **Not OIDC-compliant.** There is no client registration, no PKCE (the `oauth_states.code_verifier` column is inserted as `NULL`), and no ID token. `/.well-known/openid-configuration` points `authorization_endpoint` at GitHub login start, points `token_endpoint` at `POST /v1/auth/refresh`, and lists `response_types_supported: ["code"]`, `id_token_signing_alg_values_supported: ["RS256"]`, and `grant_types_supported` of `refresh_token`, `password`, and `authorization_code`. The refresh route accepts only a JSON `refresh_token`. Password login is a separate JSON route. This service does not issue authorization codes to clients. GitHub's authorization-code exchange is server-side, against GitHub, and the callback redirects to `/app` with `access_token` and `refresh_token` in the query string.
- **Helper tests only.** Refresh rotation, reuse detection, GitHub OAuth, MFA routes, organization routes, and session routes have no automated tests.
- **No automated CI.**
- **Typecheck is red.** Three TypeScript errors in `src/routes/auth.ts`, described above.
- **Signing keys live in D1.** The RSA private key is a JWK in `signing_keys`, not in a secrets manager or KMS. There is no key-rotation procedure. The active key is also cached in module memory for the life of the isolate.
- **`JWT_SECRET` is unused.** It is declared on the `Env` type and shown in `.dev.vars.example`. Signing and verification use the RS256 key in D1.
- **`ISSUER` defaults to `http://127.0.0.1:4545`** in `wrangler.jsonc`. Set it to the public origin before deploying.
- **No password-reset route, no rate limiting, and no email-verification flow.** Password registration stores `email_verified = 0`. GitHub sign-in sets `email_verified = 1` when GitHub returns a verified address.
- **Permission map is wider than the routes.** `org:delete`, `billing:read`, and `billing:manage` exist in `src/lib/rbac.ts`. No route deletes an organization or implements billing. Adding a member does not send email; the address must already belong to a user in this database. `member` and `viewer` currently hold the same permissions (`org:read`, `members:read`).
- **CORS allows any origin** (`origin: "*"` on every route).
- **`scripts/publish-to-github.sh` still pushes to `https://github.com/MadanMohan0537/Pulse-Auth.git`.** The repository remote for this project is `https://github.com/MadanMohan0537/edge-oidc-auth-service.git`.

## Quick start

Wrangler 4.125.0, locked in this repo, declares `engines.node` of `>=22.0.0`.

```bash
git clone https://github.com/MadanMohan0537/edge-oidc-auth-service.git
cd edge-oidc-auth-service
npm install
cp .dev.vars.example .dev.vars
npm run db:migrate:local
npm test
npm run dev          # wrangler dev on http://127.0.0.1:4545
```

`npm run dev` is `wrangler dev --port 4545 --ip 0.0.0.0`. The session console is `http://127.0.0.1:4545/app`.

Register a local user (the password meets the 10-character letter-and-number rule):

```bash
curl -s http://127.0.0.1:4545/v1/auth/register \
  -H 'content-type: application/json' \
  -d '{"email":"you@example.com","password":"CorrectHorse1","name":"You"}'
```

Optional GitHub login: create a GitHub OAuth App whose callback is `http://127.0.0.1:4545/v1/oauth/github/callback`, then set `GITHUB_CLIENT_ID` and `GITHUB_CLIENT_SECRET` in `.dev.vars`. The start route builds that callback from `ISSUER`.

## Deployment

Publish with a direct Workers deploy:

```bash
npx wrangler deploy
```

That command uploads the Worker named `pulse-auth` whose entry point is `src/index.ts` (`main` in `wrangler.jsonc`). Keep the name `pulse-auth`. `npm run deploy` runs `wrangler deploy`, the same publish. This README does not claim that command has been run against a live account.

`wrangler.jsonc` bindings:

| Binding | What it is |
| --- | --- |
| `DB` | D1 database name `pulse-auth`. The checked-in `database_id` is the placeholder `00000000-0000-0000-0000-000000000001`. Replace it with the id from your account. |
| `SESSION_MANAGER` | Durable Object class `SessionManager`. The SQLite class migration is already declared in `wrangler.jsonc` and is applied by `npx wrangler deploy`. |
| `AUTH_EVENTS` | Queue producer for `pulse-auth-events`. Create that queue before deploying. This repo does not consume the queue. |

Plain vars in the same file, not secrets: `ISSUER` (`http://127.0.0.1:4545`), `ACCESS_TOKEN_TTL_SECONDS` (`900`), `REFRESH_TOKEN_TTL_SECONDS` (`2592000`), `GITHUB_CLIENT_ID` (empty), and `APP_NAME` (`Pulse Auth`, used as the TOTP issuer label). Set `ISSUER` to the public origin before deploying. The account's `workers.dev` hostname follows the Worker name `pulse-auth`.

`GITHUB_CLIENT_SECRET` is required only for GitHub login. Put it on the Worker with `npx wrangler secret put GITHUB_CLIENT_SECRET`. Password sign-in does not read `JWT_SECRET`.

```bash
npx wrangler login
npx wrangler d1 create pulse-auth
# Write the returned database_id into wrangler.jsonc in place of the placeholder.
npx wrangler queues create pulse-auth-events
# Set ISSUER in wrangler.jsonc to the public origin.
npx wrangler d1 migrations apply pulse-auth --remote
# GitHub login only:
# npx wrangler secret put GITHUB_CLIENT_SECRET
npx wrangler deploy
```

## License

MIT. See [LICENSE](LICENSE).
