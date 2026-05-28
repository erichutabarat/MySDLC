# API Reference: MySDLC

**Base URL:** `https://api.mysdlc.com`
**Version:** v1
**Format:** All request and response bodies are `application/json` unless stated otherwise.

---

## Conventions

### Response Envelope

Every response — success or error — uses the same wrapper shape:

```json
{
  "success": true,
  "data": { ... },
  "error": null,
  "meta": { ... }
}
```

| Field     | Type            | Description                                              |
|-----------|-----------------|----------------------------------------------------------|
| `success` | `boolean`       | `true` on 2xx, `false` on 4xx/5xx                       |
| `data`    | `object\|null`  | The response payload. `null` on error.                   |
| `error`   | `object\|null`  | Error detail. `null` on success.                         |
| `meta`    | `object\|null`  | Pagination info or extra context. Omitted if not needed. |

**Error object shape:**
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "fields": [
      { "field": "email", "message": "Must be a valid email address" }
    ]
  }
}
```

### Authentication

Protected routes require a Bearer token in the `Authorization` header:
```
Authorization: Bearer <access_token>
```

Admin routes additionally require `role: "admin"` in the JWT payload. Regular user tokens on admin routes return `403 Forbidden`.

### HTTP Status Codes

| Code  | Meaning                                                              |
|-------|----------------------------------------------------------------------|
| `200` | OK — successful read or update                                       |
| `201` | Created — resource successfully created                              |
| `400` | Bad Request — validation failed or malformed payload                 |
| `401` | Unauthorized — missing, invalid, or expired token                    |
| `403` | Forbidden — authenticated but insufficient permissions               |
| `404` | Not Found — resource does not exist                                  |
| `409` | Conflict — duplicate resource or constraint violation                |
| `422` | Unprocessable — business rule violation (e.g. phase gate not met)    |
| `500` | Internal Server Error — unexpected server failure                    |

### Field Naming

- All JSON keys are `snake_case`
- All IDs are `string` (UUID v4)
- All timestamps are ISO 8601 UTC: `"2025-01-15T10:23:44Z"`
- Optional fields in request bodies are marked with `?`
- Nullable fields in responses are marked with `| null`

---

## Endpoint Index

| Group            | Method   | Path                                                         | Auth     |
|------------------|----------|--------------------------------------------------------------|----------|
| **Auth**         | POST     | `/api/auth/register`                                         | Public   |
|                  | POST     | `/api/auth/login`                                            | Public   |
|                  | POST     | `/api/auth/refresh`                                          | Public   |
|                  | POST     | `/api/auth/logout`                                           | User     |
| **Users**        | GET      | `/api/users/me`                                              | User     |
|                  | PATCH    | `/api/users/me`                                              | User     |
| **Projects**     | GET      | `/api/projects`                                              | User     |
|                  | POST     | `/api/projects`                                              | User     |
|                  | GET      | `/api/projects/:project_id`                                  | User     |
|                  | PATCH    | `/api/projects/:project_id`                                  | Owner    |
|                  | DELETE   | `/api/projects/:project_id`                                  | Owner    |
| **Members**      | GET      | `/api/projects/:project_id/members`                          | Member   |
|                  | POST     | `/api/projects/:project_id/members`                          | Owner    |
|                  | PATCH    | `/api/projects/:project_id/members/:user_id`                 | Owner    |
|                  | DELETE   | `/api/projects/:project_id/members/:user_id`                 | Owner    |
| **Phases**       | GET      | `/api/projects/:project_id/phases`                           | Member   |
|                  | GET      | `/api/projects/:project_id/phases/:phase_id`                 | Member   |
|                  | POST     | `/api/projects/:project_id/phases/:phase_id/complete`        | Owner    |
| **Tasks**        | GET      | `/api/projects/:project_id/phases/:phase_id/tasks`           | Member   |
|                  | POST     | `/api/projects/:project_id/phases/:phase_id/tasks`           | Member+  |
|                  | GET      | `/api/tasks/:task_id`                                        | Member   |
|                  | PATCH    | `/api/tasks/:task_id`                                        | Member+  |
|                  | DELETE   | `/api/tasks/:task_id`                                        | Owner    |
| **Documents**    | GET      | `/api/projects/:project_id/phases/:phase_id/documents`       | Member   |
|                  | POST     | `/api/projects/:project_id/phases/:phase_id/documents`       | Member+  |
|                  | GET      | `/api/documents/:document_id/download`                       | Member   |
|                  | DELETE   | `/api/documents/:document_id`                                | Owner    |
| **Analytics**    | GET      | `/api/projects/:project_id/analytics`                        | Member   |
| **Admin: SDLC**  | GET      | `/api/admin/sdlc`                                            | Admin    |
|                  | POST     | `/api/admin/sdlc`                                            | Admin    |
|                  | GET      | `/api/admin/sdlc/:sdlc_id`                                   | Admin    |
|                  | PUT      | `/api/admin/sdlc/:sdlc_id`                                   | Admin    |
|                  | DELETE   | `/api/admin/sdlc/:sdlc_id`                                   | Admin    |
| **Admin: Steps** | GET      | `/api/admin/sdlc/:sdlc_id/steps`                             | Admin    |
|                  | POST     | `/api/admin/sdlc/:sdlc_id/steps`                             | Admin    |
|                  | PUT      | `/api/admin/sdlc/:sdlc_id/steps/:step_id`                    | Admin    |
|                  | PATCH    | `/api/admin/sdlc/:sdlc_id/steps/reorder`                     | Admin    |
|                  | DELETE   | `/api/admin/sdlc/:sdlc_id/steps/:step_id`                    | Admin    |
| **System**       | GET      | `/api/health`                                                | Public   |

> **Auth levels:** `Public` = no token required · `User` = any valid JWT · `Member` = project member · `Member+` = owner or contributor · `Owner` = project owner only · `Admin` = admin role in JWT

---

## 1. Auth

### POST `/api/auth/register`

Register a new user account.

**Auth:** Public

**Request Body:**
```json
{
  "name": "Jane Doe",
  "email": "jane@example.com",
  "password": "Str0ng!Pass"
}
```

| Field      | Type     | Rules                                              |
|------------|----------|----------------------------------------------------|
| `name`     | `string` | Required. 2–100 chars.                             |
| `email`    | `string` | Required. Valid email format.                      |
| `password` | `string` | Required. Min 8 chars, 1 uppercase, 1 number.      |

**Response `201 Created`:**
```json
{
  "success": true,
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIs...",
    "user": {
      "id": "uuid-v4",
      "name": "Jane Doe",
      "email": "jane@example.com",
      "role": "user",
      "created_at": "2025-01-15T10:00:00Z"
    }
  },
  "error": null
}
```

> `refresh_token` is set as an `httpOnly` cookie (`Set-Cookie` header), not in the response body.

**Errors:**

| Status | Code                 | Condition                      |
|--------|----------------------|--------------------------------|
| `400`  | `VALIDATION_ERROR`   | Invalid email, weak password   |
| `409`  | `EMAIL_EXISTS`       | Email already registered       |

---

### POST `/api/auth/login`

Authenticate an existing user.

**Auth:** Public

**Request Body:**
```json
{
  "email": "jane@example.com",
  "password": "Str0ng!Pass"
}
```

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIs...",
    "user": {
      "id": "uuid-v4",
      "name": "Jane Doe",
      "email": "jane@example.com",
      "role": "user",
      "created_at": "2025-01-15T10:00:00Z"
    }
  },
  "error": null
}
```

