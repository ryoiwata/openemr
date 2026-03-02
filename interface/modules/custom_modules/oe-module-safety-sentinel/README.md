# oe-module-safety-sentinel

OpenEMR custom module that integrates the Safety Sentinel AI agent into the patient chart. It adds a **Safety Check** tab to the patient navigation menu and exposes the REST API endpoints that the Safety Sentinel FastAPI service calls to read and write clinical data.

## Overview

Safety Sentinel is an AI-powered clinical decision support tool built on a separate FastAPI/LangGraph service. This module is the **OpenEMR side** of that integration: it registers the chart tab, provides the PHP data layer (services + REST controllers), and stores persistent module data in three custom tables.

```
OpenEMR patient chart
  └── Safety Check tab  (public/index.php)
        └── iframe → Safety Sentinel FastAPI service (port 8001)
              └── calls back to OpenEMR REST API
                    └── /api/safety-sentinel/*  (this module)
```

## Features

| Feature | What it does |
|---|---|
| **Safety Check tab** | Embeds the Safety Sentinel UI in the patient chart via an iframe pre-loaded with the patient's UUID and name |
| **Audit log** | Persists every AI safety check result (severity, allergy conflicts, formulary status, pharmacist review flags) |
| **Pharmacist review queue** | Surfaces pending high-severity checks for pharmacist acknowledgement |
| **Conversation history** | Stores multi-turn LangGraph message history so conversations survive page reloads and server restarts |
| **Scribe encounters** | Saves ambient AI scribe encounters (transcript, SOAP note JSON, accepted ICD-10/CPT codes) |
| **Billing integration** | Writes clinician-accepted ICD-10 and CPT codes to OpenEMR's native `billing` table |
| **Draft prescriptions** | Creates draft prescription proposals in `prescriptions` for clinician signature |

## Requirements

- OpenEMR 7.0.2+
- PHP 8.2+
- Safety Sentinel FastAPI service running and reachable (default: `http://localhost:8001`)
- The `safety_sentinel_url` global set in OpenEMR's `globals` table (see Configuration)

## Installation

### 1. Place the module

The module directory must live at:

```
{openemr_root}/interface/modules/custom_modules/oe-module-safety-sentinel/
```

In the Docker development environment this is already volume-mounted from the repo. For production, copy or symlink the directory into place.

### 2. Run the SQL migrations

Execute the migration files in order against the OpenEMR database:

```bash
# Docker dev
docker exec -i development-easy-mysql-1 mariadb -u openemr -popenemr openemr \
  < sql/005_create_scribe_encounters.sql

docker exec -i development-easy-mysql-1 mariadb -u openemr -popenemr openemr \
  < sql/006_create_billing_service.sql
```

The migrations create these tables (all use `InnoDB`, `utf8mb4`):

| Table | Purpose |
|---|---|
| `safety_audit_log` | One row per AI safety check — severity, allergy flags, formulary, pharmacist review status |
| `safety_conversations` | One row per LangChain message — multi-turn conversation history keyed by `conversation_id` |
| `scribe_encounters` | One row per AI scribe encounter — transcript, SOAP JSON, accepted billing codes |

> **Note:** The module intentionally avoids foreign keys to OpenEMR's native tables. Patient identity is stored as a UUID string (`patient_uuid`) to remain stable across OpenEMR version upgrades.

### 3. Register the module

Insert the module registration row if it is not already present:

```sql
INSERT INTO modules
    (mod_name, mod_directory, mod_active, mod_ui_name, mod_relative_link,
     mod_ui_order, mod_ui_active, mod_enc_menu, directory, date, sql_run, type)
VALUES
    ('oe-module-safety-sentinel', 'oe-module-safety-sentinel', 1,
     'Safety Sentinel', 'index.php', 0, 0, 'no',
     '/var/www/localhost/htdocs/openemr/interface/modules/custom_modules/oe-module-safety-sentinel',
     NOW(), 1, 0);
```

### 4. Configure the backend URL

