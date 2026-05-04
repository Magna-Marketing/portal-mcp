# Salesforce Core MCP Tools: Summary & Tribal Knowledge

These tools target a single Salesforce org through the MCP server at
`https://portal-api.magnatech.io/mcp/sfc`. The MCP server handles OAuth and
talks to the org over Salesforce's HTTP APIs (REST, Tooling, Metadata, UI,
Analytics REST, GraphQL, Bulk v2). Tool specs in this folder describe the
**user-facing tool contract** and the **upstream Salesforce API** each tool
maps to so behavior, errors, and limits are predictable.

Tools are split into two top-level directories by **access mode**:

- `read/` — read-only tools (no DML, no metadata writes, no Apex execution).
- `write/` — tools that perform DML, metadata changes, deploys, anonymous
  Apex, debug-log enablement, or permission assignments.

Inside each, tools are grouped by domain. Top-level directories under
both `read/` and `write/`:

- `analytics/` — reports, dashboards, folders, report types, instances,
  reporting snapshots, subscriptions
- `apex/` — Apex classes / triggers, anonymous Apex, debug logs
- `apex-async/` (write only) — schedule / abort scheduled Apex jobs
- `aura-vf/` — Aura bundles, Visualforce pages and components
- `automation/` — validation rules, workflow rules, approval processes,
  quick actions, assignment / auto-response / escalation rules
- `communities/` — Experience sites (Network), members, ExperienceBundle
- `data-config/` — Custom Metadata Types and records, Custom Settings,
  Static Resources, Global Value Sets, Field Sets
- `data-ops/` (write only) — Bulk API 2.0 ingest + query, Composite,
  Composite Graph, sObject Tree, GraphQL
- `devops/` — Metadata API deploy / retrieve, Change Sets, Sandboxes,
  Scratch Orgs
- `email/` — templates, letterheads, custom notification types, send
  email, send mass email, publish custom notification
- `files/` — ContentVersion / ContentDocument / Attachment /
  ContentAsset / ContentDistribution
- `flows/` — Flow CRUD + activate / deactivate / validate
- `integration/` — Named Credentials, External Credentials, Connected
  Apps, External Client Apps, OAuth tokens, Remote Site Settings, CORS,
  CSP, Outbound Messages, Platform Event channels, CDC, Event Relay
- `objects/` — sObject + field metadata, picklists, FLS
- `org/` (read-heavy) — limits, org info, API versions, Setup Audit
  Trail, AsyncApexJob, CronTrigger, EventLogFile, Recycle Bin, Identity
  Verification History, OAuth usage
- `permissions/` — Permission Sets, Permission Set Groups, Profiles,
  assignments, audits, migrations, bulk ops
- `platform-extras/` — Big Objects, External Data Sources / Objects,
  Topics, Surveys, Knowledge, Duplicate Rules, Reporting Snapshots,
  Subscriptions, Field History tracking, Apex REST endpoints,
  Recycle Bin undelete, Lightning Usage metrics
- `query-search/` (read only) — SOQL/SOSL/aggregates/object discovery
- `record-types/` (read only) — record type metadata
- `sharing/` — OWD, sharing rules, manual shares, sharing recalc, login
  IPs, session settings, password policies, login history
- `testing/` — Apex test classes / runs / coverage / test suites
- `ui-components/` — LWC, FlexiPage / Lightning Pages
- `ui-navigation/` — Page Layouts, Compact Layouts, Custom Tabs,
  Custom Apps, MyDomain, Custom Labels, Translations, Search Layouts /
  Settings, Lightning Themes, List Views, Lightning Usage
- `users/` — User CRUD, roles, groups, queues, licenses, login history

---

## Salesforce APIs in use

The MCP tools in this folder are thin wrappers over a small set of
Salesforce HTTP APIs. Knowing which API a tool uses tells you the right
documentation, the right object/metadata names, and the right error model.

