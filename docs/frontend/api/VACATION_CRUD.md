# Frontend Vacation Create / Update / Delete

## Scope
This document describes the Angular frontend flows for submitting, modifying, and cancelling vacations. It covers component paths, API calls, payload shapes, error handling, and the payment-type lifecycle.

---

## Entry Points

| Route | Permission | Main Component |
|---|---|---|
| `/pages/vacations` | `employee` | `VacationStatementsComponent` |
| `/pages/vacation-requests` | `vacations:read` | `VacationRequestsComponent` (manager view) |

---

## Key Source Files

| File | Role |
|---|---|
| `src/app/pages/vacations/vacation-form/vacation-form.component.ts` | Employee vacation form (create + edit) |
| `src/app/pages/vacations/vacation-statements/vacation-statements.component.ts` | Employee list + delete trigger |
| `src/app/pages/vacations/vacation/vacation.component.ts` | Vacation detail/view dialog |
| `src/app/pages/vacation-requests/vacation-form-by-manager/vacation-form-by-manager.component.ts` | Manager-side form |
| `src/app/services/vacations.service.ts` | HTTP calls to `/api/vacations` |

---

## 1. Filing (Create)

### Trigger
`VacationStatementsComponent` opens `VacationFormComponent` as a Material Dialog when the user clicks "Add Vacation".

### Form Fields (`FormGroup`)

| Control | Required | Notes |
|---|---|---|
| `vacationType` | Yes | Loaded from `GET /api/references/company/vacation_type` |
| `startAt` / `endAt` | Yes | Date pickers; change triggers period recalculation |
| `vacationPaymentType` | Yes* | Hidden for unpaid types; defaults to `withSalary` |
| `manager` | Yes | Loaded from `ProfileStructureService.get()` |
| `childId` | Conditional | Required when vacation type has `options.childRequired` |
| `comment` | Optional | Required for `TLPÕP` type |
| `additionalIdentityCode` / `additionalFullname` | Conditional | Required when type has `options.additionalProfile` |
| `files` | Conditional | Required when type has `options.fileRequired` |
| `substitutes` (FormArray) | No | Max 3; each has `profile`, `startAt`, `endAt` |

