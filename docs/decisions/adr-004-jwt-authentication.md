# ADR-004: JWT Authentication and Client Token Storage Strategy

- **Status:** Accepted
- **Date:** 2026-11-04
- **Approval:** Owner Approved (Option 1 Selected)

## Context
QueueLess requires stateless, secure authentication for registered Patients and Staff (`PROVIDER`, `ORG_ADMIN`). Passwords must be hashed using a modern, memory-hard algorithm (Argon2id). 

While JSON Web Tokens (JWT) are agreed for token verification, the client-side storage mechanism, token refresh lifecycle, and cross-site threat profile require an explicit security and architectural trade-off analysis before implementation in Phase 1.

## Decision Options

### Option 1 (Selected): In-Memory Access Token + HttpOnly Refresh Cookie
- **Access Token**: Short-lived JWT (15-minute expiration). Stored in client memory (JavaScript closure or React application state). Transmitted in the standard `Authorization: Bearer <token>` header for API requests.
- **Refresh Token**: Long-lived random cryptographic token (7-day expiration). Stored in an `HttpOnly`, `Secure`, `SameSite=Strict` (or `SameSite=Lax`) cookie with `Path=/api/v1/auth`. Persisted in a database `refresh_tokens` table with revocation tracking, family tracking, and one-time rotation upon use.
- **Refresh & Logout Flow**: When the short-lived access token expires, an HTTP client interceptor calls `POST /api/v1/auth/refresh`. The browser automatically includes the HttpOnly refresh cookie, the server issues a fresh access token and rotated refresh cookie, and the client replays the original request. Calling `POST /api/v1/auth/logout` revokes the refresh token in the database and clears the cookie.
- **Cookie Transmission Scope vs. Authentication Usage**:
  - The cookie `Path=/api/v1/auth` attribute limits where browsers attach the cookie, ensuring user agents do not send it to general API routes (e.g. `/api/v1/appointments`, `/api/v1/queue-tokens`).
  - However, the `Path` attribute limits transmission across path prefixes; it does not itself guarantee that only two endpoints receive it (browsers attach matching cookies to any request under `/api/v1/auth/*`, including login or registration).
  - Therefore, the server explicitly decouples cookie transmission from credential consumption:
    1. The refresh cookie is consumed as an authentication or authorization credential strictly by:
       - `POST /api/v1/auth/refresh`
       - `POST /api/v1/auth/logout`
    2. Other authentication endpoints (such as `POST /api/v1/auth/login`, `POST /api/v1/auth/register`, or password reset) must not use the refresh cookie as an authentication or authorization credential, ignoring any received refresh cookie.
    3. All cookie-authenticated state-changing endpoints (`/refresh` and `/logout`) must strictly enforce the documented Origin/Referer validation and CSRF controls.
