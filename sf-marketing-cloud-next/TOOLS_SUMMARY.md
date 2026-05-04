# Salesforce Marketing Cloud Next MCP Tools: Summary & Tribal Knowledge

These tools target **Salesforce Marketing Cloud Next** (the Data Cloud-
native, Core-based marketing platform — distinct from Marketing
Cloud Engagement / ExactTarget which lives in `sf-marketing-cloud-engagement/`).

MC Next runs on **Salesforce Core + Data Cloud**, so most of its API
surface is the standard Salesforce platform. Per the official guidance
(["Which API Should I Use?"](https://developer.salesforce.com/docs/marketing/marketing-cloud-growth/guide/mc-connect-apis-info.html))
you'll touch six APIs:

| API                    | Where it lives in this repo                                          |
|------------------------|----------------------------------------------------------------------|
| **REST API** (sObjects, SOQL/SOSL, Composite, GraphQL) | `sf-core/` — *not duplicated here*. Use `sf-core/read/query-search`, `sf-core/write/data-ops`, etc. |
| **Bulk API 2.0**        | `sf-core/` — *not duplicated here*. Use `sf-core/read/integration` and `sf-core/write/data-ops`.   |
| **SOAP API**            | `sf-core/` — *not duplicated here*. Use the SOAP-tagged sf-core tools.                            |
| **Metadata API** (flows, etc.) | `sf-core/` — *not duplicated here*. Use `sf-core/write/devops/deploy_metadata`.            |
| **Connect API (CMS)**   | **`sf-marketing-cloud-next/{read,write}/cms/`** — emails, email templates, workspaces.            |
| **Data Cloud API**      | `sf-data-360/` is the **comprehensive** Data Cloud / Data 360 toolset (Connections, DMOs, Data Streams, Data Transforms, Segments, Activations, Identity Resolution, Calculated Insights, Data Graphs, Data Kits, Data Clean Room, Document AI, Machine Learning, Search Index, Private Network Routes, plus Streaming/Bulk Ingestion and the Query / Profile / Universal-ID-Lookup runtime). The MC Next-flavored wrappers under `sf-marketing-cloud-next/{read,write}/{data-cloud,segmentation,activations,identity-resolution,ingest}/` are convenience tools tailored for the marketer journey (publish a segment, trigger an activation, ingest event data, run IR) — they call the same underlying APIs. **For full Data 360 admin / lifecycle operations use `sf-data-360/`.** |

So this folder houses **only the MC Next-specific surfaces** —
**Connect CMS** (rich content authoring) plus a small set of
marketer-friendly Data Cloud convenience tools. Everything else
routes to `sf-core/` (sObjects, Bulk API 2.0, SOAP, Metadata) or to
`sf-data-360/` (full Data Cloud / Data 360 platform).

Tools split by access mode:

- `read/` — read-only (lookups, queries, metadata, profile reads, job
  status).
- `write/` — DML, ingestion, publish/unpublish, segmentation runs,
  activation triggers, identity-resolution runs, control plane.

---

## APIs in use

### 1. Connect CMS API (per the Postman collection)

| API                       | Base URL                                                              | Used for                                                              |
|---------------------------|-----------------------------------------------------------------------|-----------------------------------------------------------------------|
| Connect CMS               | `https://{instanceUrl}/services/data/v{ver}/connect/cms/`             | Create / update / delete / publish / clone marketing emails and email templates; manage CMS workspaces (Content Spaces). |

CMS calls reuse the standard Salesforce OAuth (`api`, `cms_api`,
`refresh_token` scopes). No special token exchange required.

### 2. Data Cloud API

Data Cloud runs on a **separate hostname** from Core. Two auth steps:

1. Mint a regular Salesforce access token via OAuth (any grant —
   `refresh_token`, `client_credentials`, or `authorization_code`).
2. Exchange it via the **CDP Auth Exchange** for a Data Cloud-scoped
   token bound to the CDP host:
   ```
   POST {instanceUrl}/services/a360/token
   Authorization: Bearer <core_access_token>
   Body: grant_type=urn:salesforce:grant-type:external:cdp&subject_token=<core_access_token>&subject_token_type=urn:ietf:params:oauth:token-type:access_token
   ```
   Response: `{ access_token, instance_url ("https://{tenant}.c360a.salesforce.com"), expires_in, token_type }`.

Use the `instance_url` as the base for every Data Cloud call.

| API                      | Base URL                                                                  | Used for                                                              |
|--------------------------|---------------------------------------------------------------------------|-----------------------------------------------------------------------|
| Streaming Ingestion      | `https://{cdp}.c360a.salesforce.com/api/v1/ingest/sources/{source}/{obj}` | Push real-time records into a Data Stream's source object.            |
| Bulk Ingestion           | `https://{cdp}.c360a.salesforce.com/api/v1/ingest/jobs`                   | Async CSV upload jobs (create/close/abort/delete + batches).          |
| Query API v2             | `https://{cdp}.c360a.salesforce.com/api/v2/query`                         | ANSI-SQL queries against DMOs, CIOs, DLOs, lakehouse views.           |
| Profile API              | `https://{cdp}.c360a.salesforce.com/api/v1/profile/{dmo}/{id}`            | Single-record + related-record reads on a DMO.                        |
| Calculated Insights      | `https://{cdp}.c360a.salesforce.com/api/v1/insight/objects`               | List / read / query CIOs.                                             |
| Metadata                 | `https://{cdp}.c360a.salesforce.com/api/v1/metadata`                      | List DMOs, DLOs, Data Streams, Data Transforms, Data Spaces.          |
| Segmentation             | `https://{cdp}.c360a.salesforce.com/api/v1/segments`                      | Read segment metadata + trigger publish runs.                         |
| Activations              | `https://{cdp}.c360a.salesforce.com/api/v1/activations`                   | Read activation metadata + trigger publish.                           |
| Identity Resolution      | `https://{cdp}.c360a.salesforce.com/api/v1/identityresolution/rulesets`   | List rulesets + trigger an IR run.                                    |
| Records                  | `https://{cdp}.c360a.salesforce.com/api/v1/dmo/{dmo}/{id}`                | Delete-by-id on a DMO record.                                         |

> All Data Cloud calls require `Authorization: Bearer <data-cloud-token>`
> (the exchanged token from step 2), not the core token.

---

## Authentication & scopes

Connected App scopes used by this tool set:

- `api` — generic REST access (Connect CMS, sObjects).
- `cms_api` — Connect CMS write access (publish, create, delete content).
- `cdp_query_api_read` — Data Cloud Query API.
- `cdp_profile_api_read` — Profile API.
- `cdp_ingest_api` — Streaming + Bulk Ingestion.
- `cdp_segment_api` — Segmentation publish.
- `cdp_calculated_insight_api` — Calculated Insights queries.
- `cdp_identityresolution_api` — Identity Resolution rulesets.
- `cdp_api` — broad Data Cloud access (covers most of the above).
- `refresh_token offline_access` — so the MCP server can refresh.

Missing scopes return `400 invalid_scope` at the auth step or
`403 INSUFFICIENT_ACCESS_OR_READONLY` on the API call.

---

## Identity model

| Concept                | Format / location                                            | Notes                                                         |
|------------------------|--------------------------------------------------------------|---------------------------------------------------------------|
| Org Id                 | 18-char SFDC org id (`00D...`)                                | Tenant identifier.                                            |
| Data Cloud Tenant      | `https://{tenant}.c360a.salesforce.com`                       | Returned by the cdpAuthExchange.                              |
| User Id                | 18-char SFDC user id (`005...`)                               | Run-as user for the Connected App.                            |
| DMO API Name           | String, suffix `__dlm` (data lake), `__dlm`/`__cio` (insights), `__dll` for DLO | Stable across orgs.                                           |
| Data Stream API Name   | String, suffix `__dlm` for the resulting DLM                  | One stream → many run instances.                              |
| Data Space             | String (`default` or custom)                                  | Tenant within a tenant; isolates DMOs / segments.             |
| Segment Id             | String GUID                                                   | Used for `publish` actions and related metadata.              |
| Activation Id          | String GUID                                                   | Used for `publish` and target-row queries.                    |
| Identity Resolution Id | String GUID                                                   | The ruleset id; one per resolved profile graph.               |
| Content Key            | String                                                        | CMS content stable key (set on create or generated).          |
| Variant Id             | String                                                        | A specific language / channel variant of a Content row.       |
| Workspace / Space Id   | String GUID (or developer name)                               | A CMS Content Space.                                          |

---

## Common patterns

- **Pagination**:
  - Connect CMS: `?pageParam=<token>` cursor; response includes `nextPageUrl`.
  - Data Cloud Query: `nextBatchId` returned for >100k row results;
    pass back as `next_batch_id`.
  - Bulk Ingestion: poll `GET /jobs/{id}` until `state=JobComplete`.
  - Profile API: 1 record + relationships per call; use Query API for batches.
- **Filtering / projection**:
  - CMS Search: `q={text}&types[]=cms_email&languages[]=en_US&contentSpaceId=...`.
  - Data Cloud Query: SQL `WHERE` clauses (case-sensitive on string columns).
  - Profile API: `?relationships=<comma-list>` to expand related DMOs.
- **Async jobs**: Bulk Ingest, Segment publish, IR run, Activation
  publish are all async. Each tool returns a `jobId`/`runId` to poll.
- **Idempotency**:
  - Streaming Ingest is **not** idempotent at the API level; deduplicate
    upstream using a primary-key column on the DMO.
  - Bulk Ingest is idempotent if you re-supply the same `jobId` and a
    deterministic primary key.
  - CMS publish is idempotent on `(contentKey, variantId)`.
- **Soft delete**: CMS `DELETE variants/{variantId}` is destructive; no
  built-in undelete (use export + re-create).

---

## Idempotency & limits

- **Connect CMS**:
  - 200 simultaneous publish / unpublish workers per tenant.
  - 1MB per content variant.
  - 100 publish actions per minute per workspace.
- **Data Cloud Query**:
  - 100k rows per synchronous response; >100k is chunked via `nextBatchId`.
  - 1000 concurrent queries per tenant.
- **Streaming Ingestion**:
  - 200 records per call; up to 200 calls/sec/source.
- **Bulk Ingestion**:
  - 150GB per job; 100MB per batch upload.
- **Segment publish**:
  - 1 in-flight publish per segment.
- **Activation publish**:
  - 1 in-flight publish per activation; activations are gated by
    Data Action targets (Marketing Cloud Engagement BU, Data Action,
    AppExchange connector, etc.).

---

## Error model

Connect CMS errors follow the standard Salesforce REST envelope:

```
[
  { "errorCode": "INVALID_FIELD", "message": "No such column 'frob' on entity 'CmsContent'" }
]
```

Data Cloud API errors:

```
{
  "error": "INVALID_QUERY",
  "message": "Failed to compile SQL: ...",
  "requestId": "abc-123-..."
}
```

Common error codes:

| Code (HTTP)              | Meaning                                                   |
|--------------------------|-----------------------------------------------------------|
| 401 / `INVALID_SESSION_ID` | Token expired / wrong host (CDP token used against Core or vice versa). |
| 403 / `INSUFFICIENT_ACCESS_OR_READONLY` | Missing scope on the Connected App.            |
| 404 / `NOT_FOUND`         | DMO / segment / content key not found in this Data Space.|
| 409 / `JOB_IN_PROGRESS`   | Bulk ingest / segment publish already running.           |
| 413 / `PAYLOAD_TOO_LARGE` | Single-record stream payload >1MB.                       |
| 429 / `REQUEST_LIMIT_EXCEEDED` | Per-minute rate limit hit; honor `Retry-After`.    |
| 500 / `INTERNAL_ERROR`    | Re-issue with the same `requestId` for support.          |

---

## CMS content-type model

The `Content-Type` (resource type) on a `cms/contents` row is one of:

| Type                                  | Used for                                              |
|---------------------------------------|-------------------------------------------------------|
| `cms_email__c`                        | Marketing email (a sendable instance).                |
| `cms_email_template__c`               | Reusable email template.                              |
| `cms_email_html__c` / `cms_email_components__c` | HTML-only or component-based body.          |
| `cms_image`                           | Image asset.                                          |
| `cms_document`                        | PDF or document.                                      |

The `manage_email` / `manage_email_template` tools cover both the
"with html" and "with components" flavors via the `body` parameter
(string `htmlBody` for HTML, structured `components[]` for component-based).

---

## Index of tools

| Folder                       | Tools                                                                                                            |
|------------------------------|------------------------------------------------------------------------------------------------------------------|
| `read/auth`                  | `get_user_info`                                                                                                  |
| `read/cms`                   | `get_content`, `search_contents`, `list_workspaces`, `get_workspace`                                             |
| `read/data-cloud`            | `query_data_cloud_sql`, `get_unified_profile`, `list_data_streams`, `get_data_stream`, `list_data_model_objects`, `list_data_lake_objects`, `list_data_transforms`, `list_data_spaces`, `list_calculated_insights`, `get_calculated_insight`, `list_identity_resolution_rulesets`, `list_segments`, `get_segment`, `list_activations`, `get_activation` |
| `read/ingest`                | `list_ingest_jobs`, `get_ingest_job`                                                                             |
| `write/auth`                 | `exchange_data_cloud_token`                                                                                      |
| `write/cms`                  | `manage_email`, `manage_email_template`, `publish_content`, `manage_workspace`                                   |
| `write/data-cloud`           | `stream_ingest`, `delete_dmo_record`                                                                             |
| `write/ingest`               | `manage_bulk_ingest_job`, `upload_bulk_ingest_data`                                                              |
| `write/segmentation`         | `publish_segment`                                                                                                |
| `write/activations`          | `trigger_activation`                                                                                             |
| `write/identity-resolution`  | `trigger_identity_resolution`                                                                                    |

> **Total:** 22 read tools across 4 folders, 12 write tools across 7
> folders. **34 tools.**

> **Postman parity:** Every endpoint in `Salesforce Marketing Cloud
> Next APIs.postman_collection.json` (27 Connect CMS requests across
> 4 groups: Email, Email Template, Search, Workspace) is covered under
> `read/cms/` and `write/cms/`. The Data Cloud API (not in the Postman)
> is sourced from the [official Marketing Cloud Next API guide](https://developer.salesforce.com/docs/marketing/marketing-cloud-growth/guide/mc-connect-apis-info.html).
