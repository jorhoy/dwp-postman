# dwp-postman

Postman collections and environments for AkerBP DWP microservices.

## Contents

```
collections/
  microservice-casing-design.postman_collection.json   # 13 endpoints (Quarkus/Java)
  microservice-akerbp-cbd.postman_collection.json      # 14 endpoints (Hapi.js/Node.js)

environments/
  akerbp-localhost.postman_environment.json            # localhost direct access
  akerbp-tiger.postman_environment.json                # Tiger dev cluster
  akerbp-preprod.postman_environment.json              # Pre-prod cluster
```

## Importing into Postman

1. Open Postman
2. Click **Import** (top left)
3. Select all files from `collections/` and `environments/`
4. Click **Import**

## Environment Setup

Each environment has two service-specific base URLs (since each microservice runs as a separate container):

| Variable | Purpose |
|---|---|
| `casingDesignBaseUrl` | Base URL for `microservice-casing-design` |
| `cbdBaseUrl` | Base URL for `microservice-akerbp-cbd` (includes `/msp` prefix) |
| `authToken` | Bearer token for authentication |
| `wellId` | Well identifier (10-char format, e.g. `NO 15/9-F-5`) |
| `wellboreId` | Wellbore identifier |
| `designId` | Design identifier (5-char format) |
| `holeSectionId` | Hole section identifier |
| `reportId` | Report identifier (for Charts endpoint) |
| `revisionId` | Revision identifier (for Pore Pressure endpoint) |

### Localhost

Runs each service directly on port 8080 (run one at a time, or on different ports):

| Variable | Value |
|---|---|
| `casingDesignBaseUrl` | `http://localhost:8080` |
| `cbdBaseUrl` | `http://localhost:8080/msp` |

### Tiger

OEC dev cluster at `tiger.dazlmkengdev02.ienergycloud.solutions`:

| Variable | Value |
|---|---|
| `casingDesignBaseUrl` | `https://tiger.dazlmkengdev02.ienergycloud.solutions/services/abp-quarkus-de-microservice-casing-design` |
| `cbdBaseUrl` | `https://tiger.dazlmkengdev02.ienergycloud.solutions/services/microservice-akerbp-cbd/msp` |

### Pre-prod

OEC pre-prod cluster at `dsif.dazlmkabpprd06.ienergycloud.solutions`:

| Variable | Value |
|---|---|
| `casingDesignBaseUrl` | `https://dsif.dazlmkabpprd06.ienergycloud.solutions/services/abp-quarkus-de-microservice-casing-design` |
| `cbdBaseUrl` | `https://dsif.dazlmkabpprd06.ienergycloud.solutions/services/microservice-akerbp-cbd/msp` |

## Authentication

- **microservice-casing-design**: Bearer token auth set at collection level — set `authToken` in your environment. Health endpoints (`/msp/home/*`) have auth disabled.
- **microservice-akerbp-cbd**: `Authorization: Bearer {{authToken}}` header on each request (validated by Joi).

To get a token, authenticate against the Keycloak server at:
- Tiger: `https://tiger.dazlmkengdev02.ienergycloud.solutions`
- Pre-prod: `https://dssecurity.dsif.dazlmkabpprd06.ienergycloud.solutions/auth`

## Collections

### microservice-casing-design (Quarkus/Java, port 8080)

| Folder | # | Method | Path |
|---|---|---|---|
| Loads | 2 | GET/POST | `/api/v1/loads/standardLoads/:wellId/:wellboreId/:designId/:holeSectionId` |
| Scenario | 2 | GET | `/api/v1/scenario/wells/:wellId/wellbores/:wellboreId/designs[WithDetails]` |
| PPFG | 1 | GET | `/api/v1/ppfg/wells/:wellId/wellbores/:wellboreId/designs/:designId/has-pore-pressure` |
| Charts | 1 | GET | `/api/v1/charts/:reportId/:wellId/:wellboreId/:designId` |
| Lithology | 1 | GET | `/api/v1/lithology/wells/:wellId/wellbores/:wellboreId/designs/:designId/formations` |
| User | 1 | GET | `/me` |
| Health | 5 | GET | `/msp/home/`, `/msp/home/health`, `/msp/home/readiness`, `/msp/home/codecoverage`, `/msp/home/build` |

### microservice-akerbp-cbd (Hapi.js / nodemsp, port 8080)

CBD = Critical Barrier Depth. All paths are relative to `{{cbdBaseUrl}}` which already includes `/msp`.

| Folder | # | Method | Path |
|---|---|---|---|
| SOI | 5 | GET/POST/DELETE | `/soi/:wellId/:wellboreId/:designId`, `/soi/list/:wellId`, `/soi/import`, `/soi/save/...`, `/soi/delete/...` |
| Gas Lift | 3 | GET/POST | `/gaslift/:wellId/:wellboreId/:designId`, `/gaslift/list/:wellId`, `/gaslift/import` |
| CBD | 1 | POST | `/cbd/save/:wellId/:wellboreId/:designId` |
| PPFGSH | 1 | GET | `/ppfgsh/:wellId/:wellboreId/:designId` |
| Casings | 1 | GET | `/casings/:wellId/:wellboreId/:designId` |
| Riser Margin | 1 | GET | `/riser-margin/:wellId/:wellboreId/:designId` |
| Revisions | 1 | GET | `/revisions/names/:wellId/:wellboreId/:designId` |
| Pore Pressure | 1 | GET | `/pp/revision/:revisionId` |

## Proxy Context Paths (dev server only)

When running through `workbench-casing-design` dev server proxy, requests are rewritten:

| Service | Dev Server Context | Rewrites To |
|---|---|---|
| Casing Design | `/api/casing-design` | `/services/abp-quarkus-de-microservice-casing-design` |
| CBD | N/A (direct via nodemsp) | `/services/microservice-akerbp-cbd/msp` |
