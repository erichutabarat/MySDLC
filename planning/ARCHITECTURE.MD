# System Architecture: MySDLC

**Document Version:** 1.0
**Stack:** Next.js · Go · MySQL · Docker · Nginx · Cloudflare · VPS

---

## 1. High-Level Overview

```
                        ┌─────────────────────────────────────────┐
                        │            CLOUDFLARE EDGE              │
                        │   DDoS protection · WAF · CDN · DNS     │
                        └───────────────────┬─────────────────────┘
                                            │ HTTPS (443)
                        ┌───────────────────▼─────────────────────┐
                        │                  VPS                     │
                        │  ┌──────────────────────────────────┐   │
                        │  │           NGINX                  │   │
                        │  │  Reverse Proxy · SSL Termination │   │
                        │  │  Rate Limiting · Static Serving  │   │
                        │  └────────┬──────────────┬──────────┘   │
                        │           │              │               │
                        │  ┌────────▼───┐  ┌──────▼──────────┐   │
                        │  │  Next.js   │  │   Go API        │   │
                        │  │  (Port     │  │   (Port 8080)   │   │
                        │  │   3000)    │  │                 │   │
                        │  └────────────┘  └──────┬──────────┘   │
                        │                         │               │
                        │              ┌──────────▼──────────┐    │
                        │              │       MySQL          │    │
                        │              │   (Port 3306)        │    │
                        │              └─────────────────────┘    │
                        │                                         │
                        │  ┌──────────────────────────────────┐   │
                        │  │     Object Storage (MinIO)       │   │
                        │  │         (Port 9000)              │   │
                        │  └──────────────────────────────────┘   │
                        └─────────────────────────────────────────┘
```

All services run as Docker containers managed by Docker Compose on a single VPS. Cloudflare sits in front as the public-facing edge layer.

---

## 2. Tech Stack

### 2.1 Frontend — Next.js

| Concern              | Choice / Notes                                                    |
|----------------------|-------------------------------------------------------------------|
| Framework            | Next.js (App Router)                                              |
| Rendering            | Server-Side Rendering (SSR) for dashboard pages; Static Generation (SSG) for marketing/landing pages |
| State Management     | React Context + SWR for server-state caching and revalidation     |
| API Communication    | REST calls to the Go backend via `fetch` / Axios                  |
| Auth Tokens          | `access_token` stored in memory; `refresh_token` in `httpOnly` cookie |
| Styling              | Tailwind CSS                                                      |

**Key Next.js conventions used:**
- `app/` directory with nested route layouts
- Server Components for data-heavy pages (dashboard, analytics)
- Client Components for interactive elements (Kanban board, task forms)
- API Route Handlers (`app/api/...`) only used for BFF (Backend for Frontend) patterns when needed (e.g. proxying file uploads)
- Middleware (`middleware.ts`) for client-side auth guard — redirects unauthenticated users before page renders

---

### 2.2 Backend — Go (Golang)

| Concern              | Choice / Notes                                                    |
|----------------------|-------------------------------------------------------------------|
| HTTP Router          | `gin` — lightweight, idiomatic REST routing             |
| Architecture         | Layered: Handler → Service → Repository                          |
| Database             | MySQL                                                            |
| ORM / Query          | `GORM` for typed SQL queries (no heavy ORM; raw SQL for control) |
| Validation           | `go-playground/validator`                                         |
| Auth Middleware      | Custom JWT middleware; Admin Guard middleware (role check)        |
| Password Hashing     | `golang.org/x/crypto/bcrypt`                                      |
| File Uploads         | Multipart parsing → streamed to MinIO via AWS S3 SDK (MinIO is S3-compatible) |

**Package structure:**
```
/cmd
  /server          ← main entrypoint
/internal
  /handler         ← HTTP handlers (thin layer, bind + respond)
  /service         ← business logic
  /repository      ← DB queries (sqlx)
  /middleware      ← JWT auth, admin guard, logging, CORS
  /model           ← structs: User, Project, SDLCMethodology, Step, Task...
  /config          ← env loading (godotenv)
/pkg
  /jwt             ← token generation & validation
  /bcrypt          ← password hashing helpers
  /storage         ← MinIO client wrapper
  /response        ← standardized JSON response helpers
```

---

### 2.3 Database — MySQL

| Concern              | Choice / Notes                                                    |
|----------------------|-------------------------------------------------------------------|
| Engine               | MySQL 8.x (InnoDB)                                               |
| Charset              | `utf8mb4` (full Unicode support)                                  |
| Migrations           | `golang-migrate/migrate` — versioned, up/down SQL files           |
| Connection Pool      | Managed by `database/sql` (Go stdlib) via `sqlx`; `MaxOpenConns`, `MaxIdleConns` configured |
| Backups              | Daily `mysqldump` to a compressed file; stored in MinIO or remote SFTP |

