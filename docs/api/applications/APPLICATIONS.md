# Applications Module (`ee.andevis.api.application`)

## Purpose

The module implements configurable internal applications (requests/forms) with:
- dynamic form templates (`application_properties` + components),
- multi-step approval workflows by role,
- optional file attachments,
- status transitions (`waiting`, `accepted`, `rejected`),
- notification and integration side effects.

## Package Structure

Base path: `api/src/main/java/ee/andevis/api/application/`

- `api/src/main/java/ee/andevis/api/application/controller/`: REST endpoints for applications, templates, workflow config, and files.
- `api/src/main/java/ee/andevis/api/application/service/`: business logic for creation, filtering, decision flow, notifications, and producer integration.
- `api/src/main/java/ee/andevis/api/application/entity/`: JPA models for applications, workflows, templates, and files.
- `api/src/main/java/ee/andevis/api/application/repository/`: Spring Data repositories and workflow lookup helpers.
- `api/src/main/java/ee/andevis/api/application/payload/`: DTOs for requests, responses, sorting, filtering, and decisions.
- `api/src/main/java/ee/andevis/api/application/spec/`: JPA Specifications used for dynamic filtering.

## Main Functional Flows

### 1) Create application

`POST /applications`
- Resolves `companyId` and requester profile from JWT.
- Supports creating for self or another employee (`employeeId` in payload).
- Copies profile snapshot fields into the application (`name`, `identityCode`, `structure`, `position`).
- Sets status to `WAITING`, stores content/context, creates workflow rows from template workflow.
- Links uploaded files (`files` array of UUIDs).
- Sends submission notifications.

### 2) List/search applications

`GET /applications`
- Supports filtering by:
  - person name,
  - type code,
  - structure,
  - status,
  - profile,
  - date range (`startAt`, `endAt`),
  - paging/sort (`page`, `size`, `sort`).
- Role-sensitive visibility:
  - `assistant` and `manager`: scoped by `AccessManagerService` available profiles.
  - `admin`, `personal`: broad access.
  - others: filtered by matching workflow role.
- Special mode: `isConfirmationExpected=true` returns waiting items where current role can act.

### 3) Approval decision

`PUT /applications/{id}/decide`
- Permission: `applications:confirm`.
- Writes decision (`isAccepted`, `comment`, `submittedAt`) into workflow row.
- Supports personal role behavior via setting `PERSONAL_CAN_CONFIRM_APPLICATIONS`.
- Final status calculation:
  - any rejected workflow -> application `REJECTED`,
  - all workflows accepted -> application `ACCEPTED`,
  - otherwise remains `WAITING`.
- Side effects:
  - mail notifications on each state update,
  - on final accept, producer integration (`ApplicationProducerService.produce(...)`).

### 4) Download PDF

`POST /applications/{id}/download`
- If archived PDF exists (`archivedAt` set), returns stored bytes.
- Otherwise renders PDF from application HTML content with embedded fonts.
- Output file name format:
  `<title>_<firstName>-<lastName>_<createdDate>.pdf`

### 5) Soft delete

`DELETE /applications/{id}`
- Sets `deletedAt` timestamp (soft delete).
- `Application` entity uses `@SQLRestriction("deleted_at IS NULL")`.

## Template Configuration Capabilities

### Application properties (`/application-properties`)

Admin-configurable template metadata:
- `code`, `title`, `note`, active flag,
- email notification config (`isNotify`, `email`),
- document sections (`header`, `body`, `footer`),
- file policy (`isFiles`, `minFiles`, `maxFiles`, allowed extensions),
- lock protection (`isLocked`) for protected templates.

Deletion protection:
- locked templates cannot be deleted,
- templates in use by existing applications cannot be deleted.

### Dynamic form components (`/application-property-components`)

Per-template form schema entries:
- component type/tag/label/placeholder,
- JSON `options` and JSON `rules`,
- sortable position,
- lock protection on update/delete.

### Workflow templates (`/application-property-workflows`)

Defines approval chain per template:
- ordered by `position`,
- role-based stages (`manager`, `personal`, `accountant`, etc.),
- per-stage notification flags and destination emails.

On application creation, module materializes these into `application_workflows`.

### Template files (`/application-property-files`)

Associates static/reference files with an application template using file UUID links.

## Runtime Notifications and Integrations

### Email notifications (`ApplicationNotifierService`)

- `WAITING`: notify next pending workflow stage.
  - manager stage: sends to manager profile email.
  - non-manager stage: sends to configured workflow email.
- `REJECTED`: sends rejection email to applicant with rejection comment.
- `ACCEPTED`: sends acceptance to applicant; optionally also to template-level email if template `isNotify=true`.

### Producer integration (`ApplicationProducerService`)

Runs on accepted applications and currently handles specific type codes:
- `ITEA`: sends non-taxable minimum event (`profile_non_taxable_minimums`) with calculated amount/date/contract external ID.
- `KLS`: sends `applications` event with training context and base64-encoded attachments.

Payloads are hashed (SHA-256) before producing via shared `ProducerService`.

## Data Model Snapshot

- `Application`: main request record, status and profile snapshot, content/context, archived file ID.
- `ApplicationWorkflow`: per-application approval entries, decisions and comments.
- `ApplicationProperty`: template definition per company.
- `ApplicationPropertyComponent`: dynamic form component definitions (JSON options/rules).
- `ApplicationPropertyWorkflow`: template workflow chain.
- `ApplicationFile`: uploaded file links for a concrete application.
- `ApplicationPropertyFile`: file links attached to a template.

## Key Endpoints and Authorization

- `GET /applications`: `applications:read` or `applications:read:scoped`
- `POST /applications`: `applications:create` or scoped/own variants
- `PUT /applications/{id}/decide`: `applications:confirm`
- `DELETE /applications/{id}`: `applications:delete`
- `GET /applications/{id}`, `POST /applications/{id}/download`, `GET /applications/types`: authenticated
- `GET /profile/applications`: authenticated
- Template management endpoints (`/application-properties*`, `/application-property-*`): mostly `admin`

`/application_files` endpoints do not declare `@PreAuthorize` in this controller and rely on global security configuration.

## Notable Implementation Caveats

- `ApplicationService.get(...)` contains `// TODO validate company`; current read path does not explicitly enforce company ownership.
- `ApplicationService.delete(...)` performs soft delete but returns `null` instead of deleted ID.
- `ApplicationDecisionPayload` field is named `externalDesition` (typo in code).
- `ApplicationProfileController` returns `null` if JWT lacks `companyId` or `profileId` (not an empty list / explicit error).
