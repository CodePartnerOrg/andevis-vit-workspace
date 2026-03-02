# Profile Module (`ee.andevis.api.profile`)

## Purpose

This module manages employee profile data and profile-adjacent domains:
- base profile identity/work snapshot and photo,
- profile search/filter for UI pickers and management views,
- profile settings,
- profile addresses/contacts/bookmarks,
- profile personal records (children, education, documents, bank accounts),
- profile health/sick leave and non-taxable minimum read APIs,
- profile manager/deputy and structure context helpers.

The module is also a major integration point with:
- `issue` + `audit` flow for confirmation-required changes,
- producer events for confirmed profile data sync,
- periodic profile activation/deactivation recalculation.

## Package Map

Base path: `api/src/main/java/ee/andevis/api/profile/`

- `api/src/main/java/ee/andevis/api/profile/controller/`: REST endpoints for profile and related subdomains.
- `api/src/main/java/ee/andevis/api/profile/service/`: business logic per subdomain.
- `api/src/main/java/ee/andevis/api/profile/entity/`: JPA models (`Profile`, `ProfileAddress`, `ProfileDocument`, `ProfileChild`, etc.).
- `api/src/main/java/ee/andevis/api/profile/repository/`: Spring Data repositories.
- `api/src/main/java/ee/andevis/api/profile/spec/`: dynamic filtering specifications (`ProfileSpec`, etc.).
- `api/src/main/java/ee/andevis/api/profile/payload/`: request/response DTOs and criteria objects.
- `api/src/main/java/ee/andevis/api/profile/task/`: scheduled profile maintenance task.

## Core Domain Model

### `Profile`

Main employee snapshot entity with:
- identity and name,
- active company/structure/position and active contract pointers,
- `isActive`, photo key,
- JSONB `settings` map.

Important behavior:
- `getFullName()` derives from first/last name.
- `getLanguage()` defaults to `"et"` when not set.

### Profile sub-record domains

Main sub-record aggregates under `profile/*`:
- addresses (`ProfileAddress`),
- documents (`ProfileDocument`),
- education (`ProfileEducation`),
- children (`ProfileChild`),
- bank accounts (`ProfileBankAccount`),
- deputies (`ProfileDeputy`),
- health/sick leave (`ProfileHealthInspection`, `ProfileSickLeave`),
- non-taxable minimum (`ProfileNonTaxableMinimum`).

## Main Endpoints

### Base profiles (`ProfilesController`)

- `GET /profiles` (`profiles:read` or `profiles:read:scoped`)
  - paginated filter with full or scoped access behavior.
- `GET /profiles/search` (`isAuthenticated`)
  - search endpoint for user pickers with manager/assistant/admin-specific scoping.
- `GET /profiles/by_user/{id}` (`profiles:read*`)
  - profile by user ID.
- `GET /profiles/{id}` (`profiles:read*`)
  - detailed mapped `ProfileResponse`.
- `GET /profiles/profile` (`isAuthenticated`)
  - current profile; for `guest` role returns candidate-like projection.
- `PUT /profiles/photo` / `DELETE /profiles/photo` (`isAuthenticated`)
  - updates photo and triggers producer event.

### Settings

- `PUT /profile_settings/{key}` (`isAuthenticated`)
  - update a single profile setting value.

### Contacts and bookmarks

- `GET /profile_contacts?search=...` (`isAuthenticated`)
  - contact search (gated by company setting `PROFILE_BOOKMARKS`).
- `GET/POST/DELETE /profile_contact_bookmarks...` (`isAuthenticated`)
  - manage bookmarked contacts.

### Address

- `GET /profile-addresses` (`isAuthenticated`)
- `PUT /profile-addresses/{id}` (`isAuthenticated`)

### Profile records requiring dedicated permissions

- Children: `/profile_children*` (`profile:children:read` + auth)
- Documents: `/profile_documents*` (`profile:documents:read` + auth)
- Educations: `/profile_educations*` (`profile:educations:read` + auth)
- Bank accounts: `/profile_bank_accounts*` (`profile:payment_accounts:read` + auth)
- Sick leaves: `/profile_sick_leaves*` (`profile:medical_records:read` + auth)
- Health inspections: `/health_inspection*` (`profile:health_check:read` + auth)

### Other profile-context APIs