### Period Recalculation
Every change to `startAt`, `endAt`, `vacationPaymentType`, or `childId` triggers:
- `POST /api/vacation_calculation`
- Request: `VacationPeriodRequest { typeCode, startAt, endAt, vacationPaymentType, childId, id }`
- Response: `VacationCalculation { totalDays, holidays, bookedDays, periods[], error? }`
- If `error` is returned the form is disabled and a toast is shown (see [Error Handling](#error-handling)).

### Save Payload
`POST /api/vacations`

```typescript
VacationRequest {
    type: string;                  // Type key, e.g. 'TLPP'
    startAt: string;               // YYYY-MM-DD
    endAt: string;                 // YYYY-MM-DD
    vacationPaymentType: string;   // 'WITH_SALARY' | 'BEFORE_VACATION' | 'UNPAID' | ...
    managerId: string;             // Manager profile UUID
    comment: string;
    childId?: string;
    profileId?: string;            // Set when manager creates vacation for another employee
    decreeNumber?: string;
    additionalIdentityCode?: string;
    additionalFullname?: string;
    files: string[];               // File UUIDs
    substitutes: { profileId, startAt, endAt }[];
}
```

**Payment type transform** (done client-side before send):
```
'withSalary'     → 'WITH_SALARY'
'beforeVacation' → 'BEFORE_VACATION'
'unpaid'         → 'UNPAID'
```

### After Save
1. Show success toast `message.vacationRequestWasAdded`.
2. `VacationFileService.linkVacationId(vacation.id)` — links any pending files to the new vacation.
3. `vacationService.setReload(null)` — triggers list refresh.
4. Dialog closes.

---

## 2. Modification (Update)

### Two Paths

#### Path A — Employee Edits Own Vacation
1. Confirmation dialog: `"message.doYouWantToChangeVacation"`.
2. `PATCH /api/vacations/{id}/reset` — resets status to `SUBMITTED`, clears workflow decisions.
3. `VacationFormComponent` opens with existing vacation data pre-populated.
4. Save calls `PUT /api/vacations/{id}` with the same `VacationRequest` shape.

#### Path B — Manager Edits Employee Vacation
1. `GET /api/vacations/{id}` — fetches current vacation data.
2. `VacationFormByManagerComponent` opens (separate component).
3. Save calls `PUT /api/vacations/{id}` with `VacationRequest`.

### After Update
1. Show success toast `message.vacationRequestWasUpdated`.
2. `vacationService.setReload(null)` — triggers list refresh.
3. Dialog closes.

---

## 3. Deletion (Cancel)

### Trigger
`VacationStatementsComponent.onDelete(vacation)`:
1. Shows `ConfirmDialogComponent` with `'message.sureDeleteThis'`.
2. On confirm: `vacationService.delete(vacation.id)`.

### API Call
`DELETE /api/vacations/{id}`

No request body. Returns the deleted `Vacation` object on success.

### After Delete
- Success toast: `'message.deletedSuccessfully'`.
- List is refreshed.

### Error Handling on Delete
If the `VACATION_BEFORE_START_RESTRICTION` setting is enabled and the vacation has `BEFORE_VACATION` payment type with a start date within N days, the delete is **blocked client-side** before the confirm dialog is shown. A snackbar with the `vacationBeforeStartRestriction` message (interpolated with `{{ days }}`) is displayed. If the backend returns `400 BAD_REQUEST` for any other reason, the error key is translated via `message.validations.{key}` with details interpolation.

---

## 4. Payment Type Lifecycle

### Available Values
```
withSalary      → WITH_SALARY
beforeVacation  → BEFORE_VACATION
unpaid          → UNPAID
minSalary       → MIN_SALARY
keepSalary      → KEEP_SALARY
averageSalary   → AVERAGE_SALARY
```

### Selection Logic (`togglePaymentType`)
1. If vacation type has `type.options.paymentTypes[]`, only those options are shown.
2. If vacation type is unpaid (`!isPaid`), payment type is forced to `unpaid` and selector is hidden.
3. Otherwise defaults: `['withSalary', 'beforeVacation']`.
4. Changing payment type triggers period recalculation.

### Display Translation Keys
- `vacationPaymentMethod.withSalary`
- `vacationPaymentMethod.beforeVacation`
- `vacationPaymentMethod.unpaid`
- etc.

---

## 5. Error Handling

### Backend Validation Error Shape
```json
{
  "error": {
    "key": "vacationBeforeStartRestriction",
    "details": {}
  }
}
```

### During Period Calculation (POST `/api/vacation_calculation`)
- `calculation.error` is returned in the response body (not HTTP error).
- Form is disabled; error toast shown via:
  ```
  translate.get('message.validations.' + calculation.error.key, calculation.error?.details)
  ```
- Special interpolation: `minimumVacationAmount` error replaces `[number]` in translation.

### During Save (POST/PUT `/api/vacations`)
- HTTP error response caught in `error` callback.
- Toast: `translate.instant('message.validations.' + error?.error.key)`.
- Form remains open for correction.

### Known Error Keys (from backend)
| Key | Cause |
|---|---|
| `vacationCantBeInPast` | Start date is in the past (employee role) |
| `vacationAlreadyExists` | Date range overlaps another vacation |
| `sickLeaveExistsOnThisDates` | Sick leave exists on these dates |
| `substituteInVacation` | Chosen substitute has a confirmed vacation in this range |
| `substituteOnSickLeave` | Chosen substitute is on sick leave |
| `vacationByDateMoreThenContract` | End date exceeds contract end date |
| `vacationBeforeStartRestriction` | BEFORE_VACATION payment type violates the N-day lead-time restriction (VIT-5344) |
| `minimumVacationAmount` | Duration shorter than type minimum |

---

## 6. Settings Consumed by Vacation Forms

Settings are fetched via `SettingService.list()` → `GET /api/settings`.

| Setting Key | Where Used | Effect |
|---|---|---|
| `CHANGE_CONFIRMED_VACATIONS` | `VacationStatementsComponent.setEditableVacations()` | Controls whether Edit button appears on confirmed vacations |
| `VACATION_CALENDAR` | Route guard | Enables/disables `/employee-vacation-calendar` route |
| `HIDE_REJECTED_VACATIONS_IN_EMPLOYEE_CALENDAR` | Calendar view | Hides rejected items for `employee` role |
| `VACATION_BEFORE_START_RESTRICTION` | `VacationFormComponent`, `VacationStatementsComponent` | Loaded on component init via `GET /api/settings/VACATION_BEFORE_START_RESTRICTION`. When enabled: (1) form triggers `calculatePeriods()` on load in edit mode so the restriction error is shown immediately if applicable; (2) delete pre-checked client-side — blocks action and shows error if `beforeVacation` payment type and start < N days. |

---

## 7. Reactive State Pattern

`VacationsService` uses a Subject to coordinate list reloads after mutations:
```typescript
private isReload = new Subject();
reload(): Observable<any> { return this.isReload.asObservable(); }
setReload(value: any) { this.isReload.next(value); }
```
All write operations call `setReload(null)` on success. `VacationStatementsComponent` subscribes and calls `loadData()`.

---

## 8. API Call Summary

| Operation | Method | Endpoint | Payload |
|---|---|---|---|
| List own vacations | GET | `/api/profile/vacations` | — |
| Get single vacation | GET | `/api/vacations/{id}` | — |
| Create | POST | `/api/vacations` | `VacationRequest` |
| Update | PUT | `/api/vacations/{id}` | `VacationRequest` |
| Delete | DELETE | `/api/vacations/{id}` | — |
| Calculate periods | POST | `/api/vacation_calculation` | `VacationPeriodRequest` |
| Reset to SUBMITTED | PATCH | `/api/vacations/{id}/reset` | — |
| Confirm/Reject | PATCH | `/api/vacations/{id}/confirm` | `VacationConfirmation` |
| Withdraw confirmation | POST | `/api/vacations/{id}/withdraw` | `VacationDraftComment` |
| Interrupt | POST | `/api/vacation_interruptions` | `VacationInterruptionRequest` |
| Delete file link | DELETE | `/api/vacations/{id}/files/{fileUuid}` | — |
