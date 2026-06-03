# DATA FLOW FOR MySDLC

---

## 1. AUTH (Register and Login)

### 1.1 Register

```
[Client: Registration Form]
    │
    │  POST /api/auth/register
    │  { name, email, password }
    ▼
[API Gateway / Auth Service]
    │
    ├── Validate input (email format, password strength)
    │       │
    │       └── [FAIL] → 400 Bad Request { errors[] }
    │
    ├── Check: Does email already exist?
    │   ┌────────────────────────┐
    │   │  DB: users             │
    │   │  SELECT WHERE email=?  │
    │   └────────────────────────┘
    │       │
    │       └── [EXISTS] → 409 Conflict { message: "Email already registered" }
    │
    ├── Hash password (bcrypt)
    │
    ├── INSERT new user record
    │   ┌─────────────────────────────────────────────────┐
    │   │  DB: users                                      │
    │   │  { id, name, email, password_hash, created_at } │
    │   └─────────────────────────────────────────────────┘
    │
    ├── Generate JWT (access_token) + Refresh Token
    │
    └── [SUCCESS] → 201 Created
            { access_token, refresh_token, user: { id, name, email } }

[Client]
    └── Store tokens (memory / httpOnly cookie)
        └── Redirect → Dashboard
```

---

### 1.2 Login

```
[Client: Login Form]
    │
    │  POST /api/auth/login
    │  { email, password }
    ▼
[API Gateway / Auth Service]
    │
    ├── Validate input
    │       └── [FAIL] → 400 Bad Request
    │
    ├── Lookup user by email
    │   ┌────────────────────────┐
    │   │  DB: users             │
    │   │  SELECT WHERE email=?  │
    │   └────────────────────────┘
    │       └── [NOT FOUND] → 401 Unauthorized
    │
    ├── Compare password with password_hash (bcrypt.compare)
    │       └── [MISMATCH] → 401 Unauthorized
    │
    ├── Generate JWT (access_token, short-lived ~15m)
    ├── Generate Refresh Token (long-lived ~7d, stored in DB)
    │   ┌──────────────────────────────────────────────────┐
    │   │  DB: refresh_tokens                              │
    │   │  { id, user_id, token_hash, expires_at }        │
    │   └──────────────────────────────────────────────────┘
    │
    └── [SUCCESS] → 200 OK
            { access_token, refresh_token, user: { id, name, email } }

[Client]
    └── Store tokens → Redirect → Dashboard
```

---

### 1.3 Token Refresh

```
[Client: access_token expired]
    │
    │  POST /api/auth/refresh
    │  { refresh_token }
    ▼
[Auth Service]
    ├── Validate refresh_token exists & not expired
    │   ┌───────────────────────────┐
    │   │  DB: refresh_tokens       │
    │   │  SELECT WHERE token=?     │
    │   └───────────────────────────┘
    │       └── [INVALID/EXPIRED] → 401 → Force Logout
    │
    ├── Issue new access_token
    └── [SUCCESS] → 200 OK { access_token }
```

---

## 2. CREATE PROJECT

