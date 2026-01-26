# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**VulnArk** (v0.1.3) is a modern vulnerability management platform with a separated frontend/backend architecture:
- **Backend**: Go 1.18+ (Gin framework) + MySQL/GORM
- **Frontend**: Vue 3 + Element Plus + ECharts
- **Deployment**: Docker Compose (3 containers: frontend, backend, mysql)

### Core Features
- Vulnerability lifecycle management (discovery to remediation)
- Asset inventory tracking
- Knowledge base for security documentation
- Known vulnerability database (CVE/CWE)
- Automated scanning integration (Nessus, Xray, AWVS, ZAP)
- CI/CD pipeline integration (Jenkins, GitLab CI, GitHub Actions)
- AI-powered risk assessment
- Multi-channel notifications (DingTalk, Feishu, Work WeChat, Email)

## Quick Commands

### Docker Deployment (Recommended)

```bash
# Interactive deployment
chmod +x deploy.sh && ./deploy.sh

# Manual deployment
docker-compose up -d --build

# View logs
docker-compose logs -f backend
docker-compose logs -f frontend
docker-compose logs -f mysql

# Restart / Stop / Remove
docker-compose restart
docker-compose stop
docker-compose down -v  # Removes volumes
```

### Backend Development

```bash
cd backend
go mod download          # Install dependencies
go run main.go           # Run development server
go build -o vulnark main.go  # Build binary

# Admin user creation (if needed)
mysql -u vulnark -p vulnark < create_admin.sql
```

### Frontend Development

```bash
cd frontend
npm install              # Install dependencies
npm run serve            # Dev server with hot reload (port 8081)
npm run build            # Production build
npm run lint             # Code linting
```

## Architecture

### Project Structure

```
VulnArk/
├── backend/
│   ├── cmd/create_admin/     # CLI tool for admin creation
│   ├── config/
│   │   ├── config.yaml       # Local development config
│   │   └── config.docker.yaml # Docker environment config
│   ├── controllers/          # HTTP request handlers (11 files)
│   ├── middleware/           # Auth, admin, CORS (3 files)
│   ├── models/               # GORM data models (10 files)
│   ├── routes/routes.go      # Route definitions
│   ├── utils/                # Database, notifications, helpers
│   ├── main.go               # Application entry
│   └── Dockerfile
├── frontend/
│   ├── src/
│   │   ├── api/              # API client modules (11 files)
│   │   ├── components/       # Reusable Vue components
│   │   ├── layout/           # App layout components
│   │   ├── router/index.js   # Vue Router config
│   │   ├── store/            # Vuex state management
│   │   ├── utils/            # Request, auth, formatting
│   │   ├── views/            # Page components (31 files)
│   │   ├── App.vue
│   │   └── main.js
│   ├── nginx.conf            # Nginx reverse proxy config
│   └── Dockerfile
├── mysql/
│   ├── init/                 # SQL initialization scripts
│   └── my.cnf                # MySQL configuration
├── docs/                     # CI/CD integration docs
├── examples/                 # CI/CD config examples
├── docker-compose.yml
├── deploy.sh                 # Interactive deployment script
└── CLAUDE.md
```

### Backend Architecture

#### Controllers (`backend/controllers/`)

| File | Purpose | Key Endpoints |
|------|---------|---------------|
| `user_controller.go` | User auth & management | Login, GetUserInfo, CRUD users |
| `vulnerability_controller.go` | Vulnerability CRUD | List, Create, Update, Delete, BatchImport |
| `asset_controller.go` | Asset management | CRUD, BatchImport, Export, GetVulnerabilities |
| `vulnerability_assignment_controller.go` | Task distribution | Assign, UpdateStatus, GetMyAssignments |
| `knowledge_controller.go` | Knowledge base | CRUD articles, GetTypes, GetCategories |
| `vulndb_controller.go` | CVE database | CRUD entries, SearchByCVE, BatchImport |
| `scan_controller.go` | Scan task management | CRUD tasks, Start, Cancel, GetResults, Import |
| `integration_controller.go` | CI/CD integration | CRUD integrations, Webhook receiver |
| `dashboard_controller.go` | Analytics | Stats, Trends, Distributions, Activities |
| `settings_controller.go` | System settings | Get/Save settings, Test connections |
| `ai_controller.go` | AI features | PerformRiskAssessment |

#### Models (`backend/models/`)

