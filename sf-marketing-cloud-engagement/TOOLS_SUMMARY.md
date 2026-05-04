# Salesforce Marketing Cloud Engagement MCP Tools: Summary & Tribal Knowledge

These tools target the **Salesforce Marketing Cloud Engagement** APIs
(formerly Marketing Cloud / ExactTarget). They cover Contact Builder,
Data Extensions, Email Studio, Mobile Studio, Journey Builder,
Automation Studio, Content Builder, Tracking Events, Transactional
Messaging, and admin (Business Units, Users, Roles).

Tools are split into two top-level directories by **access mode**:

- `read/` — read-only tools (lookups, status checks, tracking events,
  schemas, list operations).
- `write/` — DML, sends, deploys, control-plane (start / stop / pause)
  operations.

Inside each, tools are grouped by domain.

---

## Marketing Cloud APIs in use

Marketing Cloud has **three** active API surfaces. The MCP server
targets all three; each tool spec calls out which.

| API                            | Base URL                                                                | Used for                                                              |
|--------------------------------|-------------------------------------------------------------------------|-----------------------------------------------------------------------|
| REST API                       | `https://{tenant}.rest.marketingcloudapis.com/...`                      | Modern endpoints: assets, journeys, automations, transactional, push, sms, contact search, async data extension, event notification |
| SOAP API                       | `https://{tenant}.soap.marketingcloudapis.com/Service.asmx`             | Legacy CRUD: classic Email, Subscriber, List, Send (`Send` and `TriggeredSend`), DataExtension SOAP CRUD |
| Auth (OAuth 2.0)               | `https://{tenant}.auth.marketingcloudapis.com/v2/token`                 | OAuth Client Credentials / Authorization Code grant; userinfo         |

> **Tenant subdomain**: every account has a unique tenant subdomain
> (the part before `.auth.marketingcloudapis.com`). The MCP server is
> configured with the tenant + Installed Package client credentials;
> tool callers don't pass auth themselves.
>
> **API version**: all REST URLs default to **v1** of the resource
> (Marketing Cloud doesn't use semantic version numbers like the core
> platform). Where multiple versions exist (e.g. `/asset/v1/`,
> `/email/v1/`, `/messaging/v1/`), the spec calls out the version.

---

## Authentication

The MCP server holds an **Installed Package** with `client_credentials`
grant. It mints a fresh token before each tool call (or refreshes a
cached one). Every upstream call sends:

- `Authorization: Bearer <access_token>`
- `Content-Type: application/json` (REST) or `text/xml; charset=utf-8` (SOAP)
- `Accept: application/json`

Permissions on the Installed Package determine what the tools can do.
Typical scopes used by this tool set:

- `email_read`, `email_write`, `email_send`
- `journeys_read`, `journeys_execute`
- `automations_read`, `automations_write`, `automations_execute`
- `audiences_read`, `audiences_write`
- `list_and_subscribers_read`, `list_and_subscribers_write`
- `data_extensions_read`, `data_extensions_write`
- `documents_and_images_read`, `documents_and_images_write`
- `tracking_events_read`
- `webhooks_read`, `webhooks_write`
- `accounts_read`, `accounts_write`
- `users_read`, `users_write`

> Missing scopes return `400 invalid_scope` or `403 unauthorized` —
> check the Installed Package's API integration component.

---

## Tenant / business unit model

| Concept              | Format / location                                  | Notes                                                              |
|----------------------|----------------------------------------------------|--------------------------------------------------------------------|
| Account (MID)        | Numeric Member ID (e.g. `5001234`)                 | Top-of-hierarchy tenant ("Enterprise 2.0" parent).                 |
| Business Unit (MID)  | Numeric Member ID per BU                            | Multi-tenant child accounts. Every API call is scoped to one MID.  |
| Token MID            | Token issued for one MID; switch with `account_id` | Some tools accept `accountId` to issue a fresh token for a BU.    |
| Subscriber Key       | Caller-defined string (max 254 chars)              | Long-term identifier. Used everywhere.                             |
| Subscriber ID        | Internal numeric id                                 | Returned but rarely used directly.                                |
| Contact Key          | Same as Subscriber Key in Contact Builder          | Single source of truth in the All Contacts model.                  |
| Contact ID           | Internal numeric id                                 |                                                                   |
| Data Extension Key   | External Key (string, set on create)               | Stable across copies; preferred over CustomerKey in new code.      |
| Data Extension Name  | Display name; not unique across folders            | Use External Key for API references.                               |
| Folder ID            | Numeric folder/category id                          | Folders apply to DEs, Emails, Triggered Sends, Automations, etc.   |
| Send Classification  | CustomerKey (string)                               | Required on send; bundles sender+delivery profile.                 |
| Triggered Send Definition| `CustomerKey` (string)                          | Identifies a send trigger; reused for many sends.                  |