Set the Safety Sentinel service URL in the OpenEMR globals table:

```sql
-- Production droplet
UPDATE globals SET gl_value='http://161.35.61.60:8001'
WHERE gl_name='safety_sentinel_url';

-- Or insert if missing
INSERT INTO globals (gl_name, gl_value) VALUES ('safety_sentinel_url', 'http://localhost:8001');
```

The `public/index.php` entry point falls back to `http://localhost:8001` if this global is not set.

### 5. Grant OAuth2 scopes

The Safety Sentinel FastAPI service authenticates against OpenEMR using the OAuth2 password grant. The client must have all required scopes:

```sql
UPDATE oauth_clients
SET scope = 'openid offline_access api:oemr
  user/patient.read user/allergy.read user/medication.read user/medical_problem.read
  user/audit-log.r user/audit-log.c
  user/conversations.r user/messages.s user/messages.c user/messages.d
  user/billing.c user/billing.r user/prescriptions.c'
WHERE client_id = '<your_client_id>';
```

## Module Structure

```
oe-module-safety-sentinel/
├── openemr.bootstrap.php          # Registers the patient menu tab and REST routes
├── _rest_routes.inc.php           # Route definitions loaded via RestApiCreateEvent
├── public/
│   └── index.php                  # Entry point: resolves patient UUID, renders iframe
├── src/
│   ├── RestControllers/
│   │   ├── AuditLogRestController.php
│   │   ├── BillingRestController.php
│   │   ├── ConversationRestController.php
│   │   ├── PrescriptionDraftRestController.php
│   │   └── ScribeRestController.php
│   └── Services/
│       ├── AuditLogService.php
│       ├── BillingService.php
│       ├── ConversationService.php
│       ├── PrescriptionDraftService.php
│       └── ScribeEncounterService.php
├── sql/
│   ├── 005_create_scribe_encounters.sql
│   └── 006_create_billing_service.sql
├── info.txt
├── ModuleManagerListener.php
└── version.php                    # v1.0.0
```

