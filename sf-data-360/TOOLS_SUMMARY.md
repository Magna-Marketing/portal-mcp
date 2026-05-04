# Salesforce Data 360 MCP Tools: Summary & Tribal Knowledge

These tools target **Salesforce Data 360** — the rebranded, expanded
Data Cloud platform that powers Customer 360, Marketing Cloud Next,
Data Cloud-Triggered Flows, Einstein Studio / Model Builder, and
Data Clean Rooms.

Data 360 has **two complementary API planes**, both covered here:

| Plane                      | Postman collection                                            | Base URL                                                     | Token                              | Use it for                                                              |
|----------------------------|---------------------------------------------------------------|--------------------------------------------------------------|------------------------------------|-------------------------------------------------------------------------|
| **Connect (control)**      | `Salesforce Data 360 Connect APIs.postman_collection.json`    | `{instanceUrl}/services/data/v{ver}/ssot/...`                | Standard SFDC access token (OAuth) | Admin CRUD on Connections, Data Streams, DMOs, DLOs, Segments, Data Transforms, Identity Resolutions, Calculated Insights, Activations, Data Actions, Data Kits, Data Clean Room, Document AI, Machine Learning, Search Index, Private Network Routes, Metadata. |
| **Runtime (data plane)**   | `Salesforce Data 360 APIs.postman_collection.json`            | `{cdpTenantUrl}/api/v{1\|2}/...`                             | **Exchanged** Data Cloud token     | Streaming + Bulk Ingestion, Profile reads, SQL Query (v1/v2), Calculated Insight queries, Data Graph data lookups, Universal ID lookup. |

Tools split by access mode:

- `read/` — read-only (lookups, queries, metadata, profile reads, job
  status, run history, member counts).
- `write/` — DML, ingest, publish/run/refresh actions, deploy/undeploy,
  control plane (cancel / abort / retry).

Inside each, tools are grouped by domain.

---

## APIs in use

### Connect (control plane) — `/services/data/v{ver}/ssot/*`

Standard Salesforce REST host. Uses the **regular SFDC OAuth access
token**. Same envelope as any sObject REST call (`401 INVALID_SESSION_ID`,
`403 INSUFFICIENT_ACCESS_OR_READONLY`, etc.). Pagination is via
`nextRecordsUrl` / `nextPageUrl`.

| Resource family            | Path                                                          |
|----------------------------|---------------------------------------------------------------|
| Connections                | `/ssot/connections`                                           |
| Connectors                 | `/ssot/connectors`                                            |
| Data Streams               | `/ssot/data-streams`                                          |
| Data Lake Objects (DLOs)   | `/ssot/data-lake-objects`                                     |
| Data Model Objects (DMOs)  | `/ssot/data-model-objects`, `/ssot/data-model-object-mappings`|
| Data Graphs                | `/ssot/data-graphs`                                           |
| Data Spaces                | `/ssot/data-spaces`                                           |
| Data Transforms            | `/ssot/data-transforms`                                       |
| Calculated Insights        | `/ssot/calculated-insights`, `/ssot/insight/...`              |
| Profile (control reads)    | `/ssot/profile/...`                                           |
| Segments                   | `/ssot/segments`                                              |
| Activations                | `/ssot/activations`, `/ssot/activation-targets`, `/ssot/activation-external-platforms` |
| Data Actions               | `/ssot/data-actions`, `/ssot/data-action-targets`             |
| Identity Resolutions       | `/ssot/identity-resolutions`                                  |
| Data Kits                  | `/ssot/data-kits`, `/ssot/datakit/...`                        |
| Data Clean Room            | `/ssot/data-clean-room/...`                                   |
| Document AI                | `/ssot/document-processing/...`                               |
| Machine Learning           | `/ssot/machine-learning/...`                                  |
| Metadata                   | `/ssot/metadata`, `/ssot/metadata-entities`                   |
| Search Index               | `/ssot/search-index`                                          |
| Private Network Routes     | `/ssot/private-network-routes`                                |
| Query (Connect)            | `/ssot/query`, `/ssot/queryv2`, `/ssot/query-sql`             |
| Universal ID Lookup        | `/ssot/universalIdLookup/...`                                 |
| Limits                     | `/limits` (the standard org limits endpoint)                  |