**User Model** (`user.go`):
```go
// Roles: admin, manager, auditor, operator, viewer
// Password auto-hashed via BeforeCreate hook
// Methods: CheckPassword(), IsAdmin(), CanManage(), CanEdit(), CanAudit()
```

**Vulnerability Model** (`vulnerability.go`):
```go
// Severity: critical, high, medium, low, info
// Status: new, verified, in_progress, fixed, closed, false_positive
// Types: sql_injection, xss, cmd_injection, ssrf, file_upload, etc.
// Has many-to-many relationship with Assets
```

**Asset Model** (`asset.go`):
```go
// Types: host, website, database, application, server, network, cloud, iot, other
// Status: active, inactive, archived
// Importance: critical, high, medium, low
```

**VulnerabilityAssignment** (`vulnerability_assignment.go`):
```go
// Status: pending, accepted, rejected, fixed, pending_retest, closed
// Tracks assignment history with VulnerabilityAssignmentHistory
```

#### Middleware (`backend/middleware/`)

| File | Purpose |
|------|---------|
| `auth.go` | JWT authentication, token generation/validation |
| `admin_auth.go` | Admin-only route protection |
| `cors.go` | CORS headers for cross-origin requests |

#### Routes Structure (`backend/routes/routes.go`)

```
/api/v1
├── /health                    # Health check (public)
├── /auth/login                # Login (public)
├── [JWT Required]
│   ├── /user/info             # Get current user
│   ├── /user/update           # Update profile
│   ├── /admin/*               # Admin-only endpoints
│   ├── /assets/*              # Asset CRUD
│   ├── /vulnerabilities/*     # Vulnerability CRUD
│   ├── /assignments/*         # Assignment management
│   ├── /knowledge/*           # Knowledge base
│   ├── /vulndb/*              # CVE database
│   ├── /scans/*               # Scan tasks
│   ├── /dashboard/*           # Analytics
│   ├── /integrations/*        # CI/CD
│   ├── /settings/*            # System settings (admin)
│   ├── /ai/risk-assessment    # AI features
│   └── /webhooks/:type        # Webhook receivers
```

#### Utilities (`backend/utils/`)

| File | Purpose |
|------|---------|
| `database.go` | MySQL/MongoDB connection with retry (5 attempts, 5s interval) |
| `notification.go` | Multi-channel notifications (750+ lines) |
| `time.go` | CST timezone helpers |
| `random.go` | Random string generation |

### Frontend Architecture

#### API Layer (`frontend/src/api/`)

11 API modules mapping to backend controllers. All use `/api/v1` base path via Axios.

#### State Management (`frontend/src/store/`)

- `modules/user.js` - Token, user info, authentication actions
- `modules/app.js` - Sidebar state, device type

#### Views (`frontend/src/views/`)

```
views/
├── Login.vue, Dashboard.vue, Profile.vue, Settings.vue
├── vulnerability/ (Index, Add, Detail + AssignVulnerability component)
├── asset/ (Index, Add, Edit, Detail)
├── assignment/ (MyAssignments)
├── knowledge/ (Index, Add, Edit, Detail)
├── vulndb/ (Index, Add, Edit, Detail)
├── scan/ (Index, Add, Edit, Detail, Results)
├── integration/ (Index, Add, Detail)
├── user/ (Index)
└── error/ (404)
```

## Configuration

### Environment Variables (Backend)

| Variable | Default | Description |
|----------|---------|-------------|
| `DB_HOST` | `localhost` / `mysql` | Database host |
| `DB_PORT` | `3306` | Database port |
| `DB_USER` | - | Database username |
| `DB_PASSWORD` | - | Database password |
| `DB_NAME` | `vulnark` | Database name |
| `SERVER_HOST` | `0.0.0.0` | Server bind address |
| `SERVER_PORT` | `8080` | Server port |

### Config Files

- **Local dev**: `backend/config/config.yaml`
- **Docker**: `backend/config/config.docker.yaml`

Priority: Environment variables > Config file

### Vue Config (`frontend/vue.config.js`)

- Dev server: port 8081
- API proxy: `/api` -> `localhost:8080`
- Path alias: `@` -> `src/`

## Key Development Patterns

### Adding a New Feature

1. **Backend**:
   - Add model in `backend/models/`
   - Add controller in `backend/controllers/`
   - Register routes in `backend/routes/routes.go`
   - Model is auto-migrated on startup in `main.go`

2. **Frontend**:
   - Add API module in `frontend/src/api/`
   - Add view components in `frontend/src/views/`
   - Register routes in `frontend/src/router/index.js`

### API Request/Response Conventions