| API                       | Base path under the instance                          | Used for                                                               |
|---------------------------|-------------------------------------------------------|------------------------------------------------------------------------|
| REST (Data) API           | `/services/data/v{version}/`                          | SOQL/SOSL, sObject CRUD, search, list views, limits                    |
| Tooling API               | `/services/data/v{version}/tooling/`                  | ApexClass, ApexTrigger, ApexCodeCoverage, FlexiPage, AuraDefinitionBundle, Lightning component bodies, anonymous Apex, debug logs |
| Metadata API              | SOAP / `/services/data/v{version}/metadata/` (composite)| Profile, PermissionSet, PermissionSetGroup, FlexiPage, LightningComponentBundle, Flow, CustomObject, CustomField, Layout |
| UI API                    | `/services/data/v{version}/ui-api/`                   | Picklist values (record-type aware), record metadata, lookup helpers   |
| Analytics REST API        | `/services/data/v{version}/analytics/`                | Reports, dashboards, folders, report types, instances                  |
| GraphQL API               | `/services/data/v{version}/graphql`                   | UI-shaped reads, mutations against UI-API-supported objects            |
| Bulk API 2.0              | `/services/data/v{version}/jobs/{ingest|query}`       | Large-volume DML and SOQL exports (used by some `dml_records` paths)   |

