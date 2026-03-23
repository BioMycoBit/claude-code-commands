# Auth System — JWT + Password Reset + OAuth

**Sprint:** v1.3_auth_system
**Status:** IN PROGRESS
**Date:** 2025-09-15

---

## Progress Tracker

| Session | Status | Date | Notes |
|---------|--------|------|-------|
| 01 - JWT Token Service | ✅ COMPLETE | 2025-09-15 | Access + refresh tokens, RS256 signing, 15min/7d expiry |
| 02 - Password Reset Flow | ✅ COMPLETE | 2025-09-15 | Email token, 1hr expiry, rate-limited to 3/hr |
| 03 - OAuth2 Google Provider | ⏳ NEXT | | Google OAuth callback, account linking, new-user creation |
| 04 - Session Management | PENDING | | Redis session store, concurrent device limits, force-logout |
| 05 - Integration Testing | PENDING | | E2E auth flows, token rotation, edge cases |

**Legend:** ✅ COMPLETE | ⏳ NEXT | PENDING | ❌ BLOCKED

---

## Executive Summary

The application currently uses stateless cookie-based auth with no token rotation, no password reset, and no OAuth. Users who forget passwords must contact support. Third-party login (Google) is the most-requested feature in the backlog. This sprint builds a complete auth layer: JWT access/refresh tokens with RS256 signing, a password reset flow with email verification, Google OAuth2 with account linking, and Redis-backed session management with concurrent device limits.

---

## Context

**Recent velocity:**
- v1.2_user_profiles closed 2025-09-14 (4 sessions, 2 days) — avatar upload, display name, timezone
- v1.1_database_migrations closed 2025-09-12 (3 sessions, 1 day) — Alembic setup, initial schemas

**This week's focus:**
1. JWT token issuance and validation middleware
2. Password reset with email delivery
3. Google OAuth2 provider integration

**Not in scope:** Apple/GitHub OAuth providers, 2FA/TOTP, admin user management UI.

---

## Sprint Map

