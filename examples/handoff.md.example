# Session 03: Login API Rate Limiting

**Sprint:** v1.3_auth_system
**Status:** ⏳ NEXT
**Prev:** 02_password_reset_handoff.md
**Next:** 04_session_management_handoff.md

---

## Goal

Add per-IP and per-user rate limiting to the login endpoint with a 5-attempt/15-minute window, returning a `429 Too Many Requests` response with a `Retry-After` header when exceeded.

---

## Context

Sessions 01–02 built the JWT token service and password reset flow. The login endpoint (`POST /api/auth/login`) currently has no rate limiting — an attacker can brute-force credentials indefinitely. The password reset endpoint (Session 02) already has rate limiting via `slowapi` keyed on email address, so the pattern exists in the codebase. This session applies the same approach to login, keyed on both IP address and username.

The `slowapi` library is already installed and configured in `src/middleware/rate_limit.py`. Redis is the backing store (shared with session management).

---

## Tasks

- [ ] Add rate limit decorator to `POST /api/auth/login` — 5 attempts per IP per 15 minutes
- [ ] Add secondary rate limit keyed on username — 10 attempts per username per 15 minutes (prevents distributed brute-force against a single account)
- [ ] Return `429 Too Many Requests` with `Retry-After` header (seconds until window reset)
- [ ] Add structured log entry on rate limit trigger — `login.rate_limited` with IP, username, and remaining lockout seconds
- [ ] Ensure successful login does NOT reset the attempt counter (prevents attackers from using a known-valid account to reset their window)

---

## Files

| File | Change |
|------|--------|
| `src/routes/auth.py` | Add rate limit decorators to login endpoint, add `Retry-After` header logic |
| `src/middleware/rate_limit.py` | Add `login_key_func` combining IP + username extraction |
| `tests/test_auth_rate_limit.py` | New file — rate limit trigger, header validation, counter persistence |

---

## Success Criteria

- [ ] 5 failed logins from same IP → 6th returns 429 with `Retry-After` header
- [ ] 10 failed logins for same username (any IP) → 11th returns 429
- [ ] `Retry-After` header contains correct seconds until window expires
- [ ] Successful login within the window does NOT reset the counter
- [ ] `login.rate_limited` log entry emitted with IP, username, lockout seconds
- [ ] Existing password reset rate limiting still works (no regression)
- [ ] Linters pass (`ruff check .`)

---

## Stop Conditions

- Do NOT modify the JWT token service (Session 01 output)
- Do NOT modify the password reset flow (Session 02 output)
- Do NOT implement account lockout (different from rate limiting — that's Session 06 scope if needed)
- Do NOT add CAPTCHA or 2FA challenges — out of sprint scope

---

## Notes

- **Pattern source:** `src/routes/auth.py` password reset endpoint already uses `@limiter.limit("3/hour", key_func=email_key_func)` — follow the same pattern.
- **Dual key strategy:** IP-based catches single-origin brute force. Username-based catches distributed attacks (botnet hitting one account from many IPs). Both are needed.
- **`Retry-After` header:** RFC 7231 §7.1.3 — value is seconds (integer), not a date. Calculate from the rate limiter's window expiry.
- **Testing:** `slowapi` uses in-memory storage in tests by default. Call `limiter.reset()` in test setup to avoid state leaking between tests (known quirk — rate limiter state persists across test fixtures).
- **Why not reset on success:** If an attacker has 4 failed attempts and then tries a known-valid credential, resetting to 0 gives them 5 more guesses. Keeping the counter prevents this "reset trick."

---

*Session 03 of v1.3_auth_system. Previous: 02_password_reset_handoff.md. Next: 04_session_management_handoff.md.*