**Schema overview** (see `FLOW.MD` for full column details):
```
users
refresh_tokens
sdlc_methodologies          ← admin-managed
sdlc_steps                  ← admin-managed, ordered by order_index
projects
project_members
project_phases              ← cloned from sdlc_steps at project creation
tasks
phase_documents
```

**Indexing strategy:**
- Primary keys: `UUID` (CHAR 36) or `BIGINT AUTO_INCREMENT` — choose BIGINT for high-write tables (tasks, logs)
- Foreign key indexes on all `_id` columns
- Composite index on `project_phases(project_id, status)` for dashboard queries
- Index on `tasks(phase_id, status)` for task list + progress calculation

---

## 3. Infrastructure & Hosting

### 3.1 VPS Specification (Recommended Baseline)

| Resource   | Minimum          | Recommended       |
|------------|------------------|-------------------|
| CPU        | 2 vCPU           | 4 vCPU            |
| RAM        | 2 GB             | 4–8 GB            |
| Disk       | 40 GB SSD        | 80 GB SSD         |
| OS         | Ubuntu 22.04 LTS | Ubuntu 22.04 LTS  |
| Network    | 1 Gbps port      | 1 Gbps port       |

Providers: DigitalOcean, Hetzner, Linode, or Vultr.

---

### 3.2 Networking & Port Mapping

```
Public Internet → Cloudflare → VPS Public IP

VPS Port Layout (Docker internal → Host):
┌───────────────────┬──────────────┬─────────────────────────────────┐
│ Service           │ Host Port    │ Notes                           │
├───────────────────┼──────────────┼─────────────────────────────────┤
│ Nginx             │ 80, 443      │ Only ports exposed to internet  │
│ Next.js           │ 3000         │ Internal only (Nginx proxies)   │
│ Go API            │ 8080         │ Internal only (Nginx proxies)   │
│ MySQL             │ 3306         │ Internal only (no host binding) │
│ MinIO API         │ 9000         │ Internal only                   │
│ MinIO Console     │ 9001         │ Internal only (admin access)    │
└───────────────────┴──────────────┴─────────────────────────────────┘
```

MySQL and MinIO are **never** bound to the host network — only reachable within the Docker Compose network.

---

## 4. DevOps & Deployment

### 4.1 Docker Compose

All services are defined in `docker-compose.yml` with a shared internal network (`mysdlc_net`):

```yaml
# docker-compose.yml (abridged)
services:

  nginx:
    image: nginx:alpine
    ports: ["80:80", "443:443"]
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/certs:/etc/nginx/certs        # Cloudflare Origin Cert
    depends_on: [frontend, api]
    networks: [mysdlc_net]

  frontend:
    build: ./frontend
    environment:
      - NEXT_PUBLIC_API_URL=https://api.mysdlc.com
    networks: [mysdlc_net]

  api:
    build: ./backend
    environment:
      - DB_DSN=mysdlc:secret@tcp(db:3306)/mysdlc
      - JWT_SECRET=${JWT_SECRET}
      - MINIO_ENDPOINT=minio:9000
    depends_on: [db, minio]
    networks: [mysdlc_net]

  db:
    image: mysql:8.0
    environment:
      - MYSQL_DATABASE=mysdlc
      - MYSQL_USER=mysdlc
      - MYSQL_PASSWORD=${DB_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
    networks: [mysdlc_net]

  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    environment:
      - MINIO_ROOT_USER=${MINIO_USER}
      - MINIO_ROOT_PASSWORD=${MINIO_PASSWORD}
    volumes:
      - minio_data:/data
    networks: [mysdlc_net]

volumes:
  mysql_data:
  minio_data:

networks:
  mysdlc_net:
    driver: bridge
```

---

### 4.2 Nginx Configuration

Nginx acts as the reverse proxy, handling TLS termination and routing by subdomain:

```
mysdlc.com        →  Next.js  (port 3000)    ← frontend
api.mysdlc.com    →  Go API   (port 8080)    ← backend REST
```

**Key Nginx directives:**
- `proxy_pass` to upstream services by Docker service name
- `client_max_body_size 20m` — for document uploads
- `proxy_read_timeout 60s` — for long analytics queries
- `add_header X-Frame-Options DENY` and other security headers
- Rate limiting: `limit_req_zone` on `/api/auth/` routes (e.g. 10 req/min per IP)

**TLS:** Cloudflare Origin Certificate installed on Nginx. Cloudflare ↔ VPS connection uses Full (Strict) SSL mode.

---

### 4.3 CI/CD — GitHub Actions

