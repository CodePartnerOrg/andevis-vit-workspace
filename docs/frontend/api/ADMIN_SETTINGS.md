# Admin Settings API Context

## Scope
This document describes API behavior behind `/admin/settings` (admin settings tabs) for AI agents.

## Feature Entry
- Route: `/admin/settings`
- Access: `admin` permission (via `SettingRoutingModule`)
- Shell component: `SettingsComponent`
- Tabs: General, Contact Settings, Vacation Settings, Notifications, Agreements, Application Settings, Candidates (conditional), Personal Data Permissions.

## Core Settings API
### `GET /api/settings`
- Source: `SettingService.get()`
- Used by: General tab, Vacation Settings tab.
- Returns list of `Setting { id, key, isEnabled, options }`.

### `GET /api/settings/{key}`
- Source: `SettingService.getByKey(key)`
- Used by:
  - `PLANNED_VACATIONS` (Vacation Settings)
  - `CANDIDATES_MODULE` (Candidates tab)

### `PUT /api/settings`
- Source: `SettingService.update(setting)`
- Used for toggles and key-based options updates.
- Typical payloads:
  - Generic module toggle: `{ key, isEnabled }`
  - Planned vacations: `{ key: 'PLANNED_VACATIONS', isEnabled, options: { startDate, endDate, isFourteenDays, isSevenDays } }`
  - Candidates module: `{ key: 'CANDIDATES_MODULE', isEnabled, options: { email, validDays, translations[] } }`

## Tab-by-Tab API Map

### 1. General
- `GET /api/settings`
- `PUT /api/settings`
- Important keys handled in UI:
  - `ABSENCES_MODULE`, `ABSENCES_ENTRY_COMPANY`, `AUTO_ACTIVE_USERS`, `STOCK_MODULE`, `SURVEY_MODULE`
  - `DELETE_BANK_ACCOUNT`, `CHANGE_CONFIRMED_VACATIONS`, `TIME_SHEETS`, `TIME_SHEET_TOTALS`
  - `VACATIONS_PARALLEL_STRUCTURES`, `VACATION_CALENDAR`, `PROFILE_BOOKMARKS`, `CANDIDATES_MODULE`, `ABSENCE_NOTICE`
  - `PERSONAL_CAN_CONFIRM_APPLICATIONS`, `PERSONAL_CAN_CREATE_APPLICATIONS`, `MANAGER_CAN_READ_SALARY`
  - `HIDE_REJECTED_VACATIONS_IN_EMPLOYEE_CALENDAR` — hides rejected vacations from the employee team calendar and legend (employee role only)

### 2. Contact Settings
- `GET /api/companies/company/contact` (`CompanyService.getCompanyInfo`)
- `POST /api/companies` (`CompanyService.update`)
- Payload structure is `CompanyInfoVisibleField` with per-field visibility/value objects:
  - `address`, `phone`, `email`, `accountantEmail`, `accountantPhone`, `personalEmail`, `personalPhone`, `personalManagerEmail`
  - each as `FieldCustomVisibility { fieldName, value, isVisible }`

### 3. Logo Settings
- `GET /api/companies/company/contact` (current logo + `fileId`)
- `POST /api/storage/upload` (upload new image file)
- `GET /api/storage/thumbnail/{h}/{w}/{hash}` (preview)
- `POST /api/companies/logo` (persist selected `fileId` + optional `logoWidthSize`)
- `DELETE /api/companies/logo`

### 4. Vacation Settings
- `GET /api/vacation_settings` (`VacationSettingsValidationService.get`)
- `PUT /api/vacation_settings` (`VacationSettingsValidationService.update`)
- Also uses standard settings API:
  - `GET /api/settings` for `WITH_WAITING_VACATIONS`
  - `GET /api/settings/PLANNED_VACATIONS`
  - `PUT /api/settings` for planned vacations options/toggles
- `vacation_settings` payload is list of `FieldCustomVisibility` objects.

### 5. Notifications Settings (SMTP)
- `GET /api/email_configs`
- `POST /api/email_configs`
- `POST /api/email_configs/test`
- Payload model: `SmtpEmailSettings` (`host`, `port`, `starttls`, `username`, `password`, `sender`, `senderName`, ...)

### 6. Agreements
- `GET /api/agreements`
- `POST /api/agreements`
- Agreement payload includes status/start date and localized content list.

### 7. Application Settings (form builder/admin)
- Properties:
  - `GET /api/application-properties`
  - `GET /api/application-properties/{id}`
  - `PUT /api/application-properties/{id}`
  - `DELETE /api/application-properties/{id}`
  - `POST /api/application-properties/positions` (drag-drop order)
- Workflows:
  - `GET /api/application-property-workflows/{id}`
  - `POST /api/application-property-workflows/{id}`
- Components:
  - `GET /api/application-property-components?propertyId={id}`
  - `GET /api/application-property-components/{id}`
  - `POST /api/application-property-components`
  - `PUT /api/application-property-components/{id}`
  - `DELETE /api/application-property-components/{id}`
  - `POST /api/application-property-components/positions`
- Template files:
  - `GET /api/application-property-files?propertyId={id}`
  - `POST /api/application-property-files`
  - `DELETE /api/application-property-files/{id}`
- Chunked upload path used by form editor:
  - `POST /api/storage/chunk_upload`
  - `POST /api/storage/chunks_done`

### 8. Candidates (tab shown for selected company reg numbers)
- `GET /api/settings/CANDIDATES_MODULE`
- `PUT /api/settings` with key `CANDIDATES_MODULE`

### 9. Personal Data Permissions
- `GET /api/admin/role-permission-config`
- `PUT /api/admin/role-permission-config/{role}`
- Request body: `RolePermissions { id, role, permissions[] }`

## Operational Notes
- Most writes are wrapped in confirm dialogs and then full list reloads.
- This area is admin-only; route-level access prevents non-admin usage.
- `SettingsComponent` persists active tab in URL fragment (`#general`, `#vacation-settings`, etc.).
- Candidates tab visibility is hardcoded by company registration number in frontend (`10765896`).

## Key Source Files
- `frontend/src/app/modules/setting/setting-routing.module.ts`
- `frontend/src/app/modules/setting/pages/settings/settings.component.ts`
- `frontend/src/app/modules/setting/services/setting.service.ts`
- `frontend/src/app/modules/setting/pages/*` (tab-specific components)
- `frontend/src/app/services/company.service.ts`
- `frontend/src/app/services/vacation-settings-validation.service.ts`
- `frontend/src/app/services/smtp-config.service.ts`
- `frontend/src/app/services/agreement.service.ts`