**Errors:**

| Status | Code                 | Condition                         |
|--------|----------------------|-----------------------------------|
| `400`  | `VALIDATION_ERROR`   | Missing fields                    |
| `401`  | `INVALID_CREDENTIALS`| Email not found or wrong password |

---

### POST `/api/auth/refresh`

Issue a new `access_token` using the `refresh_token` cookie.

**Auth:** Public (uses `httpOnly` cookie automatically)

**Request Body:** _(empty)_

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIs..."
  },
  "error": null
}
```

> A new `refresh_token` cookie is also rotated on every successful refresh.

**Errors:**

| Status | Code               | Condition                          |
|--------|--------------------|------------------------------------|
| `401`  | `INVALID_TOKEN`    | Cookie missing, invalid, or expired|

---

### POST `/api/auth/logout`

Invalidate the current session. Deletes the refresh token from DB and clears the cookie.

**Auth:** User

**Request Body:** _(empty)_

**Response `200 OK`:**
```json
{
  "success": true,
  "data": { "message": "Logged out successfully" },
  "error": null
}
```

---

## 2. Users

### GET `/api/users/me`

Get the currently authenticated user's profile.

**Auth:** User

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "id": "uuid-v4",
    "name": "Jane Doe",
    "email": "jane@example.com",
    "role": "user",
    "created_at": "2025-01-15T10:00:00Z"
  },
  "error": null
}
```

---

### PATCH `/api/users/me`

Update the current user's profile.

**Auth:** User

**Request Body:**
```json
{
  "name?": "Jane Smith",
  "password?": {
    "current": "Str0ng!Pass",
    "new": "N3wStr0ng!Pass"
  }
}
```

