# Asana MCP Tools: Summary and Reference

## Base URL and authentication

- **Base URL**: `https://app.asana.com/api/1.0`
- **Authentication**: All requests require `Authorization: Bearer <access_token>` (Personal Access Token or OAuth access token). Create PATs in Asana: My Settings > Apps > Personal Access Tokens.
- **Content-Type**: `application/json` for request/response.

## Response format

- **Single resource**: `{ "data": { ... } }`
- **List**: `{ "data": [ ... ], "next_page": { "offset": "...", "path": "...", "uri": "..." } }`
- **Pagination**: Use `limit` and `offset`; for next page use `next_page.offset` from the previous response.

## IDs (GID)

- All Asana IDs are **GID** (globally unique identifier), string type (e.g. `"1213485594364562"`).
- **Workspace GID**: Use in Get Workspaces, Get Projects for Workspace, Get Users for Workspace, Create Project, Create Task.
- **Project GID**: Use in project, section, and task endpoints.
- **Task GID**: Use in task and dependency endpoints.
- **Section GID**: Use in section and task-in-section endpoints.
- **User GID**: Use for assignee, followers, members. Use `"me"` for the authenticated user in Get User.
- **Portfolio GID**: Use in portfolio and portfolio-membership endpoints.

## Common query parameters

- **opt_fields**: Comma-separated list of field names to return (reduces payload; default is compact representation).
- **opt_expand**: Comma-separated relation names to expand.
- **opt_pretty**: Set for pretty-printed JSON (debugging).
- **limit**: Results per page (default 20, max 100).
- **offset**: Pagination offset; use `next_page.offset` when present.

---

## Tool summary

### Portfolios

| Tool                                      | Endpoint                                              | Purpose |
|-------------------------------------------|--------------------------------------------------------|---------|
| get_portfolios                            | `GET /portfolios`                                     | List portfolios. Optional: workspace, owner, opt_fields, limit, offset. |
| get_portfolio                             | `GET /portfolios/{portfolio_gid}`                     | Single portfolio by GID. |
| get_items_for_portfolio                   | `GET /portfolios/{portfolio_gid}/items`                | List projects (items) in a portfolio. |
| add_members_for_portfolio                 | `POST /portfolios/{portfolio_gid}/addMembers`          | Add users to portfolio. Body: members (user GIDs), optional role. |
| get_portfolio_memberships_for_portfolio   | `GET /portfolios/{portfolio_gid}/portfolio_memberships` | List portfolio memberships (who has access). |

### Project briefs

| Tool                    | Endpoint                                      | Purpose |
|-------------------------|-----------------------------------------------|---------|
| get_project_brief       | `GET /projects/{project_gid}/project_brief`   | Get project brief for a project. |
| update_project_brief    | `PATCH /project_briefs/{project_brief_gid}`   | Update a project brief. Body: title, html_text, status. |
| create_project_brief    | `POST /projects/{project_gid}/project_brief`  | Create a project brief. Body: title, html_text, status. |

### Project memberships

| Tool                                  | Endpoint                                                | Purpose |
|---------------------------------------|---------------------------------------------------------|---------|
| get_project_memberships_for_project   | `GET /projects/{project_gid}/project_memberships`        | List project memberships (who has access to the project). |

### Project statuses

| Tool                              | Endpoint                                                    | Purpose |
|-----------------------------------|-------------------------------------------------------------|---------|
| get_project_status                | `GET /project_statuses/{project_status_gid}`                | Single project status update. |
| delete_project_status             | `DELETE /project_statuses/{project_status_gid}`             | Delete a project status update. |
| get_project_statuses_for_project  | `GET /projects/{project_gid}/project_statuses`              | List project status updates for a project. |
| create_project_status_for_project | `POST /projects/{project_gid}/project_statuses`             | Create a project status. Body: title, text, color, status_type. |

### Projects

