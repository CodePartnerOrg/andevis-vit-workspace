# Settings Module (`ee.andevis.api.setting`)

## Purpose

This module manages company-level feature flags and module options used across the API.

It provides:
- CRUD-like API for settings (list/get/upsert),
- typed options mapping for selected keys,
- convenience checks (`isEnabled`) for runtime branching,
- scheduled behavior tied to specific setting keys.

## Package Map

Base path: `api/src/main/java/ee/andevis/api/setting/`

- `SettingController`: `/settings` endpoints.
- `SettingService`: upsert/read/mapping logic.
- `SettingRepository`: persistence access.
- `api/src/main/java/ee/andevis/api/setting/entity/Setting.java`: `_settings` table model.
- `api/src/main/java/ee/andevis/api/setting/entity/SettingKey.java`: enum of supported keys.
- `api/src/main/java/ee/andevis/api/setting/payload/SettingDTO.java`: transport wrapper for key + enabled + typed options.
- `api/src/main/java/ee/andevis/api/setting/payload/`: typed `options` payloads for selected keys.
- `SettingTask`: periodic action based on `AUTO_ACTIVE_USERS`.

## Data Model

`Setting` fields:
- `id: UUID`
- `companyId: UUID`
- `isEnabled: Boolean`
- `key: String`
- `options: jsonb`
- audit timestamps (`createdAt`, `updatedAt`)

Persistence notes:
- repository lookup is by `(companyId, key)` pattern,
- table definition in migrations allows multiple rows per company with different keys,
- service behaves as upsert for one row per `(companyId, key)`.

## API Endpoints

Base route: `/settings`

All controller methods require `isAuthenticated()`, and each route additionally requires `settings:manage`.

- `GET /settings`
  - returns all `SettingKey` values as DTOs for the company,
  - missing keys are synthesized as disabled (`isEnabled=false`) with no DB row.

- `GET /settings/{key}`
  - returns one mapped setting DTO,
  - returns `404` when no row exists for that key/company.

- `PUT /settings`
  - upserts by `companyId + key`,
  - payload: `SettingDTO<?>` (`key` required, `isEnabled`, optional `options`),
  - returns mapped DTO after save.

No delete endpoint is implemented.

## Setting Keys (`SettingKey`)

Current enum:
- `ABSENCES_MODULE`
- `ABSENCE_NOTICE`
- `ABSENCES_ENTRY_COMPANY`
- `AUTO_ACTIVE_USERS`
- `STOCK_MODULE`
- `SURVEY_MODULE`
- `DELETE_BANK_ACCOUNT`
- `CHANGE_CONFIRMED_VACATIONS`
- `TIME_SHEETS`
- `TIME_SHEET_TOTALS`
- `VACATIONS_PARALLEL_STRUCTURES`
- `PERSONAL_CAN_CONFIRM_APPLICATIONS`
- `PERSONAL_CAN_CREATE_APPLICATIONS`
- `VACATION_CALENDAR`
- `PROFILE_BOOKMARKS`
- `CANDIDATES_MODULE`
- `WITH_WAITING_VACATIONS`
- `PLANNED_VACATIONS`
- `MANAGER_CAN_READ_SALARY`
- `VACATION_BEFORE_START_RESTRICTION`

## Typed Options Support

`SettingService` currently maps typed `options` only for:

1. `CANDIDATES_MODULE` -> `CandidateOptionDTO`
- `email`
- `validDays`
- `translations: List<LocaleDTO(locale, content)>`

2. `PLANNED_VACATIONS` -> `PlannedVacationsOptionDTO`
- `startDate`
- `endDate`
- `isFourteenDays`
- `isSevenDays`
- `isPlannedToActual`
- `paymentType`

3. `VACATION_BEFORE_START_RESTRICTION` -> `VacationBeforeStartRestrictionOptionDTO`
- `days` (Integer, default `14`)

For all other keys, `options` is ignored by mapper and DTOs are effectively boolean toggle only.

## Service Behavior

### `update(SettingDTO<?> dto, UUID companyId)`

- Finds existing setting by `(companyId, key)`.
- If not found, creates new row.
- Writes `isEnabled`, `key`, `companyId`.
- Converts `options` to typed DTO only for `CANDIDATES_MODULE`/`PLANNED_VACATIONS`/`VACATION_BEFORE_START_RESTRICTION`.
- Saves and returns mapped DTO.

### `get(UUID companyId)`

- Loads existing rows for company.
- Iterates all enum keys and returns full key catalog.
- Missing rows are represented as disabled DTO entries.

### `get(UUID companyId, String key)`

- Returns mapped DTO if row exists, else `null`.

### `isEnabled(UUID companyId, SettingKey key)`

- Fast boolean lookup helper used across modules.

## Scheduled Behavior

`SettingTask.activateUsers()` runs every 5 hours:
- reads all settings with key `AUTO_ACTIVE_USERS`,
- for enabled companies, finds active profiles,
- activates inactive users for those profiles (`UserService.activate(..., true)`).

## Cross-Module Usage (Important for Agents)

Settings are widely consumed as feature toggles. Notable usages:

- `VACATION_CALENDAR`: vacation schedule visibility mode.
- `VACATIONS_PARALLEL_STRUCTURES`: scheduled vacation/profile structure scope logic.
- `CHANGE_CONFIRMED_VACATIONS`: controls whether owner may edit confirmed vacations.
- `WITH_WAITING_VACATIONS`: vacation available-days calculation behavior.
- `PERSONAL_CAN_CONFIRM_APPLICATIONS`: affects application/vacation confirmation flows.
- `PLANNED_VACATIONS`: enables planned vacation flows and contributes permission `CREATE_PLANNED_VACATIONS` when in date window.
- `PROFILE_BOOKMARKS`: enables profile contacts/bookmarks features.
- `DELETE_BANK_ACCOUNT`: controls bank account deletion capability.
- `MANAGER_CAN_READ_SALARY`: allows/disallows manager salary access.
- `ABSENCES_ENTRY_COMPANY`: affects absence listing scope.
- `CANDIDATES_MODULE`: candidate module behavior via typed options.
- `STOCK_MODULE`: role filtering and notifications behavior in other services.

`PermissionService` also injects enabled setting keys directly into JWT permission payload as plain strings.

## Where to Change Behavior

- Add/remove exposed keys: `api/src/main/java/ee/andevis/api/setting/entity/SettingKey.java`.
- Change API contract/security: `SettingController`.
- Add typed options for a key: `SettingService.update(...)` + `SettingService.map(...)` + new option DTO.
- Change persistence/query behavior: `SettingRepository` and `Setting` entity.
- Modify auto-activation schedule: `SettingTask`.
- Adjust setting-driven authorization side effects: `api/src/main/java/ee/andevis/api/auth/service/PermissionService.java`.

## Notable Caveats

- `SettingService` constructor receives an `ObjectMapper` but ignores it and creates a new mapper instance; global mapper config/custom modules are not reused.
- `Setting.options` typing is partial: only two keys map typed options; other keys silently drop/unmap custom `options` in DTO mapping.
- `Setting` entity marks `companyId` as `unique=true`, while runtime logic expects multiple settings per company (distinguished by `key`). This mismatch can cause schema conflicts if DDL is generated from entity metadata.
- `SettingDTO` validates `key` only; `isEnabled` is nullable and not explicitly validated.