| Field              | Type     | Rules                                           |
|--------------------|----------|-------------------------------------------------|
| `name`             | `string` | Optional. 2–100 chars.                          |
| `password.current` | `string` | Required if changing password.                  |
| `password.new`     | `string` | Required if changing password. Min 8 chars.     |

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "id": "uuid-v4",
    "name": "Jane Smith",
    "email": "jane@example.com",
    "role": "user",
    "updated_at": "2025-01-15T11:00:00Z"
  },
  "error": null
}
```

**Errors:**

| Status | Code                   | Condition                     |
|--------|------------------------|-------------------------------|
| `400`  | `VALIDATION_ERROR`     | Invalid fields                |
| `401`  | `INVALID_CREDENTIALS`  | Current password is wrong     |

---

## 3. Projects

### GET `/api/projects`

List all projects the authenticated user is a member of.

**Auth:** User

**Query Parameters:**

| Param     | Type     | Default | Description                                  |
|-----------|----------|---------|----------------------------------------------|
| `status`  | `string` | —       | Filter by status: `active`, `archived`       |
| `page`    | `int`    | `1`     | Page number                                  |
| `limit`   | `int`    | `10`    | Items per page (max 50)                      |

**Response `200 OK`:**
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid-v4",
      "name": "Final Year Project",
      "description": "My thesis project",
      "sdlc_id": "uuid-v4",
      "sdlc_name": "Agile",
      "current_phase": "Design",
      "status": "active",
      "progress_percent": 35,
      "role": "owner",
      "created_at": "2025-01-15T10:00:00Z"
    }
  ],
  "error": null,
  "meta": {
    "page": 1,
    "limit": 10,
    "total": 3,
    "total_pages": 1
  }
}
```

---

### POST `/api/projects`

Create a new project. Clones SDLC steps into project phases automatically.

**Auth:** User

**Request Body:**
```json
{
  "name": "Final Year Project",
  "description": "My thesis management project",
  "sdlc_id": "uuid-v4",
  "members?": [
    { "user_id": "uuid-v4", "role": "contributor" },
    { "user_id": "uuid-v4", "role": "viewer" }
  ]
}
```

| Field         | Type       | Rules                                                              |
|---------------|------------|--------------------------------------------------------------------|
| `name`        | `string`   | Required. 3–150 chars.                                             |
| `description` | `string`   | Optional. Max 500 chars.                                           |
| `sdlc_id`     | `string`   | Required. Must reference an active SDLC methodology.              |
| `members`     | `array`    | Optional. Each: `user_id` (UUID) + `role` (contributor\|viewer).  |

**Response `201 Created`:**
```json
{
  "success": true,
  "data": {
    "id": "uuid-v4",
    "name": "Final Year Project",
    "description": "My thesis management project",
    "sdlc_id": "uuid-v4",
    "sdlc_name": "Agile",
    "owner_id": "uuid-v4",
    "current_phase": "Planning",
    "status": "active",
    "progress_percent": 0,
    "phases": [
      {
        "id": "uuid-v4",
        "name": "Planning",
        "order_index": 1,
        "status": "active",
        "is_required": true
      },
      {
        "id": "uuid-v4",
        "name": "Design",
        "order_index": 2,
        "status": "locked",
        "is_required": true
      }
    ],
    "created_at": "2025-01-15T10:00:00Z"
  },
  "error": null
}
```

**Errors:**

| Status | Code                  | Condition                             |
|--------|-----------------------|---------------------------------------|
| `400`  | `VALIDATION_ERROR`    | Missing/invalid fields                |
| `404`  | `SDLC_NOT_FOUND`      | `sdlc_id` does not exist              |
| `422`  | `SDLC_INACTIVE`       | Referenced SDLC is inactive           |

---

### GET `/api/projects/:project_id`

Get full project detail including phases and task summary.

