# Task Management Platform — System Design

## 1. Overview and Requirements

Task Management is a multi-tenant project and task tracker. A user can own or join
projects, create and assign tasks within those projects, tag tasks for categorization,
and discuss tasks through comments. Access to a project's data is governed by the
member's role on that project (owner / admin / member / viewer).

**Functional requirements**
- FR1: A user can create, view, update, and delete tasks in any project they belong to
  (subject to their role), including assigning a task to another member of the same project.
- FR2: A user can attach and remove tags on a task, and browse tasks by tag or status.
- FR3: A user can comment on a task; only the comment's author can edit or delete it
  (project owners/admins may also remove a comment for moderation).

**Non-functional requirements**
- NFR1: Task list endpoints (`GET /projects/:id/tasks`) return within 200ms at p95 for
  projects with up to 10,000 tasks.
- NFR2: Read endpoints maintain 99.9% availability; a single database failover must not
  take writes down for more than 30 seconds.
- NFR3: No API response ever includes `password_hash`, and every write or role-restricted
  read is authorized against the caller's `project_members` role before it executes.

## 2. Entity-Relationship Diagram

```mermaid
erDiagram
    USERS ||--o{ PROJECTS : owns
    USERS ||--o{ PROJECT_MEMBERS : "has membership"
    USERS ||--o{ TASKS : "assigned to"
    USERS ||--o{ COMMENTS : writes
    PROJECTS ||--o{ PROJECT_MEMBERS : has
    PROJECTS ||--o{ TASKS : contains
    TASKS ||--o{ COMMENTS : has
    TASKS ||--o{ TASK_TAGS : "tagged via"
    TAGS ||--o{ TASK_TAGS : "applied via"

    USERS {
        int id PK
        string name
        string email UK "NOT NULL"
        datetime created_at
    }
    PROJECTS {
        int id PK
        string name "NOT NULL"
        int owner_id FK "-> users.id, NOT NULL"
        datetime created_at
    }
    PROJECT_MEMBERS {
        int user_id FK "-> users.id, PK part"
        int project_id FK "-> projects.id, PK part"
        string role "owner/admin/member/viewer, NOT NULL"
    }
    TASKS {
        int id PK
        string title "NOT NULL"
        string description
        string status "todo/in_progress/done"
        int priority "1-5"
        int project_id FK "-> projects.id, NOT NULL"
        int assignee_id FK "-> users.id, NULLABLE"
        date due_date
        datetime created_at
    }
    TAGS {
        int id PK
        string name UK "NOT NULL"
    }
    TASK_TAGS {
        int task_id FK "-> tasks.id, PK part"
        int tag_id FK "-> tags.id, PK part"
    }
    COMMENTS {
        int id PK
        int task_id FK "-> tasks.id, NOT NULL"
        int author_id FK "-> users.id, NOT NULL"
        string body "NOT NULL"
        datetime created_at
    }
```

Notes on cardinality: `project_members` and `task_tags` are pure join tables with
composite primary keys (`user_id, project_id` and `task_id, tag_id` respectively) — no surrogate `id`. `tasks.assignee_id` is nullable: a task can exist with zero assignees, which the `USERS ||--o{ TASKS` relation captures (a user is assigned to zero or many tasks).
