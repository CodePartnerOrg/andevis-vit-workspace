# Vacation Module (`ee.andevis.api.vacation`)

## Purpose

This module handles employee vacation lifecycle in runtime (not annual planning):
- vacation request creation/update/delete,
- multi-role confirmation workflow,
- vacation day calculations and validation by leave type,
- schedule/calendar views,
- interruptions/revocations of already confirmed vacations,
- notifications and producer integration to external systems.

Related but separate module:
- `ee.andevis.api.scheduledvacation` for planned/scheduled vacations.

## Package Map

Base path: `api/src/main/java/ee/andevis/api/vacation/`

- `api/src/main/java/ee/andevis/api/vacation/controller/`: REST endpoints (`/vacations`, `/vacation_calculation`, `/vacation_schedule`, `/vacation_interruptions`, `/vacation_settings`).
- `api/src/main/java/ee/andevis/api/vacation/service/`: core business logic (`VacationService`, `VacationValidationService`, `VacationInterruptionService`, `VacationScheduleService`, notification/workflow helpers).
- `api/src/main/java/ee/andevis/api/vacation/validation/`: leave-type-specific calculation and validation strategies.
- `api/src/main/java/ee/andevis/api/vacation/entity/`: JPA entities (`Vacation`, `VacationWorkflow`, `VacationPeriod`, `VacationInterruption`, enums).
- `api/src/main/java/ee/andevis/api/vacation/repository/`: Spring Data repositories.
- `api/src/main/java/ee/andevis/api/vacation/spec/`: JPA Specification filters used across list/search/stat endpoints.
- `api/src/main/java/ee/andevis/api/vacation/task/`: scheduled jobs (manager sync, locking, reminder notifications).

## Status and Workflow Model

### Vacation Status (`VacationStatus`)

- `DRAFT`
- `SUBMITTED`
- `CONFIRMED`
- `REJECTED`
- `CANCELED`
- `CLOSED`

Notes:
- create flow sets `SUBMITTED` immediately (no draft creation endpoint in controller);
- soft deletion uses `deletedAt` (`@SQLRestriction("deleted_at IS NULL")` on `Vacation`).

### Approval Workflow (`VacationWorkflow`)

Rows are stored per vacation + role code (`MANAGER`, `PERSONAL`, optional `ACCOUNTANT`, etc.), with:
- approver profile snapshot fields,
- `isAccepted` (`null` = pending),
- `submittedAt` decision timestamp.

Final vacation status is derived in `VacationService.confirmVacation(...)`:
- all required roles accepted (based on vacation type confirm rules) => `CONFIRMED`;
- any rejection => `REJECTED`.

## Main Endpoints

### Core vacations (`VacationController`)

- `GET /vacations` (`vacations:read`)
  - paginated/filterable list with role-sensitive visibility.
- `GET /vacations/{id}` (`isAuthenticated`)
  - returns `403` when access checks fail.
- `POST /vacations` (`vacations:create` or `vacations:create:own`)
  - creates vacation, periods, workflows, comment, substitutes, notifications.
- `PUT /vacations/{id}` (`vacations:update` or `vacations:update:own`)
  - recalculates periods and updates workflows depending on status.
- `DELETE /vacations/{id}` (`vacations:delete` or `vacations:delete:own`)
  - soft delete + producer delete event + substitute cleanup.
- `PATCH /vacations/{id}/confirm` (`vacations:confirm`)
  - single approval/rejection decision.
- `PATCH /vacations/multi_confirm` (`vacations:confirm`)
  - batch decision for multiple vacation IDs.
- `POST /vacations/{id}/withdraw` (`vacations:withdraw`)
  - clears confirmation(s), sets status back to `SUBMITTED` if needed.
- `PATCH /vacations/{id}/reset` (`vacations:update` or `vacations:update:own`)
  - resets confirmed vacation back to submitted and rebuilds workflow.
- `PUT /vacations/{id}/send` (`vacations:send`)
  - manually sends create message to producer.
- `GET /vacations/latest` (`isAuthenticated`)
  - last confirmed vacation per type for user/profile.
