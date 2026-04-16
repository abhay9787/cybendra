# CybendraScanner — Master Documentation

**Project:** Web-Based Vulnerability Scanner  
**Team:** Abhay Sharma (2206950100013) | Arin Dhiman (2206950100031) | Khushi (2206950100061)  
**Mentor:** Ms. Apeksha Nayak  
**College:** Shri Ram Group of Colleges, Muzaffarnagar | AKTU | B.Tech CSE 4th Year 8th Sem

---

## 1. Project Structure

```
CybendraScanner/
├── quickscan/
│   ├── backend/          ← Node.js + Express API server
│   │   └── src/
│   │       ├── index.js          ← Entry point
│   │       ├── models/           ← MongoDB schemas
│   │       ├── routes/           ← API route handlers
│   │       ├── services/         ← Business logic
│   │       └── utils/            ← Helpers
│   ├── frontend/         ← React + Vite + Tailwind UI
│   │   └── src/
│   │       ├── App.jsx           ← Root router
│   │       ├── pages/            ← Page components
│   │       ├── components/       ← Shared UI components
│   │       └── services/         ← API client + state
│   ├── start.sh          ← Start both servers
│   ├── start-backend.sh  ← Start backend only
│   └── start-frontend.sh ← Start frontend only
└── extras/               ← Docs, zips, non-runtime files
```

---

## 2. How to Run

```bash
# 1. Start MongoDB
brew services start mongodb/brew/mongodb-community@7.0

# 2. Seed admin user (first time only)
cd quickscan/backend && node src/utils/seed.js

# Terminal 1 — Backend
cd quickscan/backend && node src/index.js

# Terminal 2 — Frontend
cd quickscan/frontend && npm run dev
```

**URLs:**
- Frontend: http://localhost:5173
- Backend API: http://localhost:3001
- Swagger Docs: http://localhost:3001/api-docs

**Default Admin Login:**
- Email: `admin@example.com`
- Password: `ChangeMe@123!`

---

## 3. Environment Variables

**`quickscan/backend/.env`**

| Variable | Default | Purpose |
|---|---|---|
| PORT | 3001 | Backend server port |
| MONGODB_URI | mongodb://localhost:27017/quickscan | Database connection |
| JWT_SECRET | quickscan-dev-secret-key-2024 | JWT signing key |
| JWT_EXPIRES_IN | 8h | Token expiry |
| NODE_ENV | development | Environment mode |
| LOG_LEVEL | info | Winston log level |
| MAX_CONCURRENT_SCANS | 3 | Parallel scan limit |
| SCAN_RATE_LIMIT | 100 | ms between requests |

**`quickscan/frontend/.env`**

| Variable | Default | Purpose |
|---|---|---|
| VITE_API_URL | http://localhost:3000 | Backend API base URL |

---

## 4. Backend — File by File

### `src/index.js` — Entry Point
- Loads all middleware: `helmet` (security headers), `cors`, `express-rate-limit`
- Rate limit: 1000 requests per 15 min per IP (production only)
- Mounts all routes under `/api/`
- Connects to MongoDB via Mongoose
- Serves Swagger UI at `/api-docs`
- Health check endpoint: `GET /health`

---

### `src/models/User.js` — User Schema

**Fields:**

| Field | Type | Description |
|---|---|---|
| email | String | Unique, lowercase, required |
| password | String | Bcrypt hashed (salt rounds: 12) |
| role | String | `user` or `admin` (default: user) |
| isActive | Boolean | Account enabled/disabled |
| emailNotifications | Boolean | Email alerts toggle |
| lastLogin | Date | Last login timestamp |
| mustChangePassword | Boolean | Force password change on login |
| oauthProvider | String | google / microsoft / github |
| features | Object | Per-user feature flags (see below) |