**Auth:** Member

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "id": "uuid-v4",
    "name": "Final Year Project",
    "description": "My thesis management project",
    "sdlc_id": "uuid-v4",
    "sdlc_name": "Agile",
    "owner_id": "uuid-v4",
    "current_phase": "Design",
    "status": "active",
    "progress_percent": 35,
    "my_role": "contributor",
    "phases": [
      {
        "id": "uuid-v4",
        "name": "Planning",
        "order_index": 1,
        "status": "complete",
        "is_required": true,
        "task_summary": {
          "total": 5,
          "done": 5,
          "in_progress": 0,
          "blocked": 0,
          "todo": 0
        },
        "completed_at": "2025-01-20T09:00:00Z"
      },
      {
        "id": "uuid-v4",
        "name": "Design",
        "order_index": 2,
        "status": "active",
        "is_required": true,
        "task_summary": {
          "total": 8,
          "done": 2,
          "in_progress": 3,
          "blocked": 1,
          "todo": 2
        },
        "unlocked_at": "2025-01-20T09:00:00Z",
        "completed_at": null
      }
    ],
    "member_count": 3,
    "created_at": "2025-01-15T10:00:00Z"
  },
  "error": null
}
```

---

### PATCH `/api/projects/:project_id`

Update project name, description, or status.

**Auth:** Owner

**Request Body:**
```json
{
  "name?": "Updated Project Name",
  "description?": "Updated description",
  "status?": "archived"
}
```

| Field         | Type     | Rules                                        |
|---------------|----------|----------------------------------------------|
| `name`        | `string` | Optional. 3–150 chars.                       |
| `description` | `string` | Optional. Max 500 chars.                     |
| `status`      | `string` | Optional. `active` or `archived`.            |

**Response `200 OK`:** _(returns updated project object, same shape as GET)_

---

### DELETE `/api/projects/:project_id`

Permanently delete a project and all its phases, tasks, and documents.

**Auth:** Owner

**Response `200 OK`:**
```json
{
  "success": true,
  "data": { "message": "Project deleted successfully" },
  "error": null
}
```

---

## 4. Project Members

### GET `/api/projects/:project_id/members`

List all members of a project.

**Auth:** Member

**Response `200 OK`:**
```json
{
  "success": true,
  "data": [
    {
      "user_id": "uuid-v4",
      "name": "Jane Doe",
      "email": "jane@example.com",
      "role": "owner",
      "joined_at": "2025-01-15T10:00:00Z"
    },
    {
      "user_id": "uuid-v4",
      "name": "John Smith",
      "email": "john@example.com",
      "role": "contributor",
      "joined_at": "2025-01-16T08:00:00Z"
    }
  ],
  "error": null
}
```

---

### POST `/api/projects/:project_id/members`

Invite a user to the project by email.

**Auth:** Owner

**Request Body:**
```json
{
  "email": "john@example.com",
  "role": "contributor"
}
```

| Field   | Type     | Rules                                           |
|---------|----------|-------------------------------------------------|
| `email` | `string` | Required. Must match a registered user account. |
| `role`  | `string` | Required. `contributor` or `viewer`.            |

**Response `201 Created`:**
```json
{
  "success": true,
  "data": {
    "user_id": "uuid-v4",
    "name": "John Smith",
    "email": "john@example.com",
    "role": "contributor",
    "joined_at": "2025-01-16T08:00:00Z"
  },
  "error": null
}
```

**Errors:**

| Status | Code                | Condition                            |
|--------|---------------------|--------------------------------------|
| `404`  | `USER_NOT_FOUND`    | No account with that email           |
| `409`  | `ALREADY_MEMBER`    | User is already a project member     |

---

### PATCH `/api/projects/:project_id/members/:user_id`

Change a member's role.

**Auth:** Owner

**Request Body:**
```json
{
  "role": "viewer"
}
```

| Field  | Type     | Rules                                 |
|--------|----------|---------------------------------------|
| `role` | `string` | Required. `contributor` or `viewer`.  |

**Response `200 OK`:** _(returns updated member object)_

**Errors:**

| Status | Code                  | Condition                               |
|--------|-----------------------|-----------------------------------------|
| `400`  | `CANNOT_CHANGE_OWNER` | Cannot change the owner's role          |
| `404`  | `MEMBER_NOT_FOUND`    | User is not a member of this project    |

---

### DELETE `/api/projects/:project_id/members/:user_id`

Remove a member from the project.

**Auth:** Owner

**Response `200 OK`:**
```json
{
  "success": true,
  "data": { "message": "Member removed successfully" },
  "error": null
}
```

**Errors:**

| Status | Code                   | Condition                          |
|--------|------------------------|------------------------------------|
| `400`  | `CANNOT_REMOVE_OWNER`  | Owner cannot remove themselves     |
| `404`  | `MEMBER_NOT_FOUND`     | User is not a member               |

---

## 5. Phases

### GET `/api/projects/:project_id/phases`

List all phases for a project with their task summaries.

**Auth:** Member

**Response `200 OK`:**
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid-v4",
      "name": "Planning",
      "order_index": 1,
      "status": "complete",
      "is_required": true,
      "allowed_doc_types": ["PRD", "other"],
      "estimated_duration_days": 7,
      "task_summary": { "total": 5, "done": 5, "in_progress": 0, "blocked": 0, "todo": 0 },
      "unlocked_at": "2025-01-15T10:00:00Z",
      "completed_at": "2025-01-20T09:00:00Z"
    },
    {
      "id": "uuid-v4",
      "name": "Design",
      "order_index": 2,
      "status": "active",
      "is_required": true,
      "allowed_doc_types": ["mockup", "other"],
      "estimated_duration_days": 10,
      "task_summary": { "total": 8, "done": 2, "in_progress": 3, "blocked": 1, "todo": 2 },
      "unlocked_at": "2025-01-20T09:00:00Z",
      "completed_at": null
    }
  ],
  "error": null
}
```

---

### GET `/api/projects/:project_id/phases/:phase_id`

Get a single phase with full task list and uploaded documents.

