HRMS — Multi-Subsidiary HR Management System

This is my submission for the HRMS assignment (built against the SRS —
"HR Management System, Multi-Subsidiary, International Web Platform"). It's
a Next.js app for a company running multiple subsidiaries across different
countries, covering the org structure, role-based dashboards, and a
multi-level leave approval workflow.

The full SRS describes a much bigger product vision than what's buildable
in the 2-day window, so this README follows Section 9 of that doc — it
lays out clearly what's fully working, what's partially there, and what
I documented but didn't build.

## Tools I used

I used AI tools — Claude and GitHub Copilot — while building this. They
helped with scaffolding boilerplate, working through a couple of tricky
Prisma/Next.js issues, and speeding up repetitive UI work. The schema
design, architecture decisions, and final review of everything were mine.

## Tech Stack

Per the spec's required stack:

- **Framework:** Next.js 14, App Router (not Pages Router), TypeScript, Server Components + Server Actions
- **Styling/UI:** Tailwind CSS + shadcn/ui-style primitives (Button, Input, Card, Table, Badge, Dialog)
- **Charts:** Recharts — all dashboards read live data from the database, nothing hardcoded
- **Database:** SQLite by default (zero-config, file-based) via Prisma ORM — swappable to MySQL by changing one line, satisfying the "candidate's choice of MySQL or SQLite" requirement
- **Auth:** Email/password with bcrypt password hashing, JWT session in an httpOnly cookie (`jose`), role-based middleware protecting routes
- **Icons:** lucide-react

No paid or licensed third-party services are required to run this locally.

## Setup & Run (SQLite — default, no external DB needed)

\`\`\`bash
npm install
cp .env.example .env          # sets up DATABASE_URL + JWT_SECRET
npm run db:push               # creates prisma/dev.db and applies the schema
npm run db:seed               # seeds countries/subsidiaries/employees/leave/demo users
npm run dev                   # http://localhost:3000
\`\`\`

That's it — no Docker, no external credentials, no manual data entry needed.

## Setup & Run (MySQL instead)

1. In \`prisma/schema.prisma\`, change:
   \`\`\`prisma
   datasource db {
     provider = "mysql"   // was "sqlite"
     url      = env("DATABASE_URL")
   }
   \`\`\`
2. In \`.env\`, set \`DATABASE_URL="mysql://user:password@localhost:3306/hrms"\`.
3. Run the same commands: \`npm install\`, \`npm run db:push\`, \`npm run db:seed\`, \`npm run dev\`.

## Demo Login Credentials

Seeded automatically by \`prisma/seed.ts\` — one account per role, as required by Section 9.1:

| Role | Email | Password |
|---|---|---|
| Super Admin | \`admin@hrms.com\` | \`Admin@123\` |
| HR Manager | \`hr@hrms.com\` | \`Hr@12345\` |
| Manager | \`manager@hrms.com\` | \`Manager@123\` |
| Employee | \`employee@hrms.com\` | \`Employee@123\` |

The seed also builds a realistic 3-level reporting chain across 2 countries
(US, India) and 2 subsidiaries (Globex Americas / Globex India), 4
departments, ~12 employees, leave types with balances, and a few sample
leave requests already sitting at different stages of approval — so the
dashboards and queues aren't empty on first login.

## What's fully working (Section 9.1 — Mandatory)

- Auth with 4 distinct roles (spec asked for at least 3), session-protected routes via middleware, role-scoped navigation
- Org data model: 2 countries, 2 subsidiaries, 4 departments, designations, ~12 seeded employees with a real multi-level reporting chain
- Leave management end-to-end: employee submits → routes to direct manager → auto-escalates to Department Head when duration exceeds the leave type's threshold → HR Manager can view/override → status visible on the employee's dashboard, with a full timestamped approval audit trail
- 3 role-based dashboards (Admin/global, HR/subsidiary, Manager/team) — spec asked for at least 2 — with Recharts visualizations reading live Prisma queries, no hardcoded arrays
- Organization chart: recursive tree view, global for Super Admin, subsidiary-scoped for other roles

## What's partially there (Section 9.2 — Should Include)

- Holiday calendar per subsidiary (view only)
- Attendance: simplified daily status model with 14-day seeded history feeding an attendance-rate chart on the HR dashboard
- Notifications for leave status changes: the data model supports it — every status transition is already logged in \`LeaveApproval\` — but there's no in-app notification bell/UI yet. I ran out of time in the 2-day window.

## Documented but not built (Section 9.3)

**Payroll, Recruitment/ATS, Performance Management.** Per the spec, a short
note on how the schema would extend is enough here:

- **Payroll** — add a \`PayrollRun\` + \`Payslip\` model keyed to \`Employee\`, driven by \`Designation\`-level base pay and deductions from existing \`Attendance\`/\`LeaveBalance\` data.
- **Recruitment/ATS** — add \`JobRequisition\` (owned by \`Department\`) and \`Candidate\`/\`Application\` models; on hire, an \`Application\` converts into an \`Employee\` + \`User\`, reusing the existing onboarding path.
- **Performance Management** — add a \`ReviewCycle\` + \`Review\` model referencing the existing \`manager\`/\`directReports\` self-relation on \`Employee\`, since the reporting chain needed for review routing already exists.

## Project Structure

\`\`\`
prisma/schema.prisma      Data model (org hierarchy, users, leave workflow, attendance)
prisma/seed.ts            Seed data + demo users
src/lib/auth.ts           Password hashing, JWT session cookie
src/lib/rbac.ts           Role guards for Server Components
src/lib/leave.ts          Leave submission + multi-level approval/escalation state machine
src/middleware.ts         Route-level session + role enforcement
src/app/login             Login page + server action
src/app/admin             Super Admin: global dashboard, employee directory, org/dept management
src/app/hr                HR Manager: subsidiary dashboard, full leave register + override, holidays
src/app/manager           Manager: team dashboard, leave approvals queue
src/app/employee          Employee: personal dashboard, leave request form
src/app/org-chart         Shared recursive org chart (scoped by role)
src/components/ui         Tailwind/shadcn-style primitives (Button, Card, Table, Badge, Input)
src/components/charts     Recharts wrappers (bar, pie, line)
\`\`\`

## Design/Implementation Notes

- **RBAC is enforced twice** — \`middleware.ts\` blocks cross-role route access at the edge, and each Server Component also checks the role before querying data, so a route that somehow slipped past middleware still can't return another role's data.
- **Leave escalation is data-driven, not hardcoded** — each \`LeaveType\` has its own \`escalationDays\` threshold; a request longer than that automatically gets a second \`LeaveApproval\` step at the Department Head level, per Section 3.5.
- **No paid third-party services** required — SQLite needs zero setup, and nothing depends on a licensed API key for core functionality to run.
