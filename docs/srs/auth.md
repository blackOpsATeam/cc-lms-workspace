# 🧩 MODULE: AUTHENTICATION & AUTHORIZATION

## 1. Purpose

Enable secure user access to CC-LMS for all roles (Admin, Teacher, Student, Accountant), verify identity at login, manage active sessions via JWT, and enforce role-based access on every protected API. The module owns *who is allowed in and what they can call*; it does not own user records or profile data — those belong to User Management and Profile Management respectively.

## 2. Scope

**In Scope**

- Login by email + password or phone + password
- Logout and session/token invalidation
- Password reset via email or SMS
- JWT issuance, expiry, and validation
- Refresh-token rotation (per ADR-0002 — access + refresh)
- Optional OTP-based 2FA (configurable per role or per user)
- Role-based API access enforcement
- Failed-attempt tracking and temporary account lockout
- Rate limiting on authentication endpoints

**Out of Scope**

- User creation, edit, and deactivation — handled by User Management and Admission Management
- Profile data (name, photo, address) — handled by Profile Management
- Role definition and permission catalog beyond the role enum — handled by User Management
- Audit log storage — written via the central `audit_log` table (schema location TBD; conventions noted in `codebases/CLAUDE.md`)
- SMS/email delivery infrastructure — Notification Management owns the channel; this module hands off a message and recipient

## 3. Actors

- **Admin** — logs in to manage the system; may have mandatory 2FA per business decision
- **Teacher** — logs in to access classes, attendance, assessment
- **Student** — logs in to access their workspace
- **Accountant** — logs in to access billing and payroll
- **System** — generates and signs tokens, evaluates expiry, dispatches OTP/reset messages to Notification, enforces lockout

## 4. Core Concepts

### Credential

The pair `(identifier, password)` a user presents at login. `identifier` is either a verified email address or a Bangladesh phone number in E.164 form (`+8801XXXXXXXXX`). Each user has both a hashed password and zero, one, or both of email / phone.

### Session

A server-acknowledged state tied to one issued access token + refresh token pair. A session is identified by the refresh-token row in `AuthSession`. A user may have multiple concurrent sessions across devices; each session can be revoked independently.

### Access Token

Short-lived signed JWT (default 24 h, configurable) carried in the `Authorization: Bearer` header. Claims: `user_id`, `tenant_id`, `role`, `session_id` (= `AuthSession.id`, used as the JWT `jti` for revocation lookup), `iat`, `exp`. Signature + expiry are verified statelessly; `session_id` is additionally checked against a revocation deny-list (see FR-AUTH-012); every downstream API guard cross-checks the JWT `tenant_id` against the resource's `tenant_id` and returns `404` on mismatch (existence-hiding) per ADR-0003.

> **Multi-tenancy:** Per ADR-0003 (accepted, shared-schema), every domain table in this module — `User`, `AuthSession`, `PasswordResetToken`, `OtpChallenge`, `RolePolicy` — carries a `tenant_id` column (UUID, NOT NULL, FK → `Tenant.id`). Identifier uniqueness on `User.email` / `User.phone` is scoped per tenant (partial unique index on `(tenant_id, email)` and `(tenant_id, phone)`). Login resolves the user's tenant from the identifier; the `tenant_id` is captured in the issued token. Prisma middleware injects `tenant_id` on every query.

### Refresh Token

Long-lived opaque token stored server-side in `AuthSession`. Used to mint a new access token without re-authenticating. Rotated on every use.

### Reset Token

One-time, time-limited token (default 15 min) sent to the user's email or phone for password reset. Single-use; invalidated on consumption or expiry.

### OTP (One-Time Password)

Six-digit numeric code, valid for 5 minutes, sent via SMS or email as the second factor when 2FA is enabled for a user or role.

### Lockout

A temporary block on a user account after a configurable number of consecutive failed login attempts (default 5). Lockout duration is configurable (default 15 min). Lockout is per-account, not per-IP.

## 5. Functional Requirements

### 5.1 Authentication

- **FR-AUTH-001:** System shall allow login using email + password.
- **FR-AUTH-002:** System shall allow login using phone (E.164 `+8801XXXXXXXXX`) + password.
- **FR-AUTH-003:** System shall validate credentials by comparing the submitted password against the stored hash before granting access.
- **FR-AUTH-004:** System shall reject login for users whose `is_active = false` with HTTP 403 and error code `ACCOUNT_INACTIVE` (chosen over 401 because the credentials themselves were valid).
- **FR-AUTH-005:** System shall reject login for users currently in a lockout window and return error code `ACCOUNT_LOCKED` with the unlock-at timestamp.