```
[Client: New Project Form]
    │  Inputs: { name, description, sdlc_methodology, team_members[] }
    │
    │  POST /api/projects
    │  Headers: { Authorization: Bearer <access_token> }
    ▼
[API Gateway]
    │
    ├── Authenticate JWT → Extract user_id
    │       └── [INVALID] → 401 Unauthorized
    │
    ├── Validate payload
    │       └── [FAIL] → 400 Bad Request { errors[] }
    │
    ├── INSERT project record
    │   ┌───────────────────────────────────────────────────────────────┐
    │   │  DB: projects                                                 │
    │   │  { id, name, description, methodology, owner_id, created_at, │
    │   │    current_phase, status: "active" }                          │
    │   └───────────────────────────────────────────────────────────────┘
    │
    ├── Assign Owner role to creator
    │   ┌────────────────────────────────────────────────────┐
    │   │  DB: project_members                               │
    │   │  { project_id, user_id, role: "owner" }           │
    │   └────────────────────────────────────────────────────┘
    │
    ├── [Template Engine] Generate phases based on methodology
    │
    │   ┌─ IF Agile ───────────────────────────────────────────────────┐
    │   │   Phases: Planning → Design → Sprint Cycles → Testing →      │
    │   │           Review → Deployment → Maintenance                  │
    │   │   Extras: Backlog table, Sprint table, Kanban board          │
    │   └──────────────────────────────────────────────────────────────┘
    │
    │   ┌─ IF Waterfall ───────────────────────────────────────────────┐
    │   │   Phases: Requirements → System Design → Implementation →    │
    │   │           Testing → Deployment → Maintenance                 │
    │   │   Extras: Sequential dependency locks, Gantt timeline        │
    │   └──────────────────────────────────────────────────────────────┘
    │
    │   ┌─ IF RAD ─────────────────────────────────────────────────────┐
    │   │   Phases: Requirements Planning → User Design →              │
    │   │           Rapid Construction (cycles) → Cutover              │
    │   │   Extras: Prototype iteration tracker, Feedback loop logs    │
    │   └──────────────────────────────────────────────────────────────┘
    │
    │   ┌─ IF Custom ──────────────────────────────────────────────────┐
    │   │   Phases: User-defined (add/remove/rename)                   │
    │   └──────────────────────────────────────────────────────────────┘
    │
    ├── INSERT generated phases
    │   ┌──────────────────────────────────────────────────────────────┐
    │   │  DB: project_phases                                          │
    │   │  { id, project_id, name, order_index, status: "locked" |    │
    │   │    "active" | "complete", unlocked_at }                      │
    │   └──────────────────────────────────────────────────────────────┘
    │   (First phase is set to "active"; all subsequent → "locked")
    │
    ├── Invite team members (if provided)
    │   ┌───────────────────────────────────────────────────────┐
    │   │  DB: project_members                                  │
    │   │  { project_id, user_id, role: "contributor"|"viewer" }│
    │   └───────────────────────────────────────────────────────┘
    │   └── [Email Service] Send invitation emails
    │
    └── [SUCCESS] → 201 Created
            { project_id, name, methodology, phases[], dashboard_url }

[Client]
    └── Redirect → Project Dashboard
```

---

## 3. MANAGE PROJECT (SDLC STEP FOR PROJECT)

### 3.1 View Project Dashboard

```
[Client: Open Project]
    │
    │  GET /api/projects/:project_id
    │  Headers: { Authorization: Bearer <access_token> }
    ▼
[API Gateway]
    ├── Authenticate JWT
    ├── Authorize: Is user a member of this project?
    │   ┌─────────────────────────────────┐
    │   │  DB: project_members            │
    │   │  SELECT WHERE project_id=?      │
    │   │         AND user_id=?           │
    │   └─────────────────────────────────┘
    │       └── [NOT MEMBER] → 403 Forbidden
    │
    ├── Fetch project record + phases + tasks summary
    │   ┌──────────────────────────────────────────────────────────┐
    │   │  DB: projects JOIN project_phases JOIN tasks             │
    │   │  Returns: current_phase, phase statuses,                 │
    │   │           task counts (todo/in-progress/blocked/done),   │
    │   │           overall_progress_%, upcoming_deadlines         │
    │   └──────────────────────────────────────────────────────────┘
    │
    └── [SUCCESS] → 200 OK { project, phases[], tasks_summary, analytics }
```

---

### 3.2 Task Management (within a Phase)

#### Create Task
```
[Client: Add Task Form in Phase]
    │
    │  POST /api/projects/:project_id/phases/:phase_id/tasks
    │  { title, description, assignee_id, priority, due_date }
    ▼
[API Gateway]
    ├── Authenticate + Authorize (owner or contributor)
    ├── Validate phase is "active" (not locked)
    │       └── [LOCKED] → 403 { message: "Phase not yet unlocked" }
    │
    ├── INSERT task
    │   ┌──────────────────────────────────────────────────────────────┐
    │   │  DB: tasks                                                   │
    │   │  { id, phase_id, project_id, title, description,            │
    │   │    assignee_id, priority: low|medium|high|critical,         │
    │   │    status: "todo", due_date, created_at }                    │
    │   └──────────────────────────────────────────────────────────────┘
    │
    └── [SUCCESS] → 201 Created { task }
```

