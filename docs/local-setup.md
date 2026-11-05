# Local setup, integration, and troubleshooting

## Prerequisites and scope

Use Node.js **22.12+ on the Node 22 line, or a newer supported LTS release**, npm,
Git, and curl. The frontend README requires 22.12+; the backend manifest requires
22.9+. Use one version satisfying both. GitHub repository access is also required.

Run each application's commands in that repository's directory. No Docker
Compose, database, shared-auth server, or repository-wide startup script exists.
These instructions run the two applications independently; the integration gaps
below still prevent the default live browser flow from working.

For a reproducible architecture baseline, clone and select these revisions in
a parent working directory:

```bash
git clone https://github.com/enterprise-ai-engineering-demo/customer-management-api.git
git -C customer-management-api checkout --detach bd38112fe5726d24a0a5e507dea5a1eea7dc371a

git clone https://github.com/enterprise-ai-engineering-demo/customer-portal-frontend.git
git -C customer-portal-frontend checkout --detach 2509fb2739db00ce3b287e42b6ad98bbbd171690
```

A detached checkout is for reproduction; create a branch before implementing
changes. `shared-auth-platform` need not be cloned to run either application,
because neither consumes it.

## Start the API

In the `customer-management-api` directory:

```bash
npm ci
cp .env.example .env
node -e "console.log(require('node:crypto').randomBytes(32).toString('hex'))"
```

Paste the generated random value into `JWT_SECRET` in `.env`, keeping it local.
Do not overwrite a previously configured `.env` unnecessarily.

| Variable | Default | Requirements |
| --- | --- | --- |
| `PORT` | `4000` | Integer from 1 to 65535 |
| `JWT_SECRET` | None | Required, at least 32 bytes |
| `JWT_ISSUER` | `customer-management-demo` | Nonempty string |
| `JWT_AUDIENCE` | `customer-management-api` | Nonempty string |

`npm start` and `npm run token` use Node's `--env-file-if-exists=.env`; existing
process environment variables take precedence. `src/config.js` fails fast on
invalid configuration.

```bash
npm start
```

Expected log: `Customer Management API listening on port 4000` with the default
port. In a second terminal in the same repository:

```bash
curl -i http://localhost:4000/health
export TOKEN="$(npm run --silent token)"
curl -i http://localhost:4000/api/customers \
  -H "Authorization: Bearer $TOKEN"
```

Expect 200 for both requests. Token generation uses the same server configuration;
the token lasts one hour. The variable is local to this shell. If the running
server has different exported JWT variables, the helper's token will not match.

For browser entry, copy the token locally from the helper's output; paste only
the JWT, not the `Bearer` prefix, into API settings. Do not publish the token.
Remove `TOKEN` from the shell when finished:

```bash
unset TOKEN
```

## Start the frontend

In a separate terminal in `customer-portal-frontend`:

```bash
npm ci
cp .env.example .env
npm run dev
```

| Variable | Default | Meaning |
| --- | --- | --- |
| `VITE_API_BASE_URL` | `http://localhost:4000/api` | Public build-time URL, not a secret |
| `VITE_DEMO_MODE` | `false` in the example | Only the exact value `true` selects demo mode initially |

Open the URL Vite reports, normally `http://localhost:5173`. Vite may choose a
different port if that port is occupied; the origin must match any future CORS
allowlist. Restart Vite after changing `.env`. For a deployment, rebuild after
changing `VITE_*`; modifying a runtime environment after build does not rewrite
the bundle.

For a working standalone presentation, choose **Explore with demo data** from
the error state or **API settings > Demo workspace**, or set
`VITE_DEMO_MODE=true` before starting. The demo adapter requires no token or API
server. It has eight initial records. Demo writes survive navigation and mode
switching within the mounted provider but disappear on refresh.

To exercise the live connection attempt, choose **API settings > Live API**,
paste a demo JWT without `Bearer`, and save. This does not resolve CORS or schema
incompatibilities. A valid token is necessary but insufficient for integration.
Refreshing clears the token and re-applies the environment's initial mode.

## Integration instructions and acceptance gates

The following are **required future changes**, not existing configuration
switches. Implement them in the relevant application repositories, not here.

1. **Agree on the customer contract.** Either extend the backend model,
   validation, storage, and responses to support the frontend's business fields,
   or revise the frontend types, forms, runtime validators, filters, charts, and
   CSV export to use the smaller backend model. Make `company` semantics explicit.
   Changing just a TypeScript interface does not change runtime validation.
2. **Enable a browser transport path.** Implement API CORS handling for the exact
   frontend origin, including unauthenticated OPTIONS preflight before customer
   authentication. Permit `GET`, `POST`, `OPTIONS`, and the `Authorization` and
   `Content-Type` headers. Alternatively, implement a same-origin reverse proxy
   and configure the frontend base URL accordingly. Neither exists in the
   current `src/app.js` or `vite.config.ts`; no `CORS_ORIGIN` variable is defined.
3. **Align JWT configuration.** Use the helper with the running API's secret,
   issuer, and audience; never put the secret in the frontend. A future shared
   platform must meet an explicitly agreed issuance/verification contract.
