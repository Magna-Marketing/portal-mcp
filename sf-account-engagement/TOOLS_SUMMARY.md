# Salesforce Account Engagement (Pardot) MCP Tools: Summary & Tribal Knowledge

These tools target the **Salesforce Account Engagement** APIs (formerly
**Pardot**). They cover Prospects, Visitors, Lists, Forms / Form
Handlers, Landing Pages, Emails, Email Templates, List Emails,
Engagement Studio Programs, Custom Fields, Custom Redirects, Dynamic
Content, Files / Folders, Imports / Exports, Bulk Actions, Tags,
Opportunities, Tracker Domains, Lifecycle Stages, External Activities,
and Users.

Tools are split into two top-level directories by **access mode**:

- `read/` — read-only tools (lookups, queries, status checks, tracker
  visits, exports download, lifecycle history, bulk-action read).
- `write/` — DML, sends, file uploads, imports, exports, ingestion,
  visitor↔prospect linkage, control-plane (cancel / undelete) operations.

Inside each, tools are grouped by domain (one folder per object family).

---

## Account Engagement APIs in use

Account Engagement currently ships **two** active API surfaces. The MCP
server prefers v5 (modern REST/JSON) and only falls back to v4 (legacy
XML) when v5 doesn't expose the operation. Each tool spec calls out
which.

| API           | Base URL                               | Used for                                                                      |
|---------------|----------------------------------------|-------------------------------------------------------------------------------|
| **v5 REST**   | `https://{pi-domain}/api/v5/...`       | Modern CRUD on every object family. Default for all new tools. JSON over HTTPS. |
| **v4 (XML)**  | `https://{pi-domain}/api/{object}/version/4/do/{verb}` | Legacy operations that v5 doesn't yet cover: prospect assignment, list email send / test, sendBatch. |
| **OAuth 2.0** | `https://login.salesforce.com/services/oauth2/token` | Token endpoint (Salesforce SSO). Use `test.salesforce.com` for sandboxes.     |

`{pi-domain}` is your business unit's Account Engagement domain — one of:
- `pi.pardot.com` (US production legacy)
- `pi.demo.pardot.com` (US sandbox / demo)
- `pi.pardot.eu` (EU production)
- `go.pardot.com` (My Domain users; e.g. `go.{myDomain}.my.pardot.com`)

> **API version**: v5 is the recommended surface. Pardot v3 is
> end-of-life as of Q1 2025; v4 still works but is deprecated.

---

## Authentication

Account Engagement uses **Salesforce SSO** for auth. The MCP server is
configured with a Salesforce **Connected App** (Pardot scope) and
mints a fresh token before each tool call:

1. `POST https://login.salesforce.com/services/oauth2/token`
   - `grant_type=refresh_token` (server-to-server) **or**
   - `grant_type=client_credentials` (Connected App with "Issue tokens
     for client credentials flow" enabled and a run-as user)
2. Send the resulting access token on every API call:
   - `Authorization: Bearer <salesforce-access-token>`
   - `Pardot-Business-Unit-Id: <0Uv...>`  ← **required on every call**
   - `Content-Type: application/json` (REST) or
     `application/x-www-form-urlencoded` (v4 actions)

The `Pardot-Business-Unit-Id` is the 18-character ID of the Pardot
Business Unit (find it in Salesforce Setup → Pardot Account Setup).

Connected App OAuth scopes typically required:
- `pardot_api` (full Account Engagement REST/v4 access)
- `refresh_token offline_access` (so the MCP server can refresh)

> Missing scopes return `400 invalid_grant` at the token endpoint or
> `403 invalid_session_id` on the API call.

---

## Business unit / identity model

| Concept                | Format / location                                              | Notes                                                                                  |
|------------------------|----------------------------------------------------------------|----------------------------------------------------------------------------------------|
| Business Unit ID       | 18-char Salesforce ID, prefix `0Uv`                            | Sent on every call via `Pardot-Business-Unit-Id` header. One BU per Account Engagement instance. |
| Prospect ID            | 64-bit numeric                                                 | The "person" record. Created from forms, imports, or API. Cross-references a CRM Contact/Lead. |
| Prospect `crmId`       | Salesforce 18-char ID                                          | Mirrors Lead/Contact/Account on the connected org.                                    |
| Visitor ID             | 64-bit numeric                                                 | Anonymous web visitor. Promotes to a Prospect via `assign_visitor_to_prospect`.       |
| Campaign ID            | Numeric (Pardot) **or** Salesforce 18-char (post-SFDC connect) | After "Connected Campaigns" is enabled, Pardot campaigns share an ID with SFDC.        |
| List ID                | 64-bit numeric                                                 | Static or dynamic. Dynamic lists rely on `criteria`.                                  |
| File / Folder ID       | 64-bit numeric                                                 | Folders are hierarchical, scoped per object family.                                   |
| Tag                    | Free-text string (case-preserving, comparison case-insensitive)| Cross-object label. Use `merge_tags` to consolidate.                                  |
| Custom Field `apiFieldId` | String (snake_case)                                         | Stable API name. Use this in Prospect writes — not the display `fieldId`.            |
| Tracker Domain ID      | 64-char numeric                                                | Owns the vanity URL used for emails / forms / custom redirects.                       |

---

## Common patterns

- **Field projection**: every v5 GET supports `?fields=a,b,c.relation.d`.
  Tools use a curated default `fields` list and let callers pass `fields`
  to override. Without `fields`, only `id` is returned.
- **Filtering**: `?filter[name]=eq,Welcome` (operators: `eq`, `notEq`,
  `gt`, `gte`, `lt`, `lte`, `in`, `notIn`, `like`, `notLike`, `isNull`,
  `notNull`).
- **Sorting**: `?orderBy=updatedAt desc,name asc`.
- **Pagination**: cursor based via `nextPageUrl` and `nextPageToken`
  in the response. Tools return them and accept `nextPageToken` as
  input.
- **Tag mutation**: every taggable object has `do/addTag` and
  `do/removeTag` actions. Tools expose a single `manage_*_tags`
  with `operation: "add" | "remove"`.
- **Folders / categories**: every folder-aware object has a `folderId`
  field; root folder per object family. Use `read/folders/list_folders`
  and `read/folders/list_folder_contents` to browse.
- **Soft delete vs hard delete**:
  - Most write operations soft-delete (`isDeleted=true`) with a 90-day
    recovery window. Use `do/undelete` (Forms, Prospects) to restore.
  - `DELETE` on Custom Fields / Lists / Form Handlers / Layout
    Templates / Email Templates / Files is permanent.

---

## Idempotency & limits

- **Daily API limit**: published on `read/account/get_account`
  (`maximumDailyApiCalls`). Pro: 25k/day. Plus: 50k/day. Advanced /
  Premium: 100k/day. Concurrency cap: 5 requests in flight.
- **Bulk Action**: 50,000 prospect updates per job; status polled with
  `get_bulk_action`. Async; not idempotent — supply your own dedupe key.
- **Imports**: 50,000 rows per import file, multiple files per import
  via `imports/{id}/batches`. Idempotent on prospect email when
  `operation=upsert`.
- **Exports**: created async; poll `get_export` until `status=Complete`,
  then download via `download_export_results`.
- **Engagement Studio Programs**: created from a YAML structure file;
  `download_program_structure` round-trips the same YAML.
- **Prospect upsert by email**: `upsert_prospect_by_email` is the
  recommended idempotent write. Repeat calls return the same prospect.
- **List email send**: idempotent only if the same `clientReference`
  is supplied via the v4 endpoint.
- **External Activity ingestion**: idempotent on
  `(extensionSalesforceId, externalId)` — re-sending the same payload
  is a no-op.

---

## Error model

v5 returns:

```
{
  "code": 38,
  "message": "API user must be a verified user.",
  "errorRef": "01af2a8e-...",
  "details": [{ "field": "salesforceId", "message": "..." }]
}
```

Common v5 codes:

| Code   | Meaning                                                       |
|--------|---------------------------------------------------------------|
| 1      | Internal error.                                               |
| 4      | Invalid API key / business unit (rotated token, wrong BU id). |
| 15     | Login failed (bad creds).                                     |
| 20     | Invalid input parameter.                                      |
| 38     | API user not verified — set `Verify Email` on the user.       |
| 49     | Trying to write a read-only field.                            |
| 53     | Operation not permitted (Connected App / scope issue).        |
| 59     | Object not found.                                             |
| 65     | Invalid field projection (`fields=` mentions a missing field).|
| 76     | Concurrency limit exceeded — retry with backoff.              |
| 184    | Daily API limit reached — wait until midnight UTC.            |
| 600    | Validation error on payload.                                  |

v4 returns XML with `<rsp stat="fail" version="1.0"><err code="N">msg</err></rsp>`.
The MCP server normalizes these into the same `{ code, message }` envelope.

OAuth token errors:

| HTTP   | Body `error`              | Meaning                                            |
|--------|---------------------------|----------------------------------------------------|
| 400    | `invalid_grant`           | Refresh token revoked / user deactivated.          |
| 400    | `invalid_client_id`       | Connected App ID wrong.                            |
| 400    | `unsupported_grant_type`  | CC flow not enabled on Connected App.              |
| 401    | `invalid_session_id`      | Expired access token; refresh and retry.           |

---

## Visitor → Prospect identity flow

1. Anonymous web traffic emits **Visitors** (`piVID` cookie). Page hits
   become `VisitorPageView` and grouped into `Visit`.
2. Cookie identification (form submission, email click, MAID match)
   converts a Visitor into a Prospect.
3. `assign_visitor_to_prospect` is the API to manually staple a visitor
   to an existing prospect.
4. `identify_visitor_company` enriches the visitor with reverse-IP
   firmographics (Visitor Insight).

---

## Prospect lifecycle

- **Create / update**: `manage_prospect` (POST/PATCH on
  `/api/v5/objects/prospects`). Use `apiFieldId` for custom fields.
- **Idempotent upsert**: `upsert_prospect_by_email`
  (`POST /objects/prospects/do/upsertLatestByEmail`) — preferred for
  inbound integrations.
- **Soft delete & restore**: `manage_prospect operation="delete"` +
  `undelete_prospect` (`POST /objects/prospects/do/undelete`).
- **Assignment**: `assign_prospect` uses v4 endpoints
  (`/api/prospect/version/4/do/assign|unassign/id/{id}`). Sets `userId`,
  `groupId`, or `roleId`.
- **Tags**: `manage_prospect_tags` toggles `do/addTag` / `do/removeTag`.

---

## List / list-membership / list-email model

- **List**: `manage_list` CRUD; supports static and dynamic. Dynamic
  lists pull `criteria` (a Pardot DSL) and recompute hourly.
- **List Membership**: `manage_list_membership` (CRUD on
  `/objects/list-memberships`) — `(listId, prospectId)` is the natural
  key; `optedOut` flips opt-out scoped to that list.
- **List Email** (a one-off send to a list):
  - `create_list_email_draft` creates the email shell (v5).
  - `send_list_email` actually fires it (v4
    `/api/email/version/4/do/send`) — v5 doesn't yet expose `do/send`.
  - Track stats with `get_list_email` (read tool — surfaces opens,
    clicks, bounces, opt-outs, deliveries).

---

## Engagement Studio program model

- A program is a YAML graph of "steps" (rules / actions / triggers /
  email sends) attached to a list of prospects.
- `read/engagement-programs/list_engagement_programs`,
  `read/engagement-programs/get_engagement_program`, and
  `read/engagement-programs/download_program_structure` (returns the
  YAML) cover the read side.
- `write/engagement-programs/upload_engagement_studio_program` POSTs a
  multipart payload with the program YAML to create a new program. v5
  doesn't yet expose `update`; round-trip via download → modify →
  upload-as-new.

---

## File & folder model

- Files live in `objects/files`; the upload payload is multipart
  (`file` part + `name`, `folderId`).
- Folders are typed (the same folder hierarchy is **not** shared
  across object families). Use `list_folder_contents` to browse a
  given folder regardless of object type.
- `Layout Templates` (HTML+CSS for forms / landing pages) are
  managed via `manage_layout_template`.

---

## Custom fields & tags

- **Custom fields** (`/objects/custom-fields`) extend the Prospect
  object. `apiFieldId` is the API name used on prospect writes.
- **Tags** are not first-class custom fields; they're labels on
  taggable objects (Prospect, Campaign, Email, EmailTemplate, File,
  Form, FormField, FormHandler, LandingPage, LayoutTemplate, List,
  Opportunity, User, CustomField, CustomRedirect, DynamicContent).

---

## Imports vs Bulk Actions vs Exports

| Job type      | Endpoint                        | Use when                                                    |
|---------------|---------------------------------|-------------------------------------------------------------|
| **Import**    | `/api/v5/imports`               | Upserting prospects from CSV (1–10M rows, multi-batch).     |
| **Bulk Action**| `/api/v5/bulk-actions`         | One-shot operation on a filtered set of prospects (e.g. set field, add to list). Up to 50k records. |
| **Export**    | `/api/v5/exports`               | Pulling a filtered set of records back out as JSON/CSV. Async. |

All three are async: poll the read tool until `status=Complete` (or
`Error`), then download or pull errors.

---

## External Activities

- **External Activity Types** are configured by the admin in Setup
  ("External Activity Types").
- The MCP `submit_external_activity`
  (`POST /api/v5/external-activities`) ingests a single event with
  `extensionSalesforceId`, `typeName`, `prospectEmail` (or
  `prospectFid` / `prospectId`), `value`, and `activityDate`.
- Once ingested, the event becomes a `VisitorActivity` row that
  Engagement Studio rules and scoring can reference.

---

## Rate limits & retries

- **Daily**: tracked on `read/account/get_account.apiCallsUsed`.
- **Per-minute**: 100 requests/minute on v5 endpoints by default; the
  server returns `429 Too Many Requests` with a `Retry-After` header.
- **Concurrency**: 5 concurrent v5 requests per BU. The MCP server
  serializes / queues writes if it sees `code: 76`.
- **Recommended retry**: exponential backoff on `429`, `5xx`, and
  `code: 76`; max 5 retries.

---

## Index of tools

> Counts reflect the current shape of the directory. Each `.txt` file
> is a complete tool spec.

### Read tools

| Folder                 | Tools                                                                                                            |
|------------------------|------------------------------------------------------------------------------------------------------------------|
| `read/account`         | `get_account`                                                                                                    |
| `read/auth`            | `get_user_info`                                                                                                  |
| `read/bulk-actions`    | `list_bulk_actions`, `get_bulk_action`                                                                           |
| `read/campaigns`       | `list_campaigns`, `get_campaign`                                                                                 |
| `read/custom-fields`   | `list_custom_fields`, `get_custom_field`                                                                         |
| `read/custom-redirects`| `list_custom_redirects`, `get_custom_redirect`                                                                   |
| `read/dynamic-content` | `list_dynamic_content`, `get_dynamic_content`, `list_dynamic_content_variations`, `get_dynamic_content_variation`|
| `read/emails`          | `list_sent_emails`, `get_sent_email`, `list_email_templates`, `get_email_template`, `list_list_emails`, `get_list_email` |
| `read/engagement-programs` | `list_engagement_programs`, `get_engagement_program`, `download_program_structure`                            |
| `read/exports`         | `list_exports`, `get_export`, `download_export_results`                                                          |
| `read/external-activities` | `list_external_activities`, `get_external_activity`                                                          |
| `read/files`           | `list_files`, `get_file`                                                                                         |
| `read/folders`         | `list_folders`, `get_folder`, `list_folder_contents`, `get_folder_content`                                       |
| `read/forms`           | `list_forms`, `get_form`, `list_form_fields`, `get_form_field`, `list_form_handlers`, `get_form_handler`, `list_form_handler_fields`, `get_form_handler_field` |
| `read/imports`         | `list_imports`, `get_import`, `download_import_errors`                                                           |
| `read/landing-pages`   | `list_landing_pages`, `get_landing_page`                                                                         |
| `read/layout-templates`| `list_layout_templates`, `get_layout_template`                                                                   |
| `read/lifecycle`       | `list_lifecycle_stages`, `get_lifecycle_stage`, `list_lifecycle_history`, `get_lifecycle_history`                |
| `read/lists`           | `list_lists`, `get_list`, `list_list_memberships`, `get_list_membership`                                         |
| `read/opportunities`   | `list_opportunities`, `get_opportunity`                                                                          |
| `read/prospects`       | `list_prospects`, `get_prospect`, `list_prospect_accounts`, `get_prospect_account`                               |
| `read/tags`            | `list_tags`, `get_tag`, `list_tagged_objects`, `get_tagged_object`                                               |
| `read/tracker-domains` | `list_tracker_domains`, `get_tracker_domain`                                                                     |
| `read/users`           | `list_users`, `get_user`                                                                                         |
| `read/visitors`        | `list_visitors`, `get_visitor`, `list_visits`, `get_visit`, `list_visitor_activities`, `get_visitor_activity`, `list_visitor_page_views`, `get_visitor_page_view` |

### Write tools

| Folder                 | Tools                                                                                                            |
|------------------------|------------------------------------------------------------------------------------------------------------------|
| `write/auth`           | `refresh_access_token`, `revoke_token`                                                                           |
| `write/bulk-actions`   | `manage_bulk_action`                                                                                             |
| `write/campaigns`      | `manage_campaign_tags`, `connect_salesforce_campaign`                                                            |
| `write/custom-fields`  | `manage_custom_field`, `manage_custom_field_tags`                                                                |
| `write/custom-redirects`| `manage_custom_redirect`, `manage_custom_redirect_tags`                                                         |
| `write/dynamic-content`| `manage_dynamic_content`, `manage_dynamic_content_variation`, `manage_dynamic_content_tags`                      |
| `write/emails`         | `send_one_to_one_email`, `manage_email_tags`, `manage_email_template`, `manage_email_template_tags`, `create_list_email_draft`, `send_list_email` |
| `write/engagement-programs` | `upload_engagement_studio_program`                                                                          |
| `write/exports`        | `create_export`, `cancel_export`                                                                                 |
| `write/external-activities` | `submit_external_activity`                                                                                  |
| `write/files`          | `manage_file`, `manage_file_tags`                                                                                |
| `write/forms`          | `manage_form`, `undelete_form`, `reorder_form_fields`, `manage_form_tags`, `manage_form_field`, `manage_form_field_options`, `reorder_form_field_values`, `manage_form_field_tags`, `manage_form_handler`, `manage_form_handler_field`, `manage_form_handler_tags` |
| `write/imports`        | `manage_import`, `upload_import_batch`, `cancel_import`                                                          |
| `write/landing-pages`  | `manage_landing_page`, `manage_landing_page_tags`                                                                |
| `write/layout-templates`| `manage_layout_template`, `manage_layout_template_tags`                                                         |
| `write/lists`          | `manage_list`, `manage_list_tags`, `manage_list_membership`                                                      |
| `write/opportunities`  | `manage_opportunity_tags`                                                                                        |
| `write/prospects`      | `manage_prospect`, `upsert_prospect_by_email`, `undelete_prospect`, `manage_prospect_tags`, `assign_prospect`    |
| `write/tags`           | `manage_tag`, `merge_tags`                                                                                       |
| `write/users`          | `manage_user_tags`                                                                                               |
| `write/visitors`       | `assign_visitor_to_prospect`, `identify_visitor_company`                                                         |

> **Total:** 79 read tools across 25 folders, 56 write tools across 21
> folders. **135 tools.**

> **Postman parity:** Every endpoint in `Marketing Cloud Account
> Engagement API (fka Pardot API).postman_collection.json` (238 v5
> requests across 38 object groups) maps to one or more tools above.
> The 60+ Export `<Object>.FilterBy[Created/Updated/Activity/Prospect]At`
> requests are all the same `POST /api/v5/exports`; `create_export`
> covers them via the `object` + `procedure` parameters. The Import
> "Create with File" multipart variant is folded into `manage_import`
> (pass `csv` / `csvBase64` alongside `operation="create"`).