#### Update Task Status
```
[Client: Drag-drop or Status Change]
    │
    │  PATCH /api/tasks/:task_id
    │  { status: "in_progress" | "blocked" | "done" }
    ▼
[API Gateway]
    ├── Authenticate + Authorize
    ├── UPDATE task status
    │   ┌──────────────────────────────────┐
    │   │  DB: tasks                       │
    │   │  UPDATE SET status=?, updated_at │
    │   └──────────────────────────────────┘
    │
    ├── [Analytics Service] Recalculate phase progress %
    │
    └── [SUCCESS] → 200 OK { task }
```

---

### 3.3 Phase Gate Transition

```
[Client: Click "Mark Phase Complete" / "Advance to Next Phase"]
    │
    │  POST /api/projects/:project_id/phases/:phase_id/complete
    │  Headers: { Authorization: Bearer <access_token> }
    ▼
[API Gateway]
    │
    ├── Authenticate + Authorize (owner only)
    │
    ├── Validate current phase is "active"
    │
    ├── [Phase Gate Check]
    │   ├── Are all required tasks in status "done"?
    │   ├── Are all required documents uploaded?
    │   └── [GATE FAIL] → 422 Unprocessable
    │           { message: "Phase gate requirements not met", blockers[] }
    │
    ├── UPDATE current phase → status: "complete", completed_at: now
    │   ┌──────────────────────────┐
    │   │  DB: project_phases      │
    │   │  UPDATE SET status=      │
    │   │  "complete"              │
    │   └──────────────────────────┘
    │
    ├── Determine next phase (order_index + 1)
    │   ┌─────────────────────────────────────┐
    │   │  DB: project_phases                 │
    │   │  SELECT WHERE project_id=?          │
    │   │  ORDER BY order_index               │
    │   │  Find next "locked" phase           │
    │   └─────────────────────────────────────┘
    │
    ├── UPDATE next phase → status: "active", unlocked_at: now
    │
    ├── UPDATE projects.current_phase = next phase name
    │
    ├── [Notification Service] Notify all project members
    │       "🚀 Project X has moved to [Next Phase]"
    │
    └── [SUCCESS] → 200 OK
            { completed_phase, next_phase, project_progress_% }

[Client]
    └── Dashboard re-renders with new active phase unlocked
```

---

### 3.4 Documentation Hub (per Phase)

```
[Client: Upload Document in Phase]
    │
    │  POST /api/projects/:project_id/phases/:phase_id/documents
    │  Content-Type: multipart/form-data
    │  { file, doc_type: "PRD"|"mockup"|"test_script"|"other", title }
    ▼
[API Gateway]
    ├── Authenticate + Authorize
    ├── Validate file type & size
    │
    ├── [Storage Service] Upload to object storage (e.g. S3)
    │       Returns: file_url
    │
    ├── INSERT document record
    │   ┌────────────────────────────────────────────────────────┐
    │   │  DB: phase_documents                                   │
    │   │  { id, phase_id, project_id, title, doc_type,         │
    │   │    file_url, uploaded_by, uploaded_at }                │
    │   └────────────────────────────────────────────────────────┘
    │
    └── [SUCCESS] → 201 Created { document }
```

---

### 3.5 Analytics & Dashboard Aggregation

```
[Client: View Dashboard / Phase Insights]
    │
    │  GET /api/projects/:project_id/analytics
    ▼
[Analytics Service]
    │
    ├── Aggregate task counts per phase
    │   ┌──────────────────────────────────────────────────────────┐
    │   │  DB: tasks GROUP BY phase_id, status                    │
    │   │  → todo_count, in_progress_count, blocked_count,        │
    │   │     done_count per phase                                 │
    │   └──────────────────────────────────────────────────────────┘
    │
    ├── Calculate overall_progress_%
    │       formula: (done_tasks / total_tasks) * 100
    │
    ├── Calculate time_in_phase
    │       formula: now() - phase.unlocked_at  (for active phase)
    │                phase.completed_at - phase.unlocked_at (for done)
    │
    ├── Identify bottlenecks
    │       flag phases where blocked_count > 0
    │       flag phases where time_in_phase > estimated_duration
    │
    └── [SUCCESS] → 200 OK
            {
              overall_progress_%,
              current_phase,
              phases: [{ name, status, time_spent, task_summary }],
              bottlenecks[],
              upcoming_deadlines[]
            }
```

---

## Data Store Summary

