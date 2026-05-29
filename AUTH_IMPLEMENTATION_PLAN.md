# Authentication Gaps — Implementation Plan (API + Web)

> Branch: `signup` | Scope: `apps/api` + `apps/web` only (no mobile)
> Current date: 2026-05-19

---

## Current State Summary

| Layer | What exists |
|-------|-------------|
| API   | `POST /v1/auth/register`, `POST /v1/auth/login`, `POST /v1/auth/refresh`, wallet SIWE login |
| DB    | `User` (id, email, name, passwordHash, role, createdAt), `RefreshToken` (rotation + revocation) |
| Web   | Single `apps/web/app/login/page.tsx` — login + register form, wallet connect |
| Missing | Email verification, password reset, 2FA/TOTP, SSO, RBAC middleware, logout endpoint, session list/revoke |

---

## Feature 1 — Email Verification (Priority: HIGH)

### Goal
Block login until the user confirms their email address. Account is created but locked until the verification link is clicked.

### API Changes

**Schema additions — `prisma/schema.prisma`**
```prisma
model User {
  // add these two fields
  emailVerified     Boolean  @default(false)
  verifyToken       String?  @unique   // short-lived hex token
  verifyTokenExpiry DateTime?
}
```

**New route — `POST /v1/auth/verify-email`**
```
Body: { token: string }
- Look up User by verifyToken
- Check verifyTokenExpiry > now()
- Set emailVerified = true, clear verifyToken + expiry
- Return 200 { message: "Email verified" }
```

**New route — `POST /v1/auth/resend-verification`**
```
Body: { email: string }
- Rate-limit: 3 requests per hour per email
- If user exists and emailVerified === false, regenerate token + resend email
- Always return 200 (don't leak whether email exists)
```

**Modified `register()` in `auth-service.ts`**
- After `prisma.user.create`, generate a `crypto.randomBytes(32).toString('hex')` token
- Save `verifyToken` + `verifyTokenExpiry` (now + 24 hours) on the user
- Send verification email via email service (see Email Service section below)
- Do NOT return an `accessToken` — return `{ message: "Check your email to verify your account" }`

**Modified `login()` in `auth-service.ts`**
- After password check, before `buildSession`:
  ```ts
  if (!user.emailVerified) throw app.httpErrors.forbidden("Please verify your email before logging in");
  ```

### Email Service — `apps/api/src/lib/email.ts`
Use **Resend** (recommended) or **Nodemailer + SMTP**.

```ts
// Minimal interface
export async function sendVerificationEmail(to: string, token: string): Promise<void>
export async function sendPasswordResetEmail(to: string, token: string): Promise<void>
export async function sendMagicLinkEmail(to: string, token: string): Promise<void>  // used by SSO fallback
```

**Environment variables needed:**
```env
RESEND_API_KEY=re_...
EMAIL_FROM=noreply@quantts.ai
NEXT_PUBLIC_APP_URL=https://app.quantts.ai
```

### Web Changes

**New page — `apps/web/app/verify-email/page.tsx`**
- Reads `?token=` from URL query params
- Calls `POST /v1/auth/verify-email` on mount
- Shows: loading → success ("Email verified, redirecting…") → error ("Link expired, resend?")
- On success, redirect to `/dashboard` after 2 seconds

**Modified `apps/web/app/login/page.tsx`**
- After register API call returns 200, show: "Check your inbox — we sent a verification link to `{email}`"
- Add "Resend verification email" button (calls `POST /v1/auth/resend-verification`)

**New page — `apps/web/app/verify-email/resend/page.tsx`** _(optional, simple form)_
- Email input + submit → calls resend endpoint

---

## Feature 2 — Password Reset / Forgot Password (Priority: HIGH)

### Goal
Allow users to reset their password via an emailed token link.

### API Changes

**Schema additions — `prisma/schema.prisma`**
```prisma
model PasswordResetToken {
  id        String   @id @default(cuid())
  userId    String
  tokenHash String   @unique
  expiresAt DateTime
  usedAt    DateTime?
  createdAt DateTime @default(now())
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

// Add to User model:
  passwordResets PasswordResetToken[]
```