**Auth:** Member

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "id": "uuid-v4",
    "name": "Design",
    "order_index": 2,
    "status": "active",
    "is_required": true,
    "allowed_doc_types": ["mockup", "other"],
    "estimated_duration_days": 10,
    "tasks": [
      {
        "id": "uuid-v4",
        "title": "Create wireframes",
        "description": "Low-fidelity screens for all main flows",
        "assignee": { "id": "uuid-v4", "name": "John Smith" },
        "priority": "high",
        "status": "in_progress",
        "due_date": "2025-01-25T00:00:00Z",
        "created_at": "2025-01-20T10:00:00Z"
      }
    ],
    "documents": [
      {
        "id": "uuid-v4",
        "title": "UI Mockup v1",
        "doc_type": "mockup",
        "uploaded_by": { "id": "uuid-v4", "name": "Jane Doe" },
        "uploaded_at": "2025-01-21T14:00:00Z"
      }
    ],
    "unlocked_at": "2025-01-20T09:00:00Z",
    "completed_at": null
  },
  "error": null
}
```

---

### POST `/api/projects/:project_id/phases/:phase_id/complete`

Mark the current phase as complete and unlock the next phase.

**Auth:** Owner

**Request Body:** _(empty)_

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "completed_phase": {
      "id": "uuid-v4",
      "name": "Design",
      "status": "complete",
      "completed_at": "2025-02-01T09:00:00Z"
    },
    "next_phase": {
      "id": "uuid-v4",
      "name": "Implementation",
      "status": "active",
      "unlocked_at": "2025-02-01T09:00:00Z"
    },
    "project_progress_percent": 50
  },
  "error": null
}
```

**Errors:**

| Status | Code                       | Condition                                   |
|--------|----------------------------|---------------------------------------------|
| `400`  | `PHASE_NOT_ACTIVE`         | Phase is not currently active               |
| `422`  | `GATE_INCOMPLETE_TASKS`    | Required tasks are not all done             |
| `422`  | `GATE_MISSING_DOCUMENTS`   | Required documents not uploaded             |
| `422`  | `FINAL_PHASE`              | This is the last phase; project completes   |

> When `FINAL_PHASE` is returned alongside `200 OK`, `next_phase` is `null` and the project `status` becomes `"complete"`.

---

## 6. Tasks

### GET `/api/projects/:project_id/phases/:phase_id/tasks`

List all tasks in a phase.

**Auth:** Member

**Query Parameters:**

| Param      | Type     | Description                                        |
|------------|----------|----------------------------------------------------|
| `status`   | `string` | Filter: `todo`, `in_progress`, `blocked`, `done`   |
| `priority` | `string` | Filter: `low`, `medium`, `high`, `critical`        |
| `assignee` | `string` | Filter by `user_id`                                |

**Response `200 OK`:**
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid-v4",
      "title": "Create wireframes",
      "description": "Low-fidelity screens for all main flows",
      "assignee": { "id": "uuid-v4", "name": "John Smith" },
      "priority": "high",
      "status": "in_progress",
      "due_date": "2025-01-25T00:00:00Z",
      "created_at": "2025-01-20T10:00:00Z",
      "updated_at": "2025-01-21T09:00:00Z"
    }
  ],
  "error": null
}
```

---

### POST `/api/projects/:project_id/phases/:phase_id/tasks`

Create a new task in a phase. Phase must be `active`.

**Auth:** Member+ (owner or contributor)

**Request Body:**
```json
{
  "title": "Create wireframes",
  "description?": "Low-fidelity screens for all main flows",
  "assignee_id?": "uuid-v4",
  "priority": "high",
  "due_date?": "2025-01-25T00:00:00Z"
}
```

| Field         | Type        | Rules                                                    |
|---------------|-------------|----------------------------------------------------------|
| `title`       | `string`    | Required. 3–200 chars.                                   |
| `description` | `string`    | Optional. Max 1000 chars.                                |
| `assignee_id` | `string`    | Optional. Must be a project member.                      |
| `priority`    | `string`    | Required. `low`, `medium`, `high`, or `critical`.        |
| `due_date`    | `string`    | Optional. ISO 8601 date.                                 |

**Response `201 Created`:** _(returns created task object)_

**Errors:**

| Status | Code                  | Condition                                |
|--------|-----------------------|------------------------------------------|
| `403`  | `PHASE_LOCKED`        | Phase is not active                      |
| `404`  | `ASSIGNEE_NOT_MEMBER` | Assignee is not a project member         |

---

### GET `/api/tasks/:task_id`

Get a single task by ID.

**Auth:** Member (of the task's project)

**Response `200 OK`:** _(returns single task object with full detail)_

---

### PATCH `/api/tasks/:task_id`

Update a task's fields or status.

**Auth:** Member+ (owner or contributor)

**Request Body:**
```json
{
  "title?": "Updated title",
  "description?": "Updated description",
  "assignee_id?": "uuid-v4",
  "priority?": "critical",
  "status?": "blocked",
  "due_date?": "2025-01-28T00:00:00Z"
}
```

| Field         | Type        | Rules                                                      |
|---------------|-------------|----------------------------------------------------------|
| `title`       | `string`    | Optional. 3–200 chars.                                   |
| `description` | `string`    | Optional. Max 1000 chars.                                |
| `assignee_id` | `string`    | Optional. Must be a project member. `null` to unassign.  |
| `priority`    | `string`    | Optional. `low`, `medium`, `high`, `critical`.           |
| `status`      | `string`    | Optional. `todo`, `in_progress`, `blocked`, `done`.      |
| `due_date`    | `string`    | Optional. ISO 8601 date. `null` to clear.                |

**Response `200 OK`:** _(returns updated task object)_

---

### DELETE `/api/tasks/:task_id`

Delete a task permanently.

**Auth:** Owner

**Response `200 OK`:**
```json
{
  "success": true,
  "data": { "message": "Task deleted successfully" },
  "error": null
}
```

---

## 7. Documents

### GET `/api/projects/:project_id/phases/:phase_id/documents`

List all documents uploaded to a phase.

**Auth:** Member

**Response `200 OK`:**
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid-v4",
      "title": "UI Mockup v1",
      "doc_type": "mockup",
      "file_name": "mockup_v1.pdf",
      "file_size_bytes": 2048000,
      "uploaded_by": { "id": "uuid-v4", "name": "Jane Doe" },
      "uploaded_at": "2025-01-21T14:00:00Z"
    }
  ],
  "error": null
}
```