### 5.2 Session and Token Management

- **FR-AUTH-006:** System shall issue a signed JWT access token and an opaque refresh token on successful login.
- **FR-AUTH-007:** System shall set the access-token TTL from a configurable value (default 24 hours).
- **FR-AUTH-008:** System shall set the refresh-token TTL from a configurable value (default 30 days).
- **FR-AUTH-009:** System shall persist one `AuthSession` row per refresh token, recording `user_id`, `device_info`, `issued_at`, `expires_at`, `revoked_at`.
- **FR-AUTH-010:** System shall accept a refresh token at `POST /auth/refresh`, issue a new access + refresh pair, and mark the old refresh token revoked (rotation).
- **FR-AUTH-011:** System shall invalidate the current session on logout by setting `revoked_at` on the corresponding `AuthSession`.
- **FR-AUTH-012:** System shall reject any access token whose `session_id` claim corresponds to a revoked `AuthSession`. Implementation: a Redis-backed deny-list keyed by `session_id`, populated on every revocation event (logout, refresh rotation, password reset, user deactivation, admin revoke), with entry TTL equal to the access-token TTL so entries expire naturally.
- **FR-AUTH-013:** Authenticated user shall be able to list and revoke their own active sessions.
- **FR-AUTH-034:** Admin shall be able to list any user's active sessions and revoke any session.
- **FR-AUTH-035:** System shall expose an internal endpoint `POST /internal/users/{user_id}/revoke-sessions`, callable by User Management when a user is deactivated, that sets `revoked_at = now` on all non-revoked `AuthSession` rows for the given `user_id` and adds each `session_id` to the revocation deny-list. The internal endpoint is authenticated by a shared service token (out of scope for this module).

### 5.3 Password Management

- **FR-AUTH-014:** System shall accept a forgot-password request by email or phone identifier.
- **FR-AUTH-015:** System shall generate a cryptographically secure `PasswordResetToken` and store its hash with `user_id`, `expires_at`, `used = false`.
- **FR-AUTH-016:** System shall hand the reset link / token to Notification Management for delivery via email or SMS based on the identifier used.
- **FR-AUTH-017:** System shall set reset-token TTL from a configurable value (default 15 minutes).
- **FR-AUTH-018:** System shall accept a password reset using a valid, unused, unexpired token and update `password_hash` for the owning user.
- **FR-AUTH-019:** System shall mark a reset token `used = true` on successful consumption and reject any subsequent use.
- **FR-AUTH-020:** System shall revoke all of a user's active `AuthSession` rows on successful password reset.
- **FR-AUTH-021:** System shall not disclose whether a submitted forgot-password identifier matches a real account (response is uniform).

### 5.4 Optional 2FA (OTP)

- **FR-AUTH-022:** System shall support a per-user `two_fa_enabled` flag and a per-role `two_fa_required` policy.
- **FR-AUTH-023:** When 2FA applies, System shall, after successful password verification, generate a 6-digit OTP and dispatch it via Notification Management (SMS for phone identifiers, email for email identifiers).
- **FR-AUTH-024:** System shall set OTP TTL from a configurable value (default 5 minutes).
- **FR-AUTH-025:** System shall complete login (issue tokens) only after the user submits a valid, unexpired OTP.
- **FR-AUTH-026:** System shall invalidate an OTP on first use, on expiry, or after a configurable number of failed verification attempts (default 5).

### 5.5 Role-Based Access

- **FR-AUTH-027:** System shall assign exactly one role per user from the enum `{ ADMIN, TEACHER, STUDENT, ACCOUNTANT }`.
- **FR-AUTH-028:** System shall include the user's role in the access-token claims.
- **FR-AUTH-029:** System shall enforce role-based API access via a guard that reads the role claim and rejects requests outside the endpoint's allow-list with HTTP 403.
- **FR-AUTH-030:** System shall reject any request bearing a missing, malformed, or expired access token with HTTP 401.

### 5.6 Lockout and Rate Limiting

