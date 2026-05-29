# Google OAuth — React Native integration

This document describes how the Quantt **API** exposes Google sign-in and how a **React Native** app should integrate. Web flow is summarized for context only.

For **email/password** auth (signup, login, refresh, forgot password), see [auth-api-react-native.md](./auth-api-react-native.md).

---

## Base URL

Use the Fastify API origin (no trailing slash):

| Environment | Example |
|---------------|---------|
| Local | `http://localhost:4000` (or your dev host/port) |
| Production | `https://api.yourdomain.com` |

All paths below are relative to this base.

---

## API endpoints (Google-related)

| Method | Path | Intended client | Purpose |
|--------|------|-------------------|---------|
| `GET` | `/v1/auth/google` | **Web browser** | Redirects to Google, then to `/v1/auth/google/callback`, then to the Next.js app with HTTP-only cookies. **Not the primary native flow.** |
| `GET` | `/v1/auth/google/callback` | **Google → server** | OAuth redirect target for the **web** authorization flow. Registered in Google Cloud as `GOOGLE_REDIRECT_URI`. |
| `POST` | `/v1/auth/google/token` | **React Native (recommended)** | Exchanges Google’s **authorization code** (and optional PKCE) for Quantt **JSON** session tokens. |

### `POST /v1/auth/google/token`

**Headers:** `Content-Type: application/json`

**Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `code` | string | Yes | Authorization code returned by Google after user consent. |
| `redirectUri` | string | Yes | **Exact** same `redirect_uri` used in the Google authorize request. Must be allowlisted on the server (see [Backend environment variables](#backend-environment-variables)). |
| `codeVerifier` | string | If PKCE was used | Length 43–128 characters (RFC 7636). Required when the authorize URL included `code_challenge` / PKCE. |
| `clientId` | string | No | Google `client_id` used in the authorize step. Omit to use the server’s default `GOOGLE_CLIENT_ID`. Set when using a separate native client registered as `GOOGLE_MOBILE_CLIENT_ID` on the server. |

**Example request:**

```http
POST /v1/auth/google/token HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "code": "4/0A...",
  "redirectUri": "com.yourcompany.quantt:/oauth",
  "codeVerifier": "dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk",
  "clientId": "123456789-abc.apps.googleusercontent.com"
}
```

**Success (`200`):** Either:

- **Full session** (same shape as email/password login): `accessToken`, `refreshToken`, `user` (`id`, `email`, `role`, `name`), **or**
- **MFA required** (when the user has TOTP enabled): `{ "mfaRequired": true, "mfaPendingToken": "<jwt>" }` — then call `POST /v1/auth/totp/challenge` with `mfaPendingToken` and the 6-digit code (same as password login); response includes tokens to store.

**Error responses:**

| Status | Meaning |
|--------|---------|
| `400` | `redirect_uri` or `client_id` is not allowed for this server. |
| `401` | Invalid/expired code, or Google rejected the token exchange. |
| `503` | Google OAuth not configured (missing client id and/or no allowlisted redirect URIs). |

**Rate limiting:** 20 requests per minute (per route configuration).

### Refreshing the session (not Google-specific)

After Google login, use the normal refresh endpoint:

- `POST /v1/auth/refresh`  
- Prefer sending `refreshToken` in the JSON body from secure storage (the web app may use cookies; native should use the body).

---

## Recommended native integration flow

### 1. Google Cloud Console

1. Create/configure the **OAuth consent screen**.
2. Create **OAuth 2.0 Client ID** credentials as needed:
   - **Web** client: used by existing browser sign-in (`GOOGLE_CLIENT_ID` + secret + server callback).
   - For the app, either:
     - Use the **same Web client** and add **native redirect URIs** (custom scheme or HTTPS app link) and use **authorization code + PKCE** from the app, **or**
     - Use **separate iOS / Android** OAuth clients; then the backend must list `GOOGLE_MOBILE_CLIENT_ID` and the app must send `clientId` in `POST /v1/auth/google/token`.
3. Register **every** `redirect_uri` the app will use (dev, staging, prod). They must match **exactly** (including trailing slashes if any).

### 2. Deep linking

Define a stable OAuth return URL, for example:

- Custom scheme: `com.yourcompany.quantt:/oauth`
- Or universal / app link: `https://app.yourcompany.com/oauth`

The same string must appear in:

- Google Cloud Console (authorized redirect URIs),
- the Google **authorize** request as `redirect_uri`,
- the **`redirectUri`** field in `POST /v1/auth/google/token`,
- backend env allowlist (`GOOGLE_REDIRECT_URI` or `GOOGLE_OAUTH_ADDITIONAL_REDIRECT_URIS`).

### 3. React Native libraries (suggested approaches)

**Authorization code + PKCE (matches this backend):**

- **Expo:** `expo-auth-session` + `expo-web-browser` (or equivalent) to open Google and capture the redirect with `code`.
- **Bare React Native:** `react-native-app-auth`, or a WebBrowser / `ASWebAuthenticationSession`-style flow with PKCE.

**Google Sign-In SDK (`@react-native-google-signin/google-signin`):**

- Typically yields **ID tokens** on the device. This backend endpoint is built around **authorization `code` exchange**. Using the SDK would require a **different** server contract (e.g. verify ID token) unless you still drive a code+PKCE flow. For alignment with `POST /v1/auth/google/token`, prefer **browser / in-app browser + PKCE**.

### 4. Building the Google authorize URL

Open in the system browser or an auth session (not a plain WebView that shares cookies with untrusted content if you can avoid it).

Minimum query parameters (align with product needs):

| Parameter | Value / notes |
|-----------|----------------|
| `client_id` | Same id the backend will accept (default server `GOOGLE_CLIENT_ID`, or mobile client if using `GOOGLE_MOBILE_CLIENT_ID` + `clientId` in POST). |
| `redirect_uri` | Your app deep link (allowlisted). |
| `response_type` | `code` |
| `scope` | e.g. `openid email profile` |
| `state` | **Recommended:** random opaque value; validate on return. |
| PKCE | If used: `code_challenge`, `code_challenge_method=S256`; store `code_verifier` until the token `POST`. |

Authorize URL pattern:

```text
https://accounts.google.com/o/oauth2/v2/auth?client_id=...&redirect_uri=...&response_type=code&scope=openid%20email%20profile&state=...&code_challenge=...&code_challenge_method=S256
```

### 5. After Google redirects to the app

1. Parse `code` from the query string; validate `state`.
2. Call `POST /v1/auth/google/token` with `code`, `redirectUri`, and `codeVerifier` (if PKCE).
3. Store `accessToken` and `refreshToken` in **secure storage** (iOS Keychain, Android EncryptedSharedPreferences / Keystore).
4. Send `Authorization: Bearer <accessToken>` on API requests.
5. On `401`, call `POST /v1/auth/refresh` with the stored `refreshToken`.

### 6. Coordination checklist (app ↔ backend)

- [ ] Share exact **redirect URIs** per environment.
- [ ] Backend sets `GOOGLE_OAUTH_ADDITIONAL_REDIRECT_URIS` (and Google Console matches).
- [ ] Agree on **single Web client id + PKCE** vs **separate `GOOGLE_MOBILE_CLIENT_ID`** and document `clientId` in POST when needed.

---

## Backend environment variables

These variables live on the **API server** (`apps/api`), in `.env` or your secrets manager. **Never** ship `GOOGLE_CLIENT_SECRET` in the mobile app.

| Variable | Web Google login | `POST /v1/auth/google/token` | Description |
|----------|------------------|------------------------------|-------------|
| `GOOGLE_CLIENT_ID` | Required | Required unless only `GOOGLE_MOBILE_CLIENT_ID` is configured | Primary Google OAuth client id. |
| `GOOGLE_CLIENT_SECRET` | Required for web callback | Sent to Google **only** when the exchange uses `GOOGLE_CLIENT_ID` (web client). | Confidential; server-only. |
| `GOOGLE_REDIRECT_URI` | Required | Allowlisted when set | Web callback URL, e.g. `https://api.example.com/v1/auth/google/callback`. |
| `GOOGLE_OAUTH_ADDITIONAL_REDIRECT_URIS` | Optional | **Needed for typical native redirects** unless `redirectUri` equals `GOOGLE_REDIRECT_URI` | Comma-separated list of extra allowed `redirect_uri` values (e.g. `com.app:/oauth,https://app.example.com/oauth`). |
| `GOOGLE_MOBILE_CLIENT_ID` | Optional | Optional | Second allowed Google client id; app may send it as JSON `clientId` when the authorize step used that client. |

**Allowlist rule:** The `redirectUri` sent in `POST /v1/auth/google/token` must **exactly** match `GOOGLE_REDIRECT_URI` **or** one of the comma-separated values in `GOOGLE_OAUTH_ADDITIONAL_REDIRECT_URIS`.

See `apps/api/.env.example` for a template.

---

## React Native app configuration (typical)

Store only non-secret values in the app config / env:

| Name | Example | Notes |
|------|---------|--------|
| `QUANTT_API_BASE_URL` | `https://api.example.com` | Origin for `POST /v1/auth/google/token` and other APIs. |
| Google `client_id` | Same as used in authorize URL | Public. |
| OAuth `redirect_uri` | e.g. `com.yourcompany.quantt:/oauth` | Must match Google + backend allowlist. |

Do **not** embed `GOOGLE_CLIENT_SECRET` in the app.

---

## References in this repository

- OAuth routes: `apps/api/src/routes/oauth.ts`
- Token exchange logic: `apps/api/src/services/oauth-service.ts`
- Env template: `apps/api/.env.example`

---

## Document history

| Date | Change |
|------|--------|
| 2026-05-20 | Initial version: native `POST /v1/auth/google/token`, env vars, RN flow. |