- **Security Profile & Threat Analysis**:
  - **XSS Considerations**: Neither in-memory storage nor HttpOnly cookies provide complete immunity to Cross-Site Scripting (XSS). An `HttpOnly` flag prevents client-side JavaScript from reading the cookie value directly, preventing trivial exfiltration of the persistent refresh token string. However, if malicious JavaScript executes in the application context (via XSS), an attacker can still execute authenticated actions in the browser context (e.g. invoking API endpoints using the active session or issuing requests to `/api/v1/auth/refresh` to obtain valid access tokens). Furthermore, an in-memory access token is not directly persistent across page reloads or stored on disk, but it can be accessed or intercepted while held in memory by malicious code executing within the application context.
  - **CSRF Considerations**: General API endpoints authenticated via `Authorization: Bearer <token>` headers are resistant to ambient-credential cross-site request forgery because browsers do not automatically attach custom `Authorization` headers on cross-origin requests. Although the browser attaches the cookie to requests under `/api/v1/auth`, only endpoints that consume ambient credentials for authentication are subject to CSRF risks. Because the server strictly restricts refresh cookie authentication to `POST /api/v1/auth/refresh` and `POST /api/v1/auth/logout`, CSRF exposure is confined to these two cookie-authenticated endpoints, which must enforce appropriate protections:
    - **Primary Server-Side Origin/Referer Validation**: The server enforces this exact validation sequence for cookie-authenticated state-changing endpoints:
      1. If an `Origin` header is present, validate it against an exact server-side allowlist of trusted application origins.
      2. If `Origin` is absent, validate the `Referer` origin against the same allowlist.
      3. If both `Origin` and `Referer` headers are absent, reject the request.
      4. Reject malformed origins, untrusted origins, and any requests that fail validation.
      5. Do not use substring matching or permissive wildcard matching for trusted origins.
    - **Defense in Depth via SameSite Attribute**: The `SameSite` cookie attribute acts as defense in depth, not as a replacement for explicit server-side Origin/Referer validation and other required CSRF controls:
      - `SameSite=Strict` provides the strongest restriction and generally prevents cookies from being sent in cross-site contexts.
      - `SameSite=Lax` permits cookies in certain cross-site top-level navigations using safe HTTP methods (e.g. following a top-level link), but generally excludes cookies from cross-site subrequests (e.g. images, iframes, embedded resources, scripts) and cross-site `POST` requests.
      - Because QueueLess refresh and logout endpoints use `POST` (`POST /api/v1/auth/refresh` and `POST /api/v1/auth/logout`), `SameSite=Lax` helps reduce CSRF exposure by preventing browsers from attaching the refresh cookie on cross-site `POST` requests. However, `SameSite` must not be treated as a standalone protection or assumed to make endpoints CSRF-proof on its own due to potential browser inconsistencies or edge-case navigation contexts.
    - **Mandatory Anti-CSRF Token Verification**: Both `POST /api/v1/auth/refresh` and `POST /api/v1/auth/logout` strictly require anti-CSRF token verification to ensure requests originate from legitimate client application instances. The implementation must use either a synchronizer token pattern or a correctly implemented double-submit cookie pattern. The choice between a synchronizer token and a double-submit cookie remains an available implementation option, but anti-CSRF token verification itself is a mandatory requirement for both endpoints and is not optional.
    - **Custom Request Headers**: Custom headers (such as `X-Requested-With` or `X-QueueLess-Request`) must not be presented as standalone CSRF protection; their effectiveness strictly depends on mandatory server-side enforcement (rejecting requests lacking the header) paired with a restrictive CORS policy that prevents unauthorized cross-origin preflights from setting custom headers.

### Option 2: Dual HttpOnly Cookies (Access & Refresh)
- Both access and refresh tokens are stored in `HttpOnly`, `Secure`, `SameSite=Strict` cookies.
- The browser automatically attaches authentication cookies on all API requests.
- **Security Profile & Threat Analysis**:
  - **XSS Considerations**: `HttpOnly` prevents direct JavaScript token string extraction, but does not prevent an attacker with XSS execution from issuing authenticated browser requests using the victim's ambient credentials.
  - **CSRF Considerations**: Because ambient credentials are sent automatically on all requests, **all state-changing API endpoints** across the entire application are exposed to CSRF risks. This requires comprehensive CSRF mitigation across every route (strict Origin allowlist verification, synchronizer tokens, and restrictive CORS rules).

### Option 3: LocalStorage Access & Refresh Tokens
- Both tokens stored in browser `localStorage`.
- **Security Profile**: Persistent tokens stored in browser `localStorage` are directly accessible via the DOM storage API (`localStorage.getItem(...)`). Malicious JavaScript executing in the application's origin during an XSS compromise can read and exfiltrate persistent tokens, granting an attacker continued access even after the browser session closes. Strongly discouraged for production-grade authentication.

## Password Hashing Decision
- All user passwords must be hashed using **Argon2id** (`argon2` npm package) with parameters calibrated for local development and production security (memory cost: 64MB, time cost: 3 iterations, parallelism: 1).
- Plain SHA-256 and unsalted MD5/bcrypt are prohibited.

## Consequences & Trade-offs
- **Option 1 (Memory + HttpOnly Refresh)**: Selected by project owner on 2026-11-04. Storing short-lived access tokens in memory limits token persistence. While the browser's cookie `Path=/api/v1/auth` transmits the cookie to requests matching that path prefix, the server consumes the refresh cookie for authentication strictly on `POST /api/v1/auth/refresh` and `POST /api/v1/auth/logout` (ignoring it on all other endpoints). This confines CSRF exposure to those two cookie-authenticated endpoints (where explicit Origin allowlisting, `SameSite` defense in depth, and anti-CSRF token verification are enforced), avoiding ambient-credential CSRF risks across the rest of the application. In-memory access tokens require a silent refresh flow on page reload, handled via TanStack Query and HTTP client interceptors.
- **Status**: Accepted on 2026-11-04. The project owner has explicitly selected Option 1 (In-Memory Access Token + HttpOnly Refresh Cookie) with all documented security controls (mandatory Origin/Referer allowlist validation, SameSite defense in depth, and anti-CSRF verification).