**Workflow: `.github/workflows/deploy.yml`**

```
Trigger: push to main branch
                │
    ┌───────────▼────────────┐
    │   1. Lint & Test        │
    │   - go vet, go test     │
    │   - next build (type    │
    │     check)              │
    └───────────┬────────────┘
                │ [PASS]
    ┌───────────▼────────────┐
    │   2. Build Docker       │
    │      Images             │
    │   - docker build api    │
    │   - docker build        │
    │     frontend            │
    └───────────┬────────────┘
                │
    ┌───────────▼────────────┐
    │   3. Push to Registry  │
    │   GitHub Container     │
    │   Registry (ghcr.io)   │
    └───────────┬────────────┘
                │
    ┌───────────▼────────────┐
    │   4. SSH into VPS      │
    │   - git pull           │
    │   - docker compose     │
    │     pull               │
    │   - docker compose     │
    │     up -d --build      │
    │   - run DB migrations  │
    └───────────┬────────────┘
                │
    ┌───────────▼────────────┐
    │   5. Health Check      │
    │   curl /api/health     │
    │   [FAIL] → rollback    │
    │   notify via Slack/    │
    │   email                │
    └────────────────────────┘
```

**Secrets stored in GitHub Actions Secrets:** `VPS_SSH_KEY`, `VPS_HOST`, `JWT_SECRET`, `DB_PASSWORD`, `MINIO_PASSWORD`

---

### 4.4 Cloudflare Configuration

| Feature           | Configuration                                                       |
|-------------------|---------------------------------------------------------------------|
| DNS               | A record → VPS IP (proxied, orange cloud)                          |
| SSL Mode          | Full (Strict) — requires Origin Certificate on Nginx               |
| WAF               | OWASP Core Ruleset enabled; custom rule to block non-Cloudflare IPs hitting VPS directly |
| Rate Limiting     | Cloudflare-level rate limit on `/api/auth/*` (extra layer above Nginx) |
| Caching           | Cache static Next.js assets (`/_next/static/*`) at edge            |
| DDoS              | Automatic DDoS protection (Cloudflare default)                      |

**VPS Firewall (ufw):** Only accept incoming traffic from Cloudflare IP ranges on port 80/443. Block direct access to the VPS IP.

---

## 5. Identity & Session Management

### 5.1 Password Hashing — Bcrypt

- Algorithm: `bcrypt` with cost factor `12`
- Passwords are **never stored in plaintext** — only `password_hash` persisted in `users` table
- On login: `bcrypt.CompareHashAndPassword(storedHash, inputPassword)`
- On register: `bcrypt.GenerateFromPassword(password, 12)`

### 5.2 JWT Session Strategy

```
┌────────────────────────────────────────────────────────────────┐
│ access_token                                                    │
│  - Algorithm: HS256 (HMAC-SHA256)                              │
│  - Payload: { user_id, role, exp }                             │
│  - TTL: 15 minutes                                             │
│  - Storage: JavaScript memory (never localStorage)             │
└────────────────────────────────────────────────────────────────┘
┌────────────────────────────────────────────────────────────────┐
│ refresh_token                                                   │
│  - Opaque random string (crypto/rand, 32 bytes, hex-encoded)   │
│  - TTL: 7 days                                                 │
│  - Storage: httpOnly, Secure, SameSite=Strict cookie           │
│  - Persisted: hashed (SHA-256) in DB refresh_tokens table      │
│  - Rotation: new refresh_token issued on every use             │
└────────────────────────────────────────────────────────────────┘
```

**Token refresh flow:** Next.js middleware checks `access_token` expiry on every request. If expired, it silently calls `POST /api/auth/refresh` using the `httpOnly` cookie before retrying the original request.

**Logout:** Deletes the `refresh_token` record from DB + clears the cookie. All active sessions for a user can be invalidated by deleting all their refresh_token rows.

---

## 6. Media & Object Storage

### 6.1 Strategy — Self-Hosted MinIO

MinIO is an S3-compatible object store running as a Docker container on the same VPS. It handles all user-uploaded files (PRDs, mockups, test scripts, etc.).

**Bucket layout:**
```
mysdlc-documents/
  └── {project_id}/
        └── {phase_id}/
              └── {document_id}_{original_filename}
```