```
┌─────────────────────────────────────────────────────────┐
│                v1.3_auth_system Sprint Map               │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  PHASE A: Token Foundation                              │
│  ┌─────────────────────────┐                            │
│  │ S01: JWT Token Service  │  (foundation — all else    │
│  │                         │   depends on this)         │
│  └────────────┬────────────┘                            │
│               │                                         │
│  PHASE B: Auth Flows                                    │
│  ┌────────────▼────────────┐                            │
│  │ S02: Password Reset     │  (needs JWT for reset      │
│  │                         │   token signing)           │
│  └────────────┬────────────┘                            │
│               │                                         │
│  ┌────────────▼────────────┐                            │
│  │ S03: OAuth2 Google      │  (needs JWT for post-OAuth │
│  │                         │   session creation)        │
│  └────────────┬────────────┘                            │
│               │                                         │
│  PHASE C: State Management                              │
│  ┌────────────▼────────────┐                            │
│  │ S04: Session Mgmt       │  (needs all auth flows     │
│  │                         │   to manage sessions for)  │
│  └────────────┬────────────┘                            │
│               │                                         │
│  PHASE D: Validation                                    │
│  ┌────────────▼────────────┐                            │
│  │ S05: Integration Test   │                            │
│  └─────────────────────────┘                            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Phase A: Token Foundation

### Session 01: JWT Token Service

| Aspect | Detail |
|--------|--------|
| **Goal** | Create a JWT service that issues and validates access + refresh token pairs with RS256 signing |
| **Effort** | Medium |
| **Blast radius** | `src/auth/`, `src/middleware/`, `config/keys/` |

**Tasks:**

1. **Generate RS256 key pair** — `openssl genrsa` for private key, extract public key. Store in `config/keys/` (gitignored).
2. **Token service module** — `src/auth/token_service.py`: `create_access_token(user_id, roles)` → 15min expiry, `create_refresh_token(user_id)` → 7-day expiry. Both include `jti` (unique token ID) for revocation.
3. **Validation middleware** — `src/middleware/auth.py`: Extract Bearer token from `Authorization` header, verify signature, check expiry, attach `request.user` with decoded claims.
4. **Refresh endpoint** — `POST /api/auth/refresh`: Accept refresh token, validate, issue new access + refresh pair (token rotation).
5. **Revocation list** — Redis SET `revoked_tokens` with TTL matching token expiry. Middleware checks `jti` against this set.

> ### ELI5: JWT Token Pairs
> Imagine a movie theater. The access token is your ticket stub — it gets you into the screening room for the next 15 minutes. The refresh token is your receipt — when the stub expires, you show the receipt at the box office to get a new stub without buying another ticket. If someone steals your receipt, we can add it to a "lost receipts" list (revocation) so they can't use it.

---

## Phase B: Auth Flows

### Session 02: Password Reset Flow

| Aspect | Detail |
|--------|--------|
| **Goal** | Allow users to reset their password via an emailed one-time link with a 1-hour expiry |
| **Effort** | Medium |
| **Blast radius** | `src/auth/`, `src/routes/auth.py`, `templates/email/` |

**Tasks:**

1. **Reset request endpoint** — `POST /api/auth/forgot-password`: Accept email, generate a signed reset token (JWT, 1hr expiry, `purpose: "password_reset"`), send email with reset link. Always return 200 (don't leak whether email exists).
2. **Reset confirmation endpoint** — `POST /api/auth/reset-password`: Accept token + new password, validate token purpose and expiry, hash new password with bcrypt, update database, revoke all existing refresh tokens for that user.
3. **Email template** — Clean HTML email with reset button linking to `/reset-password?token=<jwt>`. Plain text fallback included.
4. **Rate limiting** — 3 reset requests per email per hour. Uses `slowapi` with custom key function keyed on email address.

> ### ELI5: Why Always Return 200
> If the "forgot password" endpoint returned "email not found" for unknown emails, an attacker could try thousands of emails to discover which ones have accounts. By always saying "if that email exists, we sent a link," we don't reveal anything. The user with a real account gets the email; the attacker gets nothing useful.

---

### Session 03: OAuth2 Google Provider

| Aspect | Detail |
|--------|--------|
| **Goal** | Allow users to sign in with their Google account and link it to an existing or new local account |
| **Effort** | Medium-High |
| **Blast radius** | `src/auth/oauth/`, `src/routes/auth.py`, `config/settings.py` |

**Tasks:**

1. **OAuth config** — Add `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` to settings. Create `src/auth/oauth/google.py` with authorization URL builder and token exchange logic.
2. **Authorization redirect** — `GET /api/auth/oauth/google`: Generate `state` parameter (CSRF protection, stored in Redis with 10min TTL), redirect to Google's authorization endpoint with `openid email profile` scopes.
3. **Callback handler** — `GET /api/auth/oauth/google/callback`: Validate `state`, exchange `code` for Google access token, fetch user info from Google's userinfo endpoint.
4. **Account linking** — If email matches existing user → link Google ID, issue JWT pair. If new email → create user account, issue JWT pair. If Google ID already linked to different email → reject with clear error.
5. **Frontend button** — "Sign in with Google" button on login page, styled per Google's branding guidelines.

> ### Deep Dive: OAuth2 State Parameter
> The `state` parameter prevents CSRF attacks on the OAuth callback. Without it, an attacker could craft a URL like `/callback?code=ATTACKER_CODE` and trick a logged-in user into clicking it — linking the attacker's Google account to the victim's local account. The flow: (1) server generates random `state`, stores in Redis with 10min TTL, (2) `state` is included in the Google redirect URL, (3) Google echoes `state` back in the callback, (4) server verifies the returned `state` matches what's in Redis. If it doesn't match (or is missing), the callback is rejected. The Redis TTL ensures stale authorization attempts expire automatically.

---

## Phase C: State Management

### Session 04: Session Management

| Aspect | Detail |
|--------|--------|
| **Goal** | Track active sessions per user in Redis with a 5-device concurrent limit and force-logout capability |
| **Effort** | Medium |
| **Blast radius** | `src/auth/session.py`, `src/routes/auth.py`, `src/middleware/auth.py` |

**Tasks:**

1. **Session store** — Redis HASH `sessions:{user_id}` with fields per device: `{jti: {device_name, ip, created_at, last_seen}}`. TTL matches refresh token expiry (7 days).
2. **Session creation** — On login/OAuth/refresh, add session entry. If user has 5+ active sessions, reject with `429 Too Many Sessions` and list active devices.
3. **Active sessions endpoint** — `GET /api/auth/sessions`: Return list of active sessions with device info and last-seen timestamps.
4. **Force-logout endpoint** — `DELETE /api/auth/sessions/{jti}`: Revoke a specific session's refresh token and remove from session store. `DELETE /api/auth/sessions` (no jti): Revoke all sessions except current.
5. **Middleware update** — Auth middleware updates `last_seen` timestamp on each authenticated request (debounced to once per minute to reduce Redis writes).

---

## Phase D: Validation

### Session 05: Integration Testing

| Aspect | Detail |
|--------|--------|
| **Goal** | End-to-end validation of all auth flows working together |
| **Effort** | Low-Medium |
| **Blast radius** | Test scripts only |

**Tasks:**

1. **Token lifecycle** — Login → access token works → wait 15min (or mock expiry) → 401 → refresh → new access token works → revoke refresh → refresh fails
2. **Password reset** — Request reset → token received → reset password → old password fails → new password works → all sessions revoked
3. **OAuth flow** — Redirect to Google → callback with valid code → JWT issued → account linked → subsequent Google login reuses account
4. **Session limits** — Login from 5 devices → 6th rejected → force-logout device 1 → 6th succeeds
5. **Cross-feature** — OAuth login → password reset (user has no password yet → skip or set initial) → session shows both OAuth and password login methods

---

## Success Criteria

| Milestone | Validation |
|-----------|------------|
| JWT tokens issued on login | `POST /api/auth/login` returns `access_token` + `refresh_token` |
| Token refresh works | `POST /api/auth/refresh` returns new token pair, old refresh token revoked |
| Password reset email sent | `POST /api/auth/forgot-password` triggers email with valid reset link |
| Password actually resets | `POST /api/auth/reset-password` with valid token updates password hash |
| Google OAuth creates account | Full OAuth flow results in new user with linked Google ID |
| Google OAuth links existing | OAuth with matching email links to existing account |
| Session limit enforced | 6th concurrent login returns 429 with active device list |
| Force-logout works | `DELETE /api/auth/sessions/{jti}` revokes that session's tokens |
| Linters pass | `ruff check .` and `npm run lint` clean |

---

## Handoff Documents

```
work/v1.3_auth_system/
├── 00_auth_system_overview_v4.md     ← this document
├── 01_jwt_token_service_handoff.md
├── 02_password_reset_handoff.md
├── 03_oauth_google_handoff.md
├── 04_session_management_handoff.md
└── 05_integration_test_handoff.md
```

---

## Risks

| Risk | Prob | Impact | Mitigation |
|------|------|--------|------------|
| RS256 key rotation not handled | Medium | High | Document manual rotation process. Automated rotation is v2.0 scope. |
| Email delivery fails silently | Medium | Medium | Log all email send attempts. Add health check endpoint for email service. |
| Google OAuth callback timing out | Low | Medium | 10-second timeout on token exchange. Retry once before failing. |
| Refresh token stolen from client | Low | Critical | Token rotation on every refresh (old token revoked). Short access token TTL (15min). |

---

## Dependencies

```
S01 (JWT Token Service)
         │
         ├──────────────────────┐
         │                      │