- **FR-AUTH-031:** System shall increment a per-user `failed_login_count` on each invalid-password attempt and reset it to 0 on a successful login.
- **FR-AUTH-032:** System shall set `locked_until = now + lockout_duration` (default 15 min) when `failed_login_count` reaches the threshold (default 5), and clear `failed_login_count`.
- **FR-AUTH-033:** System shall rate-limit `/auth/login`, `/auth/forgot-password`, and `/auth/refresh` per IP and per identifier using the limits defined in section 14 (Non-Functional Requirements). Exceeding any limit returns HTTP 429.

## 6. Business Rules

- Password must be **minimum 8 characters (proposed; pending confirmation — see OQ-4)**, contain at least one letter and one digit.
- Phone identifier must be **Bangladesh E.164 format `+8801XXXXXXXXX`** (13 chars total). Display format `+880 1712 345 678` is for UI only.
- Reset token validity: **15 minutes** (configurable).
- OTP validity: **5 minutes** (configurable).
- Maximum failed login attempts: **5** → temporary lockout for **15 minutes** (both configurable).
- Refresh-token TTL: **30 days** (configurable); rotated on every use.
- Access-token (JWT) TTL: **24 hours** (configurable).
- Concurrent sessions per user: **unlimited across devices** by default; Admin may revoke any.
- A successful password reset revokes all of that user's existing sessions.
- A user with `is_active = false` cannot log in and any existing sessions are revoked.

## 7. User Flow / Process Flow

### 7.1 Login (no 2FA)

1. User enters email or phone + password on the login screen.
2. Client validates identifier format and password presence.
3. Client `POST /auth/login` with `{ identifier, password }`.
4. System resolves the user by email or phone; if not found, returns generic `INVALID_CREDENTIALS`.
5. System checks `is_active`; if false, returns `ACCOUNT_INACTIVE`.
6. System checks `locked_until > now`; if true, returns `ACCOUNT_LOCKED` with `unlock_at`.
7. System verifies password against `password_hash`.
8. On failure: increment `failed_login_count`; if threshold reached, set `locked_until` and return `ACCOUNT_LOCKED`; otherwise return `INVALID_CREDENTIALS`.
9. On success: reset `failed_login_count`, update `last_login_at`, check 2FA policy.
10. If 2FA not required: issue access + refresh tokens, create `AuthSession`, return tokens + user summary.
11. Client stores tokens and routes to the role-appropriate dashboard.

### 7.2 Login (with 2FA)

1. Steps 1–9 of 7.1.
2. System detects 2FA applies (`user.two_fa_enabled = true` or role policy requires it).
3. System generates OTP, stores it hashed with `expires_at = now + 5 min`, dispatches via Notification.
4. System returns `OTP_REQUIRED` with a short-lived `otp_challenge_id` (does **not** issue tokens yet).
5. User enters OTP; client `POST /auth/verify-otp` with `{ otp_challenge_id, otp }`.
6. System validates OTP (matches, not expired, not exceeded failure count).
7. On success: issue access + refresh tokens, create `AuthSession`, return tokens + user summary.
8. On failure: increment OTP attempt count; if exceeded, invalidate challenge and require login restart.

### 7.3 Forgot Password

1. User clicks "Forgot password" and submits email or phone.
2. Client `POST /auth/forgot-password` with `{ identifier }`.
3. System resolves user; whether or not found, returns the same generic 200 response.
4. If found: generate `PasswordResetToken`, store hash + `expires_at`, hand link/token to Notification (email or SMS based on identifier type).
5. User opens link, sees password-reset form.
6. Client `POST /auth/reset-password` with `{ token, new_password }`.
7. System verifies token validity + unused + unexpired, updates `password_hash`, marks token `used = true`, revokes all of the user's `AuthSession` rows, returns success.

### 7.4 Logout

1. User clicks logout (or session is force-revoked by Admin).
2. Client `POST /auth/logout` with current access token in header.
3. System reads `session_id` from the access-token claims, sets `revoked_at = now` on the matching `AuthSession`, and adds `session_id` to the revocation deny-list with TTL equal to the access token's remaining lifetime.
4. Client clears local tokens and routes to login screen.

### 7.5 Token Refresh

1. Access token nears expiry or returns 401.
2. Client `POST /auth/refresh` with `{ refresh_token }`.
3. System verifies refresh token: exists, not revoked, not expired.
4. System mints new access + refresh tokens, revokes the old refresh token (rotation), updates the `AuthSession` row.
5. Client replaces stored tokens and retries the original request.