**Feature Flags inside `features`:**
- `notifications.email` — email alerts on/off
- `notifications.scanComplete` — notify when scan finishes
- `scanning.autoScan` — auto-scan on schedule
- `scanning.scanDepth` — basic / standard / deep
- `dashboard.defaultView` — grid / list
- `security.twoFactorAuth` — 2FA toggle
- `security.sessionTimeout` — minutes (default 480)

**Methods:**
- `comparePassword(candidate)` — bcrypt compare, returns boolean
- `toJSON()` — strips password, resetToken from output

**Hooks:**
- `pre('save')` — auto-hashes password if modified

---

### `src/models/Target.js` — Target Schema

**Fields:**

| Field | Type | Description |
|---|---|---|
| url | String | Target website URL (lowercased) |
| name | String | Human-readable label |
| description | String | Optional notes |
| owner | ObjectId | Ref to User |
| isActive | Boolean | Target enabled |
| lastScanned | Date | Last scan timestamp |
| scanCount | Number | Total scans run |
| highestSeverity | String | Worst finding severity ever |

**Index:** Compound unique index on `(owner, url)` — prevents duplicate URLs per user.

---

### `src/models/Scan.js` — Scan Schema

**Fields:**

| Field | Type | Description |
|---|---|---|
| target | ObjectId | Ref to Target |
| user | ObjectId | Ref to User |
| status | String | pending / running / completed / failed / cancelled |
| progress | Number | 0–100 percentage |
| findings | Array | Array of finding objects |
| summary | Object | Count by severity |
| scanConfig | Object | Which checks to run |
| startedAt | Date | When scan started |
| completedAt | Date | When scan finished |
| duration | Number | ms taken |
| error | String | Error message if failed |

**Finding Object:**
```json
{
  "type": "header | tls | directory | robots | status | general",
  "severity": "info | low | medium | high | critical",
  "title": "Missing X-Frame-Options Header",
  "description": "...",
  "recommendation": "...",
  "url": "https://target.com",
  "evidence": {}
}
```

**Indexes:** `(target, createdAt)`, `(user, createdAt)`, `(status)`

---

### `src/models/ScheduledScan.js` — Scheduled Scan Schema

**Fields:**

| Field | Type | Description |
|---|---|---|
| target | ObjectId | Ref to Target |
| user | ObjectId | Ref to User |
| name | String | Schedule label |
| cronExpression | String | Cron format e.g. `0 9 * * 1` |
| isActive | Boolean | Schedule enabled |
| scanConfig | Object | Which checks to run |
| lastRun | Date | Last execution time |
| nextRun | Date | Next scheduled time |
| runCount | Number | Total executions |

---

## 5. Backend Routes — API Reference

