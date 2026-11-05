# Integration risks and change management

## Current review disposition

**Independent demos are implemented; the integrated live customer workspace is
blocked.** There is no evidence of an implemented shared identity platform.
This document records architecture and integration risks, not a penetration-test
report or a commitment to implement the proposals.

All findings refer to the [source baseline](architecture.md#source-baseline).

## Risk register

Priority here expresses delivery impact: **Blocker** prevents the current live
integration; **High** requires explicit resolution before use beyond a controlled
demo; **Medium** requires coordinated operational or contract handling.

| ID | Priority | Observed condition and source | Impact | Proposed treatment |
| --- | --- | --- | --- | --- |
| INT-01 | Blocker | Frontend requires `status`, `createdAt`, and string `company`; API returns only id/name/email and optional company (`src/types/customer.ts`, `src/services/customers.ts` versus API `src/customers.js`) | Successful API reads are rejected by the UI | Agree a domain model and change every affected producer/consumer surface |
| INT-02 | Blocker | Creation form sends six fields; API allows only name/email/company (`src/pages/CreateCustomer.tsx` versus API `validateCustomer`) | UI creation returns 400 even with valid authentication | Align write model and response model together |
| INT-03 | Blocker | API has no CORS middleware/preflight path; Vite has no proxy (`src/app.js`, frontend `vite.config.ts`) | Default cross-origin browser requests fail before customer data can be used | Implement explicit CORS handling before auth or a deliberate same-origin deployment path |
| INT-04 | High | `shared-auth-platform` has only README and ignore file | An assumed central issuer or reusable auth package does not exist | Keep local JWT behavior explicit; define interfaces before shared-platform work |
| INT-05 | High | API data lives in `createCustomerStore()` memory | Restart loses writes; replicas disagree; uniqueness is not global | Keep demo-only scope or add persistent shared storage with migration and uniqueness rules |
| INT-06 | High | Verifier checks JWT authenticity but routes never authorize claims (`src/auth.js`, `src/app.js`) | Every accepted token can read/create every customer | Define roles/scopes, subject/tenant policy, and server-side checks before multi-user use |
| INT-07 | High | HS256 secret is shared by issuer and verifier; API does not require `exp` (`src/auth.js`, `scripts/create-token.js`) | Secret holders can mint tokens; externally minted nonexpiring tokens can be accepted | Constrain issuance, require expiry policy, and design rotation/provider integration |
| INT-08 | Medium | No refresh/revocation/logout subsystem; token is memory-only in frontend | Reload/expiry interrupts the workspace; no immediate per-token invalidation | Document manual renewal for demos; specify lifecycle for a future identity integration |
| INT-09 | Medium | Frontend tests mock responses; API tests use isolated instances | Both suites can pass despite schema/CORS incompatibility | Add cross-repository browser/API contract checks |
| INT-10 | Medium | Frontend maps errors by status, ignores backend error details (`src/services/api.ts`) | Validation and duplicate-email diagnostics become generic UI errors | Agree a safe error mapping, retaining structured field details where useful |
| INT-11 | Medium | Full collection fetched and aggregated in browser (`src/pages/Customers.tsx`, `src/pages/Dashboard.tsx`) | Larger datasets increase payload, memory, rendering, and data exposure | Define server-side query/pagination/reporting requirements before scale-up |
| INT-12 | Medium | No idempotency support; POST response can be lost (`src/app.js`, frontend `src/services/customers.ts`) | A retry may duplicate an attempted operation or receive a conflict | Keep automatic retries disabled; define recovery/idempotency if required |
| INT-13 | High | Local HTTP defaults; no production deployment or abuse-control configuration in trees | A local demo is not evidence of production readiness | Define HTTPS termination, secret delivery, origin policy, rate limiting, audit and monitoring requirements |

## Breaking-change analysis

These are **potential future breaking changes**, not an assertion that such
changes have already shipped.

| Change | Consumers affected | Compatibility consideration |
| --- | --- | --- |
| Rename `/api/customers`, change base prefix, or relocate `/health` | Frontend adapter, environment configuration, scripts, health checks | Coordinate base URL and path changes; avoid double `/api` prefixes |
| Change envelope or introduce paginated `data` objects | `unwrap` and array validation in frontend `src/services/customers.ts` | Current list consumer expects the complete array; partial results also invalidate dashboard totals and CSV semantics |
| Add required customer fields | API clients, fixtures, demo adapter, UI forms | Existing input may become invalid; define defaults/backfill only where business meaning is valid |
| Remove or rename `status` / `createdAt` / `company` | Frontend runtime validator, filters, charts, profile, CSV | These are currently assumed by UI behavior, not merely display metadata |
| Make company required server-side | Existing API clients that omit company | Currently valid POSTs would fail; coordinate validation and migration |
| Relax unknown-field rejection | Backend write contract | May hide client/API drift; deliberate compatibility policy is needed |
| Change ID type/format | Routes, stored links, detail lookup | Frontend accepts numbers or strings but existing backend IDs and comparisons are strings |
| Change issuer/audience/secret or move to asymmetric JWTs | Token helper, running API instances, all existing tokens | No multi-key transition exists; HS256-only verifier cannot accept new algorithms without code changes |
| Add roles/scopes/tenant checks | Every currently accepted token | Helper currently has no such claims; valid tokens may newly receive denials |
| Require expiration/max age | Tokens from issuers other than the helper | Currently optional `exp` becomes a mandatory claim; test rejection explicitly |
| Restrict allowed origins or move frontend hostname/port | Browser clients | Origin includes scheme, host, and port; curl success cannot prove browser compatibility |
| Introduce durable storage | API lifecycle and tests | Define migration, global email uniqueness, cleanup/reset behavior, and whether old demo IDs remain meaningful |
| Move auth into `shared-auth-platform` | API manifest/imports/configuration and release flow | Creates a package/runtime dependency that does not currently exist |
| Change Vite environment at deployment | Frontend release artifact | Public values are compiled into the bundle; rebuilding is required |
| Change HTTP errors or success statuses | Client status mapping and schema parsing | 204 is not a valid customer payload for the current adapter; no backend 403 path exists today |

## Proposed integration decision process

There is no recorded approved target design or implemented migration plan in the
inspected repositories. The following decisions should be made explicitly:

1. Choose whether the richer frontend model or the minimal API model is the
   authoritative customer contract. Include field meaning, optionality, dates,
   status lifecycle, identifiers, unknown fields, and error semantics.
2. Decide whether this remains a single-user ephemeral demo or needs durable,
   multi-user customer management. Do not imply production guarantees from the
   current implementation.
3. Select direct cross-origin API access versus an implemented same-origin
   proxy. Document the exact allowed origin and transport configuration.
4. Define the shared platform's role before adding it as a dependency: library,
   identity-provider integration, or service are materially different designs.

For the richer model, adding fields affects API storage, validation, samples,
responses, and tests as well as documentation. For the minimal model, removing
frontend requirements affects types, runtime guards, forms, demo fixtures,
filters, chart calculations, profiles, and exports. Fabricating creation dates
or customer statuses to bypass the validator would conceal the mismatch.

## Proposed release acceptance checklist

- Record reviewed revisions and the agreed request/response contract together.
- Verify public health, authenticated list/detail/create, and exact envelopes.
- Verify missing/expired/wrong-issuer/wrong-audience/wrong-algorithm tokens are
  rejected; add role/tenant/expiry-presence tests if those policies are introduced.
- Verify frontend-origin preflight succeeds without requiring a token and that
  unauthorized origins are not allowed.
- Load list, profile, dashboard, and creation in a browser against the real API;
  assert no schema errors and that newly created records appear after navigation.
- Verify malformed JSON, unknown fields, duplicate email, body limits, and error
  presentation without exposing sensitive server internals.
- Confirm demo mode makes no API calls and never mixes demo data with live data.
- Verify reload/expiry behavior, API restart data loss, and uncertain-write
  recovery match the documented scope.
- Run existing repository checks and new cross-repository checks; mocked unit
  responses alone cannot satisfy acceptance.
- Coordinate rollout/rollback across affected repositories. Record which
  frontend build matches which API contract and JWT configuration. A rollback
  cannot restore lost in-memory writes; secret rollback may re-enable old tokens.

These checks are recommendations for subsequent implementation, not completed
integration validation at this baseline.