4. **Verify independently, then end-to-end.** Check public health, authenticated
   list/detail/create, negative authentication, browser preflight, and response
   schema validation. Verify actual list, profile, and creation UI against the
   real API, not only mocked Fetch responses.

Useful current diagnostics:

```bash
# Must be public and return 200.
curl -i http://localhost:4000/health

# Must return 401 without a token.
curl -i http://localhost:4000/api/customers

# Shows the current preflight gap; it is not expected to succeed at this baseline.
curl -i -X OPTIONS http://localhost:4000/api/customers \
  -H 'Origin: http://localhost:5173' \
  -H 'Access-Control-Request-Method: GET' \
  -H 'Access-Control-Request-Headers: authorization'
```

After CORS is implemented, verify an allowed origin receives appropriate
`Access-Control-Allow-Origin`, allowed methods, and allowed headers, and that
disallowed origins are not authorized. Curl reports headers but does not enforce
browser CORS; repeat the real request in the browser's Network panel.

For authenticated GET, detail, and POST examples and their exact expected payloads,
use the [API reference](api-reference.md). On a fresh API process, a successful
create changes the collection from two records to three; a restart restores two.

## Existing validation commands

| Repository | Commands | Scope |
| --- | --- | --- |
| `customer-management-api` | `npm test` | Node test runner and Supertest; app instances and test-only signing config; no `.env` or running server needed |
| `customer-portal-frontend` | `npm test`, `npm run typecheck`, `npm run lint`, `npm run build` | Mocked API/UI tests, TypeScript, ESLint, production bundle |
| `shared-auth-platform` | None | No package or tests |

The existing tests do not start both repositories together. Passing them
independently is not an end-to-end compatibility result.

The frontend build produces `dist/`. Production hosting needs an SPA fallback to
`index.html` for paths such as `/customers/:id`. `npm run preview` is a local
preview, not a production hosting solution. No deployment manifests are supplied
by the inspected repositories.

## Troubleshooting guide

| Symptom | Evidence to inspect | Explanation / action |
| --- | --- | --- |
| API exits before listening | Startup exception; `src/config.js`, `src/auth.js` | Set a secret of at least 32 bytes, nonempty issuer/audience, and a valid port; check exported environment overrides |
| Listener cannot bind | `src/server.js` error output | Resolve occupied port or choose `PORT`; update the frontend API URL and restart Vite if port changes |
| Health succeeds but customers return 401 | Response status and `WWW-Authenticate`; helper configuration | Supply a token, without duplicate `Bearer` prefix; regenerate after expiry or configuration changes |
| Fresh token still returns 401 | Server environment versus helper environment | Compare secret/issuer/audience privately; verify HS256 and system clock; do not paste secrets into tickets |
| Browser says "Cannot reach the API" but curl works | Browser preflight and response headers | Current API lacks CORS. Implement the integration prerequisite; do not disable browser security |
| Browser cannot connect at all | Vite URL, API listener, configured base URL | Confirm both processes run; check host, port, HTTP/HTTPS, and mixed-content blocking |
| UI shows unexpected customer list/customer format | Raw 200 JSON; `src/services/customers.ts` | Current API lacks `status` and `createdAt`; `company` can also be absent. Fix the contract, not the token |
| Creating from UI returns a generic 400 | API error body in local Network panel | `phone`, `industry`, `status` are rejected; the frontend does not show backend `details` |
| Direct POST returns 400 | `error.code` and `details` | Use a JSON object, valid fields, nonblank strings, proper email; omit company rather than sending `""` or `null` |
| Direct POST returns 409 | `CUSTOMER_EXISTS`; current customer list | Email is normalized and unique within this app; check for an earlier successful request before retrying |
| POST returns 413 or 415 | Error envelope; content headers/body | Stay within the `16kb` JSON limit and use supported encoding; do not send oversized test data in normal demos |
| Customer detail returns 404 | API list and requested ID | Use an ID from the same API instance; demo IDs and API IDs are unrelated; restart discards created IDs |
| Customers disappear or differ between requests | Mode label and API process topology | Demo is tab-local; backend is process-local; multiple backend instances diverge without shared persistence |
| Token vanishes on refresh | API settings | Expected memory-only design; generate/paste a valid token again |
| Demo works but live does not | Connection mode and browser Network panel | Demo uses no HTTP; it cannot prove CORS, JWT, or API contract compatibility |
| New `.env` URL seems ignored | API endpoint shown in settings | Restart dev server or rebuild deployed frontend; the endpoint field is read-only |
| Deep link works in dev but host returns 404 | Hosting fallback configuration | Serve SPA paths through `index.html`; do not route API requests through the SPA fallback |
| Shared auth cannot be started | Repository tree | No implementation or start script exists; use the API's local demo utility |
| POST fails after a network interruption | List by email using an authenticated request | Server may have committed the in-memory write; investigate before manually retrying |

For unexpected 500 errors, inspect the API's `Unhandled request error:` log
locally. Record method, path, status, error code, and reproduction steps; redact
authorization headers and customer information. No correlation-ID mechanism or
centralized observability configuration is implemented.
