# Everhour MCP Tools: Summary & Tribal Knowledge

## Tribal knowledge (Asana + Everhour integration)

### Identity and mapping

- **Everhour clients = Asana Portfolios**: 1:1. A client's `projects` array lists Asana Project IDs as `as:<asana_project_gid>`. Client name = Portfolio name in Asana.
- **Everhour projects**: Synced from Asana; `platform` is `"as"`. Project `id` is `as:<asana_project_gid>`. `workspaceId` is Asana workspace ID (value after `as:`).
- **Everhour tasks = Asana tasks**: 1:1. Task `id` is `as:<asana_task_gid>`. Task updates (name, status, etc.) happen in Asana and sync to Everhour. **Estimates** are set in Everhour and displayed in Asana.
- **User IDs**: Numeric user IDs (e.g. in `users`, `time.users`, `assignees.userId`, time record `user`) are **Everhour user IDs** and are **the same** across clients, projects, tasks, and time records. Asana user is referenced as `accountId`: `as:<asana_user_gid>`.

### IDs and prefixes

- **Client ID**: Everhour numeric (e.g. `10414383`). Use in `Get a Client` and in project `client`.
- **Project ID**: `as:<asana_project_gid>` (e.g. `as:1211038673444900`). Use in project and task endpoints.
- **Task ID**: `as:<asana_task_gid>`. Use in Get Task, Get Task Billing, Update Task Estimate.
- **Platform**: For projects, always `"as"` (Asana) in this setup.

### API units (convert for display)

| Concept  | API unit | Display  | Conversion |
|----------|----------|----------|------------|
| Time     | seconds  | hours    | / 3600     |
| Budget   | seconds  | hours    | / 3600     |
| Progress | seconds  | hours    | / 3600     |
| Rate     | cents    | dollars  | / 100      |
| Cost     | cents    | dollars  | / 100      |

Examples: `budget.budget` 17388000 sec = 4830 h. `rate.rate` 18500 cents = $185/hour. Task `estimate.total` and `time.total` in seconds.

### Billable vs non-billable

- **Projects**: Billable projects include `billing`, `budget`, `rate` fields. Non-billable projects omit those three fields. All other fields are the same.
- **Tasks**: If project is non-billable, all its tasks are non-billable (no flag needed). If project is billable, each task can be billable or unbillable: task has `unbillable: true` when unbillable; when absent, task is billable. Flag not present when the project itself is non-billable.
- **Time records (Get All Time Records, Get Project Time Records, Get User Time Records, Get Task Time Records)**: Responses do not say whether the project is billable. Call **Get a Project** (`get_project`) for the relevant `as:` project id. If `get_project` omits `billing` (and `budget`, `rate`), treat every time entry on that project as **non-billable**; do not use `task.unbillable` to override that. **Only** when `get_project` includes `billing` apply task-level rules: entry is non-billable if embedded `task.unbillable` is `true`; if `unbillable` is absent, treat the entry as billable. For team-wide time, resolve each distinct project in `task.projects` via `get_project`.
- **Get Task Billing**: Use when you need task-level billing info; call with `opts_include_billing=1`.

### Task estimates

- **Overall vs user-level**: A task has **either** an overall estimate (single total) **or** user-level estimate splits (per Everhour user), not both. Use **Update Task Estimate** with `type: "overall"` (and `users: {}`) or `type: "users"` (and `users: { everhourUserId: seconds }`). All values in seconds.

### Pagination and parameter constraints

- **Get All Projects**: Do **not** use `limit`. Use optional `query` and `platform` (`"as"`).
- **Get Project Tasks**: Do **not** use `page` or `limit`. Use optional `exclude-closed` and `query` (task name).
- **Get All Time Records**: `from` and `to` (YYYY-MM-DD) are **required**. Do **not** use `limit`. Use `page` (1, 2, 3, ...) and keep requesting until the response is empty.

### Authentication

All Everhour API calls require:
- **X-Api-Key** (required): API key for authentication.
- **Content-Type: application/json**

### Project attributes (from Asana)

Custom fields on projects (e.g. Manager, Project Lead, Salesforce Opportunity Id, Key Account, Project Folder, Project Due On, Project Status, Sprint, Priority) come from Asana. **Salesforce Opportunity Id** maps 1:1 to Salesforce Opportunity.

---

## Tool summary

### Users