**Upload flow:**
```
[Client]
    │  POST /api/projects/:id/phases/:id/documents
    │  multipart/form-data { file, title, doc_type }
    ▼
[Go API]
    ├── Validate file type (allowlist: pdf, doc, docx, png, jpg, md)
    ├── Validate file size (max 20 MB)
    ├── Generate unique object key:
    │       {project_id}/{phase_id}/{uuid}_{filename}
    ├── Stream file directly to MinIO
    │       (never written to local disk)
    ├── MinIO returns ETag on success
    ├── INSERT phase_documents record { file_url, ... }
    └── Return { document_id, file_url }

[Client accesses file]
    │  GET /api/documents/:document_id/download
    ▼
[Go API]
    ├── Authorize: user is member of the project
    ├── Generate presigned MinIO URL (TTL: 5 minutes)
    └── Redirect 302 → presigned URL
```

Presigned URLs mean file bytes **never pass through the Go API** on download — the client fetches directly from MinIO. Access is still gated by the API authorization check.

---

## 7. Monitoring & Logging

### 7.1 Health Checks

```
GET /api/health
→ 200 OK {
    status: "ok",
    db: "connected",
    storage: "connected",
    uptime_seconds: 3600
  }
```

- Go API pings MySQL (`SELECT 1`) and MinIO (`BucketExists`) on each health check
- Nginx `health_check` directive polls `/api/health` every 10 seconds
- Docker Compose `healthcheck` restarts containers that fail 3 consecutive checks
- GitHub Actions post-deploy step calls `/api/health` and rolls back if non-200

### 7.2 Application Logging

**Go API — Structured Logging:**
- Library: `zerolog` or `slog` (stdlib, Go 1.21+)
- Format: JSON (machine-readable, easy to parse)
- Log levels: `DEBUG` (local), `INFO` (production), `ERROR` always
- Every HTTP request logs: `method`, `path`, `status`, `latency_ms`, `user_id`, `request_id`
- Sensitive fields (passwords, tokens) are **never logged**

```json
{
  "level": "info",
  "time": "2025-01-15T10:23:44Z",
  "request_id": "a1b2c3d4",
  "user_id": "uuid-here",
  "method": "POST",
  "path": "/api/projects",
  "status": 201,
  "latency_ms": 42
}
```

**Log storage:**
- Logs written to `stdout` → captured by Docker
- Rotate with `docker logs` or forward to a log file via Docker logging driver
- On VPS: `journald` or a bind-mounted log volume

### 7.3 Performance Tracking

| Metric                        | How Tracked                                                  |
|-------------------------------|--------------------------------------------------------------|
| HTTP response latency         | Logged per-request in Go middleware                         |
| Slow DB queries               | MySQL `slow_query_log` enabled (threshold: 1s)              |
| Container resource usage      | `docker stats` (manual) or Netdata agent on VPS             |
| Disk usage (MySQL + MinIO)    | Cron job alerts when disk > 80%                             |
| Uptime                        | Uptime Robot (free tier) — pings `/api/health` every 5 min |

### 7.4 Error Tracking

- Go panics recovered by middleware → logged as `ERROR` with stack trace
- Unhandled errors return `500 Internal Server Error` with a `request_id` (never expose internal error details to client)
- Optional: integrate **Sentry** Go SDK for automatic error capture and alerting

---

## 8. Environment Configuration

All secrets and environment-specific values are managed via `.env` files on the VPS (never committed to Git). Docker Compose reads them via `env_file` or `environment` blocks.

**`.env` (VPS only):**
```env
# App
JWT_SECRET=<strong-random-256-bit-secret>
APP_ENV=production

# Database
DB_DSN=mysdlc:password@tcp(db:3306)/mysdlc?parseTime=true

# MinIO
MINIO_ENDPOINT=minio:9000
MINIO_USER=admin
MINIO_PASSWORD=<strong-password>
MINIO_BUCKET=mysdlc-documents

# BCrypt
BCRYPT_COST=12
```

**`.env.local` (Next.js, VPS):**
```env
NEXT_PUBLIC_API_URL=https://api.mysdlc.com
```

---

## 9. Architecture Decision Notes

| Decision                        | Rationale                                                                                       |
|---------------------------------|-------------------------------------------------------------------------------------------------|
| Go for backend                  | High performance, low memory footprint, excellent concurrency — well-suited for a VPS deployment |
| MySQL over PostgreSQL            | Simpler ops on a single VPS; wide hosting support; sufficient for relational SDLC data          |
| MinIO over S3                    | Self-hosted = zero egress costs; S3-compatible API means easy migration to S3/R2 later          |
| Docker Compose over K8s          | Single VPS doesn't need orchestration overhead; Compose is simple, reproducible, and sufficient |
| Cloudflare in front of Nginx     | Free DDoS protection, WAF, and CDN without additional infra cost                               |
| Snapshot isolation for phases    | Cloning `sdlc_steps` into `project_phases` prevents admin edits from breaking live projects     |
| httpOnly cookies for refresh token | Prevents XSS access to long-lived tokens; access_token in memory limits exposure window       |