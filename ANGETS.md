# ANGETS.md

## Purpose
This workspace is a coordination root for four related codebases (`api`, `frontend`, `connector`, `docker`) plus cross-project documentation in `docs/`.

Use this file as a fast onboarding map before changing code.

## Repository Structure
- `docs/`: canonical project documentation index and module-level context for API/frontend domains.
- `api/`: Spring Boot 3 (Java 21, Gradle) HR/workforce REST backend with Flyway migrations and Kafka producer events.
- `frontend/`: Angular 19 (TypeScript, npm) multi-tenant self-service SPA.
- `connector/`: Spring Boot 3.5 (Java 21, Gradle) Kafka consumer that syncs data to external HR/payroll integrations.
- `docker/`: Docker Compose deployment stack (Nginx app, API, connector, PostgreSQL, Kafka).

Important: there is no single root build. Run build/test commands inside the target subproject directory.

## Docs-First Navigation (Read In Order)
1. `docs/INDEX.md` — entry point and document index.
2. API backend domain docs:
   - `docs/api/applications/APPLICATIONS.md`
   - `docs/api/vacations/VACATIONS.md`
   - `docs/api/profile/PROFILE.md`
   - `docs/api/settings/SETTINGS.md`
3. Frontend behavior docs:
   - `docs/frontend/application/APPLICATIONS.md`
   - `docs/frontend/api/ADMIN_SETTINGS.md`
   - `docs/frontend/api/EMPLOYEE_VACATION_CALENDAR.md`

## High-Level Architecture
- `frontend` calls `api` over REST/JWT.
- `api` writes business data to PostgreSQL and emits Kafka producer events.
- `connector` consumes Kafka events and maps/syncs data with external systems.
- `docker` provides runtime wiring and mounted configs for all services.

## Domain Hotspots (Most Frequently Changed)
- Application workflows/forms: `api/src/main/java/ee/andevis/api/application` and related frontend pages/services.
- Vacation lifecycle/calculation/confirmation: `api/src/main/java/ee/andevis/api/vacation` plus frontend vacation modules.
- Profile and confirmation-required personal data updates: `api/src/main/java/ee/andevis/api/profile`.
- Feature flags/options: `api/src/main/java/ee/andevis/api/setting` and frontend admin settings pages.

## Quick Commands
### API (`api/`)
- `./gradlew build`
- `./gradlew test`
- `./gradlew bootRun`

### Frontend (`frontend/`)
- `npm install`
- `npm run start` (remote API)
- `npm run dev` (local API)
- `npm run build`
- `npm run lint`

### Connector (`connector/`)
- `./gradlew build`
- `./gradlew test`
- `./gradlew bootRun`

### Docker (`docker/`)
- `docker compose up -d`
- `docker compose down`
- `docker compose logs -f <service>`

## Agent Guardrails
- Check permissions/role scope before changing API or frontend behavior; many endpoints are role-sensitive.
- Check setting flags (`SettingKey`) before assuming a feature is globally enabled.
- For vacation/application changes, validate side effects: workflow transitions, notifications, and producer/Kafka integration.
- Prefer updating docs with behavior changes; start with the relevant file in `docs/` and keep `docs/INDEX.md` consistent.
- Because this workspace contains multiple repositories, run `git status` in the correct project directory before committing.

## Practical Change Workflow
1. Identify affected domain and open matching `docs/...` file first.
2. Update backend logic and authorization checks.
3. Update frontend service + route/component behavior if API contract changed.
4. Run targeted tests/build in changed subproject(s).
5. Update docs for any behavior/API changes.
