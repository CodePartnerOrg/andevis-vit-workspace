# Application Functionality Overview

## Purpose
This project is an Angular 19 multi-tenant self-service HR portal. A single codebase serves different client companies through environment-based builds and runtime configuration.

## Primary User Areas
- Employee self-service: profile, workplace info, salary view, health info, vacations, applications, questionnaires, stock items, time sheets.
- Manager/HR workflows: employee vacation approvals, planned vacation requests, company vacation calendar, workforce profiles, careers/workload, surveys.
- Admin operations: settings, roles, logs, structure managers, application form configuration, stock administration.
- Candidate flow: invitation-based candidate dashboard and form; HR-side candidate list and candidate detail view.

## Main Functional Domains
- Authentication and access selection
  - Supports password, Entra ID, ID-card, Smart-ID, Mobile-ID (runtime toggled via `/api/properties`).
  - Supports company/role selection after login when user has multiple accesses.
- Profile domain
  - Personal info, addresses, contacts, bank account, children, education, documents, disability, health/sick-leave/absence notices.
- Vacation domain
  - Employee vacation request creation/update/withdraw.
  - Manager confirmation/rejection and multi-confirmation.
  - Vacation balance/calculation, schedules, employee and company calendars.
  - Planned vacations and related approval workflows.
- Application domain
  - Employee application list and per-application forms.
  - Application view/status/confirmation.
  - Admin-side application property/form configuration.
- Organization/workforce domain
  - Workplaces, departments, structure managers, profile directory, workload/careers.
- Additional modules
  - Surveys (`profile-surveys`, `manage-surveys`, `admin/surveys`).
  - Stock (`stock`, `manage/stock`, `admin/stock`).
  - Time sheets (`time-sheets`, `work-schedules`).
  - Candidate module (`candidate/*`, `candidates/*`).

## Routing Map (high level)
- Root router lazy-loads: `auth`, `pages`, `time-sheet`, `survey`, `stock`, `candidate`, `setting`, `errors`.
- Authenticated shell is `LayoutComponent`.
- Key routes in `pages`:
  - `/dashboard`, `/profile`, `/workplaces`, `/health`, `/salary`
  - `/vacations`, `/employee-vacations`, `/vacations-plan`, `/vacation-calendar`, `/employee-vacation-calendar`
  - `/applications`, `/applications/:id`, `/applications/:id/view`, `/employee-applications`
  - `/profiles`, `/careers`, `/issues`, `/admin/structure_managers`, `/version`

## Authorization Model
- Global auth guard: JWT presence check (`canActive`).
- HTTP auth: `JwtInterceptor` adds bearer token and logs out on `401`.
- Fine-grained route/UI access: `ngx-permissions` with role/permission strings.
- Permissions are base64-encoded in local storage and loaded after successful authorization.

## Runtime Configuration & Multi-Tenant Behavior
- Build-time tenant switching via multiple `environment.*.ts` files and Angular `fileReplacements`.
- Runtime module config from `GET /api/properties`:
  - `auth` methods enabled/disabled.
  - `assistant` feature flag.
- `SITE_CUSTOMER_FOLDER_NAME` controls customer-specific visual assets.

## API Usage Pattern
- Services under `frontend/src/app/services` and feature-module services are thin HTTP wrappers around REST endpoints.
- Base URL comes from `environment.API_URL`.
- Examples of core API groups:
  - Auth/session: `login`, `session`, `authorize`, `token/check`, password reset/activation.
  - HR domains: `profiles/*`, `vacations*`, `vacation_schedule*`, `applications*`, `careers*`, `issues*`, `structures*`, `roles*`, `settings*`, `stock*`, `surveys*`.

## Localization & UI
- Languages: `et`, `en`, `ru` via `@ngx-translate`.
- Language persisted in local storage and synced to profile settings.
- UI stack: Angular Material + Metronic-based layout components.

## Important Code Locations
- App routing: `frontend/src/app/app-routing.module.ts`
- Main authenticated routes: `frontend/src/app/pages/pages-routing.module.ts`
- Feature module routes:
  - `frontend/src/app/modules/*/*-routing.module.ts`
- Auth and token handling:
  - `frontend/src/app/services/authentication.service.ts`
  - `frontend/src/app/services/jwt.interceptor.ts`
  - `frontend/src/app/guards/jwt.guard.ts`
- Runtime config:
  - `frontend/src/app/services/modules.service.ts`
  - `frontend/src/app/models/module-configs/auth.module.config.ts`

## Agent Quick Start
1. Check route ownership first (`pages` vs feature modules under `modules`).
2. Locate matching service in `frontend/src/app/services` or feature-module `services/`.
3. Verify permissions on route and template before changing behavior.
4. Verify tenant/runtime flags if a feature appears hidden.