| Table              | Key Columns                                                                 |
|--------------------|-----------------------------------------------------------------------------|
| `users`            | id, name, email, password_hash, created_at                                 |
| `refresh_tokens`   | id, user_id, token_hash, expires_at                                         |
| `projects`         | id, name, description, methodology, owner_id, current_phase, status        |
| `project_members`  | project_id, user_id, role (owner / contributor / viewer)                   |
| `project_phases`   | id, project_id, name, order_index, status, unlocked_at, completed_at       |
| `tasks`            | id, phase_id, project_id, title, assignee_id, priority, status, due_date   |
| `phase_documents`  | id, phase_id, project_id, title, doc_type, file_url, uploaded_by           |


---

## 4. ADMIN: SDLC MANAGEMENT

> **Access Control:** All routes in this section require the requesting user to have `role: "admin"` stored in `users.role`. Any non-admin JWT is rejected with `403 Forbidden` at the Admin Guard middleware before reaching the service layer.

---

### 4.1 Admin Guard (Shared Middleware)

```
[Any Admin Request]
    │
    │  Headers: { Authorization: Bearer <access_token> }
    ▼
[Admin Guard Middleware]
    │
    ├── Verify JWT → Extract { user_id, role }
    │       └── [INVALID TOKEN] → 401 Unauthorized
    │
    ├── Check role === "admin"
    │   ┌────────────────────────────────┐
    │   │  DB: users                     │
    │   │  SELECT role WHERE id=user_id  │
    │   └────────────────────────────────┘
    │       └── [role !== "admin"] → 403 Forbidden
    │                { message: "Admin access required" }
    │
    └── [PASS] → Continue to route handler
```

---

### 4.2 SDLC Methodology Management

#### 4.2.1 List All SDLC Methodologies

```
[Admin Client: SDLC Management Page]
    │
    │  GET /api/admin/sdlc
    │  Headers: { Authorization: Bearer <access_token> }
    ▼
[Admin Guard] → [SDLC Service]
    │
    ├── Fetch all SDLC records
    │   ┌──────────────────────────────────────────────────────────┐
    │   │  DB: sdlc_methodologies                                  │
    │   │  SELECT * ORDER BY created_at DESC                       │
    │   └──────────────────────────────────────────────────────────┘
    │
    ├── For each SDLC, count associated steps
    │   ┌──────────────────────────────────────────────────────────┐
    │   │  DB: sdlc_steps                                          │
    │   │  SELECT COUNT(*) GROUP BY sdlc_id                        │
    │   └──────────────────────────────────────────────────────────┘
    │
    └── [SUCCESS] → 200 OK
            {
              sdlc_list: [
                { id, name, description, is_active, step_count, created_at }
              ]
            }
```

---

#### 4.2.2 Create SDLC Methodology

```
[Admin Client: "New SDLC" Form]
    │
    │  POST /api/admin/sdlc
    │  { name, description, is_active: true|false }
    ▼
[Admin Guard] → [SDLC Service]
    │
    ├── Validate payload
    │   ├── name: required, unique, max 100 chars
    │   └── [FAIL] → 400 Bad Request { errors[] }
    │
    ├── Check name uniqueness
    │   ┌──────────────────────────────────────────┐
    │   │  DB: sdlc_methodologies                  │
    │   │  SELECT WHERE name=? (case-insensitive)  │
    │   └──────────────────────────────────────────┘
    │       └── [EXISTS] → 409 Conflict
    │               { message: "SDLC name already exists" }
    │
    ├── INSERT SDLC record
    │   ┌──────────────────────────────────────────────────────────────┐
    │   │  DB: sdlc_methodologies                                      │
    │   │  { id, name, description, is_active, created_by, created_at }│
    │   └──────────────────────────────────────────────────────────────┘
    │
    └── [SUCCESS] → 201 Created
            { id, name, description, is_active, created_at }
```

---

#### 4.2.3 Update SDLC Methodology