| Tool                                | Endpoint                                           | Purpose |
|-------------------------------------|----------------------------------------------------|---------|
| get_projects                        | `GET /projects`                                    | List projects. Optional: workspace, team, archived, opt_fields. |
| create_project                      | `POST /projects`                                   | Create project. Body: workspace (required), name (required), notes, color, due_on, team, public, default_view. |
| get_project                         | `GET /projects/{project_gid}`                       | Single project by GID. |
| update_project                      | `PATCH /projects/{project_gid}`                    | Update project. Body: name, notes, archived, color, due_on, public, default_view. |
| get_projects_for_workspace         | `GET /workspaces/{workspace_gid}/projects`         | All projects in a workspace. Optional: archived, opt_fields. |
| create_project_for_workspace       | `POST /workspaces/{workspace_gid}/projects`        | Create project in workspace. Body: name (required), notes, color, due_on, team, public, default_view. |
| add_custom_field_setting_for_project | `POST /projects/{project_gid}/addCustomFieldSetting` | Add a custom field to a project. Body: custom_field (GID). |
| add_members_for_project            | `POST /projects/{project_gid}/addMembers`          | Add users/teams to project. Body: members (GIDs). |
| remove_members_for_project         | `POST /projects/{project_gid}/removeMembers`       | Remove users/teams from project. Body: members (GIDs). |

### Sections

| Tool                        | Endpoint                                              | Purpose |
|-----------------------------|-------------------------------------------------------|---------|
| get_section                 | `GET /sections/{section_gid}`                         | Single section by GID. |
| update_section              | `PATCH /sections/{section_gid}`                       | Update section (e.g. name). Body: name. |
| delete_section              | `DELETE /sections/{section_gid}`                     | Delete a section. |
| get_sections_for_project    | `GET /projects/{project_gid}/sections`                | List sections in a project. |
| create_section_for_project  | `POST /projects/{project_gid}/sections`               | Create section. Body: name (required). |
| add_task_for_section        | `POST /sections/{section_gid}/addTask`               | Add task to section. Body: task (GID), optional insert_after/insert_before. |
| insert_section_for_project  | `POST /projects/{project_gid}/sections/insert`        | Move/insert section. Body: section (GID), insert_after or insert_before. |

### Status updates (generic)

| Tool                     | Endpoint                              | Purpose |
|--------------------------|---------------------------------------|---------|
| get_status               | `GET /status_updates/{status_update_gid}` | Single status update. |
| delete_status            | `DELETE /status_updates/{status_update_gid}` | Delete status update. |
| get_statuses_for_object  | `GET /status_updates?parent={gid}`     | Status updates for a project or goal. Required: parent (GID). |
| create_status_for_object | `POST /status_updates`                 | Create status update. Body: parent (GID), title, text, html_text, status_type. |

### Workspaces

| Tool                     | Endpoint                                    | Purpose |
|--------------------------|---------------------------------------------|---------|
| get_workspaces           | `GET /workspaces`                           | List workspaces (and organizations) the user can access. |
| remove_user_for_workspace | `POST /workspaces/{workspace_gid}/removeUser` | Remove a user from workspace/org. Body: user (GID). |

### Tasks

