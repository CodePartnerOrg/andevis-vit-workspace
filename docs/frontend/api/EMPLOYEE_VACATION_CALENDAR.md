# Employee Vacation Calendar API Context

## Scope
This document describes API behavior used by the `employee-vacation-calendar` feature (`/employee-vacation-calendar`) to help AI agents make safe changes.

## Feature Entry
- Route: `/employee-vacation-calendar`
- Route guard permissions (`pages-routing.module.ts`): `employee` or `VACATION_CALENDAR`
- Main component: `src/app/pages/employee-vacation-calendar/employee-vacation-calendar.component.ts`

## API Calls Used by This Screen
1. `GET /api/profiles/profile`
- Source: `UserService.getProfile()`
- Purpose: get current profile id before resolving default structure filter.

2. `GET /api/profile_structures/{profileId}`
- Source: `ProfileStructureService.get(profileId)`
- Purpose: preselect user structure in filter.

3. `GET /api/structures`
- Source: `DepartmentsService.list()`
- Purpose: load hierarchical departments/structures for tree filter.

4. `GET /api/references/company/vacation_type?filter=false`
- Source: `ReferenceService.list('company', 'vacation_type', null, false)`
- Purpose: vacation type reference list (localized labels), shown in filter UI.
- Note: current `onSearch()` implementation does not send `typeCode` to backend.

5. `GET /api/events`
- Source: `EventsService.list()`
- Purpose: decorate calendar days with holiday/shortened-workday metadata.

6. `GET /api/vacation_schedule?page={n}&size={size}&sort={sort}&...filters`
- Source: `VacationsService.getVacationSchedule(...)`
- Purpose: fetch paginated employee vacation schedule rows.
- Pagination strategy: frontend recursively requests every page and concatenates all rows.

## Vacation Schedule Request Filters
`onSearch()` builds query string manually from form values.

Supported filters sent to `/vacation_schedule`:
- `firstName`, `lastName`
  - If input has two words, first word -> `firstName`, second -> `lastName`.
  - Otherwise same value is sent to both.
- `structures`
  - Sent as selected structure ids array converted to comma-separated string.
- `status`
  - Mapped from approval dropdown to one of:
    - `isConfirmedByManager`
    - `isConfirmedByManagerOrIsConfirmedByHRManager`
    - `isConfirmedByHRManager`
- `startAt`, `endAt`
  - Sent as `YYYY-MM-DD`.

Validation constraints before request:
- `structures` required (`minLengthArrayValidator(1)`)
- date range validated by `dateYearRangeValidator`

## Response Shape Used
Endpoint returns `Pageable<VacationSchedule>`.

`VacationSchedule` fields used by UI:
- `fullName`
- `departmentCode`, `departmentName`
- `photo`, `photoKey`
- `vacations: UserVacation[]`

`UserVacation` fields used by item renderer:
- `id`, `startAt`, `endAt`, `color`, `type`, `status`, `isPlanned`

## Rendering/Behavior Notes That Affect API Work
- Day background color depends on `events` + weekend logic.
- Vacation item width is calculated from date span and visible period bounds.
- Large datasets can trigger many sequential requests because all pages are loaded.
- `onClear()` resets form but does not auto-trigger reload; user must search again.

## Filtering Rejected Vacations (VIT-5358)

When the `HIDE_REJECTED_VACATIONS_IN_EMPLOYEE_CALENDAR` admin setting is enabled:
- Only applies to users with the `employee` permission (not managers/juht).
- Rejected vacations (status `'rejected'`) are hidden in the shared team calendar view.
- The "Tagasi lükatud" legend item is removed.
- Employee's own rejected vacations in "Minu puhkused" are NOT affected.

Implementation:
- `employee-vacation-calendar.component.ts` loads the setting via `SettingService` and checks `employee` permission via `NgxPermissionsService`. Sets `hideRejectedVacations` flag.
- `employee-vacation-item.component.ts` accepts `@Input() hideRejected` and skips rendering vacations with `status === 'rejected'` when true.
- Legend item for rejected status uses `*ngIf="!hideRejectedVacations"` in template.

## Key Source Files
- `frontend/src/app/pages/employee-vacation-calendar/employee-vacation-calendar.component.ts`
- `frontend/src/app/pages/employee-vacation-calendar/vacation-item/employee-vacation-item.component.ts`
- `frontend/src/app/services/vacations.service.ts`
- `frontend/src/app/services/departments.service.ts`
- `frontend/src/app/services/profile-structure.service.ts`
- `frontend/src/app/services/reference.service.ts`
- `frontend/src/app/services/events.service.ts`
