# Authentication Flow — Octo Time

## Table of Contents

1. [Overview](#overview)
2. [Token Strategy](#token-strategy)
3. [Token Storage](#token-storage)
4. [JWT Payload Structure](#jwt-payload-structure)
5. [Database Tables](#database-tables)
6. [Redis Keys](#redis-keys)
7. [Google OAuth 2.0 Flow](#google-oauth-20-flow)
8. [Native Registration Flow](#native-registration-flow)
9. [Native Login Flow](#native-login-flow)
10. [Token Refresh Flow](#token-refresh-flow)
11. [Logout Flow](#logout-flow)
12. [Session Management](#session-management)
13. [Password Reset Flow](#password-reset-flow)
14. [Account Linking](#account-linking)
15. [Username Selection Rules](#username-selection-rules)
16. [Security Measures](#security-measures)
17. [Error Handling](#error-handling)

---

## Overview

Octo Time uses a dual-token JWT authentication system with rotating refresh tokens. The system supports:

- **Native authentication** (email + password)
- **Google OAuth 2.0** (SSO)
- **Account linking** (connect a Google account to an existing native account)
- **Concurrent sessions** across multiple devices (web, iOS, Android)
- **Sliding-window refresh tokens** with rotation on each use

All tokens are stateless JWTs with server-side invalidation via Redis for logout and token rotation tracking.

---

## Token Strategy

### Access Token

| Property | Value |
|----------|-------|
| TTL | 15 minutes |
| Algorithm | RS256 (RSA 2048-bit) |
| Transport | Authorization header (`Bearer <token>`) on mobile; httpOnly cookie on web |
| Stored server-side | No (stateless). Revocation via Redis blacklist only for edge cases. |

### Refresh Token

| Property | Value |
|----------|-------|
| TTL | 30 days (sliding window — reset on each use) |
| Algorithm | RS256 |
| Transport | httpOnly, Secure, SameSite=Strict cookie (web); SecureStore (mobile) |
| Stored server-side | Yes — a hash of each refresh token is stored in `user_sessions` and Redis |
| Rotation | Yes — on every refresh, old token is invalidated and a new one is issued |
| Family tracking | Yes — token family ID allows detection of refresh token theft |

### Sliding Window Behavior

Each time the refresh token is used, its expiry is pushed forward by 30 days from the time of use. A session that is actively used never expires. An idle session (no refresh calls) expires after 30 days.

---

## Token Storage

### Web (Browser)

| Token | Storage | Cookie Attributes |
|-------|---------|-------------------|
| Access Token | httpOnly cookie | `HttpOnly; Secure; SameSite=Strict; Path=/; Max-Age=900` |
| Refresh Token | httpOnly cookie | `HttpOnly; Secure; SameSite=Strict; Path=/api/auth/refresh; Max-Age=2592000` |

The refresh token cookie is scoped to `Path=/api/auth/refresh` so it is only sent on refresh requests, reducing exposure.

The web client does not have direct JavaScript access to either token. CSRF protection is handled via the `SameSite=Strict` attribute. For cross-origin API calls, a CSRF token (stored in a non-httpOnly cookie) is double-submitted.

### Mobile (iOS + Android)

| Token | Storage |
|-------|---------|
| Access Token | iOS: `Keychain` via `expo-secure-store`; Android: `EncryptedSharedPreferences` via `expo-secure-store` |
| Refresh Token | iOS: `Keychain` (kSecAttrAccessibleAfterFirstUnlock); Android: `EncryptedSharedPreferences` |

On mobile, tokens are stored via `SecureStore` (Expo) which maps to the OS-level secure storage. Tokens are sent in the `Authorization: Bearer <token>` header for access tokens and via a dedicated refresh endpoint body/header for refresh tokens.

---

## JWT Payload Structure

### Access Token Claims

```json
{
  "iss": "https://api.octotime.app",
  "aud": "octotime-client",
  "sub": "usr_01H9QZXMN8K3P7Y2V6W4R5T0",
  "iat": 1718352000,
  "exp": 1718352900,
  "jti": "acc_01H9QZXMN8K3P7Y2V6W4R5T0_1718352000",
  "type": "access",
  "email": "user@example.com",
  "username": "animefan42",
  "display_name": "Anime Fan",
  "avatar_url": "https://cdn.octotime.app/avatars/usr_01H9QZXMN8K3P7Y2V6W4R5T0.webp",
  "role": "user",
  "email_verified": true,
  "session_id": "sess_01H9QZXMN8K3P7Y2V6W4R5T0",
  "device_id": "dev_01H9QZXMN8K3P7Y2V6W4R5T0"
}
```

### Refresh Token Claims

```json
{
  "iss": "https://api.octotime.app",
  "aud": "octotime-client",
  "sub": "usr_01H9QZXMN8K3P7Y2V6W4R5T0",
  "iat": 1718352000,
  "exp": 1720944000,
  "jti": "ref_01H9QZXMN8K3P7Y2V6W4R5T0_1718352000",
  "type": "refresh",
  "session_id": "sess_01H9QZXMN8K3P7Y2V6W4R5T0",
  "family_id": "fam_01H9QZXMN8K3P7Y2V6W4R5T0",
  "generation": 1
}
```

**Field Descriptions:**

- `jti` — Unique token ID. Used to track individual token instances in Redis.
- `session_id` — Links the token to a specific device session row in `user_sessions`.
- `family_id` — Groups all refresh tokens generated for one device session. If a reuse of a revoked token is detected within the same family, the entire family is invalidated (all sessions for that user on that device).
- `generation` — Increments on each rotation. Allows detection of concurrent refresh races.

---

## Database Tables

### `users`

```sql
CREATE TABLE users (
  id                  TEXT PRIMARY KEY,           -- ULID: usr_01H...
  email               TEXT UNIQUE NOT NULL,
  email_verified      BOOLEAN NOT NULL DEFAULT false,
  username            TEXT UNIQUE NOT NULL,
  display_name        TEXT,
  avatar_url          TEXT,
  password_hash       TEXT,                       -- NULL for OAuth-only accounts
  role                TEXT NOT NULL DEFAULT 'user', -- 'user' | 'moderator' | 'admin'
  is_active           BOOLEAN NOT NULL DEFAULT true,
  google_id           TEXT UNIQUE,                -- NULL for native-only accounts
  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email    ON users(email);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_google_id ON users(google_id) WHERE google_id IS NOT NULL;
```

### `user_sessions`

```sql
CREATE TABLE user_sessions (
  id                  TEXT PRIMARY KEY,           -- ULID: sess_01H...
  user_id             TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  family_id           TEXT NOT NULL,              -- ULID: fam_01H...
  refresh_token_hash  TEXT NOT NULL,              -- SHA-256 of refresh JWT
  generation          INT NOT NULL DEFAULT 1,
  device_name         TEXT,                       -- "iPhone 15 Pro", "Chrome on Windows"
  device_type         TEXT,                       -- 'web' | 'ios' | 'android'
  ip_address          INET,
  user_agent          TEXT,
  last_used_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at          TIMESTAMPTZ NOT NULL,
  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  is_revoked          BOOLEAN NOT NULL DEFAULT false
);

CREATE INDEX idx_sessions_user_id          ON user_sessions(user_id);
CREATE INDEX idx_sessions_refresh_hash     ON user_sessions(refresh_token_hash);
CREATE INDEX idx_sessions_family_id        ON user_sessions(family_id);
CREATE INDEX idx_sessions_expires_at       ON user_sessions(expires_at);
```

### `email_verifications`

```sql
CREATE TABLE email_verifications (
  id          TEXT PRIMARY KEY,
  user_id     TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token       TEXT NOT NULL UNIQUE,               -- 6-digit OTP or signed URL token
  expires_at  TIMESTAMPTZ NOT NULL,
  used_at     TIMESTAMPTZ,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### `password_reset_tokens`

```sql
CREATE TABLE password_reset_tokens (
  id          TEXT PRIMARY KEY,
  user_id     TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token_hash  TEXT NOT NULL UNIQUE,               -- SHA-256 of the OTP
  expires_at  TIMESTAMPTZ NOT NULL,               -- 15 minutes from creation
  used_at     TIMESTAMPTZ,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_prt_user_id    ON password_reset_tokens(user_id);
CREATE INDEX idx_prt_token_hash ON password_reset_tokens(token_hash);
```

### `login_attempts`

```sql
CREATE TABLE login_attempts (
  id          TEXT PRIMARY KEY,
  identifier  TEXT NOT NULL,    -- email or username
  ip_address  INET NOT NULL,
  success     BOOLEAN NOT NULL,
  attempted_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_la_identifier   ON login_attempts(identifier, attempted_at DESC);
CREATE INDEX idx_la_ip           ON login_attempts(ip_address, attempted_at DESC);
```

---

## Redis Keys

| Key Pattern | Type | TTL | Purpose |
|-------------|------|-----|---------|
| `auth:blacklist:{jti}` | STRING | Remaining access token TTL | Blacklisted access token JTIs |
| `auth:session:{session_id}` | HASH | 30 days | Active session metadata (user_id, device_type, last_used) |
| `auth:refresh_family:{family_id}` | SET | 30 days | All session IDs in a refresh token family |
| `auth:rate:login:{ip}` | STRING | 15 minutes | Failed login count by IP |
| `auth:rate:login:user:{email}` | STRING | 15 minutes | Failed login count by email/username |
| `auth:rate:refresh:{ip}` | STRING | 1 minute | Refresh requests per IP |
| `auth:otp:verify:{user_id}` | STRING | 24 hours | Email verification OTP |
| `auth:otp:reset:{email}` | STRING | 15 minutes | Password reset OTP + user_id |
| `auth:otp:reset_attempts:{email}` | STRING | 15 minutes | Reset OTP attempt count |
| `auth:suspicious:{user_id}` | STRING | 24 hours | Suspicious login flag |
| `auth:google:state:{state}` | STRING | 10 minutes | OAuth state parameter (CSRF) |
| `auth:account_link:{token}` | HASH | 1 hour | Pending account link data |

---

## Google OAuth 2.0 Flow

### Overview

Octo Time uses the Authorization Code flow with PKCE for web and the native OAuth flow (via Google Sign-In SDK) for mobile.

### Web Flow — Detailed Steps

```
Client (Browser)                    Octo API                     Google OAuth
      |                                 |                               |
      |  GET /api/auth/google/init      |                               |
      |-------------------------------->|                               |
      |                                 | Generate state (random 32B)   |
      |                                 | Generate code_verifier        |
      |                                 | code_challenge = S256(verif.) |
      |                                 | Store state in Redis (10 min) |
      |  302 Redirect to Google         |                               |
      |<--------------------------------|                               |
      |                                 |                               |
      |  GET accounts.google.com/o/oauth2/v2/auth                      |
      |  ?client_id=...                                                 |
      |  &redirect_uri=https://api.octotime.app/api/auth/google/callback|
      |  &response_type=code                                            |
      |  &scope=openid email profile                                    |
      |  &state={random_state}                                          |
      |  &code_challenge={S256_challenge}                               |
      |  &code_challenge_method=S256                                    |
      |---------------------------------------------------------------->|
      |                                 |                               |
      |         User logs in and grants permission                      |
      |                                 |                               |
      |  302 Redirect back                                              |
      |  GET /api/auth/google/callback                                  |
      |  ?code={auth_code}&state={state}                                |
      |<----------------------------------------------------------------|
      |                                 |                               |
      |  Browser follows redirect       |                               |
      |  GET /api/auth/google/callback  |                               |
      |  ?code={auth_code}&state={state}|                               |
      |-------------------------------->|                               |
      |                                 | 1. Validate state vs Redis    |
      |                                 | 2. Delete state from Redis    |
      |                                 |                               |
      |                                 |  POST /token                  |
      |                                 |  code={auth_code}             |
      |                                 |  code_verifier={verifier}     |
      |                                 |------------------------------>|
      |                                 |                               |
      |                                 |  {access_token, id_token}     |
      |                                 |<------------------------------|
      |                                 |                               |
      |                                 | 3. Verify id_token signature  |
      |                                 | 4. Extract: google_id, email, |
      |                                 |    name, picture              |
      |                                 |                               |
      |                                 | 5. Lookup user by google_id   |
      |                                 |    OR email                   |
      |                                 |                               |
      |                                 | CASE A: google_id match found |
      |                                 | → user already exists, log in |
      |                                 |                               |
      |                                 | CASE B: email match found     |
      |                                 |   but google_id not linked    |
      |                                 | → link google_id to account   |
      |                                 |   (auto-link if email verified)|
      |                                 |                               |
      |                                 | CASE C: no match found        |
      |                                 | → create new user             |
      |                                 |   generate username from name |
      |                                 |   set email_verified = true   |
      |                                 |   (Google emails are verified)|
      |                                 |                               |
      |                                 | 6. Create user_sessions row   |
      |                                 | 7. Issue access + refresh JWT |
      |                                 | 8. Set httpOnly cookies       |
      |  200 OK + Set-Cookie            |                               |
      |<--------------------------------|                               |
      |                                 |                               |
```

### Mobile Flow (iOS / Android)

Mobile apps use the Google Sign-In SDK which performs the OAuth flow natively and returns an `id_token`. The app sends this token to Octo's API for verification.

```
Mobile App                          Octo API                     Google OAuth
      |                                 |                               |
      | Tap "Sign in with Google"       |                               |
      | Google SDK opens browser/sheet  |                               |
      |---------------------------------------------------------------->|
      |                                 |                               |
      |  User authenticates             |                               |
      |  SDK returns id_token + access_token                            |
      |<----------------------------------------------------------------|
      |                                 |                               |
      |  POST /api/auth/google/mobile   |                               |
      |  { id_token: "..." }            |                               |
      |-------------------------------->|                               |
      |                                 | 1. Verify id_token with       |
      |                                 |    Google public keys (JWKS)  |
      |                                 | 2. Check aud = our client_id  |
      |                                 | 3. Check exp not passed       |
      |                                 | 4. Extract user info          |
      |                                 | 5-8. Same as web flow above   |
      |                                 |                               |
      |  200 { access_token, refresh_token }                            |
      |<--------------------------------|                               |
      |                                 |                               |
      | Store in SecureStore            |                               |
```

### New User Creation via Google

When no existing user is found (CASE C), the following occurs:

1. A `username` is auto-generated from the Google display name:
   - Strip non-alphanumeric characters
   - Truncate to 18 characters
   - Append 2 random digits if the username is already taken
   - The user is prompted to change it after first login
2. `email_verified` is set to `true` (Google guarantees verified emails)
3. `password_hash` is `NULL` — the account is OAuth-only unless the user sets a password later
4. `google_id` is stored
5. Avatar is fetched from Google's `picture` URL and re-hosted on Octo CDN

---

## Native Registration Flow

```
Client                              Octo API                     Database / Redis
  |                                    |                               |
  |  POST /api/auth/register           |                               |
  |  {                                 |                               |
  |    email: "user@example.com",      |                               |
  |    password: "Hunter2!secure",     |                               |
  |    username: "animefan42"          |                               |
  |  }                                 |                               |
  |----------------------------------->|                               |
  |                                    | 1. Validate request body      |
  |                                    |    (Zod schema)               |
  |                                    |                               |
  |                                    | 2. Validate email format      |
  |                                    |    Check disposable domains   |
  |                                    |                               |
  |                                    | 3. Validate password strength |
  |                                    |    min 8 chars, 1 upper,      |
  |                                    |    1 digit, 1 special         |
  |                                    |                               |
  |                                    | 4. Validate username rules    |
  |                                    |    (see Username Rules section)|
  |                                    |                               |
  |                                    | 5. Check email uniqueness     |
  |                                    |----------------------------->>|
  |                                    |    SELECT * FROM users        |
  |                                    |    WHERE email = $1           |
  |                                    |<<-----------------------------|
  |                                    |                               |
  |                                    | 6. Check username uniqueness  |
  |                                    |----------------------------->>|
  |                                    |    SELECT * FROM users        |
  |                                    |    WHERE username = $1        |
  |                                    |<<-----------------------------|
  |                                    |                               |
  |                                    | 7. Hash password              |
  |                                    |    bcrypt(password, rounds=12)|
  |                                    |                               |
  |                                    | 8. Create user (BEGIN txn)    |
  |                                    |----------------------------->>|
  |                                    |    INSERT INTO users (...)    |
  |                                    |<<-----------------------------|
  |                                    |                               |
  |                                    | 9. Generate email OTP         |
  |                                    |    6-digit numeric code       |
  |                                    |----------------------------->>|
  |                                    |    INSERT INTO                |
  |                                    |    email_verifications (...)  |
  |                                    |    SET auth:otp:verify:{uid}  |
  |                                    |    EX 86400                   |
  |                                    |<<-----------------------------|
  |                                    |                               |
  |                                    | 10. COMMIT txn                |
  |                                    |                               |
  |                                    | 11. Send verification email   |
  |                                    |     (async via queue)         |
  |                                    |                               |
  |  201 { user_id, email,             |                               |
  |         message: "Check email" }   |                               |
  |<-----------------------------------|                               |
  |                                    |                               |
  |  --- User receives OTP email ---   |                               |
  |                                    |                               |
  |  POST /api/auth/verify-email       |                               |
  |  { user_id, otp: "482931" }        |                               |
  |----------------------------------->|                               |
  |                                    | 1. GET auth:otp:verify:{uid} |
  |                                    |    (from Redis)               |
  |                                    | 2. Compare OTP (constant-time)|
  |                                    | 3. UPDATE users               |
  |                                    |    SET email_verified = true  |
  |                                    | 4. DELETE OTP from Redis      |
  |                                    | 5. Mark verification used     |
  |                                    |                               |
  |                                    | 6. Issue access + refresh JWT |
  |                                    | 7. Create user_sessions row   |
  |                                    | 8. Set cookies / return tokens|
  |                                    |                               |
  |  200 { access_token, user }        |                               |
  |<-----------------------------------|                               |
```

### Password Hashing

- Algorithm: **bcrypt** with cost factor **12**
- On login, if `rounds < 12` (legacy), rehash and update silently
- Never store plaintext; never log passwords

### Email Validation

1. Format validation via regex (RFC 5322 simplified)
2. MX record check: the domain must have valid MX records
3. Disposable email domain blocklist (maintained list of ~50,000 known throwaway domains)
4. Normalization: lowercase the entire address; for Gmail accounts, strip dots and `+` aliases before uniqueness check

---

## Native Login Flow

```
Client                              Octo API                     Database / Redis
  |                                    |                               |
  |  POST /api/auth/login              |                               |
  |  {                                 |                               |
  |    identifier: "animefan42",       | (email or username accepted)  |
  |    password: "Hunter2!secure",     |                               |
  |    device_name: "Chrome on Mac",   |                               |
  |    device_type: "web"              |                               |
  |  }                                 |                               |
  |----------------------------------->|                               |
  |                                    | 1. Rate limit check (IP)      |
  |                                    |    GET auth:rate:login:{ip}   |
  |                                    |    If >= 10 in 15min → 429    |
  |                                    |                               |
  |                                    | 2. Rate limit check (user)    |
  |                                    |    GET auth:rate:login:user   |
  |                                    |    :{identifier}              |
  |                                    |    If >= 5 in 15min → 429     |
  |                                    |    (account lockout)          |
  |                                    |                               |
  |                                    | 3. Lookup user                |
  |                                    |    by email OR username       |
  |                                    |    (case-insensitive)         |
  |                                    |                               |
  |                                    | 4. Check is_active = true     |
  |                                    |                               |
  |                                    | 5. Verify password            |
  |                                    |    bcrypt.compare(input, hash)|
  |                                    |    [constant-time comparison] |
  |                                    |                               |
  |                                    | IF password invalid:          |
  |                                    |   INCR auth:rate:login:{ip}   |
  |                                    |   INCR auth:rate:login:user:  |
  |                                    |   {identifier}                |
  |                                    |   INSERT login_attempts       |
  |                                    |     (success=false)           |
  |                                    |   Return 401 (generic msg)    |
  |                                    |                               |
  |                                    | IF password valid:            |
  |                                    | 6. Check email_verified       |
  |                                    |    If false → 403 with code   |
  |                                    |    "EMAIL_NOT_VERIFIED"       |
  |                                    |                               |
  |                                    | 7. Suspicious login detection |
  |                                    |    Check IP geo vs history    |
  |                                    |    If suspicious:             |
  |                                    |      Set suspicious flag      |
  |                                    |      Send alert email         |
  |                                    |                               |
  |                                    | 8. Create session             |
  |                                    |    INSERT user_sessions (...)  |
  |                                    |    SET auth:session:{sess_id} |
  |                                    |                               |
  |                                    | 9. Issue tokens               |
  |                                    |    Create access JWT (15 min) |
  |                                    |    Create refresh JWT (30 day)|
  |                                    |    Store SHA256(refresh) in   |
  |                                    |    user_sessions              |
  |                                    |                               |
  |                                    | 10. INSERT login_attempts     |
  |                                    |     (success=true)            |
  |                                    |                               |
  |  200 { access_token, user }        |                               |
  |  + Set-Cookie: access_token=...    |                               |
  |  + Set-Cookie: refresh_token=...   |                               |
  |<-----------------------------------|                               |
```

---

## Token Refresh Flow

```
Client                              Octo API                     Redis / Database
  |                                    |                               |
  |  POST /api/auth/refresh            |                               |
  |  Cookie: refresh_token=<JWT>       |                               |
  |  (mobile: body or header)          |                               |
  |----------------------------------->|                               |
  |                                    | 1. Extract refresh token      |
  |                                    |    from cookie (web) or       |
  |                                    |    Authorization header (mob) |
  |                                    |                               |
  |                                    | 2. Verify JWT signature       |
  |                                    |    and exp claim              |
  |                                    |                               |
  |                                    | 3. Rate limit check           |
  |                                    |    GET auth:rate:refresh:{ip} |
  |                                    |    Max 10 per minute per IP   |
  |                                    |                               |
  |                                    | 4. Extract: session_id,       |
  |                                    |    family_id, jti, generation |
  |                                    |                               |
  |                                    | 5. Lookup session in DB       |
  |                                    |    SELECT * FROM user_sessions|
  |                                    |    WHERE id = session_id      |
  |                                    |    AND is_revoked = false     |
  |                                    |                               |
  |                                    | 6. Verify token hash          |
  |                                    |    SHA256(token) == stored    |
  |                                    |                               |
  |                                    | IF token hash mismatch        |
  |                                    | (reuse of old rotated token): |
  |                                    |   THEFT DETECTED              |
  |                                    |   Revoke all sessions in      |
  |                                    |   family_id                   |
  |                                    |   Alert user via email        |
  |                                    |   Return 401                  |
  |                                    |                               |
  |                                    | 7. Verify generation matches  |
  |                                    |                               |
  |                                    | 8. Generate new token pair    |
  |                                    |    new generation = old + 1   |
  |                                    |                               |
  |                                    | 9. UPDATE user_sessions       |
  |                                    |    SET refresh_token_hash =   |
  |                                    |        SHA256(new_refresh)    |
  |                                    |    SET generation = gen + 1   |
  |                                    |    SET last_used_at = NOW()   |
  |                                    |    SET expires_at = NOW()+30d |
  |                                    |                               |
  |                                    | 10. Blacklist old refresh jti |
  |                                    |     SET auth:blacklist:{jti}  |
  |                                    |     EX remaining_ttl          |
  |                                    |                               |
  |                                    | 11. Update Redis session cache|
  |                                    |     SET auth:session:{sess}   |
  |                                    |                               |
  |  200 { access_token }              |                               |
  |  + Set-Cookie: access_token=...    |                               |
  |  + Set-Cookie: refresh_token=...   |                               |
  |<-----------------------------------|                               |
```

### Race Condition Handling

If two refresh requests arrive simultaneously for the same session, a database row-level lock prevents duplicate rotation:

```sql
SELECT * FROM user_sessions
WHERE id = $1 AND is_revoked = false
FOR UPDATE SKIP LOCKED;
```

The second request will see `SKIP LOCKED` and return a 409 Conflict, telling the client to retry.

---

## Logout Flow

### Single Device Logout

```
Client                              Octo API                     Redis / Database
  |                                    |                               |
  |  POST /api/auth/logout             |                               |
  |  Cookie: access_token=<JWT>        |                               |
  |  Cookie: refresh_token=<JWT>       |                               |
  |----------------------------------->|                               |
  |                                    | 1. Extract session_id from    |
  |                                    |    access token claims        |
  |                                    |                               |
  |                                    | 2. Revoke session in DB       |
  |                                    |    UPDATE user_sessions       |
  |                                    |    SET is_revoked = true      |
  |                                    |    WHERE id = session_id      |
  |                                    |                               |
  |                                    | 3. Delete from Redis          |
  |                                    |    DEL auth:session:{sess_id} |
  |                                    |                               |
  |                                    | 4. Blacklist access token jti |
  |                                    |    SET auth:blacklist:{jti}   |
  |                                    |    EX remaining_access_ttl    |
  |                                    |                               |
  |                                    | 5. Clear cookies (Set-Cookie  |
  |                                    |    with Max-Age=0)            |
  |                                    |                               |
  |  200 { message: "Logged out" }     |                               |
  |<-----------------------------------|                               |
```

### Logout All Devices

```
POST /api/auth/logout-all

1. Authenticate user via access token
2. UPDATE user_sessions SET is_revoked = true WHERE user_id = $1
3. Delete all Redis session keys for user:
   SCAN for keys auth:session:* where user_id matches
   (or maintain a SET per user: auth:user_sessions:{user_id})
4. Blacklist current access token jti
5. Clear cookies
6. Return 200
```

The Redis key `auth:user_sessions:{user_id}` is a SET containing all active session IDs for a user. It is maintained on session creation and deletion.

---

## Session Management

### Concurrent Sessions

Octo Time allows unlimited concurrent sessions by default. Each device login creates a new entry in `user_sessions`.

### Session Listing API

```
GET /api/auth/sessions

Response:
{
  "sessions": [
    {
      "id": "sess_01H...",
      "device_name": "Chrome on macOS",
      "device_type": "web",
      "ip_address": "203.0.113.42",
      "location": "Cairo, Egypt",
      "last_used_at": "2024-06-14T10:00:00Z",
      "created_at": "2024-06-01T09:00:00Z",
      "is_current": true
    },
    {
      "id": "sess_01H...",
      "device_name": "iPhone 15 Pro",
      "device_type": "ios",
      "ip_address": "203.0.113.99",
      "location": "Alexandria, Egypt",
      "last_used_at": "2024-06-13T22:00:00Z",
      "created_at": "2024-06-10T14:00:00Z",
      "is_current": false
    }
  ]
}
```

### Revoke Specific Session

```
DELETE /api/auth/sessions/:session_id

1. Verify the session belongs to the authenticated user
2. UPDATE user_sessions SET is_revoked = true WHERE id = $1
3. DEL auth:session:{session_id} from Redis
4. Return 200
```

### Device Tracking

On each successful login, the following device metadata is stored:

- `User-Agent` header → parsed for browser/OS/device type
- IP address → resolved to city/country via MaxMind GeoIP database (local, no external call)
- Device name: provided by mobile apps via request body; inferred from User-Agent for web

---

## Password Reset Flow

```
Client                              Octo API                     Redis / Email
  |                                    |                               |
  |  POST /api/auth/password-reset/    |                               |
  |       request                      |                               |
  |  { email: "user@example.com" }     |                               |
  |----------------------------------->|                               |
  |                                    | 1. Rate limit: max 3 requests |
  |                                    |    per email per 15 minutes   |
  |                                    |    GET auth:otp:reset_attempts|
  |                                    |    :{email}                   |
  |                                    |                               |
  |                                    | 2. Lookup user by email       |
  |                                    |    (always return 200 to      |
  |                                    |    prevent email enumeration) |
  |                                    |                               |
  |                                    | IF user found:                |
  |                                    | 3. Generate 6-digit OTP       |
  |                                    |    (cryptographically random) |
  |                                    |                               |
  |                                    | 4. Hash OTP: SHA256(otp)      |
  |                                    |                               |
  |                                    | 5. Invalidate previous reset  |
  |                                    |    tokens for this user:      |
  |                                    |    UPDATE password_reset_tokens|
  |                                    |    SET used_at = NOW()        |
  |                                    |    WHERE user_id = $1         |
  |                                    |    AND used_at IS NULL        |
  |                                    |                               |
  |                                    | 6. Store new OTP:             |
  |                                    |    SET auth:otp:reset:{email} |
  |                                    |    "{otp_hash}:{user_id}"     |
  |                                    |    EX 900 (15 minutes)        |
  |                                    |                               |
  |                                    | 7. INSERT password_reset_     |
  |                                    |    tokens(user_id, token_hash,|
  |                                    |    expires_at = NOW()+15min)  |
  |                                    |                               |
  |                                    | 8. Queue email delivery       |
  |                                    |    (async, via email queue)   |
  |                                    |                               |
  |  200 { message: "If email exists,  |                               |
  |         check inbox" }             |                               |
  |<-----------------------------------|                               |
  |                                    |                               |
  |  --- User receives OTP email ---   |                               |
  |                                    |                               |
  |  POST /api/auth/password-reset/    |                               |
  |       verify                       |                               |
  |  { email, otp: "829401",           |                               |
  |    new_password: "NewP@ss9!" }     |                               |
  |----------------------------------->|                               |
  |                                    | 1. Rate limit: max 5 attempts |
  |                                    |    per email in 15 minutes    |
  |                                    |    (prevent OTP brute force)  |
  |                                    |                               |
  |                                    | 2. GET auth:otp:reset:{email} |
  |                                    |    from Redis                 |
  |                                    |    If missing → 400           |
  |                                    |    "OTP expired or invalid"   |
  |                                    |                               |
  |                                    | 3. Verify: SHA256(input_otp)  |
  |                                    |    == stored hash             |
  |                                    |    [constant-time compare]    |
  |                                    |    If mismatch → 400 + INCR   |
  |                                    |    attempts counter           |
  |                                    |                               |
  |                                    | 4. Validate new password      |
  |                                    |    strength                   |
  |                                    |                               |
  |                                    | 5. Hash new password          |
  |                                    |    bcrypt(new_password, 12)   |
  |                                    |                               |
  |                                    | 6. BEGIN transaction:         |
  |                                    |    UPDATE users               |
  |                                    |    SET password_hash = $1     |
  |                                    |    WHERE id = user_id         |
  |                                    |                               |
  |                                    | 7. Revoke all existing        |
  |                                    |    sessions:                  |
  |                                    |    UPDATE user_sessions       |
  |                                    |    SET is_revoked = true      |
  |                                    |    WHERE user_id = $1         |
  |                                    |                               |
  |                                    | 8. Mark OTP as used           |
  |                                    |    UPDATE password_reset_tokens|
  |                                    |    SET used_at = NOW()        |
  |                                    |                               |
  |                                    | 9. COMMIT transaction         |
  |                                    |                               |
  |                                    | 10. DEL auth:otp:reset:{email}|
  |                                    |     from Redis                |
  |                                    |                               |
  |                                    | 11. Send confirmation email   |
  |                                    |                               |
  |                                    | 12. Issue new session tokens  |
  |                                    |     (auto-login after reset)  |
  |                                    |                               |
  |  200 { access_token, user }        |                               |
  |<-----------------------------------|                               |
```

---

## Account Linking

Account linking allows a user who registered natively (email + password) to connect their Google account, and vice versa.

### Link Google to Existing Native Account

```
Client (authenticated)              Octo API                     Google
  |                                    |                               |
  |  GET /api/auth/google/link         |                               |
  |  Authorization: Bearer <token>     |                               |
  |----------------------------------->|                               |
  |                                    | 1. Verify access token        |
  |                                    | 2. Check user has no          |
  |                                    |    google_id yet              |
  |                                    | 3. Generate link_token (UUID) |
  |                                    |    SET auth:account_link:     |
  |                                    |    {token}                    |
  |                                    |    { user_id, action: "link"} |
  |                                    |    EX 3600                    |
  |                                    | 4. Build Google OAuth URL     |
  |                                    |    with link_token in state   |
  |  302 Redirect to Google            |                               |
  |<-----------------------------------|                               |
  |                                    |                               |
  |  [User authenticates with Google]  |                               |
  |                                    |                               |
  |  GET /api/auth/google/callback     |                               |
  |  ?code=...&state={link_token}      |                               |
  |----------------------------------->|                               |
  |                                    | 1. Verify state, get link_token|
  |                                    | 2. GET auth:account_link:     |
  |                                    |    {link_token} → user_id     |
  |                                    | 3. Exchange code → id_token   |
  |                                    | 4. Extract google_id          |
  |                                    | 5. Check google_id not        |
  |                                    |    already in use by another  |
  |                                    |    account                    |
  |                                    | 6. UPDATE users               |
  |                                    |    SET google_id = $1         |
  |                                    |    WHERE id = user_id         |
  |                                    | 7. DEL auth:account_link:     |
  |                                    |    {link_token}               |
  |                                    | 8. Send confirmation email    |
  |  200 { message: "Google linked" }  |                               |
  |<-----------------------------------|                               |
```

### Unlink Google

```
POST /api/auth/google/unlink

Requirements before unlinking:
1. User must have a password_hash set (can't leave account with no login method)
2. Clear google_id on the users row
3. Send email notification
```

---

## Username Selection Rules

### Rules

| Rule | Constraint |
|------|-----------|
| Length | 3–20 characters (inclusive) |
| Allowed characters | Letters (a-z, A-Z), digits (0-9), underscores (`_`) |
| Case | Stored as-is; uniqueness enforced case-insensitively |
| Start character | Must start with a letter |
| Consecutive underscores | Not allowed (`__` is blocked) |
| End character | Must not end with an underscore |
| Reserved words | Blocked (see list below) |
| Change frequency | 1 change per 30 days (tracked via `username_updated_at`) |

### Reserved Words Blocklist

The following usernames (and any that contain them as a complete word) are blocked:

```
admin, administrator, moderator, mod, staff, octotime, octo, system,
api, auth, login, logout, register, signup, me, self, null, undefined,
true, false, root, superuser, webmaster, help, support, abuse,
contact, info, security, privacy, legal, terms, about, blog,
explore, search, trending, notifications, settings, profile, user,
users, list, lists, collection, review, reviews, watchlist, anime,
movies, shows, home, index, dashboard, discover
```

Usernames are matched after lowercasing. A username like `OctoTime` is blocked because lowercased it equals `octotime`.

### Validation Implementation

```typescript
const USERNAME_REGEX = /^[a-zA-Z][a-zA-Z0-9_]{1,18}[a-zA-Z0-9]$|^[a-zA-Z]{3}$/;
const CONSECUTIVE_UNDERSCORES = /__/;
const RESERVED_WORDS = new Set([/* ... list above ... */]);

function validateUsername(username: string): ValidationResult {
  if (username.length < 3 || username.length > 20) {
    return { valid: false, error: "USERNAME_LENGTH" };
  }
  if (!/^[a-zA-Z]/.test(username)) {
    return { valid: false, error: "USERNAME_MUST_START_WITH_LETTER" };
  }
  if (!/^[a-zA-Z0-9_]+$/.test(username)) {
    return { valid: false, error: "USERNAME_INVALID_CHARS" };
  }
  if (username.endsWith("_")) {
    return { valid: false, error: "USERNAME_CANNOT_END_WITH_UNDERSCORE" };
  }
  if (CONSECUTIVE_UNDERSCORES.test(username)) {
    return { valid: false, error: "USERNAME_CONSECUTIVE_UNDERSCORES" };
  }
  if (RESERVED_WORDS.has(username.toLowerCase())) {
    return { valid: false, error: "USERNAME_RESERVED" };
  }
  return { valid: true };
}
```

---

## Security Measures

### Brute Force Protection

**Per-IP Rate Limiting:**
- Login endpoint: max 10 attempts per 15 minutes per IP
- Penalty: 429 Too Many Requests with `Retry-After` header
- Implementation: Redis `INCR` + `EXPIRE`

**Per-Account Rate Limiting:**
- Login endpoint: max 5 failed attempts per account per 15 minutes
- After 5 failures: account is temporarily locked; returns 403 with `ACCOUNT_TEMPORARILY_LOCKED`
- Lock duration: 15 minutes (matches the rate limit window)
- After unlocking: a 6th failure resets the 15-minute window

**Password Reset Brute Force:**
- Max 3 OTP requests per email per 15 minutes
- Max 5 OTP submission attempts before the OTP is invalidated and must be re-requested

### Rate Limiting Architecture

All rate limiting uses Redis with sliding window counters implemented via a Lua script for atomicity:

```lua
local key = KEYS[1]
local window = tonumber(ARGV[1])
local limit = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

-- Remove entries outside the window
redis.call('ZREMRANGEBYSCORE', key, 0, now - window * 1000)

-- Count current entries
local count = redis.call('ZCARD', key)

if count >= limit then
  return 0  -- Rate limited
end

-- Add current request
redis.call('ZADD', key, now, now .. ':' .. math.random())
redis.call('EXPIRE', key, window)
return 1  -- Allowed
```

### Suspicious Login Detection

A login is flagged as suspicious when:

1. **New country/region**: IP resolves to a country not seen in the last 30 days of logins for that user
2. **New device fingerprint**: dramatically different User-Agent (e.g., always iOS, now Windows)
3. **Time of day anomaly**: login at an unusual hour (>3 standard deviations from historical pattern)
4. **Impossible travel**: two logins from geographically distant IPs within a short timeframe (e.g., Cairo then Tokyo within 2 hours)

**Response to suspicious login:**
- Login is still allowed (we don't block, to avoid lockouts)
- `auth:suspicious:{user_id}` Redis key is set for 24 hours
- An email alert is sent: "New sign-in from [location] at [time]. If this wasn't you, secure your account."
- The alert email contains a direct link to revoke all sessions

### HTTPS / TLS

- All endpoints served over HTTPS (TLS 1.2 minimum, TLS 1.3 preferred)
- HSTS header with `max-age=31536000; includeSubDomains; preload`
- Certificates via Let's Encrypt with auto-renewal

### CSRF Protection (Web)

- `SameSite=Strict` on all auth cookies prevents CSRF for same-site requests
- For APIs called from the web client: a CSRF token is set in a non-httpOnly cookie (`csrf_token`) and must be echoed in the `X-CSRF-Token` request header
- The API verifies the header value matches the cookie value (double-submit cookie pattern)

### Timing Attack Prevention

- Password comparison via `bcrypt.compare()` (inherently constant-time)
- OTP comparison via `crypto.timingSafeEqual()` on the hash values
- User lookup always runs regardless of whether email exists (to prevent timing-based email enumeration)

### Token Signing Keys

- RS256 private key (2048-bit RSA) for signing, stored in AWS KMS
- Public key exposed at `/.well-known/jwks.json` for verification
- Key rotation: annually, with a 7-day overlap window where both old and new keys are valid
- Each key has a `kid` (Key ID) claim in the JWT header for key selection

---

## Error Handling

### Error Response Format

```json
{
  "error": {
    "code": "INVALID_CREDENTIALS",
    "message": "Invalid email or password.",
    "details": null
  }
}
```

### Error Codes by Flow

**Registration:**

| Code | HTTP | Description |
|------|------|-------------|
| `EMAIL_ALREADY_EXISTS` | 409 | Email is taken |
| `USERNAME_ALREADY_EXISTS` | 409 | Username is taken |
| `EMAIL_INVALID` | 400 | Email format invalid or disposable |
| `PASSWORD_TOO_WEAK` | 400 | Password doesn't meet requirements |
| `USERNAME_INVALID` | 400 | Username fails validation rules |
| `USERNAME_RESERVED` | 400 | Username is a reserved word |

**Login:**

| Code | HTTP | Description |
|------|------|-------------|
| `INVALID_CREDENTIALS` | 401 | Email/password wrong (generic, no detail) |
| `EMAIL_NOT_VERIFIED` | 403 | Account email not verified yet |
| `ACCOUNT_DISABLED` | 403 | Account has been disabled by admin |
| `ACCOUNT_TEMPORARILY_LOCKED` | 429 | Too many failed attempts |
| `RATE_LIMITED` | 429 | IP rate limit exceeded |

**Token Refresh:**

| Code | HTTP | Description |
|------|------|-------------|
| `REFRESH_TOKEN_INVALID` | 401 | Token is malformed, expired, or signature invalid |
| `REFRESH_TOKEN_REUSED` | 401 | Token was already rotated (possible theft) |
| `SESSION_REVOKED` | 401 | Session was explicitly logged out |
| `SESSION_EXPIRED` | 401 | Session TTL exceeded (> 30 days idle) |

**Password Reset:**

| Code | HTTP | Description |
|------|------|-------------|
| `OTP_INVALID` | 400 | OTP does not match |
| `OTP_EXPIRED` | 400 | OTP has expired (> 15 minutes) |
| `OTP_MAX_ATTEMPTS` | 429 | Too many OTP attempts |
| `PASSWORD_TOO_WEAK` | 400 | New password doesn't meet requirements |
| `RESET_RATE_LIMITED` | 429 | Too many reset requests |

**Google OAuth:**

| Code | HTTP | Description |
|------|------|-------------|
| `OAUTH_STATE_INVALID` | 400 | State parameter mismatch (CSRF attempt) |
| `OAUTH_CODE_INVALID` | 400 | Authorization code is invalid/expired |
| `GOOGLE_ID_ALREADY_LINKED` | 409 | Google account already linked to another user |
| `ACCOUNT_HAS_NO_PASSWORD` | 400 | Cannot unlink Google without a password set |

All 401 responses clear the auth cookies (via `Set-Cookie: ...; Max-Age=0`) to prevent the client from getting stuck with an invalid session.