---

### POST `/api/projects/:project_id/phases/:phase_id/documents`

Upload a document to a phase.

**Auth:** Member+ (owner or contributor)

**Request:** `multipart/form-data`

| Field      | Type     | Rules                                                                     |
|------------|----------|---------------------------------------------------------------------------|
| `file`     | `file`   | Required. Allowed types: `pdf`, `doc`, `docx`, `png`, `jpg`, `md`. Max 20 MB. |
| `title`    | `string` | Required. 3–150 chars.                                                    |
| `doc_type` | `string` | Required. Must be in phase's `allowed_doc_types`. `PRD`, `mockup`, `test_script`, `other`. |

**Response `201 Created`:**
```json
{
  "success": true,
  "data": {
    "id": "uuid-v4",
    "title": "UI Mockup v1",
    "doc_type": "mockup",
    "file_name": "mockup_v1.pdf",
    "file_size_bytes": 2048000,
    "uploaded_by": { "id": "uuid-v4", "name": "Jane Doe" },
    "uploaded_at": "2025-01-21T14:00:00Z"
  },
  "error": null
}
```

**Errors:**

| Status | Code                    | Condition                                      |
|--------|-------------------------|------------------------------------------------|
| `400`  | `INVALID_FILE_TYPE`     | File extension not in allowlist                |
| `400`  | `FILE_TOO_LARGE`        | File exceeds 20 MB limit                       |
| `400`  | `DOC_TYPE_NOT_ALLOWED`  | `doc_type` not in phase's `allowed_doc_types`  |
| `403`  | `PHASE_LOCKED`          | Phase is not active                            |

---

### GET `/api/documents/:document_id/download`

Get a short-lived presigned download URL for a document. Does not stream the file through the API.

**Auth:** Member (of the document's project)

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "download_url": "https://storage.mysdlc.com/mysdlc-documents/...",
    "expires_at": "2025-01-21T14:05:00Z"
  },
  "error": null
}
```

> The client should immediately redirect to `download_url`. It expires in 5 minutes.

---

### DELETE `/api/documents/:document_id`

Delete a document and remove its object from storage.

**Auth:** Owner

**Response `200 OK`:**
```json
{
  "success": true,
  "data": { "message": "Document deleted successfully" },
  "error": null
}
```

---

## 8. Analytics

### GET `/api/projects/:project_id/analytics`

Get aggregated progress data and phase insights for a project.

**Auth:** Member

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "project_id": "uuid-v4",
    "overall_progress_percent": 35,
    "current_phase": "Design",
    "status": "active",
    "phases": [
      {
        "id": "uuid-v4",
        "name": "Planning",
        "status": "complete",
        "order_index": 1,
        "task_summary": { "total": 5, "done": 5, "in_progress": 0, "blocked": 0, "todo": 0 },
        "time_spent_days": 5,
        "estimated_duration_days": 7,
        "is_overdue": false
      },
      {
        "id": "uuid-v4",
        "name": "Design",
        "status": "active",
        "order_index": 2,
        "task_summary": { "total": 8, "done": 2, "in_progress": 3, "blocked": 1, "todo": 2 },
        "time_spent_days": 3,
        "estimated_duration_days": 10,
        "is_overdue": false
      }
    ],
    "bottlenecks": [
      {
        "phase_id": "uuid-v4",
        "phase_name": "Design",
        "reason": "blocked_tasks",
        "blocked_task_count": 1
      }
    ],
    "upcoming_deadlines": [
      {
        "task_id": "uuid-v4",
        "task_title": "Create wireframes",
        "due_date": "2025-01-25T00:00:00Z",
        "days_remaining": 4,
        "assignee": { "id": "uuid-v4", "name": "John Smith" }
      }
    ]
  },
  "error": null
}
```

---

## 9. Admin — SDLC Methodologies

> All routes in this section require `role: "admin"` in the JWT.

### GET `/api/admin/sdlc`

List all SDLC methodologies.