**New route — `POST /v1/auth/forgot-password`**
```
Body: { email: string }
Rate-limit: 5 requests per hour per IP
- Find user by email
- Generate crypto.randomBytes(32).toString('hex') token
- Hash it with sha256 before storing (same pattern as refreshToken)
- Set expiry: now + 1 hour
- Send password reset email with link: {APP_URL}/reset-password?token={rawToken}
- Always return 200 (no email enumeration)
```

**New route — `POST /v1/auth/reset-password`**
```
Body: { token: string, newPassword: string (min 10 chars) }
- sha256 hash the token, look up PasswordResetToken
- Check: not expired, not used, user still exists
- bcrypt.hash newPassword (rounds: 12)
- Update user.passwordHash
- Mark token as usedAt = now()
- Revoke ALL active refresh tokens for this user (security: new password = new session)
- Return 200 { message: "Password updated. Please log in." }
```

**New service — `apps/api/src/services/password-reset-service.ts`**
Contains `requestPasswordReset(app, email)` and `resetPassword(app, token, newPassword)`.

### Web Changes

**New page — `apps/web/app/forgot-password/page.tsx`**
- Email input form
- On submit → `POST /v1/auth/forgot-password`
- Show: "If that email exists, we sent a reset link"

**New page — `apps/web/app/reset-password/page.tsx`**
- Reads `?token=` from URL
- Shows new password + confirm password fields
- Validates passwords match client-side before submit
- On submit → `POST /v1/auth/reset-password`
- On success → redirect to `/login` with success message

**Modified `apps/web/app/login/page.tsx`**
- Add "Forgot password?" link below password field → `/forgot-password`

---

## Feature 3 — 2FA / TOTP (Priority: HIGH)

### Goal
Allow users to enable TOTP-based 2FA (Google Authenticator, Authy). Login becomes a two-step flow when 2FA is enabled.

### API Changes

**Schema additions — `prisma/schema.prisma`**
```prisma
model User {
  // add:
  totpSecret    String?   // encrypted AES-256 before storing
  totpEnabled   Boolean   @default(false)
  totpBackupCodes String[] // hashed bcrypt backup codes (store as array)
}
```

**New routes — `apps/api/src/routes/totp.ts`** (authenticated, requires valid access token)

`POST /v1/auth/totp/setup`
```
- Generate secret: speakeasy.generateSecret({ name: "Quantts", issuer: "quantts.ai" })
- Encrypt secret with AES-256 key from env before saving to DB (not yet enabled)
- Return { otpauthUrl, qrCodeDataUrl, backupCodes: string[10] }
  (backup codes: 10 random 8-char codes, bcrypt-hash each before storing)
```

`POST /v1/auth/totp/verify-setup`
```
Body: { code: string }  // 6-digit TOTP from authenticator app
- Decrypt stored secret, verify code with speakeasy.totp.verify (window: 1)
- If valid: set totpEnabled = true
- Return 200 { message: "2FA enabled" }
```

`POST /v1/auth/totp/disable`
```
Body: { password: string }  // require password confirmation to disable
- Verify password with bcrypt
- Set totpEnabled = false, clear totpSecret + backupCodes
```

**Modified `login()` flow in `auth-service.ts`**
```
If user.totpEnabled === true:
  - Do NOT issue access/refresh tokens yet
  - Issue a short-lived "pending MFA" JWT (expiresIn: 5m, payload: { sub: userId, type: "mfa_pending" })
  - Return { mfaRequired: true, mfaPendingToken: string }

New route: POST /v1/auth/totp/challenge
  Body: { mfaPendingToken: string, code: string }
  - Verify mfaPendingToken (type must be "mfa_pending", not expired)
  - Decrypt totpSecret, verify TOTP code (or match backup code)
  - If backup code: mark it as used (remove from array)
  - Issue full access + refresh tokens → buildSession()
```

**Environment variables needed:**
```env
TOTP_ENCRYPTION_KEY=<32-byte hex>  # for AES-256 encrypting TOTP secrets at rest
```

**Dependencies to add:**
```
apps/api: speakeasy, qrcode
```

### Web Changes

