# TeamSphere API — Setup Guide & Documentation

## What is TeamSphere?

TeamSphere is a **multi-tenant SaaS Project Management System**. Multiple organizations (tenants) share the same platform, each with complete data isolation. Think of it as a per-company Jira — each company has its own users, projects, tasks, and roles.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Runtime | Node.js + TypeScript |
| Framework | Express 5 |
| Database | MongoDB + Mongoose |
| Auth | JWT (access + refresh tokens) |
| Frontend | React 19 + Vite + Redux Toolkit + MUI |
| Email | Nodemailer (SMTP) |
| Docs | Swagger (auto-generated) |

---

## Prerequisites

- Node.js v18+
- MongoDB running locally (or Atlas URI)
- npm or yarn

---

## Backend Setup

### 1. Install dependencies

```bash
cd teamsphere_backend
npm install
```

### 2. Create `.env` file

```env
PORT=8000
MONGO_URI=mongodb://localhost:27017/teamsphere
JWT_ACCESS_SECRET=your_access_secret_here
JWT_REFRESH_SECRET=your_refresh_secret_here
NODE_ENV=development
ACCESS_TOKEN_EXPIRE=15m
REFRESH_TOKEN_EXPIRE=7d
SMTP_USER=your_email@gmail.com
SMTP_PASS=your_app_password
SMTP_FROM=TeamSphere <your_email@gmail.com>
```

### 3. Seed the database (optional but recommended)

Creates system roles, permissions, and a super admin account:

```bash
npm run seed
```

Super admin credentials after seeding:
- **Email**: `superadmin@teamsphere.com`
- **Password**: `SuperAdmin@123`

### 4. Start the server

```bash
# Development (hot reload)
npm run dev

# Production
npm run build && npm start
```

Server runs at: `http://localhost:8000`

Swagger UI: `http://localhost:8000/api-docs`

---

## Frontend Setup

### 1. Install dependencies

```bash
cd teamsphere_frontend
npm install
```

### 2. Create `.env` file

```env
VITE_API_URL=http://localhost:8000/api/v1/
```

### 3. Start the dev server

```bash
npm run dev
```

Frontend runs at: `http://localhost:5173`

---

## Postman Setup

### Import the collection

1. Open Postman
2. Click **Import** → select `TeamSphere_API.postman_collection.json`
3. The collection includes all 36 endpoints organized in 8 folders

### Collection variables (auto-managed)

| Variable | Description | Auto-set by |
|----------|-------------|-------------|
| `baseUrl` | `http://localhost:8000/api/v1` | Pre-configured |
| `accessToken` | JWT access token (15 min expiry) | Login / Register |
| `refreshToken` | JWT refresh token (7 day expiry) | Login / Register |
| `tokenExpiry` | Expiry timestamp (ms) | Login / Register |
| `tenantId` | Current tenant MongoDB ID | Login / Register |
| `userId` | Current user MongoDB ID | Login / Register |
| `memberId` | Second user ID | Invite User |
| `projectId` | Current project ID | Create Project |
| `taskId` | Current task ID | Create Task |
| `commentId` | Current comment ID | Add Comment |
| `roleId` | Custom role ID | Create Role |
| `permissionId` | Custom permission ID | Create Permission |

### Recommended first-time flow

```
1. Auth → Register Tenant         (creates tenant + admin, saves all tokens)
2. User Management → Invite User  (creates a second user, saves memberId)
3. Project Management → Create Project  (saves projectId)
4. Task Management → Create Task        (saves taskId)
5. Task Management → Add Comment        (saves commentId)
6. Dashboard → Get Dashboard Stats
```

### Token auto-refresh

The collection pre-request script automatically refreshes the access token when it is about to expire (within 60 seconds). No manual refresh needed during a session.

---

## Authentication

### How it works

1. **Register** or **Login** to receive an `accessToken` (15 min) and `refreshToken` (7 days)
2. Include `Authorization: Bearer <accessToken>` on all protected routes
3. Include `X-Tenant-ID: <tenantId>` on all tenant-scoped routes
4. When the access token expires, call `POST /auth/refresh` with the refresh token

### Login types

| Login Type | X-Tenant-ID Header | Description |
|-----------|-------------------|-------------|
| Tenant User | Required | Regular users within a tenant |
| Super Admin | **Omit entirely** | Platform admin — no tenant context |

### JWT payload

```json
// Access token
{ "userId": "...", "tenantId": "..." | null, "role": "admin" }

// Refresh token
{ "userId": "..." }
```

---

## Role & Access Control

| Role | Tenant Access | Project Access | Task Access |
|------|--------------|---------------|------------|
| `super-admin` | Full CRUD on tenants | None | None |
| `admin` | All users, all projects | Create, update, delete | Create, update, delete |
| `manager` | Read users | Create, update projects | Create, update, delete tasks |
| `employee` | Read users | Read projects (member only) | Read; update status only |

---

## API Reference

**Base URL:** `http://localhost:8000/api/v1`

### Auth (`/auth`)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/auth/register` | None | Register tenant + admin |
| POST | `/auth/login` | None | Login (+ X-Tenant-ID for tenant users) |
| POST | `/auth/refresh` | None | Refresh access token |
| GET | `/auth/me` | Bearer | Get current user profile |
| POST | `/auth/logout` | Bearer | Logout + revoke refresh token |