```
[Admin Client: Edit SDLC Form]
    │
    │  PUT /api/admin/sdlc/:sdlc_id
    │  { name?, description?, is_active? }
    ▼
[Admin Guard] → [SDLC Service]
    │
    ├── Validate SDLC exists
    │   ┌──────────────────────────────────────┐
    │   │  DB: sdlc_methodologies              │
    │   │  SELECT WHERE id=:sdlc_id            │
    │   └──────────────────────────────────────┘
    │       └── [NOT FOUND] → 404 Not Found
    │
    ├── If name changed → re-check uniqueness
    │       └── [CONFLICT] → 409 Conflict
    │
    ├── [Impact Check] Is this SDLC in use by active projects?
    │   ┌──────────────────────────────────────────────────┐
    │   │  DB: projects                                    │
    │   │  SELECT COUNT(*) WHERE sdlc_id=? AND status=     │
    │   │  "active"                                        │
    │   └──────────────────────────────────────────────────┘
    │   ├── [IN USE + is_active set to false]
    │   │       → 422 Unprocessable
    │   │         { message: "Cannot deactivate: SDLC has active projects",
    │   │           active_project_count }
    │   └── [SAFE] → proceed
    │
    ├── UPDATE SDLC record
    │   ┌──────────────────────────────────────┐
    │   │  DB: sdlc_methodologies              │
    │   │  UPDATE SET name=?, description=?,   │
    │   │  is_active=?, updated_at=now()       │
    │   └──────────────────────────────────────┘
    │
    └── [SUCCESS] → 200 OK { updated sdlc }
```

---

#### 4.2.4 Delete SDLC Methodology

```
[Admin Client: Delete SDLC]
    │
    │  DELETE /api/admin/sdlc/:sdlc_id
    ▼
[Admin Guard] → [SDLC Service]
    │
    ├── Validate SDLC exists
    │       └── [NOT FOUND] → 404 Not Found
    │
    ├── [Safety Check] Are any projects using this SDLC?
    │   ┌────────────────────────────────────────────┐
    │   │  DB: projects                              │
    │   │  SELECT COUNT(*) WHERE sdlc_id=:sdlc_id   │
    │   └────────────────────────────────────────────┘
    │       └── [HAS PROJECTS] → 409 Conflict
    │               { message: "Cannot delete: projects are linked to this SDLC",
    │                 project_count }
    │
    ├── DELETE all sdlc_steps WHERE sdlc_id=:sdlc_id (cascade)
    │   ┌──────────────────────────────────┐
    │   │  DB: sdlc_steps                  │
    │   │  DELETE WHERE sdlc_id=:sdlc_id   │
    │   └──────────────────────────────────┘
    │
    ├── DELETE sdlc record
    │   ┌──────────────────────────────────────────┐
    │   │  DB: sdlc_methodologies                  │
    │   │  DELETE WHERE id=:sdlc_id                │
    │   └──────────────────────────────────────────┘
    │
    └── [SUCCESS] → 200 OK
            { message: "SDLC and all its steps deleted successfully" }
```

---

### 4.3 SDLC Steps Management

> Steps define the ordered phases that belong to an SDLC methodology. When a user creates a project, these steps are cloned into `project_phases`.

#### 4.3.1 List Steps for an SDLC

```
[Admin Client: SDLC Detail Page]
    │
    │  GET /api/admin/sdlc/:sdlc_id/steps
    ▼
[Admin Guard] → [SDLC Steps Service]
    │
    ├── Validate SDLC exists
    │       └── [NOT FOUND] → 404 Not Found
    │
    ├── Fetch all steps ordered by order_index
    │   ┌───────────────────────────────────────────────────────────┐
    │   │  DB: sdlc_steps                                           │
    │   │  SELECT * WHERE sdlc_id=:sdlc_id ORDER BY order_index    │
    │   └───────────────────────────────────────────────────────────┘
    │
    └── [SUCCESS] → 200 OK
            {
              sdlc: { id, name },
              steps: [
                { id, name, description, order_index,
                  is_required, estimated_duration_days,
                  allowed_doc_types[], created_at }
              ]
            }
```

---

#### 4.3.2 Create Step