> **API version**: Tools default to **v66.0 (Spring '26)**. Reference
> documentation lives in `salesforce-core/documentation/`. Object / field
> reference and SOQL/SOSL grammar live in `salesforce-core/supporting/`
> (used as RAG sources to resolve API names, picklists, and relationships).

---

## Authentication & headers

The MCP server exchanges its long-lived credentials for a session token and
makes calls on behalf of the org. From the tool caller's perspective, no
auth headers need to be passed; from the upstream call's perspective, all
Salesforce HTTP requests carry:

- `Authorization: Bearer <access_token>` — OAuth 2.0 bearer.
- `Content-Type: application/json` (or `application/xml` for Metadata SOAP).
- `Accept: application/json`.
- `Sforce-Auto-Assign: false` (used opportunistically on Lead/Case writes
  to disable assignment rules unless the caller asks for them).

Common optional headers the MCP layer may set:

- `Sforce-Call-Options: client=mtg-mcp`
- `Sforce-Query-Options: batchSize=2000` for large SOQL results.
- `If-Modified-Since` for cached metadata reads (`describe_object`, etc.).

---

## Identifiers & API names

Salesforce uses **API names**, not labels. Tools accept and return API
names in every metadata-shaped field.

| Concept                  | Format                                  | Examples                                              |
|--------------------------|-----------------------------------------|-------------------------------------------------------|
| sObject API name         | `Object__c` for custom; `Object` standard | `Account`, `Opportunity`, `Project__c`              |
| Field API name           | `Field__c` for custom; `Field` standard   | `Name`, `StageName`, `Salesforce_Opportunity_Id__c` |
| Record ID                | 15- or 18-char base62                   | `006Qg00000Dkc4RIAR`                                  |
| Record Type DeveloperName| `Developer_Name`                         | `Internal_Project`, `Master`                          |
| Profile / PermSet name   | `Developer_Name`                         | `System_Administrator`, `MTG_Project_Manager`         |
| Flow API name            | `Master_Label_Without_Spaces`           | `Account_Trigger_Flow`                                |
| LWC bundle name          | `camelCaseStartingLower`                | `projectCard`, `recordViewer`                          |
| Aura bundle name         | `camelCase`                             | `accountSummary`                                       |
| FlexiPage DeveloperName  | `Object_Record_Page` style              | `Account_Record_Page`, `Project__c_Record_Page`        |

> **15 vs 18 char IDs**: Salesforce returns 18-char IDs; tools accept
> either. Use 18-char in long-lived storage (case-insensitive safe).

---

## Access modes (and the `read/` vs `write/` split)

| Mode    | Tool side effects allowed                                                                                         |
|---------|-------------------------------------------------------------------------------------------------------------------|
| `read`  | SOQL/SOSL, describe, GET endpoints, analyses derived from reads only. **No DML, no metadata mutation, no anonymous Apex.** |
| `write` | DML on records, metadata create/update/delete (incl. permission-set ops), Apex/LWC/FlexiPage deploys, debug-log enablement, anonymous Apex execution. |

Treat anything that calls `POST /sobjects`, `PATCH /sobjects/{id}`, the
Metadata API mutators, `tooling/executeAnonymous`, or any `assign*` /
`unassign*` permission op as a **write** even if the immediate response
is small.

---

## SOQL & SOSL conventions used by tools

- **SOQL**: SELECT field list (use `FIELDS(STANDARD)` / `FIELDS(CUSTOM)` /
  `FIELDS(ALL)` with care — limited to 200 rows). Relationship traversal:
  parent `Account.Name` (dot, max 5 levels), child subqueries
  `(SELECT Id FROM Contacts)`. Aggregates: `COUNT()`, `COUNT(Id)`, `SUM`,
  `AVG`, `MIN`, `MAX`, with optional `GROUP BY` / `HAVING`. `WHERE` clause
  supports `LIKE`, `IN`, `INCLUDES`, `EXCLUDES`, `ALL ROWS` (for
  IsDeleted), date literals (`TODAY`, `LAST_N_DAYS:30`).
- **SOSL**: `FIND {term} IN ALL FIELDS RETURNING Account(Id, Name WHERE
  ...), Contact(Id, Name)`. Each `RETURNING` clause supports its own
  field list, `WHERE`, `ORDER BY`, `LIMIT`.
- **Tooling SOQL**: same grammar, hits Tooling sObjects (`ApexClass`,
  `ApexTrigger`, `ApexCodeCoverageAggregate`, `Flow`, `FlowDefinition`,
  `FlexiPage`, `AuraDefinitionBundle`, `LightningComponentBundle`, etc.).
  Tooling does **not** see standard data records.
- **Reserved chars**: escape `\\`, `\'`, `\"`, `\n`, `\r`, `\t`, `\b`, `\f`.
  Tools accept user-supplied strings and escape automatically; if you
  hand-craft SOQL, escape yourself.
- **API limits**: SOQL up to 50k rows per transaction; tools paginate via
  `nextRecordsUrl`. Aggregate queries cap at 2,000 grouping rows. SOSL
  caps at 2,000 records per query and 200 records per `RETURNING`.

---

## Pagination

| API                     | How tools paginate                                                            |
|-------------------------|--------------------------------------------------------------------------------|
| REST query              | Follow `nextRecordsUrl` from previous response (`done: false`).               |
| Tooling query           | Same as REST query.                                                           |
| Analytics list reports  | `?pageToken=` returned in `metadata.pageToken`; pass it back.                 |
| UI API records          | `pageToken` in response; pass as query param `pageToken=...`.                 |
| Bulk v2 query           | Poll job until `state=JobComplete`, then `GET /results?locator=&maxRecords=`. |
| GraphQL                 | Use cursor connections (`first`, `after`, `pageInfo.endCursor`).              |

---

## Idempotency & limits

- **DML**: Salesforce DML is **not** idempotent at the platform level.
  `dml_records` supports an `externalId` field for **upsert** which is the
  idempotent form. Without an external ID, a retried insert duplicates.
- **Bulk API 2.0**: Jobs themselves are idempotent on `jobId`; the data
  inside is not — re-uploading the same payload after success creates a
  second copy.
- **Daily API limits**: per-org limit returned by `Limits` (the MCP
  surfaces this on tool-call failures with `RequestLimitExceeded`). Plan
  bulk loads against `DailyApiRequests`.
- **Apex deploy time**: `write_apex` and `write_apex_trigger` use the
  Tooling API container/member pattern — saves are synchronous but
  **count against test deploy windows** in production orgs.
- **Metadata deploys**: most permission-set, profile, and flow writes use
  the Metadata API which is **asynchronous**; tools poll until the deploy
  finishes. Expect 5–60 s in sandbox, longer in production with running
  validation rules.
- **Anonymous Apex**: `execute_anonymous` is bound to the running user's
  permission set and counts toward governor limits like a synchronous
  transaction (10 s CPU, 6 MB heap).

---

## Error model

Every tool surfaces the Salesforce error array verbatim under an `errors`
key when the upstream call fails:

```json
{ "errorCode": "INVALID_FIELD",
  "message": "No such column 'Foo__c' on entity 'Account'.",
  "fields": ["Foo__c"] }
```

Common error codes to recognize:

| Code                              | Meaning / typical fix                                                |
|-----------------------------------|----------------------------------------------------------------------|
| `INVALID_SESSION_ID`              | Token expired; MCP retries once after refresh.                       |
| `INVALID_FIELD` / `INVALID_TYPE`  | Wrong API name or unauthorized field — check `describe_object`.      |
| `MALFORMED_QUERY`                 | SOQL syntax error.                                                   |
| `INSUFFICIENT_ACCESS_OR_READONLY` | Profile/permission set blocks the action.                            |
| `FIELD_CUSTOM_VALIDATION_EXCEPTION` | Validation rule fired on write.                                    |
| `REQUEST_LIMIT_EXCEEDED`          | Daily API limit; retry next day or contact admin.                    |
| `STORAGE_LIMIT_EXCEEDED`          | Org out of storage for this object.                                  |
| `INVALID_CROSS_REFERENCE_KEY`     | Bad lookup ID; record may not exist or be in different org.          |
| `DUPLICATE_VALUE`                 | Unique field/external ID collision.                                  |
| `ENTITY_IS_DELETED`               | Record is in Recycle Bin (or hard-deleted).                          |
| `UNABLE_TO_LOCK_ROW`              | Concurrent write contention; tools retry once.                       |

---

## Permissions model (for permission tools)

The org grants effective permissions through a layered model. Tools in
`read/permissions/` and `write/permissions/` operate on these:

| Layer               | sObject(s)                                  | What it controls                                     |
|---------------------|---------------------------------------------|------------------------------------------------------|
| Profile             | `Profile`                                   | Baseline tab, app, system, object, field perms.      |
| Permission Set      | `PermissionSet`                             | Additive grants on top of profile. License-bounded.   |
| Permission Set Group| `PermissionSetGroup`, `PermissionSetGroupComponent` | Bundle multiple permission sets; faster recalc.|
| Assignment          | `PermissionSetAssignment`                   | User ↔ permission set / group link.                  |
| Object permission   | `ObjectPermissions`                         | CRUD + viewAll/modifyAll on an object per permset.   |
| Field permission    | `FieldPermissions`                          | Read/Edit on a field per permset.                    |
| Setup audit trail   | `SetupAuditTrail`                           | Who changed what, used by audit/compare tools.       |

> **Profiles vs Permission Sets**: Salesforce is moving away from
> profile-based permissions. The `migration` and `extract_permissions_*`
> tools are designed to help reach a profile-minimal posture.

---

## Lightning UI model (for ui-components tools)

| Concept              | Tooling sObject(s)                                 | Notes                                              |
|----------------------|----------------------------------------------------|----------------------------------------------------|
| LWC bundle           | `LightningComponentBundle`, `LightningComponentResource` | Files: `*.js`, `*.html`, `*.css`, `*.js-meta.xml`, `*.svg`. |
| Aura bundle          | `AuraDefinitionBundle`, `AuraDefinition`           | Component / Application / Event / Interface.       |
| Visualforce          | `ApexPage`, `StaticResource`                       | Legacy UI.                                         |
| FlexiPage            | `FlexiPage`                                        | Lightning App / Record / Home Page; XML metadata.   |
| Page assignment      | `FlexiPageRegionInfo`, `AppMenuItem`, `ProfileLayout` | Where a page is activated.                       |
| App                  | `CustomApplication`                                |                                                    |

LWCs **must** include both `*.js` and `*.js-meta.xml`. Bundles can be
**deprecated** by renaming with a `DEPRECATED_` prefix and setting
`isExposed=false` (used by `deprecate_lwc_component`).

---

## Reports & analytics model

Reports and dashboards are first-class metadata. They are read with the
**Analytics REST API**:

- `GET /services/data/v66.0/analytics/reports` — list reports.
- `GET /services/data/v66.0/analytics/reports/{reportId}` — run a report
  (synchronous; up to 2,000 rows).
- `GET /services/data/v66.0/analytics/reports/{reportId}/describe` — get
  metadata + filters + columns.
- `POST /services/data/v66.0/analytics/reports/{reportId}/instances` —
  start an async run.
- `GET /services/data/v66.0/analytics/reports/{reportId}/instances/{instanceId}`
  — fetch instance status / results.
- `GET|PATCH /services/data/v66.0/analytics/dashboards/{dashboardId}` —
  metadata; `PUT /refresh` triggers a refresh run.
- `GET /services/data/v66.0/folders` — list folders (Report or
  Dashboard type).
- Report types via `GET /services/data/v66.0/analytics/reportTypes`.

> Reports run as the **Salesforce running user** for the OAuth session,
> not the MCP caller. Sharing rules and FLS still apply.

---

## Tool index

**Total: 305 tools (157 read + 148 write).** Organized by domain
folder under `read/` and `write/`.

### read/ — 157 tools

- `read/query-search/` (4) — `query_records`, `aggregate_query`,
  `search_objects`, `search_all`
- `read/objects/` (2) — `describe_object`, `get_picklist_values`
- `read/record-types/` (1) — `query_record_types`
- `read/apex/` (2) — `read_apex`, `read_apex_trigger`
- `read/flows/` (2) — `read_flow`, `list_flows`
- `read/permissions/` (14) — view / list / audit / analyze / compare /
  find / generate / preview tools across permission sets, permission
  set groups, profiles, and migrations.
- `read/ui-components/` (21) — LWC / Aura / VF / FlexiPage discovery,
  bundle reads, dependency / impact / usage analysis, duplicate
  detection.
- `read/analytics/` (4) — `run_report`, `describe_report`, `list_reports`,
  `list_report_types`
- **`read/users/` (7)** — `list_users`, `get_user`, `list_user_roles`,
  `list_public_groups`, `list_queues`, `list_licenses`,
  `list_login_history`
- **`read/sharing/` (6)** — `get_sharing_settings`, `list_sharing_rules`,
  `list_record_shares`, `list_login_ip_ranges`, `get_session_settings`,
  `list_apex_managed_sharing`
- **`read/automation/` (8)** — `list_validation_rules`, `list_workflow_rules`,
  `list_approval_processes`, `list_pending_approvals`, `list_quick_actions`,
  `list_assignment_rules`, `list_auto_response_rules`,
  `list_escalation_rules`
- **`read/ui-navigation/` (12)** — `list_page_layouts`, `list_compact_layouts`,
  `list_custom_tabs`, `list_custom_apps`, `get_mydomain_settings`,
  `list_custom_labels`, `list_translations`, `list_search_layouts`,
  `get_search_settings`, `list_lightning_themes`, `list_list_views`,
  `list_lightning_usage`
- **`read/email/` (4)** — `list_email_templates`, `list_letterheads`,
  `list_custom_notification_types`, `list_email_messages`
- **`read/data-config/` (7)** — `list_custom_metadata_types`,
  `query_custom_metadata_records`, `list_custom_settings`,
  `list_static_resources`, `read_static_resource`,
  `list_global_value_sets`, `list_field_sets`
- **`read/files/` (7)** — `list_files`, `get_file`, `download_file_content`,
  `list_file_shares`, `list_attachments`, `list_content_assets`,
  `list_content_distributions`
- **`read/org/` (12)** — `get_org_limits`, `get_org_info`,
  `list_api_versions`, `list_setup_audit_trail`, `list_async_jobs`,
  `list_scheduled_jobs`, `list_event_log_files`, `get_event_log_file`,
  `get_record_counts`, `get_oauth_app_usage`, `list_recycle_bin`,
  `list_identity_verification_history`
- **`read/testing/` (6)** — `list_apex_test_classes`,
  `list_apex_test_runs`, `get_apex_test_run_status`,
  `get_code_coverage`, `get_org_coverage_summary`,
  `list_apex_test_suites`
- **`read/integration/` (12)** — `list_named_credentials`,
  `list_external_credentials`, `list_connected_apps`,
  `list_external_client_apps`, `list_oauth_tokens`,
  `list_remote_site_settings`, `list_cors_origins`,
  `list_csp_trusted_sites`, `list_outbound_messages`,
  `list_platform_event_channels`,
  `list_change_data_capture_channels`, `list_event_relays`
- **`read/aura-vf/` (5)** — `list_aura_bundles`, `read_aura_bundle`,
  `list_visualforce_pages`, `read_visualforce_page`,
  `read_visualforce_component`
- **`read/communities/` (4)** — `list_experience_sites`,
  `view_experience_site`, `list_experience_site_members`,
  `list_experience_bundles`
- **`read/devops/` (6)** — `list_deployments`, `get_deployment_status`,
  `list_change_sets`, `list_sandboxes`, `view_sandbox`,
  `list_scratch_orgs`
- **`read/platform-extras/` (11)** — `list_big_objects`,
  `list_external_data_sources`, `list_external_objects`,
  `list_topics`, `list_surveys`, `list_knowledge_articles`,
  `list_duplicate_rules`, `list_reporting_snapshots`,
  `list_report_subscriptions`, `list_dashboard_subscriptions`,
  `list_apex_rest_endpoints`

### write/ — 148 tools

- `write/objects/` (4) — `dml_records`, `manage_object`, `manage_field`,
  `manage_field_permissions`
- `write/apex/` (4) — `write_apex`, `write_apex_trigger`,
  `execute_anonymous`, `manage_debug_logs`
- `write/flows/` (3) — `create_flow`, `update_flow`, `deploy_flow`
- `write/permissions/` (15) — full permset / PSG / profile mgmt + bulk
  assignment ops + extract / diff helpers.
- `write/ui-components/` (17) — LWC bundle CRUD + per-file ops, FlexiPage
  CRUD + region/component ops, deprecate, clone.
- `write/analytics/` (6) — `create_report`, `update_report`,
  `manage_report_instance`, `manage_dashboard`, `manage_report_folder`,
  `manage_dashboard_component`
- **`write/users/` (9)** — `create_user`, `update_user`, `reset_password`,
  `manage_user_role`, `manage_public_group`, `manage_queue`,
  `assign_permission_set_license`, `unassign_permission_set_license`,
  `bulk_manage_users`
- **`write/sharing/` (8)** — `update_sharing_settings`,
  `manage_sharing_rule`, `create_record_share`, `delete_record_share`,
  `recalculate_sharing`, `update_login_ip_ranges`,
  `update_session_settings`, `update_password_policies`
- **`write/automation/` (10)** — `manage_validation_rule`,
  `manage_workflow_rule`, `submit_for_approval`,
  `process_approval_action`, `manage_approval_process`,
  `manage_quick_action`, `run_quick_action`, `manage_assignment_rule`,
  `manage_auto_response_rule`, `manage_escalation_rule`
- **`write/ui-navigation/` (9)** — `manage_page_layout`,
  `manage_compact_layout`, `manage_custom_tab`, `manage_custom_app`,
  `update_mydomain_settings`, `manage_custom_label`,
  `update_translation`, `update_search_settings`, `manage_list_view`
- **`write/email/` (6)** — `send_email`, `send_mass_email`,
  `manage_email_template`, `manage_letterhead`,
  `manage_custom_notification_type`, `publish_custom_notification`
- **`write/data-config/` (5)** — `manage_custom_metadata_record`,
  `manage_custom_setting_record`, `manage_static_resource`,
  `manage_global_value_set`, `manage_field_set`
- **`write/files/` (6)** — `upload_file`, `update_file_version`,
  `delete_file`, `share_file`, `manage_attachment`,
  `create_content_distribution`
- **`write/testing/` (3)** — `run_apex_tests`, `abort_apex_test_run`,
  `manage_apex_test_suite`
- **`write/devops/` (6)** — `retrieve_metadata`, `deploy_metadata`,
  `validate_deploy`, `cancel_deploy`, `manage_sandbox`,
  `manage_scratch_org`
- **`write/aura-vf/` (4)** — `manage_aura_bundle`,
  `update_aura_definition`, `manage_visualforce_page`,
  `manage_visualforce_component`
- **`write/communities/` (3)** — `manage_experience_site`,
  `publish_experience_site`, `manage_network_member`
- **`write/integration/` (13)** — `manage_named_credential`,
  `manage_external_credential`, `manage_connected_app`,
  `manage_external_client_app`, `revoke_oauth_token`,
  `publish_platform_event`, `manage_platform_event_channel`,
  `manage_change_data_capture_channel`, `manage_event_relay`,
  `manage_remote_site_setting`, `manage_cors_origin`,
  `manage_csp_trusted_site`, `manage_outbound_message`
- **`write/data-ops/` (6)** — `manage_bulk_ingest_job`,
  `manage_bulk_query_job`, `composite_request`, `composite_graph`,
  `sobject_tree_insert`, `graphql_query`
- **`write/apex-async/` (2)** — `schedule_apex_job`,
  `abort_scheduled_job`
- **`write/platform-extras/` (9)** — `manage_external_data_source`,
  `manage_topic_assignments`, `manage_survey`,
  `manage_knowledge_article`, `manage_duplicate_rule`,
  `undelete_record`, `manage_reporting_snapshot`,
  `manage_subscription`, `update_field_history_tracking`

---

## Per-tool spec format

Each tool has its own `.txt` file. The fields are:

```
Tool: <tool_name>

Description
<one paragraph: what it does, when to use it, related tools>

Salesforce API
<REST | Tooling | Metadata | UI | Analytics | GraphQL | Bulk v2>, vXX.0

Endpoint
<METHOD https://{instance}/services/data/v{version}/...>

Headers
<the relevant ones; auth assumed>

Body
<JSON shape if applicable, or "None (GET)">

Input (parameters)
<MCP tool parameters: name, type, required/optional, description>

Result
<response shape: keys + types + meaning>

Required permissions
<profile / permset perms the running user must have>

Idempotency & limits
<retry safety, rate limits, governor limits, deploy semantics>

Errors
<key error codes the tool surfaces>

Example request
<JSON body or full URL>

Example result
<JSON shape>
```

For full per-API reference, see `salesforce-core/documentation/`.
For object/field/SOQL grammar reference (RAG), see
`salesforce-core/supporting/`.