- `GET /vacations/unused` (`isAuthenticated`)
  - available/unused days by type.
- `DELETE /vacations/{id}/files/{fileId}` (`isAuthenticated`)
  - unlinks file from vacation.

### Calculations (`VacationCalculationController`)

- `POST /vacation_calculation` (`vacations:create` or `vacations:create:own`)
  - validates request and returns calculated periods/errors.
- `GET /vacation_calculation/{date}/{type}` (`isAuthenticated`)
  - returns `availableTotal` for date/type.

### Schedule (`VacationScheduleController`)

- `GET /vacation_schedule` (`isAuthenticated`)
  - org/department schedule view by profile list.
- `GET /vacation_schedule/profile` (`isAuthenticated`)
  - current user profile-centered schedule.

### Interruptions (`VacationInterruptionController`)

- `POST /vacation_interruptions` (`isAuthenticated`)
  - interrupt/revoke confirmed vacations; may split into two vacations.
- `GET /vacation_interruptions/types` (`isAuthenticated`)
  - interruption enum values.

### Validation settings (`VacationSettingsController`)

- `GET /vacation_settings` (`isAuthenticated`)
- `PUT /vacation_settings` (`isAuthenticated`)

Stores annual leave validation settings under vacation type `TLPP` reference settings JSON.

### Profile-specific list (outside package)

- `GET /profile/vacations` in `profile/ProfileVacationController` (`isAuthenticated`)
  - own vacations or another profile if caller has `vacations:read`.

## Core Business Flows

### 1) Create vacation

`VacationService.create(...)`:
- maps request dates/type/payment/child/additional fields,
- resolves structure snapshot at start date,
- validates collisions (`validateDateCollisions`) for selected profile scenarios,
- calculates periods via `VacationValidationService.generateVacationPeriods(...)`,
- persists `Vacation` + `VacationPeriod` + employee comment + substitutes,
- creates workflow based on current role:
  - `assistant`: manager workflow pending,
  - `manager`: manager accepted + personal pending,
  - `personal`: manager+personal accepted => immediate `CONFIRMED`,
  - default employee: manager pending.
- sends notification and/or producer messages depending on role and status.

### 2) Update vacation

`VacationService.update(...)`:
- recalculates periods with validation,
- updates type/payment/files/child,
- workflow behavior by current status:
  - `REJECTED`: clears non-manager workflows and resubmits,
  - `SUBMITTED`: resets manager workflow decision and can replace manager assignee.
- updates employee comment and substitutes.

### 3) Confirm/reject

`VacationService.confirm(...)` -> `confirmVacation(...)`:
- decision is written to role-specific workflow,
- role-specific paths for `manager`, `personal`, optional `accountant`,
- on full acceptance by required roles:
  - sets `CONFIRMED`,
  - sends confirmation notification,
  - sends producer message(s),
  - notifies deputies.
- on rejection:
  - sets `REJECTED`,
  - sends rejection email to employee and optional type-level email.

### 4) Withdraw confirmation

`VacationService.withdrawConfirmation(...)`:
- updates comment,
- if previously confirmed, sends withdraw event to producer and returns status to `SUBMITTED`,
- clears workflow decision for current role or for all roles (`isClearAll=true` path),
- sends withdrawal notifications.

### 5) Reset confirmed vacation

`VacationService.reset(...)`:
- sends reset notifications,
- sends producer delete event,
- sets status to `SUBMITTED`, clears/recreates workflows (manager preserved/recreated),
- clears external sync metadata (`externalId`, `hash`-related metadata, lock flags).

### 6) Interrupt/revoke confirmed vacation

`VacationInterruptionService.create(...)`:
- only works for `CONFIRMED` vacation,
- validates started state and request interval,
- optional `revocation` path sets status `CANCELED`,
- when interruption ends before original end date:
  - original vacation is shortened,
  - a cloned continuation vacation is created,
  - workflows/comments are cloned,
  - both vacations are recalculated and synced to producer (`update` + `create`).
- also updates/adjusts deputies for interrupted interval.

## Calculation and Validation Architecture

### Entry point

`VacationValidationService.generateVacationPeriods(user, company, request)`.