```
[Admin Client: "Add Step" Form]
    │
    │  POST /api/admin/sdlc/:sdlc_id/steps
    │  {
    │    name,
    │    description,
    │    order_index,
    │    is_required: true|false,
    │    estimated_duration_days,
    │    allowed_doc_types: ["PRD","mockup","test_script","other"]
    │  }
    ▼
[Admin Guard] → [SDLC Steps Service]
    │
    ├── Validate SDLC exists
    │       └── [NOT FOUND] → 404 Not Found
    │
    ├── Validate payload
    │   ├── name: required, max 100 chars
    │   ├── order_index: positive integer
    │   └── [FAIL] → 400 Bad Request { errors[] }
    │
    ├── Check order_index conflict within same SDLC
    │   ┌──────────────────────────────────────────────────────────┐
    │   │  DB: sdlc_steps                                          │
    │   │  SELECT WHERE sdlc_id=? AND order_index=?               │
    │   └──────────────────────────────────────────────────────────┘
    │   └── [CONFLICT] → Shift all steps with order_index >= new value
    │       ┌──────────────────────────────────────────────────────┐
    │       │  DB: sdlc_steps                                      │
    │       │  UPDATE SET order_index = order_index + 1            │
    │       │  WHERE sdlc_id=? AND order_index >= :new_index       │
    │       └──────────────────────────────────────────────────────┘
    │
    ├── INSERT step record
    │   ┌───────────────────────────────────────────────────────────────┐
    │   │  DB: sdlc_steps                                               │
    │   │  { id, sdlc_id, name, description, order_index, is_required,  │
    │   │    estimated_duration_days, allowed_doc_types, created_at }   │
    │   └───────────────────────────────────────────────────────────────┘
    │
    └── [SUCCESS] → 201 Created { step }
```

---

#### 4.3.3 Update Step

```
[Admin Client: Edit Step Form]
    │
    │  PUT /api/admin/sdlc/:sdlc_id/steps/:step_id
    │  { name?, description?, order_index?, is_required?,
    │    estimated_duration_days?, allowed_doc_types[]? }
    ▼
[Admin Guard] → [SDLC Steps Service]
    │
    ├── Validate SDLC + Step exist (and step belongs to SDLC)
    │       └── [NOT FOUND] → 404 Not Found
    │
    ├── If order_index changed → reorder siblings
    │   ┌──────────────────────────────────────────────────────────┐
    │   │  DB: sdlc_steps                                          │
    │   │  Shift affected rows up or down to accommodate new index │
    │   └──────────────────────────────────────────────────────────┘
    │
    ├── [Impact Warning] Check if active projects are on this step
    │   ┌─────────────────────────────────────────────────────────────┐
    │   │  DB: project_phases                                         │
    │   │  SELECT COUNT(*) WHERE sdlc_step_id=? AND status="active"  │
    │   └─────────────────────────────────────────────────────────────┘
    │   └── [HAS ACTIVE] → Include warning in response but still update
    │           { warning: "N active project phases reference this step" }
    │
    ├── UPDATE step record
    │   ┌──────────────────────────────────────────────────────────┐
    │   │  DB: sdlc_steps                                          │
    │   │  UPDATE SET name=?, description=?, order_index=?,        │
    │   │  is_required=?, estimated_duration_days=?,               │
    │   │  allowed_doc_types=?, updated_at=now()                   │
    │   └──────────────────────────────────────────────────────────┘
    │
    └── [SUCCESS] → 200 OK { updated_step, warning? }
```

---

#### 4.3.4 Reorder Steps (Drag & Drop)

```
[Admin Client: Drag-and-drop step reorder]
    │
    │  PATCH /api/admin/sdlc/:sdlc_id/steps/reorder
    │  { ordered_step_ids: [uuid, uuid, uuid, ...] }
    ▼
[Admin Guard] → [SDLC Steps Service]
    │
    ├── Validate all step IDs belong to :sdlc_id
    │   ┌──────────────────────────────────────────────────┐
    │   │  DB: sdlc_steps                                  │
    │   │  SELECT id WHERE sdlc_id=?                       │
    │   └──────────────────────────────────────────────────┘
    │       └── [MISMATCH] → 400 Bad Request
    │
    ├── Bulk UPDATE order_index for each step
    │   ┌──────────────────────────────────────────────────────────┐
    │   │  DB: sdlc_steps (batch update in transaction)            │
    │   │  For each step_id at position i:                         │
    │   │  UPDATE SET order_index = i WHERE id = step_id          │
    │   └──────────────────────────────────────────────────────────┘
    │
    └── [SUCCESS] → 200 OK
            { steps: [ ...reordered list with new order_index ] }
```

---

#### 4.3.5 Delete Step