### Runtime (data plane) — `{cdpTenantUrl}/api/v{1|2}/*`

Hosted on the org's CDP tenant URL (e.g.
`https://magnatech.c360a.salesforce.com`). Uses the **exchanged
Data Cloud token** (mint via `write/auth/exchange_data_cloud_token`,
which calls `/services/a360/token` with `grant_type=urn:salesforce:grant-type:external:cdp`).

| API                     | Path                                                                   |
|-------------------------|------------------------------------------------------------------------|
| Streaming Ingestion     | `POST /api/v1/ingest/sources/{source}/{object}` (insert / delete / test) |
| Bulk Ingestion          | `/api/v1/ingest/jobs[/:jobId[/batches]]`                               |
| Query API v1            | `POST /api/v1/query`                                                   |
| Query API v2            | `POST /api/v2/query`, `GET /api/v2/query/:nextBatchId`                 |
| Calculated Insights     | `GET /api/v1/insight/calculated-insights/:ci_name`, `/api/v1/insight/metadata[/:ci_name]` |
| Profile (runtime reads) | `GET /api/v1/profile/:dmo[/:id[/:childOrCi]]`, `/api/v1/profile/metadata[/:dmo]` |
| Data Graph (runtime)    | `GET /api/v1/dataGraph/metadata`, `/api/v1/dataGraph/:name[/:recordId]` |
| Metadata (runtime)      | `GET /api/v1/metadata`                                                 |
| Universal ID Lookup     | `GET /api/v1/universalIdLookup/:entityName/:dataSourceId/:dataSourceObjectId/:sourceRecordId` |

> The two planes overlap on Profile / Calculated Insights / Metadata /
> Query / Data Graph reads. **The runtime plane is faster and
> better-suited for high-throughput data lookups**; the Connect plane
> is better when you also need admin CRUD context. Tools call out
> which plane they hit.

---

## Authentication & scopes

The MCP server runs both planes from a single Connected App. Two
tokens are minted:

1. **SFDC access token** (`grant_type=refresh_token` /
   `client_credentials` / `jwt-bearer`). Used directly against the
   Connect plane (`/services/data/v{ver}/ssot/...`).
2. **Data Cloud token** (exchanged via `write/auth/exchange_data_cloud_token`
   = `POST {instanceUrl}/services/a360/token`). Used against the
   runtime plane (`{cdpTenantUrl}/api/v1...`).

Connected App scopes used by this tool set:

- `api`, `refresh_token offline_access` — Connect plane.
- `cdp_api` — broad Data Cloud (covers most runtime calls).
- `cdp_query_api_read` — Query v1 / v2 / async SQL.
- `cdp_profile_api_read` — Profile API.
- `cdp_ingest_api` — Streaming + Bulk Ingestion.
- `cdp_segment_api` — Segment publish / count.
- `cdp_calculated_insight_api` — CI queries.
- `cdp_identityresolution_api` — IR ruleset runs.

Missing scopes: `400 invalid_scope` at exchange or
`403 INSUFFICIENT_ACCESS_OR_READONLY` on the API call.

---

## Identity model

