# Documentation Index

This is the central entry point for all project documentation.

## API Backend Documentation (`api/`)

Backend is a **Spring Boot 3 / Java 21** HR/workforce REST API (`ee.andevis.api`).

| Document | Description |
|---|---|
| [Applications Module](./api/applications/APPLICATIONS.md) | Configurable internal applications (forms, approval workflows, notifications, PDF generation) |
| [Vacation Module](./api/vacations/VACATIONS.md) | Employee vacation lifecycle — create/update/confirm/interrupt, calculation strategies, schedules |
| [Profile Module](./api/profile/PROFILE.md) | Employee profiles, sub-records (children, education, bank, health), manager/deputy context, confirmation workflows |
| [Settings Module](./api/settings/SETTINGS.md) | Company-level feature flags and typed options used as runtime toggles across all modules |

## Frontend Documentation (`frontend/`)

Frontend is an **Angular 19** multi-tenant self-service HR portal.

| Document | Description |
|---|---|
| [Application Functionality Overview](./frontend/application/APPLICATIONS.md) | High-level overview of all functional domains, routing map, authorization model, multi-tenant behavior |
| [Admin Settings API Context](./frontend/api/ADMIN_SETTINGS.md) | API calls and behavior for the `/admin/settings` admin tabs (settings, contact, vacation, SMTP, agreements, app forms, candidates) |
| [Employee Vacation Calendar API Context](./frontend/api/EMPLOYEE_VACATION_CALENDAR.md) | API calls, filters, and rendering behavior for the `/employee-vacation-calendar` screen |

## Project Components Overview

| Component | Technology | Purpose |
|---|---|---|
| `api/` | Spring Boot 3 / Java 21 / Gradle | HR/workforce REST API with Flyway DB migrations |
| `frontend/` | Angular 19 / TypeScript / npm | Multi-tenant self-service HR SPA |
| `connector/` | Spring Boot 3.5 / Java 21 / Gradle | Kafka consumer — syncs external HR/payroll data to local PostgreSQL |
| `docker/` | Docker Compose | Multi-container deployment: Nginx, API, Connector, PostgreSQL, Kafka (KRaft+TLS) |