### Auth Routes — `src/routes/auth.js`
Base: `/api/auth`

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/register` | No | Register new user |
| POST | `/login` | No | Login, returns JWT token |
| GET | `/me` | Yes | Get current user profile |
| POST | `/change-password` | Yes | Change own password |
| PUT | `/update-profile` | Yes | Update email notifications / features |

**Register body:** `{ email, password }`  
**Login body:** `{ email, password }` → returns `{ token, user }`  
**Change password body:** `{ currentPassword, newPassword }`

---

### Target Routes — `src/routes/targets.js`
Base: `/api/targets`  All routes require JWT.

| Method | Endpoint | Description |
|---|---|---|
| GET | `/` | List own targets (paginated) |
| POST | `/` | Create new target |
| GET | `/:id` | Get single target |
| PUT | `/:id` | Update target name/description |
| DELETE | `/:id` | Delete target |

**Create body:** `{ url, name, description? }`  
**Ownership check:** `requireOwnership(Target)` middleware ensures users can only access their own targets.

---

### Scan Routes — `src/routes/scans.js`
Base: `/api/scans`  All routes require JWT.

| Method | Endpoint | Description |
|---|---|---|
| GET | `/` | List own scans (paginated, filterable by status/targetId) |
| POST | `/` | Create and start a new scan |
| GET | `/:id` | Get scan details with findings |
| POST | `/:id/cancel` | Cancel running/pending scan |
| DELETE | `/:id` | Delete scan record |
| GET | `/:id/export?format=` | Export scan as `pdf`, `csv`, or `json` |

**Create body:** `{ targetId, consent: "true", config? }`  
**Consent field is mandatory** — must be `"true"` to start scan.

**Export formats:**
- `pdf` → `application/pdf` — formatted report with findings
- `csv` → `text/csv` — spreadsheet of all findings
- `json` → `application/json` — full scan object

---

### Admin Routes — `src/routes/admin.js`
Base: `/api/admin`  Requires JWT + admin role.

| Method | Endpoint | Description |
|---|---|---|
| GET | `/dashboard` | Stats: users, targets, scans, high severity count |
| GET | `/users` | All users (searchable, paginated) |
| POST | `/users` | Create user (sets mustChangePassword=true) |
| PUT | `/users/:id` | Update user email/role/isActive |
| POST | `/users/:id/reset-password` | Generate temp password |
| GET | `/targets` | All targets across all users |
| GET | `/scans` | All scans across all users |
| POST | `/scans/:id/force-run` | Reset and re-run any scan |
| POST | `/scans/:id/cancel` | Cancel any scan |
| GET | `/logs` | System logs |

---

### User Routes — `src/routes/users.js`
Base: `/api/users`  All routes require JWT.

| Method | Endpoint | Description |
|---|---|---|
| GET | `/profile` | Get own profile |
| PUT | `/settings` | Update emailNotifications toggle |
| PUT | `/features` | Update feature flags object |

---

## 6. Backend Services

### `src/services/scanner.js` — VulnerabilityScanner Class

The core scanning engine. Instantiated with `rateLimit` (ms between requests) and `timeout` (ms per request).

**Main method: `scanTarget(url, config)`**
Runs all enabled checks and returns `{ findings[], duration, summary }`.

**Check methods:**

| Method | What it checks | Findings it produces |
|---|---|---|
| `checkSecurityHeaders(url)` | Fetches HTTP headers, checks for missing security headers | Missing X-Frame-Options, X-Content-Type-Options, X-XSS-Protection, HSTS, CSP; Info disclosure via Server/X-Powered-By headers |
| `checkTLSCertificate(hostname, port)` | Opens TLS socket, reads certificate | Expired cert (critical), expiring within 30 days (medium), not yet valid (high), connection failure (high) |
| `checkRobotsTxt(url)` | Fetches `/robots.txt` | Sensitive paths exposed (/admin, /backup etc.) — low severity |
| `checkSensitiveFiles(url)` | Probes `.env`, `.git`, `config.php`, `phpinfo.php` | Exposed sensitive file — high severity |
| `checkCookieSecurity(url)` | Reads `Set-Cookie` headers | Missing Secure flag (medium), missing HttpOnly (medium), missing SameSite (low) |
| `checkFormSecurity(url)` | Parses HTML for `<form>` tags | Missing CSRF token on POST forms (medium), password autocomplete enabled (low) |
| `checkCORSMisconfiguration(url)` | Sends request with `Origin: evil.com` | Wildcard CORS with credentials (high), reflected origin (medium) |

**Disabled by default (performance):**
- `checkDirectories` — directory fuzzing
- `checkOpenRedirects` — open redirect testing
- `checkSubdomainTakeover` — subdomain takeover

**Helper:** `sleep(ms)` — rate limiting between requests.  
**`generateSummary(findings)`** — counts findings by severity, returns summary object.

---

### `src/services/scanService.js` — ScanService Class (Singleton)

Manages scan lifecycle. Exported as a singleton instance.

**Properties:**
- `runningScans` — `Map<scanId, scan>` of currently executing scans
- `scanner` — VulnerabilityScanner instance

**Methods:**

| Method | Description |
|---|---|
| `createScan(targetId, userId, config)` | Creates Scan document, starts `executeScan` via `setImmediate` |
| `executeScan(scanId)` | Runs scanner, updates progress (10% → 90% → 100%), saves findings, marks completed/failed |
| `updateProgress(scanId, progress)` | Updates scan progress field in DB |
| `updateTargetStats(targetId, summary)` | Updates `lastScanned`, `scanCount`, `highestSeverity` on Target |
| `getHighestSeverity(summary)` | Returns worst severity string from summary object |
| `getScan(scanId, userId, isAdmin)` | Fetches scan with target+user populated |
| `getUserScans(userId, options)` | Paginated scan list for a user |
| `getAllScans(options)` | Paginated scan list for admin |
| `cancelScan(scanId, userId, isAdmin)` | Sets status to cancelled, removes from runningScans map |
| `deleteScan(scanId, userId, isAdmin)` | Deletes scan document |
| `getRunningScans()` | Returns array of currently running scans |
| `getStats()` | Aggregates scan counts by status |

---

### `src/services/exportService.js` — ExportService Class (Singleton)

Handles report generation. Exported as a singleton instance.

**Methods:**

| Method | Output | Description |
|---|---|---|
| `exportToPDF(scan)` | `Buffer` (PDF) | Generates PDF using pdfkit — header, target info, severity summary boxes, findings list with badges |
| `exportToCSV(scan)` | `Buffer` (CSV) | Comma-separated findings with all fields |
| `exportToJSON(scan)` | `Buffer` (JSON) | Full scan object as formatted JSON |
| `csv(field)` | String | Escapes CSV special characters |

---

## 7. Backend Utilities

### `src/utils/auth.js`

| Export | Type | Description |
|---|---|---|
| `generateToken(userId)` | Function | Signs JWT with userId, uses JWT_SECRET, expires in JWT_EXPIRES_IN |
| `verifyToken` | Middleware | Extracts Bearer token, verifies, loads user into `req.user` |
| `requireAdmin` | Middleware | Checks `req.user.role === 'admin'`, returns 403 if not |
| `requireOwnership(Model)` | Middleware factory | Loads resource by `req.params.id`, checks `resource.owner === req.user._id`, attaches to `req.resource` |

---

### `src/utils/validation.js`

Uses `express-validator`. Each validation group is an array of middleware ending with `handleValidationErrors`.

| Export | Used on | Rules |
|---|---|---|
| `authValidation.register` | POST /auth/register | email format, password: 8+ chars, uppercase, lowercase, number, special char |
| `authValidation.login` | POST /auth/login | email format, password not empty |
| `authValidation.changePassword` | POST /auth/change-password | currentPassword not empty, newPassword meets rules |
| `targetValidation.create` | POST /targets | url must be valid HTTP/HTTPS, name 1–100 chars |
| `targetValidation.update` | PUT /targets/:id | id is MongoId, name/description optional with length limits |
| `scanValidation.create` | POST /scans | targetId is MongoId, consent must equal `"true"` |
| `adminValidation.createUser` | POST /admin/users | email, password, role in [user, admin] |
| `adminValidation.updateUser` | PUT /admin/users/:id | id MongoId, optional email/role/isActive |

---

### `src/utils/logger.js`

Winston logger with:
- Console transport always active (colorized in dev)
- File transport in production: `logs/quickscan.log` (max 5MB, 5 files)
- Auto-redacts sensitive fields: `password`, `token`, `secret`, `key`, `authorization`
- Log level controlled by `LOG_LEVEL` env var (default: `info`)

---

### `src/utils/seed.js`

Run once to create the default admin user.

```bash
node src/utils/seed.js
```

Creates: `admin@example.com` / `ChangeMe@123!` with `mustChangePassword: true`.  
Skips if admin already exists.

---

## 8. Frontend — File by File

### `src/main.jsx`
React app entry point. Mounts `<App />` into `#root`.