**Namespace:** `OpenEMR\Modules\SafetySentinel\`
**Autoloader:** PSR-4 registered in `openemr.bootstrap.php`, mapping `OpenEMR\Modules\SafetySentinel\*` → `src/`

## REST API Reference

All routes are registered under `/api/safety-sentinel/` and protected by OpenEMR's standard OAuth2 middleware. Authenticate with a bearer token obtained from `/oauth2/default/token`.

### Audit Log

| Method | Path | Scope | Description |
|---|---|---|---|
| `GET` | `/api/safety-sentinel/audit-log/:puuid` | `user/audit-log.r` | Get safety check history for a patient |
| `GET` | `/api/safety-sentinel/audit-log/pending-review` | `user/audit-log.r` | List checks pending pharmacist review |
| `POST` | `/api/safety-sentinel/audit-log` | `user/audit-log.c` | Create a new audit log entry |
| `PUT` | `/api/safety-sentinel/audit-log/:id/acknowledge` | `user/audit-log.c` | Pharmacist acknowledges a flagged check |
| `GET` | `/api/safety-sentinel/health` | `user/audit-log.r` | Module health check |

**POST body fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `patient_uuid` | string | Yes | Patient UUID |
| `drug_name` | string | Yes | Proposed drug name |
| `interaction_severity` | string | Yes | `safe`, `minor`, `moderate`, `major`, or `contraindicated` |
| `drug_rxnorm` | string | No | RxNorm concept ID |
| `insurance_plan` | string | No | Formulary plan key |
| `allergy_conflict` | bool | No | Whether an allergy conflict was detected |
| `requires_pharmacist_review` | bool | No | Whether pharmacist review is needed |
| `confidence_score` | float | No | AI confidence 0.0–1.0 |
| `agent_summary` | string | No | First 2000 chars of the AI narrative |
| `formulary_covered` | bool | No | Whether the drug is on formulary |
| `formulary_tier` | int | No | Formulary tier (1–5) |
| `prior_auth_required` | bool | No | Whether prior authorisation is required |
| `covered_alternative` | string | No | Covered alternative drug name |

### Conversation History

| Method | Path | Scope | Description |
|---|---|---|---|
| `GET` | `/api/safety-sentinel/conversations/:puuid` | `user/conversations.r` | List conversations for a patient |
| `GET` | `/api/safety-sentinel/conversations/:puuid/messages?conv_id=` | `user/messages.s` | Get all messages for a conversation |
| `POST` | `/api/safety-sentinel/conversations/:puuid/messages` | `user/messages.c` | Save (replace) messages for a conversation |
| `DELETE` | `/api/safety-sentinel/conversations/:puuid/messages?conv_id=` | `user/messages.d` | Delete a conversation |

> **Route design note:** Routes use a `/messages` static suffix to avoid OpenEMR's two-parameter scope parsing bug. The router only pops one trailing `:param` when extracting the resource name; routes ending with consecutive params would produce an invalid scope string.

### Scribe Encounters

| Method | Path | Scope | Description |
|---|---|---|---|
| `GET` | `/api/safety-sentinel/scribe-encounters/:puuid` | `user/scribe-encounters.r` | List encounters for a patient |
| `POST` | `/api/safety-sentinel/scribe-encounters` | `user/scribe-encounters.c` | Save a new encounter (draft or finalized) |
| `PUT` | `/api/safety-sentinel/scribe-encounters/:id` | `user/scribe-encounters.u` | Update an encounter (draft only) |
| `DELETE` | `/api/safety-sentinel/scribe-encounters/:id` | `user/scribe-encounters.d` | Delete a draft encounter |

Finalized encounters are immutable — `PUT` and `DELETE` return a validation error if the encounter status is `finalized`.

### Billing & Prescriptions

| Method | Path | Scope | Description |
|---|---|---|---|
| `POST` | `/api/safety-sentinel/billing` | `user/billing.c` | Write ICD-10 and CPT codes to OpenEMR `billing` table |
| `POST` | `/api/safety-sentinel/prescriptions/draft` | `user/prescriptions.c` | Create draft proposals in OpenEMR `prescriptions` |

The billing endpoint is idempotent — it rejects a second submission for an encounter that already has billing entries (`activity=1`).

## How the Tab Works

`public/index.php` runs inside the OpenEMR patient chart frame. It:

1. Reads `$_SESSION['pid']` (set by OpenEMR when a chart is opened)
2. Queries `patient_data` to convert the numeric `pid` to a UUID (stored as binary, converted via `HEX()`)
3. Builds an iframe URL pointing at the Safety Sentinel FastAPI frontend, passing `patient_id` and `patient_name` as query parameters
4. Renders a full-height borderless iframe with `allow="microphone"` so the ambient scribe can access the microphone

If no patient is selected, or if the UUID lookup fails, an appropriate error message is shown instead of the iframe.

## Development

### Copying changes into Docker

The Docker development environment does **not** volume-mount custom module PHP files. After modifying any PHP file, sync it into the running container:

```bash
docker cp interface/modules/custom_modules/oe-module-safety-sentinel/src/Services/AuditLogService.php \
  development-easy-openemr-1:/var/www/localhost/htdocs/openemr/interface/modules/custom_modules/oe-module-safety-sentinel/src/Services/AuditLogService.php
```

### Running the full stack locally

```bash
# Start OpenEMR + MySQL
cd docker/development-easy
docker compose up --detach --wait

# Start Safety Sentinel FastAPI service
cd agents/safety-sentinel
uvicorn src.api.main:app --reload --port 8001
```

Access the module at: **http://localhost:8300** → select a patient → Safety Check tab

### Checking PHP error logs

```bash
docker compose exec openemr /root/devtools php-log
```

## License

GNU General Public License 3 — same as OpenEMR.
See [LICENSE](https://github.com/openemr/openemr/blob/master/LICENSE).