| Concept                  | Format / location                                              | Notes                                                              |
|--------------------------|----------------------------------------------------------------|--------------------------------------------------------------------|
| Org Id                   | 18-char SFDC org id (`00D...`)                                  | Tenant identifier.                                                 |
| CDP Tenant URL           | `https://{tenant}.c360a.salesforce.com`                          | Returned by `exchange_data_cloud_token`.                          |
| Data Space               | String (`default` or custom)                                    | Tenant-within-tenant; isolates DMOs / segments / activations.     |
| DMO API Name             | `Foo__dlm` (data lake managed)                                  | The target schema. Used in Profile / Query / Activation.           |
| DLO API Name             | `Foo__dll`                                                      | Raw lake storage produced by Data Streams.                         |
| Calculated Insight       | `Foo__cio`                                                      | Derived metrics. Queryable via SQL.                                |
| Data Stream              | Developer name (no suffix); produces a DLO + DLM.               |
| Data Graph               | Developer name (no suffix). One-to-many entity tree.            |
| Segment                  | `apiName` (string) **and** `id` (Connect id `0YS...`).          | Postman uses `segmentApiNameOrId` interchangeably.                |
| Activation               | `id` (string) — usually a `0YA...` Connect id.                  |
| Activation Target        | `id` (string).                                                   |
| Identity Resolution      | `id` or `apiName` (`UnifiedIndividual_default`).                |
| Connection               | `id` (Connect id, prefix typically `0Z6`).                      | One per data source.                                              |
| Data Action              | `apiName` + `id`.                                                |
| Data Action Target       | `apiName` (developer name, snake_case).                          |
| Data Kit                 | `dataKitDevName`.                                                |
| Search Index             | `apiName` or `id`.                                               |
| Universal Id             | The unified record id (e.g. `UnifiedIndividual__dlm.Id__c`).     | Resolved via `lookup_universal_id`.                               |

---

## Common patterns

- **Pagination**:
  - Connect plane: `?limit=N&offset=M` and `nextPageUrl` / `nextPageToken` in response.
  - Runtime Query v2: `nextBatchId` cursor returned for >100k row results.
  - Async SQL Query (`/ssot/query-sql`): create → poll status → fetch rows.
- **Filtering / projection**:
  - Connect: standard query string filters (e.g.
    `?dataSpace=default&type=Profile`).
  - Profile: `?fields=` whitelist; `?relatedRecords=` to expand children.
  - Query: ANSI SQL.
- **Async actions**: `actions/run`, `actions/publish`, `actions/refresh`,
  `actions/run-now` all return a `runId` — poll via `get_*_run_history`
  or the resource's main GET endpoint.
- **Idempotency**:
  - Streaming Ingest is **not** idempotent at the API; dedupe via DMO
    primary key.
  - Bulk Ingest is idempotent on `(jobId, primary key column)` if the
    column is the source object's external id.
  - All `actions/run` and `actions/publish` accept (or honor) a
    `runIdempotencyKey` to suppress duplicate runs.

---

## Idempotency & limits

- **Streaming Ingestion**:
  - 200 records / call. Up to 200 calls / second / source.
  - 1MB max payload per call.
- **Bulk Ingestion**:
  - 150GB per job. 100MB per batch. 100 active jobs per tenant.
- **Query API**:
  - 100k rows per synchronous response (v2 returns `nextBatchId` for more).
  - 1000 concurrent queries per tenant.
  - Per-query timeout: 5 minutes.
- **Async SQL Query** (`/ssot/query-sql`):
  - Up to 1B rows per result, paged via `/rows`.
  - Job retention: 24 hours.
- **Profile API**: 1 record + relationships per call; for batches use
  Query.
- **Segment / Activation / IR publish**: 1 in-flight per resource.
- **Data Transform run**: scheduled or on-demand; max 1 in-flight per
  transform.

---

## Error model

Connect plane (standard Salesforce REST):

```
[
  { "errorCode": "INVALID_FIELD",
    "message": "No such column 'foo' on entity 'DataModelObject'" }
]
```

Runtime plane (Data Cloud envelope):

```
{
  "error": "INVALID_QUERY",
  "message": "Failed to compile SQL: ...",
  "requestId": "abc-123-..."
}
```

Common codes:

| HTTP / Code                 | Meaning                                                    |
|-----------------------------|------------------------------------------------------------|
| 401 INVALID_SESSION_ID      | Token expired or wrong plane (Connect token used on runtime, etc.). |
| 403 INSUFFICIENT_ACCESS_OR_READONLY | Missing scope or DMO ACL.                          |
| 404 NOT_FOUND               | Resource doesn't exist in this Data Space.                 |
| 409 JOB_IN_PROGRESS         | Bulk ingest / segment publish / IR run already running.    |
| 409 DUPLICATE_VALUE         | Developer name / api name collision on create.             |
| 413 PAYLOAD_TOO_LARGE       | Stream record / batch >limit.                              |
| 422 INVALID_INPUT           | Schema or mapping rejected.                                |
| 429 REQUEST_LIMIT_EXCEEDED  | Per-minute rate limit; honor `Retry-After`.                |
| 500 INTERNAL_ERROR          | Re-issue with `requestId` for support.                     |
| 503 SOURCE_DMO_UNAVAILABLE  | A source DMO is paused / failed (IR / activation).         |

---

## Cross-reference

- For **Marketing Cloud Next-specific** marketing content (CMS Email,
  CMS Email Template, CMS Workspace publish/clone) see
  `sf-marketing-cloud-next/{read,write}/cms/`.
- For **Salesforce Core** (sObjects, SOQL/SOSL, Bulk API 2.0, Apex
  testing, Metadata API, etc.) see `sf-core/`.
- For **Marketing Cloud Engagement** (legacy ExactTarget — Triggered
  Sends, Data Extensions, Journeys, Automations) see
  `sf-marketing-cloud-engagement/`.

---

## Index of tools