- Backend uses `snake_case` field names
- Frontend uses `camelCase` (JSON tags handle conversion)
- Date format: ISO 8601
- All responses wrapped: `{"code": 200, "data": {...}, "message": "..."}`

### Authentication Flow

1. Login via `/api/v1/auth/login` returns JWT token
2. Token stored in Cookie (7-day expiry)
3. Axios interceptor adds `Authorization: Bearer <token>`
4. JWT middleware validates on protected routes
5. User info fetched and stored in Vuex

### Role-Based Access Control

| Role | Permissions |
|------|-------------|
| `admin` | Full system access, user management, settings |
| `manager` | Team management, assignment creation |
| `auditor` | View and audit vulnerabilities |
| `operator` | Create/edit vulnerabilities and assets |
| `viewer` | Read-only access |

Methods in User model: `IsAdmin()`, `CanManage()`, `CanEdit()`, `CanAudit()`

## Vulnerability Assignment Workflow

1. Admin/Manager assigns vulnerability to user
2. User receives notification (if configured)
3. Status flow: `pending` -> `accepted` -> `fixed` -> `pending_retest` -> `closed`
4. History tracked in `VulnerabilityAssignmentHistory`
5. Retest feature: assignee can request retest after fixing

## Notification System (`backend/utils/notification.go`)

Supports multiple channels:
- **Work WeChat (企业微信)**: Webhook-based markdown
- **Feishu (飞书)**: Interactive cards with signature
- **DingTalk (钉钉)**: Markdown with HMAC-SHA256 signing
- **Email**: SMTP with HTML templates, TLS support

Events: Asset CRUD, Vulnerability CRUD, Status changes

## CI/CD Integration

### Webhook Endpoint

```
POST /api/v1/webhooks/:type
Header: X-API-Key: <integration_api_key>
Body: { scan results JSON }
```

### Supported Platforms

- Jenkins (`examples/jenkinsfile-example`)
- GitLab CI (`examples/gitlab-ci-example.yml`)
- GitHub Actions (`examples/github-actions-example.yml`)
- Custom webhooks

## Database Schema

Auto-migrated tables (12 total):
- `users`, `vulnerabilities`, `assets`
- `vulnerability_assets` (junction table)
- `vulnerability_assignments`, `vulnerability_assignment_histories`
- `knowledge`, `vuln_dbs`
- `scan_tasks`, `scan_results`
- `ci_integrations`, `integration_histories`
- `settings`

## Important Files for Common Tasks

| Task | Files to Modify |
|------|-----------------|
| Add API endpoint | `controllers/*.go`, `routes/routes.go` |
| Add database model | `models/*.go`, `main.go` (migration) |
| Add frontend page | `views/*.vue`, `router/index.js`, `api/*.js` |
| Modify auth | `middleware/auth.go`, `store/modules/user.js` |
| Add notification channel | `utils/notification.go` |
| Change settings schema | `models/settings.go`, `views/Settings.vue` |

## Security Considerations

1. **Password Hashing**: bcrypt via GORM BeforeCreate hook (never hash manually)
2. **JWT Secret**: Configure strong secret in production
3. **CORS**: Restrict `allowed_origins` in production
4. **Default Admin**: Change `admin/admin123` immediately after first deploy
5. **SQL Injection**: Use GORM parameterized queries (already implemented)
6. **XSS Prevention**: DOMPurify used for markdown rendering in frontend

## Troubleshooting

### MySQL Connection Failed (Docker)

Backend has retry mechanism (5 attempts, 5s interval). If still failing:
```bash
docker-compose logs mysql   # Check MySQL readiness
docker-compose restart backend
```

### Token/Auth Issues

- Check Cookie expiration (7 days)
- Verify JWT secret matches between requests
- Clear browser cookies and re-login

### Build Failures

```bash
# Backend
cd backend && go mod tidy

# Frontend
cd frontend && rm -rf node_modules && npm install
```

## Access URLs

After Docker deployment:
- **Frontend**: http://localhost
- **Backend API**: http://localhost/api or http://localhost:8080
- **MySQL**: localhost:3306

Default credentials: `admin` / `admin123`

## Version History

### v0.1.3 (Current)
- Added retest workflow for vulnerability assignments
- Enhanced notification system

### v0.1.2
- Fixed field name mismatches between frontend/backend
- Fixed date format parsing issues

### v0.1.1
- Added MySQL connection retry mechanism
- Fixed config file path resolution
- Fixed database connection configuration
