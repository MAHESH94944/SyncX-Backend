<!-- Backend README: SyncX-Backend -->

# SyncX Backend (Workspace + Projects + Tasks)

This backend is the API + authentication + authorization layer for SyncX. It supports:

- Workspaces (create, update, invite)
- Members (join by invite code)
- Projects (inside a workspace)
- Tasks (inside a project + workspace)
- Authentication (Email/Password + Google OAuth)
- Role-Based Access Control (RBAC) using roles + permissions

The goal in interviews: explain it as a clean **Express + TypeScript** service with a clear request flow, strong validation, structured error handling, and MongoDB data modeling.

---

## Tech stack (backend)

- Runtime: Node.js
- Framework: Express
- Language: TypeScript
- Database: MongoDB via Mongoose
- Auth: Passport (Google OAuth 2.0 + Local) + JWT
- Validation: Zod
- Utilities: bcrypt (password hashing), uuid (invite/task codes)

---

## Project structure (how I’d explain it)

The backend follows a layered pattern:

- **Routes**: define HTTP endpoints and connect them to controllers
- **Controllers**: handle HTTP concerns (req/res), parse input, call services
- **Services**: actual business logic (DB reads/writes, rules, permissions)
- **Models**: Mongoose schemas (data shape + relationships)
- **Middlewares**: cross-cutting concerns (async error handling, auth checks)
- **Utils/Config**: environment config, JWT helpers, custom errors

Key folders:

- [src/routes](src/routes)
- [src/controllers](src/controllers)
- [src/services](src/services)
- [src/models](src/models)
- [src/middlewares](src/middlewares)
- [src/config](src/config)
- [src/utils](src/utils)
- [src/validation](src/validation)

---

## High-level request lifecycle (end-to-end)

When a request hits the server:

1. **Express receives the request** in [src/index.ts](src/index.ts)
2. **Global middlewares** run (JSON parsing, urlencoded parsing, CORS, passport init)
3. Request is routed via `BASE_PATH` (example: `/api/auth/...`)
4. **Controller** validates input (Zod) and extracts context (like `req.user._id`)
5. **Service** performs business logic + database operations
6. Response is returned with a consistent HTTP status code
7. Any thrown error gets caught and formatted by the **error handler**

Important design choice: controllers are wrapped in an `asyncHandler` so async errors don’t crash the server.

---

## Configuration (env-driven)

Configuration is loaded from environment variables via [src/config/app.config.ts](src/config/app.config.ts).

Common variables (see `.env`):

- `PORT`, `NODE_ENV`, `BASE_PATH`
- `MONGO_URI`
- `JWT_SECRET`, `JWT_EXPIRES_IN`
- `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_CALLBACK_URL`
- `FRONTEND_ORIGIN`, `FRONTEND_GOOGLE_CALLBACK_URL`

The helper [src/utils/get-env.ts](src/utils/get-env.ts) fails fast if a required env var is missing.

---

## Database layer (MongoDB + Mongoose)

Database connection is in [src/config/database.config.ts](src/config/database.config.ts) using `mongoose.connect(config.MONGO_URI)`.

### Data model (simple mental model)

- `User`: main identity record
- `Account`: links a user to an auth provider (Google / Email etc.)
- `Workspace`: owned by a user; has an `inviteCode` to join
- `Member`: connects a `User` to a `Workspace` with a `Role`
- `Role`: role name + permissions array (RBAC)
- `Project`: belongs to a workspace; created by a user
- `Task`: belongs to a workspace + project; optionally assigned to a user

This is a classic collaboration-app schema: **Workspace is the top boundary**, and most entities reference it.

---

## Authentication (how login works)

SyncX supports two login paths:

### 1) Email/Password (Local Strategy)

- Client calls login endpoint with `email` + `password`
- Passport Local strategy verifies credentials
  - Passwords are hashed/checked via [src/utils/bcrypt.ts](src/utils/bcrypt.ts)
- On success, we sign a JWT using [src/utils/jwt.ts](src/utils/jwt.ts)
- Client stores JWT and sends it in future requests

### 2) Google OAuth (Google Strategy)

- Client hits `/auth/google` → redirected to Google
- Google redirects back to `/auth/google/callback`
- We create or fetch the user account based on provider ID/email
- We generate a JWT and redirect back to the frontend callback URL

### JWT handling

JWT signing is centralized in [src/utils/jwt.ts](src/utils/jwt.ts). The payload is minimal (contains user ID), which is good practice.

In TypeScript, `req.user` and `req.jwt` are extended using [src/@types/index.d.ts](src/@types/index.d.ts) so controllers can safely access:

- `req.user?._id`
- `req.user?.currentWorkspace`
- `req.jwt` (token produced in the OAuth callback flow)

---

## Authorization (RBAC with roles + permissions)

Authorization is role-based:

- Roles: `OWNER`, `ADMIN`, `MEMBER` (see [src/enums/role.enum.ts](src/enums/role.enum.ts))
- Permissions: granular strings like `CREATE_PROJECT`, `DELETE_TASK`, etc.

### Where permissions live

- Role schema: [src/models/roles-permission.model.ts](src/models/roles-permission.model.ts)
  - A role stores `name` + `permissions[]`
- Role permission mapping: [src/utils/role-permission.ts](src/utils/role-permission.ts)
  - This acts like the policy source (what each role is allowed to do)
- Guard: [src/utils/roleGuard.ts](src/utils/roleGuard.ts)
  - Given a role + required permissions, it throws an `UnauthorizedException` if not allowed

### How you explain it in interviews

“Authentication proves who you are (JWT). Authorization checks what you can do (RBAC). In this app, the user’s role is workspace-scoped via the Member record.”

---

## Validation + error handling (clean API behavior)

### Validation (Zod)

Each domain has Zod schemas in [src/validation](src/validation). Example:

- auth schemas: email/password rules
- workspace/project/task schemas: ID validation, required fields, enums, date format checks

Controllers parse inputs like:

- route params (`workspaceId`, `projectId`, `taskId`)
- request body

If validation fails, Zod throws an error.

### Errors (centralized)

- Custom errors extend `AppError` in [src/utils/appError.ts](src/utils/appError.ts)
  - Includes `statusCode` and optional `errorCode`
- Global error middleware formats:
  - Zod errors as a clean `field + message` list
  - App errors with correct status codes

This is good interview material because it shows you designed predictable error responses.

---

## Main modules (what each feature does)

### Workspaces

- Create workspace (owner becomes the workspace owner)
- Generate invite code (short code from UUID)
- Fetch workspace analytics / members

Main logic lives in [src/services/workspace.service.ts](src/services/workspace.service.ts).

### Members

- Join workspace by invite code
- Member record created with a role

See [src/services/member.service.ts](src/services/member.service.ts).

### Projects

- Create/read/update/delete projects inside a workspace
- Analytics endpoints aggregate task stats by status/priority (typical “dashboard” feature)

See [src/services/project.service.ts](src/services/project.service.ts).

### Tasks

- Create/update/delete tasks inside a project
- Supports status + priority enums
- Supports optional `assignedTo`

See [src/services/task.service.ts](src/services/task.service.ts).

---

## API routing overview

Routes are grouped by domain:

- Auth: [src/routes/auth.route.ts](src/routes/auth.route.ts)
- Workspace: [src/routes/workspace.route.ts](src/routes/workspace.route.ts)
- Member: [src/routes/member.route.ts](src/routes/member.route.ts)
- Project: [src/routes/project.route.ts](src/routes/project.route.ts)
- Task: [src/routes/task.route.ts](src/routes/task.route.ts)
- User: [src/routes/user.route.ts](src/routes/user.route.ts)

Typical interview explanation:
“Routes are thin. Controllers validate + orchestrate. Services do the business logic and DB operations.”

---

## Role seeding (why it exists)

Roles/permissions are seeded via [src/seeders/role.seeder.ts](src/seeders/role.seeder.ts).

Why this is important:

- Ensures the DB has the expected role documents
- Keeps permissions consistent across environments
- Uses a transaction to keep seed operations safe

Run:

```bash
npm run seed
```

---

## How to run the backend locally

1. Install dependencies:

```bash
npm install
```

2. Create `.env` (see `.env` sample keys already in repo)

3. Start dev server:

```bash
npm run dev
```

Build for production:

```bash
npm run build
npm start
```

---

## Interview “talk track” (60–90 seconds)

“SyncX backend is an Express + TypeScript REST API backed by MongoDB/Mongoose. It supports multi-tenant collaboration using workspaces as the boundary. Authentication is handled with Passport (Google OAuth + email/password) and the API uses JWTs for stateless sessions. Authorization is RBAC: each workspace membership maps a user to a role, and roles map to permissions. Controllers validate input with Zod, then delegate to services for business logic. Errors are normalized using a custom AppError hierarchy plus a global error handler, so clients always get predictable status codes and messages.”

---

## Next: Frontend README

If you want, I can write the same “deep but simple” interview explanation for SyncX-Frontend next.