- `GET /deputy` (`isAuthenticated`): current user deputy assignments.
- `GET /profile_structures` and `GET /profile_structures/{id}`: structure/direct manager context.
- `GET /profile_contracts`: current user contracts in selected company.
- `GET /taxes` and `GET /taxes/actual_profile` (`isAuthenticated`): non-taxable minimum reads.

## Key Business Flows

### 1) Profile details mapping

`ProfileService.getProfile(user, companyId)` builds `ProfileResponse` by combining:
- profile base fields,
- active company context,
- work address (email/phone),
- active contract + career snapshot,
- structure-position graph metadata,
- disability history list,
- field-level filtering via `SecurityUtils.filterFieldsByPermissions(...)`.

### 2) Profile list/search and access scoping

- `ProfileService.filter(...)` uses `ProfileSpec` and optional manager/assistant scoping via `AccessManagerService`.
- `ProfileService.search(...)` supports name/identity/role filtering and optional exclusion list.
- Controllers decide scoped/full mode based on caller permissions.

### 3) Photo updates

- `ProfileService.updatePhoto(...)` stores file UUID on profile and publishes profile photo update event.
- `deletePhoto(...)` clears via user update path and publishes null-photo update.

### 4) Confirmation-based profile data changes

Subdomains `ProfileAddressService`, `ProfileChildService`, `ProfileDocumentService`, `ProfileEducationService`, `ProfileBankAccountService` share this pattern:
- check company-level setting `PERSONAL_ISSUES` reference,
- if enabled:
  - mark record `PENDING`,
  - create/update `Audit`,
  - create `Issue` for approval,
  - (bank account path also sends notification email to configured issues mailbox),
- if disabled:
  - apply directly as `CONFIRMED`,
  - send producer sync message (`create`/`update`/`delete`).

These services also preserve previous-version snapshots for UI comparison when records were pending/rejected.

### 5) Deputy and manager views

- `ProfileDeputyService.index()` returns current user deputy assignments filtered to active date range and only confirmed vacations.
- `ProfileStructureService.getProfileStructureResponses(...)` resolves direct managers and their work contacts for a profile/company.

### 6) Scheduled profile normalization

`ProfilesTask` runs hourly (`0 0 * ? * *`) and calls:
- `ProfileService.updateProfiles()` to recalculate active company/structure/position from active contracts/careers,
- `UserService.deactivateUsers()` after profile recalculation.

## Filtering/Spec Layer

`ProfileSpec` drives dynamic filtering:
- company, active flags, user active flags,
- active contract existence (subquery),
- name/identity/structure/position text search,
- structure and profile include/exclude lists,
- user-role profile filtering (`hasUserIdIn`).

This spec layer is heavily reused by profile list/search and dependent modules.

## Integration and Side Effects

- `ProducerService`: used by address/document/education/child/bank/photo flows.
- `AuditService` + `IssueService`: used when personal-data confirmation workflow is enabled.
- `EmailOutboxService`: bank-account issue notifications to configured personal issues email.
- `StorageService`: base64 photo/file resolution in response DTOs.

## Where to Change Behavior

- Base profile read/filter/search behavior: `ProfileService`, `ProfilesController`, `ProfileSpec`.
- Profile settings: `ProfileSettingsService` + `ProfileSettingsController`.
- Contact directory/bookmarks: `ProfileContactService`, `ProfileContactBookmarkService`.
- Confirmation-governed personal data updates: services for address/child/document/education/bank.
- Manager/deputy context: `ProfileStructureService`, `ProfileDeputyService`.
- Scheduled lifecycle sync: `ProfilesTask` and `ProfileService.updateProfiles()`.

## Notable Caveats

- `ProfileService.getSort(...)`, `ProfileSickLeaveService.getSort(...)`, and `ProfileHealthInspectionService.getSort(...)` split `sort` by `","` and access `str[1]`; empty/invalid sort can throw at runtime.
- `ProfileService.search(...)` uses `criteria.getFirstName().equals(criteria.getLastName())` without null guard; null inputs can fail.
- `ProfileContactBookmarkService.get(...)` returns `null` when bookmarks setting is disabled (not empty list).
- `ProfileContractController` and `ProfileStructureController` do not declare `@PreAuthorize`; access relies on global security config.
- In bank account producer payload, `percentage` is hardcoded to `1` in `produce(...)` instead of using stored percentage.