### `src/App.jsx` — Root Router

Defines all routes using React Router v6.

| Route | Component | Access |
|---|---|---|
| `/` | Landing | Public (redirects to /dashboard if logged in) |
| `/login` | Login | Public |
| `/register` | Register | Public |
| `/auth/callback` | AuthCallback | Public |
| `/dashboard` | Dashboard | Protected |
| `/targets` | Targets | Protected |
| `/scans` | Scans | Protected |
| `/scans/:id` | ScanDetails | Protected |
| `/profile` | Profile | Protected |
| `/features` | Features | Protected |
| `/admin/dashboard` | AdminDashboard | Admin only |
| `/admin/users` | AdminUsers | Admin only |
| `/admin/targets` | AdminTargets | Admin only |
| `/admin/scans` | AdminScans | Admin only |
| `/admin/logs` | AdminLogs | Admin only |

Auth state is initialized from `localStorage` on mount.

---

## 9. Frontend Pages

| File | What it does |
|---|---|
| `Landing.jsx` | Public homepage — navbar (About/Services/Team/Pricing/Contact scroll links), hero with 3D globe, services grid, team section (all 3 members + mentor), pricing tiers, contact form, footer |
| `Login.jsx` | Email + password login form, calls `POST /api/auth/login`, stores token in localStorage via authStore |
| `Register.jsx` | Registration form, calls `POST /api/auth/register` |
| `Dashboard.jsx` | Shows scan stats, recent scans, quick actions |
| `Targets.jsx` | List/add/edit/delete scan targets, triggers scan with consent modal |
| `Scans.jsx` | Paginated scan history with status badges, filter by status |
| `ScanDetails.jsx` | Full scan report — target info, severity summary, all findings with recommendations. Export buttons for PDF/CSV |
| `Profile.jsx` | Update email notifications, change password |
| `Features.jsx` | Toggle feature flags (notifications, scanning depth, dashboard view, security settings) |
| `AuthCallback.jsx` | Handles OAuth redirect callback |
| `admin/AdminDashboard.jsx` | System-wide stats — total users, targets, scans, high severity findings |
| `admin/AdminUsers.jsx` | User management — create, enable/disable, reset passwords |
| `admin/AdminTargets.jsx` | View all targets across all users |
| `admin/AdminScans.jsx` | View/cancel/force-run all scans |
| `admin/AdminLogs.jsx` | View system logs |

