# Task Management Platform — REST API Contract

All routes are prefixed `/api/v1`. Every route below states its **Access** level:
`Public`, `Auth` (any authenticated user), or a specific role list (checked against
that resource's `project_members` role). Unless noted, request/response bodies are
JSON. No response body anywhere includes `password_hash`.

**Status-code convention (applies to every resource):** `401` = missing/invalid token,
`403` = authenticated and a member of the resource's project, but role is insufficient
for this action, `404` = caller is not a member of the resource's project at all (its
existence is not revealed), or the resource genuinely doesn't exist. This means a `403`
already confirms the resource exists; a `404` never does.

## 0. Pagination, Filtering, and Sorting Convention

Defined once here; every list endpoint below references it instead of redefining it.

- `page` — integer, default `1`.
- `pageSize` — integer, default `20`, maximum `100` (requests above the max are clamped, not rejected).
- `status` — optional, only meaningful on `tasks` endpoints (`todo` | `in_progress` | `done`).
- `sort` — `field:direction`, e.g. `sort=createdAt:desc`. Default is `createdAt:desc` unless a route says otherwise.

Every paged response has this envelope:

```json
{
  "data": [
    /* array of resource objects */
  ],
  "page": 1,
  "pageSize": 20,
  "total": 143
}
```

## 1. Tasks

#### `POST /api/v1/projects/:id/tasks` — owner/admin/member (not viewer)

Request: `{ "title": "Fix header spacing", "description": "...", "priority": 3, "assigneeId": 2, "dueDate": "2026-09-10", "tagIds": [1, 4] }`
Response `201`: full task object with resolved `tags`.
Errors: `400` invalid body, or `assigneeId` is not a member of this project; `403` viewer role; `404` not a member of the project.

#### `GET /api/v1/projects/:id/tasks` — owner/admin/member/viewer

Paged list per the [pagination convention](#0-pagination-filtering-and-sorting-convention); supports `status` filter and `sort` (e.g. `sort=priority:desc`).
Errors: `404` not a member.

#### `GET /api/v1/tasks/:id` — member of the task's project (any role)

Response `200`: full task object.
Errors: `404` not a member of the owning project, or task doesn't exist.

#### `PATCH /api/v1/tasks/:id` — owner/admin/member (not viewer)

Request: any subset of `{ title, description, status, priority, assigneeId, dueDate }`.
Response `200`: updated task.
Errors: `400` invalid body or invalid `assigneeId`, `403` viewer role, `404` not a member / task doesn't exist.

#### `DELETE /api/v1/tasks/:id` — owner/admin only (regular members cannot delete tasks)

Response `204`.
Errors: `403` member/viewer role, `404` not a member / task doesn't exist.