| Tool          | Endpoint          | Purpose |
|---------------|-------------------|---------|
| get_all_users | `GET /team/users` | List all team users. Returns id (Everhour user ID, consistent across all endpoints), email, name, headline, status, role, type, rate (cents), capacity (seconds), cost, costHistory, groups, timeTrackingPolicy, permissions. Use to resolve user IDs to names/emails. |

### Clients

| Tool            | Endpoint                   | Purpose |
|-----------------|----------------------------|---------|
| get_all_clients | `GET /clients`             | List clients, optional `query` filter by name. Returns array of clients (id, name, projects as `as:` IDs, status, etc.). |
| get_client      | `GET /clients/{client_id}` | Single client by Everhour client ID. Same shape as one element of get_all_clients. |

### Projects

| Tool             | Endpoint                       | Purpose |
|------------------|--------------------------------|---------|
| get_all_projects | `GET /projects`                | List projects. Optional: `query`, `platform` ("as"). Do not use `limit`. Billable projects include billing, budget, rate; non-billable omit them. |
| get_project      | `GET /projects/{project_id}`   | Single project by id (`as:<asana_project_gid>`). Same shape as get_all_projects element. |

### Tasks

| Tool                 | Endpoint                                      | Purpose |
|----------------------|-----------------------------------------------|---------|
| get_project_tasks    | `GET /projects/{project_id}/tasks`            | List tasks for a project. Optional: `exclude-closed`, `query` (task name). Do not use page/limit. |
| get_task             | `GET /tasks/{task_id}`                        | Single task by id (`as:<asana_task_gid>`). Same shape as get_project_tasks element. |
| get_task_billing     | `GET /tasks/{task_id}?opts_include_billing=1` | Same as get_task but with billing info. Requires `opts_include_billing=1`. |
| search_project_tasks | `GET /projects/{project_id}/tasks/search`     | Search tasks by title. Required: `query`. Optional: `limit`, `searchInClosed`. Same shape as get_project_tasks. |
| update_task_estimate | `PUT /tasks/{task_id}/estimate`               | Set task estimate. Body: `type` "overall" (single total) or "users" (per-user splits, Everhour user ID to seconds), `total`, `users`. |

### Time records

| Tool                     | Endpoint                              | Purpose |
|--------------------------|---------------------------------------|---------|
| get_all_time_records     | `GET /team/time`                      | Team time records in date range. **Required:** `from`, `to` (YYYY-MM-DD). Do not use `limit`; use `page` until empty response. Billability: use `get_project` per project in `task.projects` (see Billable vs non-billable). |
| get_project_time_records | `GET /projects/{project_id}/time`     | Time records for one project. **Required:** `project_id` (as:). **Optional:** `from`, `to`, `limit`, `page`. Call `get_project` for same `project_id` before interpreting billability vs `task.unbillable`. |
| get_user_time_records    | `GET /users/{user_id}/time`           | Time records for one user. **Required:** `user_id` (Everhour user ID). **Optional:** `from`, `to`, `limit`, `page`. Same shape as get_all_time_records. |
| get_task_time_records    | `GET /tasks/{task_id}/time`           | Time records for one task. **Required:** `task_id` (as:). **Optional:** `from`, `to`, `limit`, `page`. Same shape as get_all_time_records. |
| add_time                 | `POST /time`                          | Create a time record. Body: time (seconds), date (YYYY-MM-DD), task (as:), user (Everhour user ID), comment (optional). Returns created record; 400 on error. |
| update_time_record       | `PUT /time/{time_id}`                 | Update existing time record by Everhour time record ID. Body: same as add_time. Returns updated record; 400 on error. |

---

## Quick reference: result shapes

- **Users**: id (Everhour user ID), email, name, headline, status, role, type, rate (cents), capacity (seconds), cost, costHistory, groups, timeTrackingPolicy, permissions.
- **Clients**: id (number), name, projects (as: strings), status, createdAt, lineItemMask, paymentDueDays.
- **Projects**: id (as:), name, platform "as", status, workspaceId, workspaceName, client, users[], billing/budget/rate (billable only), attributes. Time in seconds, rate in cents.
- **Tasks**: id (as:), name, type, status, url, iteration, projects[], estimate (total in seconds), time (total, users, timerTime in seconds), assignees, completed, unbillable (optional).
- **Time records**: id, date, user (Everhour ID), time (seconds), comment, task (embedded), history, lockReasons, isLocked, cost (cents), costRate (cents).

For full field lists, examples, and parameter details, see the corresponding `.txt` file in `clients/`, `projects/`, `tasks/`, or `time-records/`.