**New page — `apps/web/app/dashboard/security/page.tsx`**
- Shows 2FA status (enabled/disabled)
- "Enable 2FA" button → triggers setup flow:
  1. Fetch `POST /v1/auth/totp/setup` → show QR code + backup codes
  2. Enter 6-digit code to confirm → `POST /v1/auth/totp/verify-setup`
  3. Show success + prompt to save backup codes
- "Disable 2FA" button → password confirmation modal → `POST /v1/auth/totp/disable`

**Modified `apps/web/app/login/page.tsx`**
- Detect `mfaRequired: true` in login response
- Show TOTP code input step (6-digit field)
- "Use backup code instead" toggle
- On submit → `POST /v1/auth/totp/challenge`

---

## Feature 4 — SSO: Google & Apple (Priority: MEDIUM)

### Goal
Allow users to sign in / register with Google OAuth 2.0 and Apple Sign-In.

### API Changes

**Schema additions — `prisma/schema.prisma`**
```prisma
model OAuthAccount {
  id         String   @id @default(cuid())
  userId     String
  provider   String   // "google" | "apple"
  providerId String   // the sub/id from the provider
  createdAt  DateTime @default(now())
  user       User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerId])
}

// Add to User model:
  oauthAccounts OAuthAccount[]
  // passwordHash becomes optional (SSO users may have no password)
  passwordHash  String?
```

**New routes — `apps/api/src/routes/oauth.ts`**

`GET /v1/auth/google` — redirect to Google OAuth consent screen
`GET /v1/auth/google/callback` — handle Google callback
`GET /v1/auth/apple` — redirect to Apple Sign-In
`POST /v1/auth/apple/callback` — handle Apple form_post callback

**OAuth flow (Google example):**
```
1. /v1/auth/google → redirect to accounts.google.com with client_id, redirect_uri, scope=email profile
2. /v1/auth/google/callback?code=...
   - Exchange code for tokens via Google token endpoint
   - Fetch profile: { sub, email, name, picture }
   - Find OAuthAccount by (provider="google", providerId=sub)
   - If found: load user, buildSession()
   - If not found + email matches existing user: link account, buildSession()
   - If not found + no user: create User (emailVerified=true for SSO), create OAuthAccount, buildSession()
   - Set cookies + redirect to {APP_URL}/dashboard
```

**New service — `apps/api/src/services/oauth-service.ts`**
Contains `handleGoogleCallback(app, code)` and `handleAppleCallback(app, idToken)`.

**Environment variables needed:**
```env
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
GOOGLE_REDIRECT_URI=https://api.quantts.ai/v1/auth/google/callback

APPLE_CLIENT_ID=...
APPLE_TEAM_ID=...
APPLE_KEY_ID=...
APPLE_PRIVATE_KEY=...   # base64 encoded .p8 file
APPLE_REDIRECT_URI=https://api.quantts.ai/v1/auth/apple/callback
```

**Dependencies to add:**
```
apps/api: arctic (OAuth library for Fastify), jose (Apple JWT verification)
```

### Web Changes

**Modified `apps/web/app/login/page.tsx`**
- Add "Continue with Google" button → `window.location.href = API_BASE + '/v1/auth/google'`
- Add "Continue with Apple" button → same pattern
- Styled with provider brand colors/icons (use SVG icons)

**Note:** SSO callback redirects back to the API which sets cookies then redirects to `/dashboard` — no extra web page needed for the happy path. Add an `/auth/error` page for OAuth failures.

---

## Feature 5 — Role-Based Route Guards / RBAC Middleware (Priority: HIGH)

### Goal
Enforce that authenticated routes check the user's `role` claim from the JWT, and return 403 if the role does not have permission.

### Role Definitions

| Role    | Access |
|---------|--------|
| `admin` | Full access — all routes |
| `trader`| Trading routes, own account settings, dashboard |
| `viewer`| Read-only dashboard, no trading actions |

### API Changes

**New plugin — `apps/api/src/plugins/auth-guard.ts`**
```ts
import fp from "fastify-plugin";

export default fp(async (app) => {
  // Decorator: app.authenticate — verifies JWT access token from cookie or Authorization header
  app.decorate("authenticate", async (request, reply) => {
    try {
      await request.jwtVerify();
    } catch {
      throw app.httpErrors.unauthorized("Invalid or expired token");
    }
  });

  // Decorator: app.requireRole(...roles) — use after authenticate
  app.decorate("requireRole", (...roles: string[]) => async (request, reply) => {
    const user = request.user as { role: string };
    if (!roles.includes(user.role)) {
      throw app.httpErrors.forbidden(`Requires role: ${roles.join(" or ")}`);
    }
  });
});
```