---

## Common patterns

- **Pagination**:
  - REST APIs: `?$page=2&$pageSize=50` and `count` in the response.
  - SOAP APIs: `Retrieve` returns `OverallStatus="MoreDataAvailable"`
    plus a `RequestID`; pass it back with `ContinueRequest`.
  - Async DE retrieve: poll the job ID until status `Complete`.
- **Filtering**:
  - REST: `?$filter=field eq 'value'` (OData-ish syntax).
  - SOAP: `Filter` element (SimpleFilterPart / ComplexFilterPart).
- **Folders / categories**:
  - Each object has a `category`/`categoryId` field; root folder per
    object type. Use `read/folders/list_folders` to discover.

---

## Idempotency & limits

- **Triggered sends** are not idempotent on send; pass a stable
  `ConversationId` / `MessageId` in the body to deduplicate.
- **DE row writes** are idempotent on the DE's primary key when using
  `upsert`; without a primary key, multiple inserts duplicate.
- **Journey injection** is idempotent on `ContactKey` for the current
  journey version; the same contact entering an already-running version
  is suppressed unless re-entry is enabled.
- **Send rate limits**: Email sends are subject to provisioned send
  rates (varies by edition: Pro / Corporate / Enterprise). Bulk sends
  are queued.
- **REST rate limits**: 2,500 requests / minute per tenant by default;
  raises with package configuration.
- **SOAP API limits**: 500-record batches for `Create`/`Update`/`Delete`;
  100,000 rows per `Retrieve`.

---

## Error model

Marketing Cloud surfaces errors in two shapes:

REST:
```json
{ "message": "Bad request", "errorcode": 10000, "documentation": "https://..." }
```

SOAP:
```xml
<Results>
  <StatusCode>Error</StatusCode>
  <StatusMessage>...</StatusMessage>
  <ErrorCode>...</ErrorCode>
</Results>
```

Common error codes:

| Code | Meaning |
|------|---------|
| `10000` | Generic bad request. |
| `30000` | Token invalid / expired. Tool refreshes once and retries. |
| `40001` | Rate limit. |
| `109023` | Subscriber Key collision. |
| `12014` | Triggered Send paused / inactive. |
| `30002` | Object not found (typically DE / Asset / Journey). |
| `91000` | Invalid scope on the Installed Package. |
| `400`/`invalid_scope`/`unauthorized_client` | OAuth grant rejected. |

---

## Tool index

### read/ — 77 tools

- `read/auth/` (3) — `get_token_info`, `get_user_info`, `get_tenant_info` (uses `/v2/discovery`)
- `read/accounts/` (5) — `list_business_units`, `list_users`, `get_user`, `list_user_roles`, `list_role_permissions`
- `read/address/` (1) — `validate_email`
- `read/contacts/` (7) — `list_contacts`, `get_contact`, `list_contact_attributes`, `list_attribute_groups`, `list_contact_schemas`, `find_contact_by_email`, `get_contact_delete_request_status`
- `read/data-extensions/` (6) — `list_data_extensions`, `get_data_extension`, `list_data_extension_fields`, `query_data_extension_rows`, `get_async_query_status`, `get_file_import_status`
- `read/subscribers/` (4) — `list_lists`, `get_list`, `list_list_subscribers`, `get_subscriber`
- `read/email/` (6) — `list_email_classics`, `list_send_classifications`, `list_sender_profiles`, `list_delivery_profiles`, `list_triggered_send_definitions`, `get_send_status`
- `read/content/` (5) — `list_assets`, `get_asset`, `list_asset_types`, `list_categories`, `list_templates`
- `read/journeys/` (10) — `list_journeys`, `get_journey`, `list_journey_versions`, `get_journey_history` (uses `/audit/all`), `list_journey_audiences`, `list_event_definitions`, `get_journey_publish_status`, `list_contact_journey_membership`, `get_contact_exit_status`, `download_journey_history`
- `read/automations/` (4) — `list_automations`, `get_automation`, `list_automation_runs`, `list_activity_types`
- `read/tracking/` (7) — `list_sends`, `list_opens`, `list_clicks`, `list_bounces`, `list_unsubscribes`, `list_event_callbacks`, `list_event_subscriptions`
- `read/transactional/` (4) — `list_transactional_definitions`, `get_transactional_definition`, `get_transactional_send_status`, `get_triggered_send_status`
- `read/mobile/` (7) — `list_mobile_keywords` (modern `/sms/v1/`), `list_mobile_subscribers`, `list_push_apps`, `list_push_messages`, `list_push_locations`, `get_sms_message_history`, `list_sms_subscription_status`
- `read/groupconnect/` (2) — `list_groupconnect_apps`, `list_groupconnect_messages`
- `read/folders/` (2) — `list_folders`, `list_folder_contents`
- `read/einstein/` (2) — `get_send_time_optimization`, `get_engagement_score`
- `read/audit/` (2) — `list_audit_events` (uses `/data/v1/audit/auditEvents`), `list_security_events`