## 8. Data Model

### `User` *(owned by User Management; fields below are the subset Auth reads/writes)*

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| email | string, nullable, unique | Login identifier (one of email or phone is required) |
| phone | string, nullable, unique | E.164 `+8801XXXXXXXXX` |
| password_hash | string | bcrypt hash; never returned by any API |
| role | enum(`ADMIN`,`TEACHER`,`STUDENT`,`ACCOUNTANT`) | Single role per user |
| is_active | boolean | If false, login is blocked |
| two_fa_enabled | boolean | Per-user 2FA opt-in |
| failed_login_count | integer | Reset on success, incremented on failed password |
| locked_until | Timestamp, nullable | If set and `> now`, login is blocked |
| last_login_at | Timestamp, nullable | Updated on successful login |
| created_at | Timestamp | |
| updated_at | Timestamp | |
| deleted_at | Timestamp, nullable | Soft delete |

### `AuthSession`

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key, also the refresh-token `jti` |
| user_id | UUID, FK → User.id | Owner |
| refresh_token_hash | string | SHA-256 of the opaque refresh token |
| device_info | JSONB | User-agent, IP, optional device name |
| issued_at | Timestamp | |
| expires_at | Timestamp | issued_at + refresh TTL |
| revoked_at | Timestamp, nullable | Set on logout, rotation, password reset, or admin revoke |
| created_at | Timestamp | |
| updated_at | Timestamp | |

### `PasswordResetToken`

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key |
| user_id | UUID, FK → User.id | |
| token_hash | string | SHA-256 of the opaque token |
| expires_at | Timestamp | issued_at + 15 min |
| used | boolean | Single-use flag |
| used_at | Timestamp, nullable | |
| channel | enum(`EMAIL`,`SMS`) | Where it was sent |
| created_at | Timestamp | |

### `RolePolicy`

| Field | Type | Description |
|---|---|---|
| role | enum(`ADMIN`,`TEACHER`,`STUDENT`,`ACCOUNTANT`) | Primary key |
| two_fa_required | boolean | If true, every user with this role must complete OTP at login regardless of `User.two_fa_enabled` |
| updated_at | Timestamp | |
| updated_by | UUID, FK → User.id | Admin who last edited the policy |

### `OtpChallenge`

| Field | Type | Description |
|---|---|---|
| id | UUID | Primary key, returned to client as `otp_challenge_id` |
| user_id | UUID, FK → User.id | |
| otp_hash | string | SHA-256 of the 6-digit code |
| expires_at | Timestamp | issued_at + 5 min |
| attempts | integer | Failed verification count |
| consumed | boolean | True after a successful verify |
| channel | enum(`EMAIL`,`SMS`) | |
| created_at | Timestamp | |

## 9. API Contracts

### Login

```http
POST /auth/login
```

**Request**

```json
{
  "identifier": "rahim@example.com",
  "password": "S3cretPass"
}
```

**Response (200) — no 2FA**

```json
{
  "access_token": "<jwt>",
  "refresh_token": "<opaque>",
  "access_expires_in": 86400,
  "user": {
    "id": "0192c2cb-...-...",
    "role": "STUDENT"
  }
}
```

**Response (200) — 2FA required**

```json
{
  "otp_required": true,
  "otp_challenge_id": "0192c2cb-...-...",
  "channel": "SMS"
}
```

**Errors**

- `400` — `INVALID_INPUT` (malformed identifier or empty password)
- `401` — `INVALID_CREDENTIALS`
- `403` — `ACCOUNT_INACTIVE`
- `423` — `ACCOUNT_LOCKED` (includes `unlock_at`)
- `429` — `RATE_LIMITED`

Satisfies: FR-AUTH-001, 002, 003, 004, 005, 006, 028, 031, 032, 033.

### Verify OTP

```http
POST /auth/verify-otp
```

**Request**

```json
{
  "otp_challenge_id": "0192c2cb-...-...",
  "otp": "482913"
}
```

**Response (200)**

```json
{
  "access_token": "<jwt>",
  "refresh_token": "<opaque>",
  "access_expires_in": 86400,
  "user": { "id": "...", "role": "ADMIN" }
}
```

**Errors**

- `400` — `INVALID_INPUT`
- `401` — `OTP_INVALID`
- `410` — `OTP_EXPIRED` or `CHALLENGE_EXHAUSTED`