**Auth:** Admin

**Query Parameters:**

| Param       | Type      | Description                          |
|-------------|-----------|--------------------------------------|
| `is_active` | `boolean` | Filter by active/inactive status     |

**Response `200 OK`:**
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid-v4",
      "name": "Agile",
      "description": "Iterative, sprint-based development methodology",
      "is_active": true,
      "step_count": 7,
      "created_by": "uuid-v4",
      "created_at": "2025-01-01T00:00:00Z",
      "updated_at": "2025-01-10T00:00:00Z"
    }
  ],
  "error": null
}
```

---

### POST `/api/admin/sdlc`

Create a new SDLC methodology.

**Auth:** Admin

**Request Body:**
```json
{
  "name": "Kanban",
  "description": "Continuous flow methodology with WIP limits",
  "is_active": true
}
```

| Field         | Type      | Rules                           |
|---------------|-----------|---------------------------------|
| `name`        | `string`  | Required. Unique. Max 100 chars.|
| `description` | `string`  | Optional. Max 500 chars.        |
| `is_active`   | `boolean` | Optional. Default `true`.       |

**Response `201 Created`:**
```json
{
  "success": true,
  "data": {
    "id": "uuid-v4",
    "name": "Kanban",
    "description": "Continuous flow methodology with WIP limits",
    "is_active": true,
    "step_count": 0,
    "created_at": "2025-01-15T10:00:00Z"
  },
  "error": null
}
```

**Errors:**

| Status | Code                 | Condition                        |
|--------|----------------------|----------------------------------|
| `400`  | `VALIDATION_ERROR`   | Missing/invalid fields           |
| `409`  | `NAME_EXISTS`        | SDLC name already taken          |

---

### GET `/api/admin/sdlc/:sdlc_id`

Get a single SDLC methodology with its full step list.

**Auth:** Admin

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "id": "uuid-v4",
    "name": "Agile",
    "description": "Iterative, sprint-based development",
    "is_active": true,
    "steps": [
      {
        "id": "uuid-v4",
        "name": "Planning",
        "description": "Define project scope and backlog",
        "order_index": 1,
        "is_required": true,
        "estimated_duration_days": 7,
        "allowed_doc_types": ["PRD", "other"]
      }
    ],
    "created_at": "2025-01-01T00:00:00Z",
    "updated_at": "2025-01-10T00:00:00Z"
  },
  "error": null
}
```

---

### PUT `/api/admin/sdlc/:sdlc_id`

Update an SDLC methodology's metadata.

**Auth:** Admin

**Request Body:**
```json
{
  "name?": "Agile (Scrum)",
  "description?": "Updated description",
  "is_active?": false
}
```

**Response `200 OK`:** _(returns updated SDLC object)_

**Errors:**

| Status | Code                       | Condition                                          |
|--------|----------------------------|----------------------------------------------------|
| `404`  | `SDLC_NOT_FOUND`           | SDLC does not exist                                |
| `409`  | `NAME_EXISTS`              | New name conflicts with another SDLC               |
| `422`  | `HAS_ACTIVE_PROJECTS`      | Cannot deactivate; active projects exist           |

---

### DELETE `/api/admin/sdlc/:sdlc_id`

Delete an SDLC and all its steps. Blocked if any projects reference it.

**Auth:** Admin

**Response `200 OK`:**
```json
{
  "success": true,
  "data": { "message": "SDLC and all its steps deleted successfully" },
  "error": null
}
```

**Errors:**

| Status | Code                    | Condition                                 |
|--------|-------------------------|-------------------------------------------|
| `404`  | `SDLC_NOT_FOUND`        | SDLC does not exist                       |
| `409`  | `HAS_LINKED_PROJECTS`   | Projects are using this SDLC; cannot delete|

---

## 10. Admin — SDLC Steps

### GET `/api/admin/sdlc/:sdlc_id/steps`

List all steps for an SDLC, ordered by `order_index`.

**Auth:** Admin