---

## 10. Frontend Components

| File | What it does |
|---|---|
| `Layout.jsx` | Wraps all protected pages — renders Sidebar + Header + `<Outlet />` |
| `Sidebar.jsx` | Left navigation — links to Dashboard, Targets, Scans, Profile, Features, Admin (if admin) |
| `Header.jsx` | Top bar — page title, notification bell, user menu, logout |
| `ProtectedRoute.jsx` | Redirects to `/login` if not authenticated |
| `AdminRoute.jsx` | Redirects to `/dashboard` if not admin role |
| `NotificationPanel.jsx` | Dropdown notification list, mark as read, clear all |
| `SecurityGlobe.jsx` | Animated 3D globe on landing page (Three.js) |
| `OAuthButtons.jsx` | Google/GitHub/Microsoft OAuth login buttons |

---

## 11. Frontend Services

### `src/services/api.js` — Axios API Client

Base URL: `VITE_API_URL/api`  
Auto-attaches `Authorization: Bearer <token>` from localStorage on every request.

**Interceptors:**
- Request: injects auth token
- Response: handles 401 (clears auth, redirects to login), shows toast on errors

**API modules:**

| Module | Methods |
|---|---|
| `authAPI` | `register`, `login`, `getProfile`, `changePassword`, `updateProfile` |
| `targetsAPI` | `getAll`, `getById`, `create`, `update`, `delete` |
| `scansAPI` | `getAll`, `getById`, `create`, `cancel`, `delete`, `export(id, format)` |
| `adminAPI` | `getDashboard`, `getUsers`, `createUser`, `updateUser`, `resetPassword`, `getTargets`, `getScans`, `forceRunScan`, `cancelScan`, `getLogs` |
| `usersAPI` | `getProfile`, `updateSettings`, `updateFeatures` |