| Tool                          | Endpoint                                           | Purpose |
|-------------------------------|----------------------------------------------------|---------|
| search_tasks_for_workspace   | `GET /workspaces/{workspace_gid}/tasks/search`     | Search tasks in workspace. Optional: text, resource_subtype, completed_since, opt_fields. |
| add_followers_for_task       | `POST /tasks/{task_gid}/addFollowers`             | Add followers. Body: followers (user GIDs). |
| remove_follower_for_task     | `POST /tasks/{task_gid}/removeFollowers`           | Remove followers. Body: followers (user GIDs). |
| add_project_for_task         | `POST /tasks/{task_gid}/addProject`                | Add task to project. Body: project (GID), optional section, insert_after/insert_before. |
| get_dependencies_for_task    | `GET /tasks/{task_gid}/dependencies`               | Tasks this task depends on (blocking tasks). |
| add_dependencies_for_task   | `POST /tasks/{task_gid}/addDependencies`           | Set dependencies. Body: dependencies (task GIDs). |
| remove_dependencies_for_task | `POST /tasks/{task_gid}/removeDependencies`        | Unlink dependencies. Body: dependencies (task GIDs). |
| get_dependents_for_task      | `GET /tasks/{task_gid}/dependents`                 | Tasks that depend on this task. |
| add_dependents_for_task      | `POST /tasks/{task_gid}/addDependents`             | Set dependents. Body: dependents (task GIDs). |
| remove_dependents_for_task   | `POST /tasks/{task_gid}/removeDependents`         | Unlink dependents. Body: dependents (task GIDs). |
| set_parent_for_task          | `POST /tasks/{task_gid}/setParent`                | Set parent (make subtask). Body: parent (task GID or "" for top-level). |
| create_subtask_for_task      | `POST /tasks/{task_gid}/subtasks`                 | Create subtask. Body: name (required), notes, completed, due_on, assignee. |
| get_subtasks_for_task        | `GET /tasks/{task_gid}/subtasks`                  | List subtasks. |
| get_tasks_for_section        | `GET /sections/{section_gid}/tasks`               | Tasks in a section. |
| get_tasks_for_project        | `GET /projects/{project_gid}/tasks`                | Tasks in a project. Optional: section, completed_since, opt_fields, limit, offset. |
| duplicate_task               | `POST /tasks/{task_gid}/duplicate`                | Duplicate task. Body: optional name, include, name_suffix. |
| delete_task                  | `DELETE /tasks/{task_gid}`                        | Delete a task. |
| update_task                  | `PATCH /tasks/{task_gid}`                         | Update task. Body: name, notes, completed, due_on, assignee, start_on, resource_subtype. |
| get_task                     | `GET /tasks/{task_gid}`                            | Single task by GID. Use opt_fields for full details. |
| create_task                  | `POST /tasks`                                     | Create task. Body: workspace (required), name, notes, completed, due_on, assignee, projects, parent, start_on. |
| get_tasks                    | `GET /tasks`                                      | List tasks. Optional: project, section, assignee, workspace, completed_since, opt_fields. |

### Users

| Tool                      | Endpoint                                  | Purpose |
|---------------------------|-------------------------------------------|---------|
| get_users                 | `GET /users`                              | List users. Optional: workspace, team, opt_fields. |
| get_user                  | `GET /users/{user_gid}`                   | Single user. Use user_gid "me" for authenticated user. |
| get_users_for_workspace   | `GET /workspaces/{workspace_gid}/users`   | Users in a workspace or organization. |

---

## Quick reference: result shapes

- **Portfolio**: gid, resource_type, name, archived, color, created_at, created_by, workspace, owner, due_on, start_on, custom_field_settings, members.
- **Project**: gid, resource_type, name, archived, created_at, current_status, due_on, notes, owner, workspace, team, members, custom_field_settings, sections.
- **Task**: gid, resource_type, name, notes, completed, completed_at, due_on, start_on, created_at, modified_at, assignee, followers, projects, memberships, parent, custom_fields, resource_subtype, workspace.
- **Section**: gid, resource_type, name, created_at, project.
- **User**: gid, resource_type, name, email, photo, workspaces, is_workspace_admin.
- **Workspace**: gid, resource_type, name, is_organization, email_domains.
- **Project status**: gid, resource_type, title, text, color, status_type, created_at, author, project.
- **Status update**: gid, resource_type, title, text, html_text, status_type, created_at, author, parent.
- **Project brief**: gid, resource_type, title, html_text, status.
- **Portfolio membership / Project membership**: gid, resource_type, portfolio/project, user/member, access_level/role.

For full field lists, request/response examples, and parameter details, see the corresponding `.txt` file in `portfolios/`, `project-briefs/`, `project-memberships/`, `project-statuses/`, `projects/`, `sections/`, `status-updates/`, `workspaces/`, `tasks/`, or `users/`.