### Strategy dispatch

`VacationType` enum maps type codes (`TLPP`, `TLPLHP`, `TLPEMP`, etc.) to concrete validation/calculation classes in `validation/`.

### Inputs used for computation

- active contract + careers,
- public/national holidays,
- profile children and education,
- absences/sick leaves,
- profile-company relation,
- reference settings/options,
- optional remote VClient balance integration (company/enterprise specific).

### Typical outputs

`VacationCalculationResponse` with:
- period breakdown (`VacationCalculationPeriod` list),
- totals/booked days/holidays,
- optional domain error (`Error`) when rules are violated.

## Filtering and Access Rules

`VacationService.filter(...)` combines:
- `VacationSpec` conditions (company, type, structure(s), dates, sync, status modes),
- role-scoped profile visibility via `AccessManagerService`,
- per-item flags:
  - `mayConfirm` (can current role confirm now),
  - `mayFullConfirmByManager` (enterprise-specific manager shortcut),
  - `isStarted` (note: currently set using inverse check `today.isBefore(startAt)`).

`getVacationResponseById(...)` access checks include:
- owner access,
- privileged roles (`personal`, accounting/admin variants),
- assigned workflow participant,
- manager/assistant profile-scope access.

## External Side Effects

### Producer integration

Main sends are from `VacationService`/`VacationInterruptionService`:
- create/update payloads generated by `VacationService.getMessage(...)`,
- delete/withdraw events via `ProducerService` helper methods.

Payload includes vacation core data, period split, substitutes, optional child backup id, and file attachments for create.

### Notifications

- Email notifications in `EmailNotificationService` and `VacationNotificationService`.
- Deputy notifications are emitted on confirmation and scheduled checks.

### Scheduled tasks (`VacationTask`)

- `02:00` daily: sync direct manager assignments for submitted vacations.
- `03:00` daily: lock finished confirmed vacations.
- `06:00` daily: notify deputies.
- `06:15` daily: notify users about vacations starting in 14 days.

## Key Entities for Agents

- `Vacation`: request aggregate with status, period, type, sync metadata.
- `VacationWorkflow`: per-role decision records.
- `VacationPeriod`: computed day allocations.
- `VacationInterruption`: interruption audit/event record.
- `VacationFile`: link between vacation and storage file.

## Where to Change Behavior

- Endpoint contract/authorization: `api/src/main/java/ee/andevis/api/vacation/controller/`.
- Vacation lifecycle and side effects: `VacationService`.
- Day calculation/rule logic by leave type: `api/src/main/java/ee/andevis/api/vacation/validation/` + `VacationType` mapping.
- Interruption semantics: `VacationInterruptionService`.
- Filtering/search behavior: `VacationSpec`, `VacationWorkflowSpec`, service filter methods.
- Reminder/background behavior: `api/src/main/java/ee/andevis/api/vacation/task/`.

## Before-Start Restriction

When `VACATION_BEFORE_START_RESTRICTION` setting is enabled, vacations with payment type `BEFORE_VACATION` must have a start date at least N days in the future (N = `options.days`, default 14).

- **Create / Update**: checked in `VacationValidationService.validateRequest()`, returns error key `vacationBeforeStartRestriction`.
- **Delete (cancel)**: checked in `VacationController.delete()` via `checkBeforeStartRestriction()`, returns `400 BAD_REQUEST` with error key `vacationBeforeStartRestriction`.
- Vacations with any other payment type are not affected.

## Notable Caveats

- `VacationController.update(...)`: with `vacations:update:own`, non-owner path does not explicitly return forbidden; it can return unchanged entity.
- `ProfileVacationController.index(...)`: returns `null` if `id` parameter is used without `vacations:read`.
- `VacationInterruptionRequest` uses `@NotBlank` on non-String fields (`UUID`, `LocalDate`, enum); validation annotations are likely incorrect (`@NotNull` expected).
- `VacationService.setStarted(...)` sets `isStarted` using `today.isBefore(startAt)` (semantically inverse naming).
- `VacationService.create(...)` saves `finalVacation` but later calls `save(vacation)` again; functionally works but is inconsistent.
