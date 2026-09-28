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

This repo carries collections/environments in **two parallel formats** — know which one your
Postman app is actually reading from before you edit either:

- **`collections/*.postman_collection.json` + `environments/*.postman_environment.json`** — the
  classic single-file format. Good for Newman/CLI runs and one-off manual imports. Editing these
  files does **not** update anything already imported into a Postman workspace on their own — see
  below.
- **`postman/` (plus the `.postman/resources.yaml` link file)** — Postman's "Postman for Git"
  filesystem format: one YAML file per request/folder. `.postman/resources.yaml`'s
  `cloudResources` section maps this directory to a specific **Postman Cloud collection UID**. If
  your Postman workspace is Git-linked to this repo (check Postman's workspace settings for a Git
  integration pointing here), **this is the format the app actually renders** — not the JSON
  files. Editing the JSON without also updating the matching files under `postman/` will look like
  nothing changed, even though you changed something real.

**If you're Git-linked:** edit both, or at least mirror any collection/environment change into
`postman/collections/microservice-akerbp-cbd/...` and `postman/environments/...`. Postman's
Desktop/web app picks up filesystem changes on its own sync (pull, or however your integration is
configured) — there's no manual "re-import" step, but there also isn't a fixed answer here for
every setup; check your workspace's Git integration settings if a change doesn't appear.

**If you're doing a one-off manual `collections/*.json` import** (no Git link, just Postman's
Import button) and re-importing after an update looks unchanged: Postman matches by the
collection's `_postman_id` embedded in the file, but only updates an *already-imported* collection
if you re-import into the exact same workspace it was originally imported into. If you're not
sure, delete the old imported collection from Postman first, then import fresh — that always
works.

1. Open Postman
2. Click **Import** (top left)
3. Select all files from `collections/` and `environments/`
4. Click **Import** (or delete-then-import if a previous import looks stale)

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

### Auto-refreshing `authToken`

The `microservice-akerbp-cbd` collection has a collection-level pre-request script that decodes
`authToken`'s JWT `exp` claim and, if it's missing or about to expire, requests a fresh one via
Keycloak's `password` grant (`grant_type=password`, `client_id=dsis-console` — the same pattern
`dwp-db-utility/dsis/index.js`'s `getToken()` already uses against this realm) before the request
fires. Requires three environment variables (localhost and preprod already have the keys, values
empty by default — fill them in locally, never commit real values):

| Variable | Value |
|---|---|
| `keycloakBaseUrl` | `https://dssecurity.dsif.dazlmkabpprd06.ienergycloud.solutions/auth` (preprod security server — local CBD validates against it too, since local `app-config.json` runs with `usePreProd: true`) |
| `keycloakClientId` | `dsis-console` |
| `dsisUsername` / `dsisPassword` | The **DWP Automation Test User** credentials (AkerBP LastPass) — a real DWP account meant for exactly this, so its DSIS unit-system preference can be changed in DWP itself to test different systems. Don't use your own personal account here. |

If those aren't set, the script logs a console warning and lets the request proceed as-is (it'll
401 naturally on an expired/missing token) — it never silently sends a request it knows will fail
for an unrelated reason, and never assumes a service-account identity has a personal unit
preference (it doesn't; only a real user account does).

Works the same way under Newman (`pm.sendRequest` in a pre-request script is fully supported).

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

Two top-level groups:

| Group | Folder | # | Method | Path |
|---|---|---|---|---|
| v2 (DSIS units) | — | 2 | GET | `/v2/pressure-curves/...`, `/v2/riser-margin/...` — DSIS-resolved units, see below |
| Legacy (frozen /msp API) | SOI | 5 | GET/POST/DELETE | `/soi/:wellId/:wellboreId/:designId`, `/soi/list/:wellId`, `/soi/import`, `/soi/save/...`, `/soi/delete/...` |
| Legacy (frozen /msp API) | Gas Lift | 3 | GET/POST | `/gaslift/:wellId/:wellboreId/:designId`, `/gaslift/list/:wellId`, `/gaslift/import` |
| Legacy (frozen /msp API) | CBD | 1 | POST | `/cbd/save/:wellId/:wellboreId/:designId` |
| Legacy (frozen /msp API) | PPFGSH | 1 | GET | `/ppfgsh/:wellId/:wellboreId/:designId` |
| Legacy (frozen /msp API) | Casings | 1 | GET | `/casings/:wellId/:wellboreId/:designId` |
| Legacy (frozen /msp API) | Riser Margin | 1 | GET | `/riser-margin/:wellId/:wellboreId/:designId` (legacy, frozen — bare path, no `/v2`) |
| Legacy (frozen /msp API) | Revisions | 1 | GET | `/revisions/names/:wellId/:wellboreId/:designId` |
| Legacy (frozen /msp API) | Pore Pressure | 1 | GET | `/pp/revision/:revisionId` |

Everything under "Legacy (frozen /msp API)" is untouched by the unit migration — same requests,
just nested one level deeper than before.

#### v2 (DSIS units)

Requests test the endpoints migrated off the legacy `depthUnit`/`pressureUnit`/`mudWeightUnit`
query params to server-side DSIS resolution (see `plans/plugin-unit-system-plan.md` in
`akerbp-dwp-web-components`, branch `abp350-plugins-unit-conversion`). Each has a **Tests** script:

- **GET Pressure Curves (v2, DSIS units)** — fully migrated. Asserts the response states its own
  `unitSystem` (`{id, name}`) and `units` (`{depth, pressure, mudWeight}`, each `{unit, precision}`),
  and that `curveData` values are numeric or `null` (never pre-formatted strings).
- **GET Riser Margin (v2, partially migrated)** — only asserts `edm.pp`/`edm.fg` are present
  (pressure-curves migrated internally) and that `edm.waterDepth`/`edm.riserLength` are still
  plain metric numbers (not yet migrated). Deliberately does **not** assert a top-level
  `unitSystem`/`units` block — this endpoint doesn't have one yet.

To see a different resolved system, change the **DWP Automation Test User**'s unit system in DWP
itself (there's no query param for it anymore — units are resolved server-side from the token's
identity) and re-run; the pre-request script will pick up a fresh token automatically if it expired.

## Running from the command line (Newman)

```bash
npx newman run collections/microservice-akerbp-cbd.postman_collection.json \
  -e environments/akerbp-preprod.postman_environment.json
```

Swap in `akerbp-localhost.postman_environment.json` for a local run. Newman runs the same
pre-request/test scripts as the Postman GUI, including the `authToken` auto-refresh.

## Proxy Context Paths (dev server only)

When running through `workbench-casing-design` dev server proxy, requests are rewritten:

| Service | Dev Server Context | Rewrites To |
|---|---|---|
| Casing Design | `/api/casing-design` | `/services/abp-quarkus-de-microservice-casing-design` |
| CBD | N/A (direct via nodemsp) | `/services/microservice-akerbp-cbd/msp` |
