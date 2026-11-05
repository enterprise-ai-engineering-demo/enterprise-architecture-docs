# Authentication and authorization

## Implemented trust model

Authentication is implemented **inside `customer-management-api`**, not by
`shared-auth-platform`. `scripts/create-token.js` is a local token utility;
`src/auth.js` verifies tokens within the API process. There is no network call
to an auth service, login UI, login endpoint, or shared auth package dependency.

The frontend accepts a manually supplied JWT through
`customer-portal-frontend/src/components/ConnectionDialog.tsx`. It holds the token
in React state in `src/state/PortalContext.tsx`, sends it only in live API mode,
and does not verify, decode, renew, or exchange it. It is not stored in
localStorage, sessionStorage, a URL, or a cookie and is cleared by page refresh.
Switching to demo mode does not itself clear the in-memory token; emptying the
token field and saving removes it.

## Actual authentication sequence

The diagram distinguishes the working CLI/API path from the blocked default
browser path. The final customer response is the backend schema, not a promise
of frontend compatibility.

```mermaid
sequenceDiagram
    actor Dev as Developer
    participant CLI as scripts/create-token.js
    participant Config as Server JWT configuration
    participant UI as Frontend React memory
    participant API as Express API
    participant Auth as src/auth.js
    participant Store as In-memory customer store

    Dev->>CLI: npm run --silent token
    CLI->>Config: loadConfig()
    Config-->>CLI: secret, issuer, audience
    CLI->>CLI: HS256 sign, sub demo-user, expiresIn 1h
    CLI-->>Dev: JWT on stdout
    Dev->>UI: Paste JWT in API settings, choose Live API
    UI->>UI: Trim and retain token in memory
    UI->>API: Browser OPTIONS preflight for Authorization header
    API->>Auth: Customer-router authentication runs
    Auth-->>API: No bearer token on preflight
    API-->>UI: 401, no configured CORS allow headers
    Note over UI,API: Default cross-origin browser request is blocked

    Dev->>API: curl GET /api/customers with Bearer JWT
    API->>Auth: Authenticate before JSON parsing/route handler
    Auth->>Config: Use auth configuration captured at startup
    Auth->>Auth: Verify signature, HS256, issuer, audience, time claims
    alt Missing or invalid authentication
        Auth-->>API: ApiError 401 and WWW-Authenticate header
        API-->>Dev: JSON error envelope
    else Valid authentication
        Auth->>API: req.auth = verified claims; next()
        API->>Store: list()
        Store-->>API: Backend customer objects
        API-->>Dev: 200 with data envelope
    end
    Note over UI,API: If CORS is fixed, current list/detail data still fails frontend schema checks
```

No participant represents a running shared identity platform because none exists.

## Issuance and validation rules

| Concern | Implemented behavior | Evidence |
| --- | --- | --- |
| Signing | HS256 symmetric signing | `scripts/create-token.js` |
| Secret | Required string, at least 32 bytes measured with `Buffer.byteLength` | `src/auth.js: validateAuthConfig` |
| Issuer | Default `customer-management-demo`; must be nonempty | `src/auth.js`, `src/config.js` |
| Audience | Default `customer-management-api`; must be nonempty | `src/auth.js`, `src/config.js` |
| Subject | Utility sets `sub` to `demo-user` | `scripts/create-token.js` |
| Lifetime | Utility sets `expiresIn: '1h'`; library adds time claims | `scripts/create-token.js` |
| API verification | `jwt.verify` with `algorithms: ['HS256']`, issuer, audience | `src/auth.js: createAuthMiddleware` |
| Time claims | Expiration and not-before checked when present | `jsonwebtoken` verification and `test/api.test.js` |
| Request context | Verified claims assigned to `req.auth` | `src/auth.js` |
| Unauthorized response | 401 with `UNAUTHORIZED` JSON envelope and `WWW-Authenticate` header | `src/auth.js`, `src/errors.js` |

The middleware does **not require** an `exp` claim or enforce a maximum token age.
It does not require the subject to equal `demo-user`. It does not check roles,
scopes, tenant claims, or ownership. The utility's one-hour lifetime is therefore
an issuance convention, not a universally enforced expiry requirement.

Decoding a JWT alone is not verification. The API verifies the signature and
configured constraints; the browser's possession of a string is not proof of an
authenticated identity until the API accepts it.

## Authorization boundary

Every request handled under `/api/customers` passes through the same authentication
middleware. Once a token is accepted, the list, detail, and create operations are
equally accessible; no additional authorization predicate examines `req.auth`.

`GET /health` is public. There are no administrator-only routes, 403 role-denial
paths, per-user customer filters, tenant boundaries, or revocation checks.
Frontend labels and status filters do not enforce security. Hiding UI controls
would not protect the API.

## Secret handling and deployment implications

Keep `JWT_SECRET` server-side in the API environment; do not copy it into a
frontend `VITE_*` variable. Vite configuration is public build-time content.
The same symmetric secret can both mint and verify tokens, so every holder is a
trusted issuer. Issuer/audience matching does not remove that signing capability.

Use the local utility only for this demo. Avoid sharing real tokens, persisting
them in source, recording them in screenshots, or enabling verbose request logs
that expose authorization headers. Use HTTPS for deployed frontend and API
traffic; localhost HTTP defaults are development-only.

Changing the configured secret, issuer, or audience requires issuing new matching
tokens. There is no implemented multi-key rotation window, `kid` resolver,
revocation list, or automatic refresh. Restarting with a new secret invalidates
tokens signed by the old one.

## Unimplemented capabilities and proposals

| Capability | Baseline status | Proposed direction, not implemented |
| --- | --- | --- |
| Shared authentication utilities | README-only intent | Define package exports, versioning, consumers, and tests before extraction |
| User login / SSO | Absent | Choose an identity provider and browser sign-in flow |
| JWKS / asymmetric validation | Absent; current verifier is HS256-only | Replace verifier deliberately and coordinate issuer/audience/key distribution |
| Expiry required for every token | Absent | Add explicit expiry/max-age policy and negative tests |
| RBAC / scopes / tenants | Absent | Define claim semantics and server-side authorization checks |
| Refresh, logout, revocation | Absent | Specify lifecycle and failure behavior with the chosen provider |
| Abuse controls / security audit trail | No rate-limiting or audit subsystem | Define operational requirements before non-demo use |

These are not documented roadmap commitments. The inspected repository contains
no implemented identity-provider protocol or shared-platform API to configure.