Export uses `responseType: 'blob'` for file downloads.

---

### `src/services/store.js` — Zustand State Management

Four stores:

| Store | State | Key Actions |
|---|---|---|
| `useAuthStore` | user, token, isAuthenticated | `login()`, `logout()`, `updateUser()`, `isAdmin()` — persisted to localStorage |
| `useTargetsStore` | targets[], loading, pagination | `setTargets`, `addTarget`, `updateTarget`, `removeTarget` |
| `useScansStore` | scans[], loading, pagination | `setScans`, `addScan`, `updateScan`, `removeScan`, `getRunningScans()` |
| `useUIStore` | sidebarOpen, theme, notifications[] | `toggleSidebar`, `addNotification`, `markNotificationAsRead`, `getUnreadCount()` — theme persisted |
| `useAdminStore` | dashboard, users[], adminScans[] | `setDashboard`, `setUsers`, `updateUser`, `addUser` |

---

## 12. Security Features

| Feature | Implementation |
|---|---|
| Password hashing | bcrypt with salt rounds 12 |
| JWT authentication | 8h expiry, signed with JWT_SECRET |
| Rate limiting | 1000 req/15min per IP (production) |
| Input validation | express-validator on all POST/PUT routes |
| Ownership checks | requireOwnership middleware on all resource routes |
| Admin guard | requireAdmin middleware on all /admin routes |
| Consent required | Scan creation requires `consent: "true"` field |
| Scan rate limiting | 100ms between requests per scan |
| Sensitive data masking | Logger redacts password/token/secret fields |
| Security headers | helmet middleware on all responses |
| CORS | Configured with credentials support |

---

## 13. Data Flow — How a Scan Works

```
User clicks "Run Scan"
  → Frontend sends POST /api/scans { targetId, consent: "true" }
  → scanValidation middleware checks consent
  → scanService.createScan() creates Scan document (status: pending)
  → setImmediate triggers scanService.executeScan(scanId)
  → Scan status → "running", progress → 10%
  → scanner.scanTarget(url, config) runs all enabled checks:
      checkSecurityHeaders → findings[]
      checkRobotsTxt       → findings[]
      checkSensitiveFiles  → findings[]
      checkCookieSecurity  → findings[]
  → progress → 90%
  → findings saved to Scan document
  → summary calculated (count by severity)
  → Scan status → "completed", progress → 100%
  → Target.lastScanned, scanCount, highestSeverity updated
  → Frontend polls /api/scans/:id to show results
  → User clicks Export PDF → GET /api/scans/:id/export?format=pdf
  → exportService.exportToPDF(scan) generates Buffer
  → Browser downloads file
```

---

## 14. Tech Stack Summary

| Layer | Technology | Version |
|---|---|---|
| Frontend framework | React | 18 |
| Frontend build | Vite | 5 |
| Frontend styling | Tailwind CSS | 3 |
| Frontend state | Zustand | 4 |
| Frontend routing | React Router | 6 |
| HTTP client | Axios | 1 |
| Backend runtime | Node.js | 18+ |
| Backend framework | Express | 4 |
| Database | MongoDB | 7 |
| ODM | Mongoose | 8 |
| Authentication | JWT (jsonwebtoken) | 9 |
| Password hashing | bcryptjs | 2 |
| Validation | express-validator | 7 |
| PDF generation | pdfkit | 0.15 |
| Logging | Winston | 3 |
| HTTP scanning | Axios + Node TLS | — |
| API docs | Swagger (swagger-jsdoc + swagger-ui-express) | — |
| Testing (E2E) | Cypress | 13 |
| Testing (unit) | Jest | 29 |