| Folder                          | Tools                                                                                            |
|---------------------------------|--------------------------------------------------------------------------------------------------|
| `read/auth`                     | `get_user_info`                                                                                  |
| `read/account`                  | `get_limits`                                                                                     |
| `read/connections`              | `list_connections`, `get_connection`, `list_connectors`, `get_connector_metadata`, `get_connection_schema`, `introspect_connection`, `get_connection_endpoints`, `get_connection_sitemap` |
| `read/data-streams`             | `list_data_streams`, `get_data_stream`                                                           |
| `read/data-lake-objects`        | `list_data_lake_objects`, `get_data_lake_object`                                                 |
| `read/data-model-objects`       | `list_data_model_objects`, `get_data_model_object`, `list_dmo_mappings`, `get_dmo_mapping`, `get_dmo_relationships` |
| `read/data-graphs`              | `list_data_graphs`, `get_data_graph`, `get_data_graph_record`                                    |
| `read/data-spaces`              | `list_data_spaces`, `get_data_space`, `list_data_space_members`                                  |
| `read/data-transforms`          | `list_data_transforms`, `get_data_transform`, `get_data_transform_run_history`, `get_data_transform_schedule` |
| `read/calculated-insights`      | `list_calculated_insights`, `get_calculated_insight`, `get_calculated_insight_metadata`, `query_calculated_insight` |
| `read/profile`                  | `get_profile_metadata`, `get_dmo_metadata`, `get_profile_record`                                 |
| `read/segments`                 | `list_segments`, `get_segment`, `get_segment_members`                                            |
| `read/activations`              | `list_activations`, `get_activation`, `get_activation_data`, `list_activation_targets`, `get_activation_target`, `list_activation_external_platforms` |
| `read/data-actions`             | `list_data_actions`, `list_data_action_targets`, `get_data_action_target`                        |
| `read/identity-resolutions`     | `list_identity_resolutions`, `get_identity_resolution`                                           |
| `read/data-kits`                | `list_data_kits`, `get_data_kit_manifest`, `list_available_components`, `get_component_status`, `get_component_dependency` |
| `read/data-clean-room`          | `list_collaborations`, `list_query_jobs`, `list_providers`, `get_provider`, `list_provider_templates`, `list_specifications`, `list_templates` |
| `read/document-ai`              | `list_document_ai_configurations`, `get_document_ai_configuration`, `get_global_config`          |
| `read/machine-learning`         | `list_configured_models`, `get_configured_model`, `list_model_artifacts`, `get_model_artifact`, `list_model_setup_versions`, `get_model_setup_version`, `list_model_setup_partitions`, `get_model_setup_partition` |
| `read/metadata`                 | `get_metadata`, `get_metadata_entities`                                                          |
| `read/search-index`             | `list_search_indexes`, `get_search_index`, `get_search_index_config`                             |
| `read/private-network`          | `list_private_network_routes`, `get_private_network_route`                                       |
| `read/query`                    | `query_data_cloud_sql_v1`, `query_data_cloud_sql_v2`, `get_async_sql_query`                      |
| `read/ingest`                   | `list_ingest_jobs`, `get_ingest_job`                                                             |
| `read/identity`                 | `lookup_universal_id`                                                                            |
| `write/auth`                    | `exchange_data_cloud_token`                                                                      |
| `write/connections`             | `manage_connection`, `manage_connection_schema`, `manage_connection_sitemap`, `run_connection_action` |
| `write/data-streams`            | `manage_data_stream`, `run_data_stream`                                                          |
| `write/data-lake-objects`       | `manage_data_lake_object`                                                                        |
| `write/data-model-objects`      | `manage_data_model_object`, `manage_dmo_mapping`, `manage_dmo_relationship`                      |
| `write/data-graphs`             | `manage_data_graph`, `refresh_data_graph`                                                        |
| `write/data-spaces`             | `manage_data_space`, `upsert_data_space_members`                                                 |
| `write/data-transforms`         | `manage_data_transform`, `trigger_data_transform_action`, `manage_data_transform_schedule`, `validate_data_transform` |
| `write/calculated-insights`     | `manage_calculated_insight`, `run_calculated_insight`                                            |
| `write/segments`                | `manage_segment`, `publish_segment`, `count_segment`                                             |
| `write/activations`             | `manage_activation`, `publish_activation`, `manage_activation_target`                            |
| `write/data-actions`            | `manage_data_action`, `manage_data_action_target`, `generate_data_action_target_signing_key`     |
| `write/identity-resolutions`    | `manage_identity_resolution`, `run_identity_resolution`                                          |
| `write/data-kits`               | `manage_data_kit`, `deploy_data_kit`, `undeploy_data_kit`                                        |
| `write/data-clean-room`         | `manage_collaboration`, `respond_collaboration_invitation`, `run_clean_room_query`, `manage_provider`, `manage_specification` |
| `write/document-ai`             | `manage_document_ai_configuration`, `run_document_ai`, `process_document_ai_data`                |
| `write/machine-learning`        | `manage_alert`, `manage_configured_model`, `manage_model_artifact`, `manage_model_setup_version`, `predict` |
| `write/search-index`            | `manage_search_index`                                                                            |
| `write/private-network`         | `manage_private_network_route`                                                                   |
| `write/data-cloud`              | `stream_ingest`, `stream_delete`, `validate_stream_record`                                       |
| `write/ingest`                  | `manage_bulk_ingest_job`, `upload_bulk_ingest_data`                                              |
| `write/query`                   | `cancel_sql_query`                                                                               |

> **Total:** 86 read tools across 25 folders, 56 write tools across 22
> folders. **142 tools.**

> **Postman parity:** Every endpoint in
> `Salesforce Data 360 APIs.postman_collection.json` (35 requests) and
> `Salesforce Data 360 Connect APIs.postman_collection.json` (204
> requests) maps to one or more tools above. Where the two collections
> overlap on the same operation (Profile, Calculated Insight, Metadata,
> Data Graph, Universal ID Lookup), tools accept a `plane` parameter
> ("connect" | "runtime") so callers can pick the host. By default
> tools choose the plane that's typically faster for the use case
> (runtime for data lookups, connect for admin / lifecycle).