┌────────▼──────────┐  ┌───────▼────────────┐
│ S02: Password     │  │ S03: OAuth Google   │
│ Reset             │  │                     │
└────────┬──────────┘  └───────┬─────────────┘
         │                     │
         └──────────┬──────────┘
                    │
         ┌──────────▼──────────┐
         │ S04: Session Mgmt   │
         └──────────┬──────────┘
                    │
         ┌──────────▼──────────┐
         │ S05: Integration    │
         └─────────────────────┘
```

S01 is the foundation — all other sessions depend on it. S02 and S03 are independent of each other (can run in parallel). S04 needs all auth flows complete. S05 validates everything.

---

## Go/No-Go

- [ ] Plan approved
- [ ] RS256 key pair generated and stored in `config/keys/`
- [ ] Redis running (`docker-compose up redis`)
- [ ] Email service configured (SMTP credentials in `.env`)
- [ ] Google OAuth app created (client ID + secret in `.env`)

---

## Keywords

| Term | Term | Term | Term |
|------|------|------|------|
| JWT | RS256 | access token | refresh token |
| token rotation | jti | revocation | Bearer header |
| password reset | bcrypt | email template | rate limiting |
| OAuth2 | Google provider | state parameter | account linking |
| Redis session | concurrent devices | force-logout | session TTL |
| CSRF protection | slowapi | middleware | claims |

---

*This is an example sprint overview generated by `/sprint`. It demonstrates the full document structure: Progress Tracker, Executive Summary, Sprint Map, Phase tables with session details, ELI5 and Deep Dive boxes, Success Criteria, Risks, Dependencies, and Keywords. Your actual sprint will be tailored to your project's codebase, tech stack, and current state.*
