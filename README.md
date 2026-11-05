# Multi-Repository Customer Management Demo

Enterprise architecture documentation grounded in the inspected implementations of
`customer-portal-frontend`, `customer-management-api`, and `shared-auth-platform`.

**Review baseline: 2026-11-05.** The frontend and API are independently implemented,
but their customer contracts are incompatible and browser CORS support is absent
from the API. The shared authentication repository is a placeholder, not a running
identity service. This is not yet a working three-service integration.

| Document | Contents |
| --- | --- |
| [System architecture](docs/architecture.md) | System diagram, repository responsibilities, communication, dependency map, source baseline |
| [REST API reference](docs/api-reference.md) | Every implemented endpoint, request/response examples, validation, errors, contract differences |
| [Authentication and authorization](docs/authentication.md) | Actual token issuance and verification, sequence diagram, trust boundaries, missing capabilities |
| [Local setup and operations](docs/local-setup.md) | Independent setup, integration prerequisites, smoke checks, troubleshooting |
| [Integration risks and change management](docs/integration-risks.md) | Current blockers, breaking-change analysis, proposed acceptance gates |

## Status terminology

- **Implemented** means present in the source at the recorded revision.
- **Gap** means required for the described integration but absent or incompatible.
- **Proposed** means a recommendation, not an implemented feature or committed roadmap.
- **Intended** means stated purpose only, without an executable implementation.

Source file paths are qualified by repository. Pinned source links and inspected
revisions are recorded in [the architecture baseline](docs/architecture.md#source-baseline).
Examples use fictional customer data and token placeholders, never real credentials.
