# Auth API — React Native integration guide

This document describes Quantt **email/password auth** and related endpoints from a **React Native** perspective: signup, login (including MFA), refresh tokens, forgot/reset password, email verification, and logout.

For **Google OAuth** on mobile, see [google-oauth-react-native.md](./google-oauth-react-native.md).

---

## API base URL

Use the Fastify API origin (no trailing slash):

| Environment | Example |
|---------------|---------|
| Local | `http://localhost:4000` |
| Production | `https://api.yourdomain.com` |

All paths below are relative to this base.

---

## How authentication works on the API

### Access token (JWT)

- Sent on protected routes as: **`Authorization: Bearer <accessToken>`**
- Alternatively, the web app uses the **`quantts_access`** cookie; React Native should prefer the **Bearer** header.
- Payload includes at least: `id`, `email`, `role`, `name`.
- **Lifetime:** about **15 minutes** (see `buildSession` in `apps/api/src/services/auth-service.ts`).

### Refresh token (JWT)

- A separate JWT with `type: "refresh"` in the payload, signed with the server’s refresh secret.
- **Lifetime:** about **30 days** (new session row in the database per successful login/refresh).
- **Rotation:** each successful `POST /v1/auth/refresh` **revokes** the refresh token you sent and returns **new** `accessToken` + `refreshToken`. Always persist the **new** refresh token and discard the old one.

### Cookies (optional for RN)

Login, refresh, TOTP challenge, and Google web callback may also set **`quantts_access`** / **`quantts_refresh`** cookies for the browser. **Native apps should rely on JSON bodies + secure storage**, not cookies, unless you intentionally share cookie domains with a WebView.

---

## Standard JSON session shape

Successful **login**, **refresh**, **TOTP challenge completion**, and **Google token exchange** return the same core shape:

```json
{
  "accessToken": "<jwt>",
  "refreshToken": "<jwt>",
  "user": {
    "id": "<uuid>",
    "email": "user@example.com",
    "role": "admin | trader | viewer",
    "name": "Display Name"
  }
}
```

Store `accessToken` and `refreshToken` in **secure storage** (Keychain / Keystore). Use Bearer for API calls; on **401**, try refresh once, then force re-login.

---

## 1. Sign up

### `POST /v1/auth/register`

**Rate limit:** 10 requests / minute.

**Headers:** `Content-Type: application/json`

**Body:**

| Field | Type | Constraints |
|-------|------|-------------|
| `name` | string | min length **2** |
| `email` | string | valid email |
| `password` | string | min length **10** |

**Success (`200`):**

```json
{
  "message": "Account created. Check your email to verify your account."
}
```

**Errors:**

| Status | Typical cause |
|--------|----------------|
| `409` | Email already registered (`Email already registered`). |

**Important:** New accounts must **verify email** before password login succeeds. The API sends an email with a link built from server **`APP_URL`** (web), e.g. `{APP_URL}/verify-email?token=...`. For React Native you can:

- Let users tap the link and complete verification in the **browser**, or  
- Use **app/universal links** that open the app and read `token`, then call **`POST /v1/auth/verify-email`** (below).

---

## 2. Verify email

### `POST /v1/auth/verify-email`

**Rate limit:** 20 / minute.

**Body:**

```json
{ "token": "<token from email link query string>" }
```

**Success (`200`):**

```json
{ "message": "Email verified successfully. You can now log in." }
```

**Errors:**

| Status | Meaning |
|--------|---------|
| `400` | Invalid or expired token |

### `POST /v1/auth/resend-verification`

**Rate limit:** 5 / hour.

**Body:**

```json
{ "email": "user@example.com" }
```

**Success (`200`):** Always a neutral message (no email enumeration):

```json
{
  "message": "If that email exists and is unverified, we sent a new verification link."
}
```

---

## 3. Login

### `POST /v1/auth/login`

**Rate limit:** 12 / minute.

**Body:**

```json
{
  "email": "user@example.com",
  "password": "min 8 chars"
}
```

**Password rules:** Zod requires **minimum 8** characters on this route (stricter password rules apply at **register** / **reset** — min **10** there).

### Success — full session (`200`)