Satisfies: FR-AUTH-023, 024, 025, 026.

### Refresh

```http
POST /auth/refresh
```

**Request**

```json
{ "refresh_token": "<opaque>" }
```

**Response (200)**

```json
{
  "access_token": "<jwt>",
  "refresh_token": "<new-opaque>",
  "access_expires_in": 86400
}
```

**Errors**

- `401` — `REFRESH_INVALID` (unknown, revoked, or expired)
- `429` — `RATE_LIMITED`

Satisfies: FR-AUTH-010, 012.

### Logout

```http
POST /auth/logout
```

**Headers**

```
Authorization: Bearer <access_token>
```

**Response (200)**

```json
{ "message": "Logged out" }
```

**Errors**

- `401` — missing/invalid token

Satisfies: FR-AUTH-011.

### Forgot Password

```http
POST /auth/forgot-password
```

**Request**

```json
{ "identifier": "rahim@example.com" }
```

**Response (200)** — uniform whether or not the identifier matches

```json
{ "message": "If the account exists, a reset link has been sent." }
```

**Errors**

- `400` — `INVALID_INPUT`
- `429` — `RATE_LIMITED`

Satisfies: FR-AUTH-014, 015, 016, 017, 021, 033.

### Reset Password

```http
POST /auth/reset-password
```

**Request**

```json
{
  "token": "<opaque reset token>",
  "new_password": "N3wPass!"
}
```

**Response (200)**

```json
{ "message": "Password updated. Please log in." }
```

**Errors**

- `400` — `INVALID_INPUT` (weak password, malformed token)
- `401` — `RESET_TOKEN_INVALID` (unknown, used, or expired)

Satisfies: FR-AUTH-018, 019, 020.

### List My Sessions

```http
GET /auth/sessions
```

**Headers**

```
Authorization: Bearer <access_token>
```

**Response (200)**

```json
{
  "sessions": [
    {
      "id": "0192c2cb-...",
      "device_info": { "ua": "Chrome/130 Android", "ip": "103.x.x.x" },
      "issued_at": "2026-05-18T08:12:00+06:00",
      "expires_at": "2026-06-17T08:12:00+06:00",
      "current": true
    }
  ]
}
```

Satisfies: FR-AUTH-013.

### Revoke Session

```http
DELETE /auth/sessions/{id}
```

**Headers**

```
Authorization: Bearer <access_token>
```

**Response (204)** — empty body

**Errors**

- `403` — `FORBIDDEN` (not owner and not Admin)
- `404` — `NOT_FOUND`

Satisfies: FR-AUTH-013, FR-AUTH-034.

### Admin — List Sessions for a User

```http
GET /admin/users/{user_id}/sessions
```

**Headers**

```
Authorization: Bearer <admin access_token>
```

**Response (200)** — same shape as `GET /auth/sessions`, returning the target user's sessions instead of the caller's.

**Errors**

- `403` — `FORBIDDEN` (caller is not Admin)
- `404` — `NOT_FOUND` (no such user)

Satisfies: FR-AUTH-034.

### Internal — Revoke All Sessions for a User

```http
POST /internal/users/{user_id}/revoke-sessions
```

**Headers**

```
X-Internal-Token: <shared service token>
```

**Response (204)** — empty body. Idempotent: re-calling on an already-revoked user is a no-op.

**Errors**

- `401` — `INTERNAL_UNAUTHORIZED` (missing or invalid service token)
- `404` — `NOT_FOUND` (no such user)

Satisfies: FR-AUTH-035. Called by User Management on deactivation.

## 10. UI Components

### Screen: Login

- **Components:** Identifier input (email or phone, auto-detect), password input with show/hide, "Forgot password?" link, primary Login button, language toggle (EN ↔ BN).
- **Actions:** Submit credentials, navigate to Forgot Password, switch language.
- **States:** idle, validating, submitting, error (with field-level + form-level messages), locked-out (shows unlock-at), 2FA-required (transitions to OTP screen).

### Screen: OTP Verification

- **Components:** 6-digit OTP input (auto-advance, paste-safe), countdown timer (mm:ss until expiry), "Resend code" link (disabled until cooldown elapses), Verify button.
- **Actions:** Submit OTP, request resend, cancel back to login.
- **States:** idle, submitting, error (invalid code), expired (offer resend), exhausted (back to login).

### Screen: Forgot Password