### write/ — 57 tools

- `write/auth/` (1) — `mint_token`
- `write/accounts/` (3) — `manage_user`, `assign_user_role`, `assign_user_to_business_unit`
- `write/contacts/` (4) — `update_contact_key`, `delete_contact`, `restore_contact`, `create_or_update_contact`
- `write/data-extensions/` (8) — `manage_data_extension`, `insert_rows`, `upsert_rows` (sync + async), `update_rows`, `delete_rows`, `truncate_data_extension`, `queue_file_import`, `increment_column`
- `write/subscribers/` (4) — `manage_list`, `manage_list_subscribers`, `update_subscriber_status`, `manage_subscriber`
- `write/email/` (6) — `manage_email_classic`, `send_email_now`, `manage_triggered_send_definition`, `send_triggered_email` (REST `/messageDefinitionSends/key:.../send`), `manage_send_classification`, `manage_sender_profile`
- `write/content/` (3) — `manage_asset`, `upload_asset_binary`, `manage_category`
- `write/journeys/` (7) — `manage_journey`, `publish_journey`, `control_journey`, `inject_contacts_into_journey`, `fire_journey_entry_event`, `manage_event_definition`, `remove_contact_from_journey`
- `write/automations/` (4) — `manage_automation`, `control_automation`, `schedule_automation`, `manage_automation_activity`
- `write/tracking/` (2) — `manage_event_subscription` (uses `/ens-subscriptions`), `manage_event_callback` (uses `/ens-callbacks` + `/ens-verify`)
- `write/transactional/` (4) — `manage_transactional_email_definition`, `send_transactional_email`, `manage_transactional_sms_definition`, `send_transactional_sms`
- `write/mobile/` (8) — `send_sms` (uses `/sms/v1/messageContact/...`), `send_push`, `manage_keyword` (uses `/sms/v1/keyword`), `manage_push_app`, `manage_mobile_subscriber`, `manage_push_location`, `queue_mo_message`, `manage_optin_message`
- `write/groupconnect/` (2) — `send_groupconnect_message`, `manage_groupconnect_app`
- `write/folders/` (1) — `manage_folder`

---

## Per-tool spec format

Each tool has its own `.txt` file. Same shape as `salesforce-core` and
`sf-loyalty-management`:

```
Tool: <tool_name>

Description
<one paragraph: what it does, when to use it, related tools>

Marketing Cloud API
<REST | SOAP | OAuth>, version

Endpoint
<METHOD https://{tenant}.rest.marketingcloudapis.com/...>

Headers
<auth + content-type>

Body
<JSON / XML shape if applicable, or "None (GET)">

Input (parameters)
<MCP tool parameters: name, type, required/optional, description>

Result
<response shape: keys + types + meaning>

Required scopes
<Installed Package scopes the running OAuth credential needs>

Idempotency & limits
<retry safety, rate limits, sync/async, batch caps>

Errors
<key error codes the tool surfaces>

Example request
<JSON body or full URL>

Example result
<JSON shape>
```

For full per-API reference see the bundled `marketing-cloud-api.pdf`.