**Apply guards to existing routes:**

```ts
// Example: dashboard routes (trader + admin only)
app.get("/v1/dashboard/metrics", {
  preHandler: [app.authenticate, app.requireRole("admin", "trader")]
}, handler);

// Example: admin-only
app.get("/v1/admin/users", {
  preHandler: [app.authenticate, app.requireRole("admin")]
}, handler);
```

**Audit all existing routes in:**
- `apps/api/src/routes/dashboard.ts` — add `authenticate` + role check
- `apps/api/src/routes/auth.ts` — register/login stay public; `/v1/auth/refresh` stays public
- `apps/api/src/routes/auth-wallet.ts` — wallet verify stays public; wallet-linked actions need guard

### Web Changes

**New utility — `apps/web/lib/auth.ts`**
```ts
// Server-side: parse the quantts_access cookie from Next.js request headers
export async function getSessionUser(req): Promise<{ id, email, role, name } | null>

// Client-side context / hook
export function useSession(): { user: SessionUser | null; loading: boolean }
```

**New middleware — `apps/web/middleware.ts`**
```ts
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

const PUBLIC_PATHS = ["/login", "/forgot-password", "/reset-password", "/verify-email"];
const ADMIN_PATHS = ["/admin"];
const VIEWER_BLOCKED = ["/trade", "/orders"];

export function middleware(request: NextRequest) {
  const token = request.cookies.get("quantts_access")?.value;
  const { pathname } = request.nextUrl;

  if (!token && !PUBLIC_PATHS.some(p => pathname.startsWith(p))) {
    return NextResponse.redirect(new URL("/login", request.url));
  }
  // Decode JWT payload (no verify — API verifies on data fetch)
  // Check role from payload and block accordingly
  // Return NextResponse.next() if allowed
}

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico).*)"],
};
```

---

## Feature 6 — Logout / Token Revocation (Priority: MEDIUM)

### Goal
Add `POST /v1/auth/logout` that revokes the current refresh token and clears cookies.

### API Changes

**New route — `POST /v1/auth/logout`** (add to `apps/api/src/routes/auth.ts`)
```
preHandler: [app.authenticate]

Body: { refreshToken?: string }  // optional — falls back to cookie
- sha256 hash the refresh token
- Find RefreshToken row, set revokedAt = now()
- Clear quantts_access and quantts_refresh cookies (set to expired)
- Return 200 { message: "Logged out" }
```

**New route — `POST /v1/auth/logout-all`** (bonus, needed for "session list & revoke all")
```
preHandler: [app.authenticate]
- Revoke ALL RefreshToken rows for request.user.id where revokedAt is null
- Clear cookies
- Return 200 { message: "All sessions terminated" }
```

### Web Changes

**Modified `apps/web/components/nav.tsx`**
- Add "Sign out" button / menu item
- On click → `POST /v1/auth/logout` → redirect to `/login`

---

## Feature 7 — Session List & Revoke All (Priority: LOW)

### Goal
Let users see all active sessions and revoke any or all of them.

### API Changes

**Schema additions — `prisma/schema.prisma`**
```prisma
model RefreshToken {
  // add:
  userAgent  String?
  ipAddress  String?
  lastUsedAt DateTime?
}
```

**Modified `buildSession()` in `auth-service.ts`**
- Accept `request` as a parameter to capture `userAgent` and `ipAddress` when creating the token
- Set `lastUsedAt = now()` on each `/v1/auth/refresh` call

**New route — `GET /v1/auth/sessions`**
```
preHandler: [app.authenticate]
- Return all non-revoked, non-expired RefreshToken rows for the user
- Fields: id, userAgent, ipAddress, createdAt, lastUsedAt, expiresAt
- Flag the "current" session (match via token hash from cookie)
```

**New route — `DELETE /v1/auth/sessions/:id`**
```
preHandler: [app.authenticate]
- Verify the token row belongs to request.user.id
- Set revokedAt = now()
```