Same JSON session as [Standard JSON session shape](#standard-json-session-shape). The server may also set auth cookies; **ignore cookies in RN** and use the JSON tokens.

### Success — MFA required (`200`)

If the user has TOTP enabled, the response is **not** a full session:

```json
{
  "mfaRequired": true,
  "mfaPendingToken": "<jwt>"
}
```

- **`mfaPendingToken`** is short-lived (about **5 minutes**). It is **not** a refresh token.
- Show the user a 6-digit (or backup) code field, then call **`POST /v1/auth/totp/challenge`** (below).

### Errors

| Status | Meaning |
|--------|---------|
| `401` | Wrong email/password (`Invalid credentials`) |
| `403` | Email not verified yet (message asks user to verify inbox) |

---

## 4. Complete login after MFA (`mfaRequired`)

### `POST /v1/auth/totp/challenge`

**Rate limit:** 10 / 5 minutes.

**Body:**

```json
{
  "mfaPendingToken": "<from login response>",
  "code": "123456 or backup code"
}
```

- `code`: **6–8** characters (TOTP or backup code).

**Success (`200`):** Full session JSON (`accessToken`, `refreshToken`, `user`). Cookies may also be set; use JSON in RN.

**Errors:**

| Status | Meaning |
|--------|---------|
| `401` | Invalid/expired MFA token, or invalid TOTP/backup code |

---

## 5. Refresh session

### `POST /v1/auth/refresh`

**Body (recommended for RN):**

```json
{ "refreshToken": "<stored refresh jwt>" }
```

If `refreshToken` is omitted, the server falls back to the **`quantts_refresh`** cookie (web). Native clients should **always send the body**.

**Success (`200`):** New `accessToken`, `refreshToken`, and `user` (same shape as login).

**Important:** The previous refresh token is **revoked** server-side. **Replace** stored refresh token with the new one.

**Errors:**

| Status | Meaning |
|--------|---------|
| `401` | Missing refresh token, invalid JWT, expired or revoked session |

---

## 6. Forgot password

### `POST /v1/auth/forgot-password`

**Rate limit:** 5 / hour.

**Body:**

```json
{ "email": "user@example.com" }
```

**Success (`200`):** Always the same message (do not use it to detect whether the email exists):

```json
{
  "message": "If that email is registered, we sent a password reset link."
}
```

**Email link:** Points to the **web** app: `{APP_URL}/reset-password?token=...` (see `apps/api/src/lib/email.ts`). Same options as verification: open in browser, or deep link into the app and read `token`.

---

## 7. Reset password

### `POST /v1/auth/reset-password`

**Rate limit:** 10 / hour.

**Body:**

```json
{
  "token": "<from reset link>",
  "newPassword": "min 10 chars"
}
```

**Success (`200`):**

```json
{
  "message": "Password updated. Please log in with your new password."
}
```

**Side effect:** All **existing refresh sessions** for that user are **revoked**. The user must log in again on all devices.

**Errors:**

| Status | Meaning |
|--------|---------|
| `400` | Invalid or expired reset token |

---

## 8. Logout

### `POST /v1/auth/logout`

**Auth required:** `Authorization: Bearer <accessToken>`

**Body (recommended for RN):**

```json
{ "refreshToken": "<current refresh token>" }
```

If omitted, server uses `quantts_refresh` cookie when present.

**Success (`200`):**

```json
{ "message": "Logged out successfully." }
```

Clears the current refresh token row (for that token). Also clears cookies in the response (web).

### `POST /v1/auth/logout-all`

**Auth required:** Bearer access token.

**Body:** none required.

Revokes **all** refresh sessions for the user. Use for “sign out everywhere”.

---

## 9. Calling protected APIs

Use:

```http
Authorization: Bearer <accessToken>
Content-Type: application/json
```

The server accepts **either** Bearer **or** `quantts_access` cookie (`apps/api/src/index.ts` — `authenticate` decorator).

Example mobile routes (all require auth): `GET /v1/mobile/overview`, etc. (`apps/api/src/routes/mobile.ts`).

---

## 10. Optional: session management

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| `GET` | `/v1/auth/sessions` | Bearer | List active refresh sessions (metadata). |
| `DELETE` | `/v1/auth/sessions/:id` | Bearer | Revoke one session. |
| `DELETE` | `/v1/auth/sessions` | Bearer | Revoke all sessions (same idea as logout-all). |

Cookie-based “current session” detection may not match native; treat as best-effort for listing.

---

## React Native implementation checklist

1. **Config:** `QUANTT_API_BASE_URL` (or equivalent) pointing at the API, not the Next.js site (unless you proxy).
2. **Signup** → show “check email” → **verify** via web link or `POST /v1/auth/verify-email` with parsed `token`.
3. **Login** → if `mfaRequired`, show TOTP UI → **`/v1/auth/totp/challenge`** → store session.
4. **HTTP client:** Attach `Authorization: Bearer` on each request; on **401**, call **`/v1/auth/refresh`**, retry once, else clear storage and navigate to login.
5. **Refresh:** Always send `refreshToken` in JSON; **update** secure storage with both new tokens.
6. **Forgot password:** Collect email → `forgot-password` → user completes **`reset-password`** on web or in-app with `token` + new password.
7. **Logout:** Send Bearer + `refreshToken` in body; clear local storage.

---

## Backend env vars relevant to these flows (ops / not in the app)

| Variable | Relevance |
|----------|-----------|
| `APP_URL` | Base for **verification** and **password reset** links in emails (`/verify-email`, `/reset-password`). Should be your **web** URL users can open. |
| `SMTP_*`, `EMAIL_FROM` | If unset, dev may log emails to console instead of sending (`apps/api/src/lib/email.ts`). |
| `JWT_SECRET`, `JWT_REFRESH_SECRET` | Issuing/verifying access and refresh JWTs (server only). |

Google-specific variables are documented in [google-oauth-react-native.md](./google-oauth-react-native.md).

---

## Source files in this repo

| Area | Path |
|------|------|
| Auth routes | `apps/api/src/routes/auth.ts` |
| Auth logic | `apps/api/src/services/auth-service.ts` |
| Password reset | `apps/api/src/services/password-reset-service.ts` |
| TOTP / MFA challenge | `apps/api/src/routes/totp.ts` |
| JWT / Bearer vs cookie | `apps/api/src/index.ts` |
| Email templates / links | `apps/api/src/lib/email.ts` |

---

## Document history

| Date | Change |
|------|--------|
| 2026-05-20 | Initial version: RN-focused auth API (signup, verify, login, MFA, refresh, forgot/reset, logout). |
