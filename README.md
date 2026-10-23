# CTRL

Cloudflare Pages site (`public/`) + Pages Functions backend (`functions/`) + D1 database. Auth: Google, GitHub, passwordless email via Resend. No other dependencies (wrangler is dev-only).

## Setup

1. `npm i`
2. `npx wrangler d1 create ctrl` → put the returned `database_id` in `wrangler.toml`.
3. `npm run db:migrate` (remote) / `npm run db:migrate:local`.
4. Cloudflare dashboard → Pages project → Settings → Bind D1 `DB` (already declared in `wrangler.toml`).
5. Secrets (Pages → Settings → Variables and Secrets, or `npx wrangler pages secret put NAME`):

| Name | Purpose |
|---|---|
| `SESSION_SECRET` | HMAC key for OAuth state cookie. `openssl rand -base64 48` |
| `RESEND_API_KEY` | Resend API key (sending access) |
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | Google OAuth |
| `GITHUB_CLIENT_ID` / `GITHUB_CLIENT_SECRET` | GitHub OAuth |

Non-secret vars live in `wrangler.toml`: `APP_URL` (production origin) and `AUTH_FROM_EMAIL` (`CTRL <noreply@ctrl.monster>`).

6. **Resend:** add and verify the `ctrl.monster` domain (SPF + DKIM DNS records). Sending from noreply@ctrl.monster fails until it is verified.
7. **Google:** OAuth client (Web). Authorized redirect URI: `https://ctrl.monster/api/auth/google/callback`.
8. **GitHub:** OAuth App. Callback URL: `https://ctrl.monster/api/auth/github/callback`.
9. Deploy: `npm run deploy`, or connect the repo (build output dir `public`, no build command).

Providers without credentials are reported by `GET /api/auth/providers`, and the UI shows "isn't available yet" instead of failing.

## API

| Route | |
|---|---|
| `POST /api/auth/email/request` `{email, mode: "login"\|"signup"}` | Sends a single-use link (15 min). Same response whether or not the account exists. |
| `POST /api/auth/email/verify` `{token}` | Consumes the token atomically, creates the user (email verified) if new, starts a session. Errors: `invalid`, `expired`, `used`. |
| `GET /api/auth/{google,github}/start` / `callback` | OAuth code flow, signed state cookie, PKCE (Google). Verified provider email required; links to an existing account with the same verified email. |
| `GET /api/auth/session`, `POST /api/auth/logout` | Current user / end session. |
| `GET /api/auth/providers` | Which methods are configured. |

## Security notes
- Email-link and session tokens are random 256-bit values; only SHA-256 hashes are stored. Cookie `bb_session`: HttpOnly, Secure, SameSite=Lax, 30 days, sliding renewal.
- Email links open `/#/verify?token=…` and need a button press (POST), so mail scanners that prefetch links can't consume them. The token sits in the URL fragment and isn't sent to servers.
- POST endpoints require a same-origin `Origin` header. Rate limits (D1): 5 emails/15 min per address, 20/hour per IP, 30 verifies/hour per IP.
- Strict headers/CSP in `public/_headers`. No secrets in client code.
- Schema is ready for creator profiles (`users.username`, `oauth_accounts`).

## Status
Tested locally (wrangler pages dev + mock Resend): email request → link → verify → session → logout, token reuse/invalid, origin check, rate limit, OAuth start redirect and forged-callback rejection.
**Not tested:** real Resend delivery, real Google/GitHub round trips, production Pages deployment, `_redirects` pretty URLs (`/login` → `/#/login`), UI in a browser.
Deployed: no.