**New route — `DELETE /v1/auth/sessions`** (revoke all)
```
preHandler: [app.authenticate]
- Revoke all active tokens (same as logout-all but doesn't clear cookies)
```

### Web Changes

**New page — `apps/web/app/dashboard/security/sessions/page.tsx`**
- Table: Device/Browser | IP | First seen | Last used | Action
- "Revoke" button per row (calls `DELETE /v1/auth/sessions/:id`)
- "Revoke all other sessions" button at top

---

## Database Migration Summary

Run in order after all schema changes are finalised:

```
Migration 1: email-verification
  - Add User.emailVerified, User.verifyToken, User.verifyTokenExpiry

Migration 2: password-reset
  - Add PasswordResetToken model
  - Add User.passwordResets relation

Migration 3: totp
  - Add User.totpSecret, User.totpEnabled, User.totpBackupCodes

Migration 4: oauth
  - Add OAuthAccount model
  - Make User.passwordHash optional

Migration 5: session-metadata
  - Add RefreshToken.userAgent, RefreshToken.ipAddress, RefreshToken.lastUsedAt
```

```bash
npx prisma migrate dev --name <migration-name>
npx prisma generate
```

---

## New Environment Variables (consolidated)

Add to `apps/api/.env` and production secrets:

```env
# Email
RESEND_API_KEY=re_...
EMAIL_FROM=noreply@quantts.ai

# App URL (used for email links)
NEXT_PUBLIC_APP_URL=https://app.quantts.ai

# TOTP
TOTP_ENCRYPTION_KEY=<32-byte hex>

# Google OAuth
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
GOOGLE_REDIRECT_URI=https://api.quantts.ai/v1/auth/google/callback

# Apple Sign-In
APPLE_CLIENT_ID=...
APPLE_TEAM_ID=...
APPLE_KEY_ID=...
APPLE_PRIVATE_KEY=...
APPLE_REDIRECT_URI=https://api.quantts.ai/v1/auth/apple/callback
```

---

## New Dependencies

```bash
# API
pnpm add --filter api speakeasy qrcode resend arctic jose

# Types
pnpm add --filter api -D @types/speakeasy @types/qrcode
```

---

## Implementation Order (Recommended)

| Step | Feature | Why first |
|------|---------|-----------|
| 1 | Email Service setup (Resend) | Required by features 1 and 2 |
| 2 | Email Verification | Blocks login — highest security gap |
| 3 | RBAC Middleware | Protects all existing routes now |
| 4 | Logout endpoint | Quick win, closes revocation gap |
| 5 | Password Reset | High priority, depends on email service |
| 6 | 2FA / TOTP | More complex, build after core auth is solid |
| 7 | SSO (Google) | Requires external setup, medium priority |
| 8 | SSO (Apple) | Apple is more complex, do after Google |
| 9 | Session List & Revoke | Low priority, polish feature |

---

## Files to Create / Modify

### API (`apps/api`)
| File | Action |
|------|--------|
| `prisma/schema.prisma` | Modify — add fields + models |
| `src/services/auth-service.ts` | Modify — email verification gate, 2FA check |
| `src/services/email-service.ts` | Create |
| `src/services/password-reset-service.ts` | Create |
| `src/services/totp-service.ts` | Create |
| `src/services/oauth-service.ts` | Create |
| `src/routes/auth.ts` | Modify — add logout, verify-email, forgot/reset-password |
| `src/routes/totp.ts` | Create |
| `src/routes/oauth.ts` | Create |
| `src/plugins/auth-guard.ts` | Create |

### Web (`apps/web`)
| File | Action |
|------|--------|
| `app/login/page.tsx` | Modify — SSO buttons, 2FA step, forgot-password link, post-register message |
| `app/verify-email/page.tsx` | Create |
| `app/forgot-password/page.tsx` | Create |
| `app/reset-password/page.tsx` | Create |
| `app/dashboard/security/page.tsx` | Create — 2FA setup |
| `app/dashboard/security/sessions/page.tsx` | Create |
| `middleware.ts` | Create — RBAC route guard |
| `lib/auth.ts` | Create — session utilities |
| `components/nav.tsx` | Modify — logout button |
