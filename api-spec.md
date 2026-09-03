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

## 1. Auth

#### `POST /api/v1/auth/register` — Public
Request: `{ "name": "Ada Lovelace", "email": "ada@example.com", "password": "s3cret123" }`
Response `201`: `{ "id": 1, "name": "Ada Lovelace", "email": "ada@example.com", "createdAt": "2026-09-01T10:00:00Z" }`
Errors: `400` invalid body (weak password, bad email format), `409` email already registered.

#### `POST /api/v1/auth/login` — Public
Request: `{ "email": "ada@example.com", "password": "s3cret123" }`
Response `200`: `{ "accessToken": "...", "refreshToken": "...", "user": { "id": 1, "name": "Ada Lovelace", "email": "ada@example.com" } }`
Errors: `401` wrong email or password (deliberately the same error for both, to avoid revealing which registered emails exist).

#### `POST /api/v1/auth/refresh` — Public (requires a valid refresh token in the body)
Request: `{ "refreshToken": "..." }`
Response `200`: `{ "accessToken": "...", "refreshToken": "..." }` (refresh tokens rotate on every use).
Errors: `401` expired, revoked, or malformed refresh token.

#### `POST /api/v1/auth/logout` — Auth
Request: `{ "refreshToken": "..." }`
Response `204`: no body. Revokes the given refresh token.
Errors: `401` unauthenticated.

## 2. Users

#### `GET /api/v1/users/me` — Auth
Response `200`: `{ "id": 1, "name": "Ada Lovelace", "email": "ada@example.com", "createdAt": "..." }`

#### `PATCH /api/v1/users/me` — Auth
Request: `{ "name": "Ada L." }`
Response `200`: updated user object.
Errors: `400` invalid body.

#### `GET /api/v1/users/:id` — Auth
Response `200`: `{ "id": 2, "name": "Grace Hopper" }` (email omitted for users other than yourself).
Errors: `404` no such user.

## 3. Projects

#### `POST /api/v1/projects` — Auth
Creates a project; the caller becomes `owner` (a `project_members` row is created automatically).
Request: `{ "name": "Website Revamp" }`
Response `201`: `{ "id": 42, "name": "Website Revamp", "ownerId": 1, "createdAt": "..." }`
Errors: `400` invalid body.

#### `GET /api/v1/projects` — Auth
Lists projects the caller is a member of. Uses the [pagination convention](#0-pagination-filtering-and-sorting-convention) (`status` filter not applicable).
Response `200`: paged envelope of project objects, each including the caller's `role`.

#### `GET /api/v1/projects/:id` — owner/admin/member/viewer
Response `200`: project object.
Errors: `404` not a member / doesn't exist.

#### `PATCH /api/v1/projects/:id` — owner/admin
Request: `{ "name": "Website Revamp v2" }`
Response `200`: updated project.
Errors: `400` invalid body, `403` member but not owner/admin, `404` not a member.

#### `DELETE /api/v1/projects/:id` — owner
Response `204`: no body.
Errors: `403` member but not owner, `404` not a member.

## 4. Project Members

#### `GET /api/v1/projects/:id/members` — owner/admin/member/viewer
Paged list (convention above). Response item: `{ "userId": 2, "projectId": 42, "role": "member", "user": { "id": 2, "name": "Grace Hopper" } }`
Errors: `404` not a member.

#### `POST /api/v1/projects/:id/members` — owner/admin
Request: `{ "userId": 2, "role": "member" }`
Response `201`: the created membership object.
Errors: `400` invalid role value, `403` member but insufficient role, `404` project or target user not found, `409` user is already a member.

#### `PATCH /api/v1/projects/:id/members/:userId` — owner/admin
Request: `{ "role": "admin" }`
Response `200`: updated membership.
Errors: `400` invalid role, `403` insufficient role, `404` not found, `409` cannot demote the last remaining owner.

#### `DELETE /api/v1/projects/:id/members/:userId` — owner/admin, or the member removing themself
Response `204`.
Errors: `403` insufficient role and not self, `404` not found, `409` cannot remove the last owner.


## 5. Tasks

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

## 6. Tags

#### `GET /api/v1/tags` — Auth
Paged list of all tags (tags are global, not per-project).

#### `POST /api/v1/tags` — Auth
Request: `{ "name": "backend" }`
Response `201`: created tag.
Errors: `400` invalid body, `409` tag name already exists.

#### `POST /api/v1/tasks/:id/tags` — owner/admin/member of the task's project (not viewer)
Request: `{ "tagId": 4 }`
Response `201`: `{ "taskId": 7, "tagId": 4 }`
Errors: `403` viewer role, `404` task or tag not found / not a member, `409` tag already attached to this task.

#### `DELETE /api/v1/tasks/:id/tags/:tagId` — owner/admin/member of the task's project (not viewer)
Response `204`.
Errors: `403` viewer role, `404` not attached / not a member.

## 7. Comments

#### `GET /api/v1/tasks/:id/comments` — member of the task's project (any role)
Paged list (convention above), default sort `createdAt:asc`.
Errors: `404` not a member / task doesn't exist.

#### `POST /api/v1/tasks/:id/comments` — owner/admin/member (not viewer)
Request: `{ "body": "Looks good, shipping this." }`
Response `201`: created comment, `authorId` set from the caller's token (never accepted from the body).
Errors: `400` empty body, `403` viewer role, `404` not a member / task doesn't exist.

#### `PATCH /api/v1/comments/:id` — comment author only
Request: `{ "body": "Edited: looks good, shipping Monday." }`
Response `200`: updated comment.
Errors: `400` empty body, `403` not the author, `404` not a member of the owning project / comment doesn't exist.

#### `DELETE /api/v1/comments/:id` — comment author, or owner/admin of the project (moderation)
Response `204`.
Errors: `403` neither the author nor owner/admin, `404` not a member / comment doesn't exist.