- **Components:** Identifier input (email or phone), Submit button, back-to-login link.
- **Actions:** Submit identifier.
- **States:** idle, submitting, sent (uniform success message regardless of match).

### Screen: Reset Password

- **Components:** New-password input, confirm-password input, password-strength hint, Submit button.
- **Actions:** Submit new password.
- **States:** idle, submitting, success (auto-route to login after 2 s), token-invalid (shows "request a new link"), token-expired (same).

### Screen: My Sessions *(in profile area)*

- **Components:** Table of active sessions (device, IP, last seen, current badge), Revoke button per row.
- **Actions:** Revoke a session.
- **States:** loading, empty (only the current session), populated.

## 11. Validation Rules

- **Identifier:** must match either email RFC-5322 or E.164 `+8801XXXXXXXXX` (13 chars). Trim whitespace; lowercase email before lookup.
- **Password (on set/reset):** length ≥ 8, must contain ≥ 1 letter and ≥ 1 digit, must not equal the identifier, must not be in a small project-defined common-password blocklist.
- **Password (on login):** non-empty; no other client-side rules (server-side hash comparison decides).
- **OTP:** exactly 6 digits, numeric only.
- **Reset token / refresh token:** opaque strings, must decode to a known row by hash lookup; never compared in plain text after issuance.
- **Bangla-friendly inputs:** password field accepts ASCII only (server enforces); identifier rejects Bangla digits, accepts only Western Arabic 0–9 for the phone form.

## 12. Edge Cases

- Invalid credentials — generic `INVALID_CREDENTIALS` (do not distinguish "user not found" from "wrong password").
- Expired reset token — `RESET_TOKEN_INVALID`, prompt to request a new link.
- Reset token reused after success — `RESET_TOKEN_INVALID`.
- OTP entered after expiry — `OTP_EXPIRED`, offer resend (subject to cooldown).
- OTP exhausted (too many wrong attempts) — challenge invalidated, user must restart login.
- Account locked due to failed attempts — response includes `unlock_at`; further attempts during the window also return locked.
- User deactivated mid-session — next request returns 401 because session has been revoked by the deactivation flow (User Management calls Auth to revoke).
- Password reset while another session is active — that other session is revoked; its next request returns 401.
- Network failure mid-login — client may retry; idempotent because nothing is committed before token issuance; OTP challenges expire on their own.
- Clock skew on the client — access-token expiry is server-validated; client may see "valid" tokens the server rejects; client falls back to refresh.
- Concurrent refresh from two tabs — only the first rotation succeeds; the second returns `REFRESH_INVALID`, prompting re-login.
- JWT secret rotated — old tokens fail signature check; client falls back to refresh, which also fails if signed by old secret; user re-logs in. (Document a planned rotation window.)
- SMS / email delivery failure — Notification reports failure; user may request resend after cooldown.
- Identifier collision risk — a phone string and an email string cannot collide by format, so a single `identifier` field is unambiguous.
- OTP resend while an existing challenge is still valid — issuing a new OTP invalidates the prior `OtpChallenge` for the same user (marked `consumed = true`); only the most recent challenge is honoured.
- Multiple valid reset tokens for the same user — issuing a new `PasswordResetToken` invalidates all prior unused tokens for that user (sets `used = true`); only the most recent token is honoured.
- User changes phone or email mid-flow — `AuthSession` rows are keyed by `user_id`, so existing sessions remain valid; the next password-reset SMS/email is sent to the updated identifier. A user who initiates a reset using the old identifier before the change propagates may see the link delivered to the new channel.
- Role policy `two_fa_required` toggled from false → true mid-session — existing sessions issued without OTP remain valid until their access token expires; the next refresh enforces 2FA. (Tracked as OQ — alternative is immediate revoke of affected sessions.)
- Sole-Admin lockout — if the only Admin account triggers lockout (5 failed attempts), the **only** recovery path in this SRS is the 15-minute auto-unlock. No operational override exists; tracked as OQ.
- Refresh-token presented after its `AuthSession` row is hard-deleted (e.g. by a future cleanup job) — endpoint returns the same `REFRESH_INVALID` as a revoked token; clients re-authenticate.

## 13. Dependencies

