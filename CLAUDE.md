# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Documentation

All project documentation is indexed at [`docs/INDEX.md`](./docs/INDEX.md).

## Project Layout

Four independent subprojects, each with its own git repository and `CLAUDE.md`:

| Directory | Stack | Role |
|---|---|---|
| `api/` | Spring Boot 3 / Java 21 / Gradle | HR/workforce REST API |
| `frontend/` | Angular 19 / TypeScript / npm | Multi-tenant self-service HR SPA |
| `connector/` | Spring Boot 3.5 / Java 21 / Gradle | Kafka consumer — syncs HR/payroll data |
| `docker/` | Docker Compose | Deployment infra: Nginx, PostgreSQL 14, Kafka (KRaft+TLS) |

## Commands

### API (`api/`)
```bash
./gradlew build             # full build
./gradlew build -x test     # skip tests
./gradlew test              # all tests
./gradlew test --tests "ee.andevis.api.SomeTest"  # single test class
./gradlew bootRun
```

### Frontend (`frontend/`)
```bash
npm install
npm run start    # dev server against remote API
npm run dev      # dev server against local API
npm run build    # production build
npm run lint     # ESLint
ng test          # Karma test runner
```

### Connector (`connector/`)
```bash
./gradlew build
./gradlew test --tests "ee.virosoft.sync.SomeTest"
./gradlew bootRun
```

### Docker (`docker/`)
```bash
docker compose up -d
docker compose down
docker compose logs -f [service]   # api | connector | app | db | kafka
git submodule update --recursive --remote
```

## Architecture

### Data Flow
```
frontend (Angular SPA)
    ↕ REST/JWT
api (Spring Boot)          ← Kafka producer events →    connector (Spring Boot)
    ↕                                                         ↕
PostgreSQL                                            External HR/payroll system
```

- `api` produces Kafka events; `connector` consumes them and syncs to the external HR/payroll system.
- `frontend` communicates exclusively with `api` over HTTPS. Dev server proxies `/auth/*` via `src/proxy.conf.json`.
- `docker/` mounts submodule `dist/` and `config/` directories at runtime — no image rebuild needed for config changes.

### Multi-Tenancy
- Every domain entity in `api` carries `companyId`.
- `frontend` builds are per-client via `environment.<client>.ts` file replacements in `angular.json`. Customer-specific assets are controlled by `SITE_CUSTOMER_FOLDER_NAME`.

### Auth
- `api` issues JWTs. Permissions are encoded as a base64 pipe-separated string in `localStorage['permissions']`.
- `JwtInterceptor` in the frontend attaches `Authorization: Bearer` and redirects to `/login` on 401.
- Auth methods (password, Smart-ID, Mobile-ID, ID-card, Entra ID, Dokobit) are runtime-toggled via `GET /api/properties`.
- `connector` uses mutual TLS (JKS keystores) for Kafka SSL.

### Key API Packages (`ee.andevis.api`)
- `auth` / `security` — JWT issuance, permissions, OAuth flows
- `profile` — employee records (profile, children, education, bank, health); confirmation-required changes use `Issue` + `Audit` flow
- `vacation` — vacation lifecycle, multi-role approval workflows, day-calculation strategies by leave type
- `application` — configurable HR forms with dynamic components and approval chains
- `setting` — company-level feature flags consumed as runtime toggles across all modules
- `producer` — Kafka outbox to connector

### Key Frontend Conventions
- All injectable services live in `src/app/services/` (one per domain) or inside feature-module `services/`.
- `@models/*` path alias resolves to `src/app/models/*`.
- `ngx-permissions` drives role-based UI; permissions are loaded after login via `NgxPermissionsService`.
- Lazy-loaded feature modules: `auth`, `pages`, `time-sheet`, `survey`, `stock`, `candidate`, `setting`, `errors`.

### Connector Patterns
- `*Loader` classes handle bulk JSON import; `*Consumer` classes handle real-time Kafka events.
- ActiveJDBC ORM: models have no fields — only extend `Model` with `@Table`. Connection lifecycle is manual (`openDatabase()` / `closeDatabase()`).
- External records are looked up by `(external_name, external_id)`; cross-entity mapping uses the `ServerMap` table.