**Response `200 OK`:**
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid-v4",
      "sdlc_id": "uuid-v4",
      "name": "Planning",
      "description": "Define project scope and backlog",
      "order_index": 1,
      "is_required": true,
      "estimated_duration_days": 7,
      "allowed_doc_types": ["PRD", "other"],
      "active_project_phase_count": 3,
      "created_at": "2025-01-01T00:00:00Z"
    }
  ],
  "error": null
}
```

---

### POST `/api/admin/sdlc/:sdlc_id/steps`

Create a new step in an SDLC. Sibling steps are shifted automatically if `order_index` conflicts.

**Auth:** Admin

**Request Body:**
```json
{
  "name": "Testing",
  "description": "QA, unit testing, and integration testing phase",
  "order_index": 5,
  "is_required": true,
  "estimated_duration_days": 14,
  "allowed_doc_types": ["test_script", "other"]
}
```

| Field                     | Type       | Rules                                                                |
|---------------------------|------------|----------------------------------------------------------------------|
| `name`                    | `string`   | Required. Max 100 chars.                                             |
| `description`             | `string`   | Optional. Max 500 chars.                                             |
| `order_index`             | `integer`  | Required. Positive integer. Siblings shift if conflict.              |
| `is_required`             | `boolean`  | Optional. Default `true`.                                            |
| `estimated_duration_days` | `integer`  | Optional. Positive integer.                                          |
| `allowed_doc_types`       | `string[]` | Optional. Subset of: `PRD`, `mockup`, `test_script`, `other`.       |

**Response `201 Created`:** _(returns created step object)_

**Errors:**

| Status | Code               | Condition              |
|--------|--------------------|------------------------|
| `400`  | `VALIDATION_ERROR` | Missing/invalid fields |
| `404`  | `SDLC_NOT_FOUND`   | SDLC does not exist    |

---

### PUT `/api/admin/sdlc/:sdlc_id/steps/:step_id`

Update a step. Returns a warning if active project phases reference it.

**Auth:** Admin

**Request Body:**
```json
{
  "name?": "QA & Testing",
  "description?": "Updated description",
  "order_index?": 4,
  "is_required?": false,
  "estimated_duration_days?": 10,
  "allowed_doc_types?": ["test_script", "other"]
}
```

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "id": "uuid-v4",
    "name": "QA & Testing",
    "order_index": 4,
    "is_required": false,
    "estimated_duration_days": 10,
    "allowed_doc_types": ["test_script", "other"],
    "updated_at": "2025-02-01T10:00:00Z",
    "warning": "3 active project phases reference this step and may be affected"
  },
  "error": null
}
```

> `warning` is `null` if no active project phases are affected.

**Errors:**

| Status | Code              | Condition                             |
|--------|-------------------|---------------------------------------|
| `404`  | `STEP_NOT_FOUND`  | Step not found or doesn't belong here |

---

### PATCH `/api/admin/sdlc/:sdlc_id/steps/reorder`

Bulk-reorder all steps in an SDLC via drag-and-drop. All step IDs must belong to the SDLC.

**Auth:** Admin

**Request Body:**
```json
{
  "ordered_step_ids": [
    "uuid-step-1",
    "uuid-step-3",
    "uuid-step-2",
    "uuid-step-4"
  ]
}
```

| Field              | Type       | Rules                                                          |
|--------------------|------------|----------------------------------------------------------------|
| `ordered_step_ids` | `string[]` | Required. Must contain ALL step IDs for this SDLC, no extras. |

**Response `200 OK`:**
```json
{
  "success": true,
  "data": [
    { "id": "uuid-step-1", "name": "Planning", "order_index": 1 },
    { "id": "uuid-step-3", "name": "Design", "order_index": 2 },
    { "id": "uuid-step-2", "name": "Analysis", "order_index": 3 },
    { "id": "uuid-step-4", "name": "Testing", "order_index": 4 }
  ],
  "error": null
}
```

**Errors:**

| Status | Code                    | Condition                                          |
|--------|-------------------------|----------------------------------------------------|
| `400`  | `INVALID_STEP_IDS`      | IDs don't match this SDLC's steps exactly          |

---

### DELETE `/api/admin/sdlc/:sdlc_id/steps/:step_id`

Delete a step. Blocked if active project phases reference it. Remaining steps are re-indexed.

**Auth:** Admin

**Response `200 OK`:**
```json
{
  "success": true,
  "data": { "message": "Step deleted and siblings re-indexed" },
  "error": null
}
```

**Errors:**

| Status | Code                     | Condition                                        |
|--------|--------------------------|--------------------------------------------------|
| `404`  | `STEP_NOT_FOUND`         | Step does not exist                              |
| `409`  | `STEP_IN_USE`            | Active project phases reference this step        |

---

## 11. System

### GET `/api/health`

System health check. Used by Docker, Nginx, and uptime monitors.

**Auth:** Public

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "status": "ok",
    "db": "connected",
    "storage": "connected",
    "version": "1.0.0",
    "uptime_seconds": 86400
  },
  "error": null
}
```

**Response `503 Service Unavailable`** _(if DB or storage is down)_:
```json
{
  "success": false,
  "data": null,
  "error": {
    "code": "SERVICE_UNAVAILABLE",
    "message": "Database connection failed",
    "db": "disconnected",
    "storage": "connected"
  }
}
```

---

## Appendix: Shared Types

### TaskStatus
`"todo"` · `"in_progress"` · `"blocked"` · `"done"`

### TaskPriority
`"low"` · `"medium"` · `"high"` · `"critical"`

### ProjectStatus
`"active"` · `"archived"` · `"complete"`

### PhaseStatus
`"locked"` · `"active"` · `"complete"`

### ProjectRole
`"owner"` · `"contributor"` · `"viewer"`

### UserRole
`"user"` · `"admin"`

### DocType
`"PRD"` · `"mockup"` · `"test_script"` · `"other"`