### Tenant Management (`/tenants`) — Super Admin Only

| Method | Path | Description |
|--------|------|-------------|
| GET | `/tenants` | List all tenants (paginated) |
| POST | `/tenants` | Create tenant + admin |
| GET | `/tenants/:id` | Get tenant by ID |
| PUT | `/tenants/:id` | Update name/status |
| DELETE | `/tenants/:id` | Soft-delete tenant |

### User Management (`/users`) — Tenant Scoped

| Method | Path | Role | Description |
|--------|------|------|-------------|
| GET | `/users` | All | List users (paginated) |
| POST | `/users/invite` | Admin | Invite/create user |
| GET | `/users/:id` | All | Get user by ID |
| PUT | `/users/:id` | Admin | Update name/role/status/avatar |
| PATCH | `/users/me/password` | All | Change own password |
| DELETE | `/users/:id` | Admin | Soft-delete user |

### Project Management (`/projects`) — Tenant Scoped

| Method | Path | Role | Description |
|--------|------|------|-------------|
| GET | `/projects` | All | List projects (paginated) |
| POST | `/projects` | Admin, Manager | Create project |
| GET | `/projects/:id` | All | Get project by ID |
| PUT | `/projects/:id` | Admin, Manager | Update project |
| POST | `/projects/:id/members` | Admin, Manager | Add members |
| DELETE | `/projects/:id/members/:userId` | Admin, Manager | Remove member |
| DELETE | `/projects/:id` | Admin | Soft-delete project (cascades to tasks) |

### Task Management (`/projects/:projectId/tasks`) — Tenant Scoped

| Method | Path | Role | Description |
|--------|------|------|-------------|
| GET | `/projects/:projectId/tasks` | All | List tasks (paginated, filterable) |
| POST | `/projects/:projectId/tasks` | Admin, Manager | Create task |
| GET | `/projects/:projectId/tasks/:id` | All | Get task by ID |
| PUT | `/projects/:projectId/tasks/:id` | All* | Update task (* employees: status only) |
| DELETE | `/projects/:projectId/tasks/:id` | Admin, Manager | Soft-delete task |
| POST | `/projects/:projectId/tasks/:id/comments` | All | Add comment |
| DELETE | `/projects/:projectId/tasks/:id/comments/:commentId` | Own only | Delete own comment |

### Role Management (`/roles`) — Tenant Scoped

| Method | Path | Description |
|--------|------|-------------|
| GET | `/roles` | List system + custom roles |
| POST | `/roles` | Create custom role |
| GET | `/roles/:id` | Get role by ID |
| PUT | `/roles/:id` | Update custom role |
| DELETE | `/roles/:id` | Soft-delete custom role |

### Permission Management (`/permissions`) — Tenant Scoped

| Method | Path | Description |
|--------|------|-------------|
| GET | `/permissions` | List system + custom permissions |
| POST | `/permissions` | Create custom permission |
| GET | `/permissions/:id` | Get permission by ID |
| PUT | `/permissions/:id` | Update custom permission |
| DELETE | `/permissions/:id` | Soft-delete custom permission |

### Dashboard (`/dashboard`) — Tenant Scoped

| Method | Path | Description |
|--------|------|-------------|
| GET | `/dashboard` | Aggregated project + task stats |

---

## Request & Response Format

### Standard response envelope

```json
{
  "success": true,
  "message": "Projects fetched",
  "data": {
    "projects": [...],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 45,
      "pages": 3
    }
  }
}
```

### Error response

```json
{
  "success": false,
  "message": "Project not found",
  "statusCode": 404
}
```

### Pagination query params

All list endpoints support: `?page=1&limit=20` (default). Max limit: 100.

---

## Data Models

### Task statuses (flow)

```
todo → in-progress → in-review → done
```

### Task priorities

```
low | medium (default) | high | critical
```

### Project statuses

```
active (default) | archived | completed
```

### Tenant statuses

```
active (default) | inactive | suspended
```

### User statuses

```
active | inactive | invited
```

---

## Key Design Decisions

### Multi-tenancy
Every model (except Tenant itself) has a `tenantId` field. The `tenantGuard` middleware extracts the tenant from the JWT and the `X-Tenant-ID` header, attaches it to `req.tenantId`, and all database queries filter by it.

### Soft delete
Nothing is physically deleted. All models have `isDeleted: boolean`. Deleting a project cascades a soft-delete to all its tasks.

### Token security
Refresh tokens are SHA-256 hashed before storing in the database. The raw token is sent to the client but never stored in plaintext.

### Task comments
Comments are embedded subdocuments inside the Task document (not a separate collection). The `addComment` endpoint uses `$push` and returns only the new comment.

### Rate limiting
- Auth endpoints: 20 req / 15 min
- All other endpoints: 300 req / 15 min

---

## Common Errors

| Status | Cause | Fix |
|--------|-------|-----|
| 400 | Missing required field | Check request body |
| 400 | Employee updating non-status field | Send only `{ "status": "..." }` |
| 401 | Missing/invalid token | Login again |
| 401 | Expired refresh token | Re-login |
| 403 | Wrong role | Use an account with the required role |
| 403 | Tenant inactive/not found | Check X-Tenant-ID header |
| 404 | Resource not found or soft-deleted | Verify the ID |
| 409 | Duplicate (email, slug, etc.) | Use a unique value |
