# MediCore HMS — Premium Health Management System

A complete, production-oriented hospital / clinic management platform: **React (Next.js 16) + TypeScript frontend, Node.js REST API backend, Prisma ORM database, JWT auth with 8-role RBAC, Socket.IO real-time notifications and a premium healthcare SaaS interface** (deep navy + medical teal design system, dark/light mode, fully responsive).

> **Demo environment**: the running instance is seeded with realistic sample data. All demo accounts use the password **`Demo@1234`**.

---

## 1. Quick start

```bash
# 1. Install dependencies
bun install            # or: npm install

# 2. Configure environment
cp .env.example .env   # adjust DATABASE_URL / JWT_SECRET

# 3. Create the database schema (Prisma push — no migration files needed)
bun run db:push        # or: bunx prisma migrate dev (see §6)

# 4. Seed the demo data
bun run scripts/seed.ts

# 5. Start the app (dev server, port 3000)
bun run dev

# 6. Start the real-time notification service (separate terminal)
cd mini-services/notification-service && bun install && bun run dev   # port 3003
```

Open `http://localhost:3000` and sign in with any demo account below.

### Demo accounts (password `Demo@1234`)

| Role | Email | Notes |
| --- | --- | --- |
| Super Administrator | `super@medicore.com` | Full access incl. user accounts |
| Hospital Administrator | `admin@medicore.com` | Full facility access |
| Doctor | `doctor@medicore.com` | Dr. Evelyn Carter (Cardiology) |
| Nurse | `nurse@medicore.com` | Ward & vitals workflows |
| Receptionist | `reception@medicore.com` | Registration, appointments, billing |
| Pharmacist | `pharmacist@medicore.com` | Dispensing + inventory |
| Lab Technician | `lab@medicore.com` | Lab pipeline + results |
| Patient (portal) | `patient@medicore.com` | Self-service portal |

---

## 2. Technology stack

| Layer | Technology |
| --- | --- |
| Frontend | React 19 (via Next.js 16 App Router), TypeScript 5, Tailwind CSS 4, shadcn/ui, Recharts, Lucide icons, Zustand, next-themes (dark/light), Sonner toasts |
| Backend | Node.js, Next.js Route Handlers (Express-style modular REST API), JWT (jsonwebtoken), bcrypt password hashing, RBAC middleware, rate limiting, centralized error handling, Zod validation |
| Database | Prisma ORM + SQLite (demo) — schema is portable to PostgreSQL by switching the datasource provider |
| Real-time | Socket.IO mini-service (`mini-services/notification-service`, port 3003) pushing DB notifications to clients |
| Docs | Swagger/OpenAPI 3.0 spec at `GET /api/openapi.json` + in-app viewer (**API Docs** in the sidebar) |

**Note on the backend**: the API layer uses Next.js Route Handlers instead of a standalone Express server — same Node.js runtime, same REST contract, modular `lib/` organization (`api.ts` = middleware, `crud.ts` = controller factory, `services.ts` = services, `schemas.ts` = validators). The Socket.IO service runs as its own Node process exactly like a separate Express service would.

---

## 3. Project structure

```
prisma/schema.prisma          # 20+ models, relationships, constraints
scripts/seed.ts               # realistic demo data (6 months of activity)
scripts/api-test.sh           # 36-assertion API smoke test
mini-services/notification-service/   # Socket.IO real-time service (port 3003)
src/lib/api.ts                # response envelope, error handling, rate limiting
src/lib/auth.ts               # JWT sign/verify, bcrypt, session extraction
src/lib/rbac.ts               # role → module permissions + write-role gates
src/lib/crud.ts               # generic CRUD controller factory (list/get/create/update/delete)
src/lib/schemas.ts            # Zod validators for every resource
src/lib/services.ts           # audit logging + notification service + code generator
src/app/api/**                # REST endpoints (see §5)
src/components/hms/**         # app shell, DataTable, FormDialog, charts, UI primitives
src/views/**                  # module views (login, dashboard, patients, …)
src/app/page.tsx              # SPA root: auth gate + client-side view router
```

---

## 4. User roles & permissions

Eight roles, each with a dedicated dashboard and server-enforced access:

| Module | Super Admin | Admin | Doctor | Nurse | Reception | Pharmacist | Lab Tech | Patient |
| --- | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: |
| Dashboard | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ (own) |
| Patients | ✔ | ✔ | ✔ | ✔ | ✔ | – | ✔ (read) | – |
| Doctors | ✔ | ✔ | – | – | ✔ (read) | – | – | – |
| Appointments | ✔ | ✔ | ✔ (own) | ✔ | ✔ | – | – | ✔ (own) |
| EMR | ✔ | ✔ | ✔ | ✔ (read) | – | – | – | ✔ (own) |
| Prescriptions | ✔ | ✔ | ✔ (issue) | – | – | ✔ (dispense) | – | ✔ (own) |
| Pharmacy inventory | ✔ | ✔ | – | – | – | ✔ | – | – |
| Laboratory | ✔ | ✔ | ✔ (request/review) | – | – | – | ✔ (process) | ✔ (own results) |
| Billing | ✔ | ✔ | ✔ (read) | – | ✔ | – | – | ✔ (own) |
| Wards & beds | ✔ | ✔ | ✔ (admit) | ✔ | ✔ (view) | – | – | – |
| Staff | ✔ | ✔ | – | – | – | – | – | – |
| Reports | ✔ | ✔ | ✔ | – | – | – | – | – |
| User accounts / Audit | ✔ | ✔ | – | – | – | – | – | – |

