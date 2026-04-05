# Module: `auth`

## Purpose

**auth** covers **admin** authentication: session-backed login/logout, exposing helpers on the Express `request` object, JWT-related API middleware for identifying the current user, and **GraphQL admin types** for `AdminUser`.

## How it works

1. **Bootstrap** — `bootstrap.ts` attaches to `express.request`:

   - `loginUserWithEmail(email, password, callback)` — delegates to `loginUserWithEmail` service, then `session.save`.
   - `logoutUser(callback)` — clears session via `logoutUser` service.
   - `isUserLoggedIn()` — `!!session.userID`.
   - `getCurrentUser()` — returns `res.locals.user` (populated by auth middleware).

2. **HTTP** — `api/global/` chains JWT and `getCurrentUser` handlers; token refresh lives under `api/refreshUserToken/` and `api/getUserToken/`. Admin login/logout JSON endpoints sit under `pages/admin/`.

3. **Pages** — Admin shell uses `[context]isAdmin[auth].js` to gate the admin UI.

4. **Persistence** — Migrations maintain admin user tables (see `migration/`).

## Framework implementation

| Mechanism | Location |
|-----------|----------|
| Bootstrap / `request` API | `modules/auth/bootstrap.ts` |
| Login/logout services | `services/loginUserWithEmail.ts`, `logoutUser.ts` |
| Session naming helpers | `getSessionConfig.ts`, `getAdminSessionCookieName.ts`, `getCookieSecret.ts` |
| API | `api/global/[context]jwtUserAuth[getCurrentUser].ts`, `api/getUserToken/*`, `api/refreshUserToken/*` |
| GraphQL | `graphql/types/AdminUser/AdminUser.admin.graphql` + resolvers |

Customer-facing auth is implemented in the **customer** module; `auth` is intentionally **admin-centric**.

## HTTP APIs

| Route | Method | Path | Role |
|-------|--------|------|------|
| `getUserToken` | POST | `/user/tokens` | Issue JWT for admin user |
| `refreshUserToken` | POST | `/user/token/refresh` | Refresh admin JWT |
| `adminLoginJson` | POST | `/user/login` | Admin session login |
| `adminLogoutJson` | GET, POST | `/user/logout` | Admin session logout |
| `adminLogin` | GET | `/login` | Admin login page |

Global middleware in `api/global/`: `[context]jwtUserAuth`, `getCurrentUser`, `demoAccountBlocking`, `auth`.

## Database tables

| Table | Migration | Role |
|-------|-----------|------|
| `admin_user` | `Version-1.0.0` (create) | Admin user credentials and profile |
| `user_token_secret` | `Version-1.0.0` (create), `Version-1.0.1` (dropped) | Deprecated token secret storage |
| `session` | `Version-1.0.1` (create, PK on `sid`, index on `expire`) | Express session store (admin + customer) |

## GraphQL types

| Type | File | Scope | Query fields |
|------|------|-------|-------------|
| `AdminUser`, `AdminUserCollection` | `AdminUser.admin.graphql` | Admin only | `adminUser`, `currentAdminUser`, `adminUsers` |

## User flows

### Admin login (session-based)

```mermaid
sequenceDiagram
    participant Admin
    participant Browser
    participant LoginPage as GET /login
    participant LoginAPI as POST /user/login
    participant AuthService as loginUserWithEmail
    participant DB as admin_user table
    participant Session as session table

    Admin->>Browser: Navigate to /login
    Browser->>LoginPage: GET /login
    LoginPage-->>Browser: Render login form
    Admin->>Browser: Enter email + password
    Browser->>LoginAPI: POST /user/login {email, password}
    LoginAPI->>AuthService: req.loginUserWithEmail(email, password)
    AuthService->>DB: SELECT from admin_user WHERE email
    DB-->>AuthService: User row
    AuthService->>AuthService: Verify password hash
    alt Valid credentials
        AuthService->>Session: session.userID = user.id
        Session->>DB: Persist session (session table)
        AuthService-->>LoginAPI: Success
        LoginAPI-->>Browser: 200 + redirect to dashboard
    else Invalid
        AuthService-->>LoginAPI: Error
        LoginAPI-->>Browser: 401 Unauthorized
    end
```

### Admin JWT token flow (API clients)

```mermaid
sequenceDiagram
    participant APIClient as API Client
    participant TokenAPI as POST /user/tokens
    participant RefreshAPI as POST /user/token/refresh
    participant Protected as Protected API endpoint

    APIClient->>TokenAPI: POST /user/tokens {email, password}
    TokenAPI-->>APIClient: {token, refreshToken}
    APIClient->>Protected: GET /api/... (Authorization: Bearer token)
    Protected->>Protected: jwtUserAuth middleware validates token
    Protected-->>APIClient: 200 response data
    Note over APIClient,Protected: When token expires
    APIClient->>RefreshAPI: POST /user/token/refresh {refreshToken}
    RefreshAPI-->>APIClient: {token, refreshToken} (new pair)
```

## What could be done better

- **Naming** — The module name `auth` sounds generic; new contributors may assume it covers customers. Renaming to `adminAuth` (breaking) or a prominent note in `AGENTS.md` / this doc reduces confusion.

- **Session vs JWT** — Document the split: where session ends and JWT begins for API clients, and token lifetimes/rotation policy.

- **Type safety** — `EvershopRequest` extensions are scattered; a single referenced interface document or barrel export helps extensions implement compatible middleware.

- **Security hardening** — Rate limiting and account lockout for `loginUserWithEmail` (if not present elsewhere) are typical production asks; worth auditing in one place.