```
[Admin Client: Delete Step]
    │
    │  DELETE /api/admin/sdlc/:sdlc_id/steps/:step_id
    ▼
[Admin Guard] → [SDLC Steps Service]
    │
    ├── Validate step exists and belongs to SDLC
    │       └── [NOT FOUND] → 404 Not Found
    │
    ├── [Safety Check] Is any active project_phase using this step?
    │   ┌─────────────────────────────────────────────────────────────┐
    │   │  DB: project_phases                                         │
    │   │  SELECT COUNT(*) WHERE sdlc_step_id=? AND status != "done" │
    │   └─────────────────────────────────────────────────────────────┘
    │       └── [IN USE] → 409 Conflict
    │               { message: "Step is referenced by active project phases",
    │                 active_count }
    │
    ├── DELETE step
    │   ┌──────────────────────────────────┐
    │   │  DB: sdlc_steps                  │
    │   │  DELETE WHERE id=:step_id        │
    │   └──────────────────────────────────┘
    │
    ├── Re-normalize order_index for remaining steps
    │   ┌──────────────────────────────────────────────────────────┐
    │   │  DB: sdlc_steps                                          │
    │   │  UPDATE SET order_index = order_index - 1               │
    │   │  WHERE sdlc_id=? AND order_index > :deleted_order_index  │
    │   └──────────────────────────────────────────────────────────┘
    │
    └── [SUCCESS] → 200 OK
            { message: "Step deleted and siblings re-indexed" }
```

---

### 4.4 How Admin SDLC Connects to Project Creation

```
[User Creates a Project with sdlc_id]
    │
    │  POST /api/projects { sdlc_id, ... }
    ▼
[Template Engine]
    │
    ├── Fetch steps for chosen SDLC
    │   ┌────────────────────────────────────────────────────────────┐
    │   │  DB: sdlc_steps                                            │
    │   │  SELECT * WHERE sdlc_id=? AND sdlc.is_active=true         │
    │   │  ORDER BY order_index                                      │
    │   └────────────────────────────────────────────────────────────┘
    │       └── [SDLC inactive or not found] → 422 Unprocessable
    │
    ├── Clone each step → INSERT into project_phases
    │   ┌──────────────────────────────────────────────────────────────────┐
    │   │  DB: project_phases                                              │
    │   │  For each sdlc_step:                                             │
    │   │  { id, project_id,                                               │
    │   │    sdlc_step_id  ← foreign key back to source step              │
    │   │    name          ← copied from sdlc_step.name                   │
    │   │    order_index   ← copied from sdlc_step.order_index            │
    │   │    is_required   ← copied from sdlc_step.is_required            │
    │   │    allowed_doc_types ← copied from sdlc_step.allowed_doc_types  │
    │   │    status: "active" (first) | "locked" (rest)                   │
    │   │    unlocked_at, completed_at: null }                             │
    │   └──────────────────────────────────────────────────────────────────┘
    │
    └── Project phases are now independent snapshots of the SDLC steps.
        Future admin edits to sdlc_steps do NOT retroactively affect
        existing project_phases (snapshot isolation).
```

---

## Updated Data Store Summary

| Table                  | Key Columns                                                                              | Managed By  |
|------------------------|------------------------------------------------------------------------------------------|-------------|
| `users`                | id, name, email, password_hash, role (user\|admin), created_at                          | System      |
| `refresh_tokens`       | id, user_id, token_hash, expires_at                                                      | System      |
| `sdlc_methodologies`   | id, name, description, is_active, created_by, created_at, updated_at                    | **Admin**   |
| `sdlc_steps`           | id, sdlc_id, name, description, order_index, is_required, estimated_duration_days, allowed_doc_types, created_at | **Admin**   |
| `projects`             | id, name, description, sdlc_id, owner_id, current_phase, status, created_at             | User        |
| `project_members`      | project_id, user_id, role (owner\|contributor\|viewer), status (pending, accepted, rejected)                                   | User        |
| `project_phases`       | id, project_id, sdlc_step_id, name, order_index, is_required, allowed_doc_types, status, unlocked_at, completed_at | System (cloned) |
| `tasks`                | id, phase_id, project_id, title, assignee_id, priority, status, due_date                | User        |
| `phase_documents`      | id, phase_id, project_id, title, doc_type, file_url, uploaded_by, uploaded_at           | User        |