Write-operation gates are enforced **server-side** in `src/lib/rbac.ts` (`WRITE_ROLES`) and every API call passes through `requireAuth()`.

---

## 5. REST API

Full interactive spec: **`GET /api/openapi.json`** (Swagger 3.0) and the in-app **API Docs** page. Highlights:

- **Auth** — `POST /api/auth/register` (patient self-signup) · `POST /api/auth/login` · `POST /api/auth/logout` · `POST /api/auth/forgot-password` · `POST /api/auth/reset-password` · `GET/PUT /api/auth/me` · `POST /api/auth/change-password`
- **Dashboard** — `GET /api/dashboard/statistics` · `GET /api/dashboard/revenue` · `GET /api/dashboard/appointments`
- **Patients** — CRUD + `GET /api/patients/:id` (360° profile) + `POST /api/patients/:id` (`createLogin`) + `/api/documents` (upload/download)
- **Doctors** — `GET/POST /api/doctors` (POST creates user + profile) · `GET/PUT/DELETE /api/doctors/:id`
- **Appointments** — `GET/POST /api/appointments` · `GET/PUT/DELETE /api/appointments/:id` (confirm / complete / cancel / no-show; patients cancel own)
- **EMR** — `GET/POST /api/medical-records` · `GET/PUT/DELETE /api/medical-records/:id`
- **Prescriptions** — `GET/POST /api/prescriptions` (with line items) · `POST /api/prescriptions/:id/dispense` (transactional stock decrement)
- **Pharmacy** — `GET/POST /api/medicines` · `PUT/DELETE /api/medicines/:id` (low-stock alerts)
- **Laboratory** — `GET/POST /api/lab-tests` · `PUT /api/lab-tests/:id` (status pipeline) · `POST /api/lab-tests/:id/result` · `POST /api/lab-tests/:id/review`
- **Billing** — `GET/POST /api/invoices` · `GET/PUT/DELETE /api/invoices/:id` · `GET/POST /api/invoices/:id/payments` (auto status) · `GET /api/payments`
- **Wards & beds** — `GET /api/wards` · ward CRUD · `GET/POST /api/beds` · `PUT/DELETE /api/beds/:id` · `GET/POST /api/admissions` · `PUT /api/admissions/:id` (transfer / discharge — transactional bed state)
- **Staff** — `GET/POST /api/staff` · `GET/PUT/DELETE /api/staff/:id` · `GET/POST /api/staff/attendance` (upsert)
- **Users** — `GET/POST /api/users` · `GET/PUT/DELETE /api/users/:id` (admin only)
- **Notifications** — `GET /api/notifications` · `PUT /api/notifications/:id` (read / mark-all)
- **Security** — `GET /api/audit-logs` · `GET /api/export?type=…` (CSV reports)

All list endpoints support `page`, `pageSize`, `search`, `sortBy`, `sortOrder` and declared filters. Responses use the envelope `{ success, data, meta }` / `{ success: false, error: { message, details } }` with proper HTTP codes (401/403/404/409/422/429).

---

## 6. Database setup & migrations

The demo uses SQLite (`db/custom.db`). To move to PostgreSQL:

1. In `prisma/schema.prisma` change `provider = "sqlite"` → `"postgresql"` and set `DATABASE_URL`.
2. `bunx prisma db push` (dev) or `bunx prisma migrate dev --name init` (generates versioned SQL migrations) then `bunx prisma migrate deploy` in CI/production.
3. Re-run `bun run scripts/seed.ts` to populate demo data.

Data integrity: unique codes (patient `P-xxxxxx`, doctor `D-xxxx`, `APT/RX/LAB/INV-xxxxxx`), foreign keys with cascade rules, `@@unique` constraints (bed per ward, attendance per staff/day) and transactional workflows (admission ↔ bed state, dispensing ↔ inventory, payment ↔ invoice status).

---

## 7. Security

- bcrypt (10 rounds) password hashing; strong-password policy enforced at registration/change
- JWT sessions (8-hour expiry) delivered as httpOnly SameSite cookie + Bearer support
- Role-based access control enforced server-side on every route (least privilege)
- Zod input validation + Prisma parameterized queries (SQL-injection safe)
- Login rate limiting; API rate limiting per bucket
- Centralized audit log (login, create, update, delete, export, upload) with IP capture
- Documents access-controlled (patients see only their own); upload type/size validation (PDF/PNG/JPEG/WebP ≤ 5 MB)
- Production checklist: set a strong `JWT_SECRET`, enable HTTPS (`Secure` cookie flag activates in production), configure SMTP, swap local document storage for Cloudinary/S3, and schedule encrypted DB backups

---

## 8. Deployment

```bash
bun run build        # production build (standalone output)
bun run start        # serve on :3000  (NODE_ENV=production)
# behind a reverse proxy (nginx/Caddy) with TLS + gzip/brotli
# run the notification service as a separate process (pm2/systemd):
cd mini-services/notification-service && bun run index.ts --port 3003
```

Environment checklist: `DATABASE_URL`, `JWT_SECRET`, SMTP vars; run `prisma migrate deploy` before first boot; mount `db/` on persistent storage for SQLite.

---

## 9. Verification

`scripts/api-test.sh` runs a 36-assertion smoke test across auth, RBAC denials, CRUD, validation errors and the patient booking workflow. The frontend was verified end-to-end in a real browser (login → dashboards → module workflows → real-time notifications).