- **User Management** — owns the `User` row, `role`, `is_active`. Auth reads on login and is notified on deactivation to revoke sessions.
- **Notification Management** — delivers OTP, password-reset links, and (optionally) login-from-new-device alerts via email and SMS.
- **Profile Management** — consumes the authenticated user context; not a dependency of Auth.
- **Central `audit_log`** (schema location TBD — to be defined in a shared infrastructure doc or a dedicated `audit.md` SRS; conventions noted in `codebases/CLAUDE.md`) — Auth writes events: `LOGIN_SUCCESS`, `LOGIN_FAILED`, `LOGOUT`, `PASSWORD_RESET`, `SESSION_REVOKED`, `LOCKOUT_TRIGGERED`.
- **Redis** — backs the access-token revocation deny-list (FR-AUTH-012) and the rate-limit counters (FR-AUTH-033). Already in the Track B stack per ADR-0002.

## 14. Non-Functional Requirements

- **Performance:** `/auth/login` p95 < 2 s under nominal load; refresh < 500 ms p95.
- **Security:** Passwords hashed with bcrypt (cost ≥ 12) or argon2id. JWT signed with a secret stored in env / secret manager; rotation supported. All endpoints HTTPS-only. Tokens never logged.
- **Scale:** Support 1,000+ concurrent active sessions per instance; horizontal scale through stateless access-token validation.
- **Accessibility:** Login screen meets WCAG 2.1 AA — labelled inputs, visible focus, no colour-only error indication, keyboard-first OTP entry.
- **Internationalization:** All user-facing strings translatable EN ↔ BN. Error messages safe for both languages (no concatenation that breaks Bangla grammar).
- **Observability:** Structured logs (Pino) for every auth event with `user_id`, `event`, `outcome`, `ip`; correlation id passed through.
- **Rate limits (authoritative — referenced by FR-AUTH-033):** `/auth/login` — 10 / minute / IP, 5 / minute / identifier. `/auth/forgot-password` — 3 / hour / identifier. `/auth/refresh` — 30 / minute / refresh token. **Known limitation:** per-IP limits may catch legitimate users sharing a VPN/NAT egress IP; per-identifier limits remain the primary defence.

## 15. Assumptions

- Users are pre-created via Admission or User Management; Auth does not handle signup.
- Notification Management is available; if it is down at the moment of password reset or OTP, the request fails gracefully and the user can retry.
- A single role per user is sufficient for the foreseeable launch scope (no multi-role accounts).
- Email and phone are not both required at user creation, but at least one must be present.
- HTTPS termination is at the edge; the API server trusts `X-Forwarded-For` only from known proxies.
- Time on the server is reliable (NTP-synced); client clock skew is tolerated for token validation up to the access-token TTL.

## 16. Open Questions

- Should 2FA be **mandatory for Admin** and **optional for others**, or fully per-user? *(Original SRS left this open.)*
- Will social login (Google) be supported in a later phase, and if so, how are existing email-based accounts linked?
- Should we cap **maximum concurrent sessions per user** (e.g. 5 devices) rather than leave it unlimited? Impacts `AuthSession` policy.
- Confirm raising password minimum from **6 to 8 characters** with the manager. Original SRS said 6; flagged as insufficient.
- Should login-from-new-device send a notification email/SMS by default, and is this configurable per user?
- Should `last_login_at` be exposed on the user's own profile (transparency) and on the Admin user-detail screen?
- Lockout policy: per-account is specified, but do we *also* want per-IP throttling beyond the rate limit (e.g. soft-block an IP after N distinct-account failures)?
- Token-revocation propagation: this SRS specifies a Redis-backed deny-list (FR-AUTH-012). Confirm with the manager that the operational cost of a Redis lookup on every authenticated request is acceptable, or whether shortening access-token TTL (e.g. 15 min) and accepting a small revocation window is preferred.
- ~~Multi-tenancy (ADR-0003)~~ — **resolved 2026-05-18:** ADR-0003 accepted Option B (shared-schema multi-tenant). JWT carries `tenant_id`; every Auth table has `tenant_id`; identifier uniqueness is per-tenant.
- Sole-Admin lockout recovery: do we add an operational override (CLI tool, break-glass account) for the case where the only Admin is locked out, or is the 15-minute auto-unlock acceptable?
- Role policy change mid-session: when `two_fa_required` flips from false → true for a role, should existing sessions for that role be immediately revoked, or are they allowed to live until their access token expires (current default)?
- Should the `RolePolicy` table support per-role overrides for other auth parameters (lockout threshold, access-token TTL, OTP TTL), or are those only global config?
