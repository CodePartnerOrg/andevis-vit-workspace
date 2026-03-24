# Settings Module

## Purpose

This module manages company-level feature flags and module options used across the API.

It provides:
- CRUD-like API for settings (list/get/upsert),
- typed options mapping for selected keys,
- convenience checks (`isEnabled`) for runtime branching,
- scheduled behavior tied to specific setting keys.

## Package Map

### Core / General Settings (`ee.andevis.api.setting`)

Base path: `api/src/main/java/ee/andevis/api/setting/`

- `SettingController`: `/settings` endpoints (general/core settings).
- `SettingService`: generic upsert/read/mapping logic — shared by all domain settings.
- `SettingRepository`: persistence access.
- `entity/Setting.java`: `_settings` table model.
- `entity/SettingKey.java`: enum of general setting keys.
- `entity/SettingKeyCode.java`: marker interface for domain-specific key enums.
- `entity/SettingKeySpec.java`: immutable configuration holder (options class, parent dependency).
- `entity/SettingGroup.java`: enum discriminator (`GENERAL`, `VACATIONS`).
- `payload/SettingDTO.java`: transport wrapper for key + enabled + typed options.
- `SettingTask`: periodic action based on `AUTO_ACTIVE_USERS`.

### Leave Settings (`ee.andevis.api.settings.leave`)

Base path: `api/src/main/java/ee/andevis/api/settings/leave/`

- `controller/LeaveSettingsController`: `/settings/leave` endpoints.
- `service/LeaveSettingsService`: leave-specific setting operations.
- `service/LeaveValidationSettingsService`: legacy field-visibility validation settings.
- `entity/LeaveSettingKey`: leave-specific setting keys enum (`group = VACATIONS`).
- `payload/PlannedVacationsOptionDTO`: typed options for `PLANNED_VACATIONS`.
- `payload/LeaveBeforeStartRestrictionOptionDTO`: typed options for `VACATION_BEFORE_START_RESTRICTION`.
- `payload/LeaveTypePayload`: leave type summary DTO.
- `payload/LeaveTypeDetailedPayload`: leave type detail DTO.
- `payload/LeaveValidationSetting`: legacy validation setting DTO.

### Deprecated (backward compatibility)

The old `VacationSettingsController` at `/vacation_settings` is deprecated and delegates
to `LeaveSettingsController`. Old services (`VacationSettingsService`, `VacationValidationSettingsService`)
and entity (`VacationSettingKey`) are also deprecated — use the `settings.leave` equivalents.

### Adding a New Settings Domain

To add a new settings domain (e.g., absence, request):
1. Create a new package `ee.andevis.api.settings.<domain>/` with controller, service, entity, payload.
2. Create a key enum implementing `SettingKeyCode` with its own `SettingGroup` value.
3. Add the new group to `SettingGroup` enum.
4. Register REST endpoints at `/settings/<domain>`.
No changes to existing domains or shared code are required.

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

### General Settings — `/settings`

All controller methods require `isAuthenticated()`, and each route additionally requires `settings:manage`.

- `GET /settings` — returns all `SettingKey` values as DTOs for the company. Missing keys are synthesized as disabled (`isEnabled=false`).
- `GET /settings/{key}` — returns one mapped setting DTO, or `404`.
- `PUT /settings` — upserts by `companyId + key`. Payload: `SettingDTO<?>`.

### Leave Settings — `/settings/leave`

All endpoints require `isAuthenticated()`. Most require `settings:manage`.

- `GET /settings/leave` — returns all `LeaveSettingKey` values as DTOs. (`settings:manage`)
- `PUT /settings/leave` — upserts a single leave setting. (`settings:manage`)
- `GET /settings/leave/{key}` — returns one leave setting by key. (all authenticated)
- `GET /settings/leave/types` — list all leave types. (`settings:manage`)
- `GET /settings/leave/types/{id}` — get detailed leave type. (`settings:manage`)
- `PUT /settings/leave/types/{id}` — update leave type. (`settings:manage`)
- `PATCH /settings/leave/types/{id}?isActive=true|false` — toggle active status. (`settings:manage`)
- `GET /settings/leave/validations` — get legacy field-visibility settings. (`settings:manage`)
- `PUT /settings/leave/validations` — update legacy field-visibility settings. (`settings:manage`)

### Deprecated — `/vacation_settings`

All old endpoints under `/vacation_settings` still work but are deprecated.
They delegate to the `/settings/leave` implementation. Clients should migrate to the new URLs.

No delete endpoint is implemented.

## Setting Keys

### General Keys (`SettingKey`, group: `GENERAL`)

- `ABSENCES_MODULE`, `ABSENCE_NOTICE`, `ABSENCES_ENTRY_COMPANY`
- `AUTO_ACTIVE_USERS`
- `STOCK_MODULE`, `SURVEY_MODULE`
- `DELETE_BANK_ACCOUNT`
- `TIME_SHEETS`, `TIME_SHEET_TOTALS`
- `PERSONAL_CAN_CONFIRM_APPLICATIONS`, `PERSONAL_CAN_CREATE_APPLICATIONS`
- `PROFILE_BOOKMARKS`
- `CANDIDATES_MODULE` (typed options: `CandidateOptionDTO`)
- `MANAGER_CAN_READ_SALARY`

### Leave Keys (`LeaveSettingKey`, group: `VACATIONS`)

- `PLANNED_VACATIONS` (typed options: `PlannedVacationsOptionDTO`)
- `VACATION_SCHEDULE_ACCESSIBLE`
- `VIEW_COLLEAGUES_ACTUAL_VACATIONS`
- `CHANGE_CONFIRMED_VACATIONS`
- `HIDE_REJECTED_VACATIONS_IN_EMPLOYEE_CALENDAR`
- `CANCEL_UNSTARTED_VACATION_AFTER_CONFIRMATION`
- `CANCEL_VACATION_WITHOUT_CONFIRMATION` (parent: `CANCEL_UNSTARTED_VACATION_AFTER_CONFIRMATION`)
- `WITH_WAITING_VACATIONS`
- `VACATION_BEFORE_START_RESTRICTION` (typed options: `LeaveBeforeStartRestrictionOptionDTO`, default days=14)
- `SYNC_CHANGES_AFTER_CONFIRMATION`
- `VACATION_APPLICATIONS_REPORT`
- `DELETE_CONFIRMED_VACATION`
- `VACATION_CALENDAR`
- `VACATIONS_PARALLEL_STRUCTURES`

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
