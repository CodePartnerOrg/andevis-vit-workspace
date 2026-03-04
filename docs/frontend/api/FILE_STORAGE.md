# File Storage and Document Management — Frontend

The frontend implements chunked file upload with deferred entity linking, blob-based download, and thumbnail display. File concerns are split between a shared `FileService` and domain-specific services.

---

## Architecture Overview

```
User (drag-and-drop or file picker)
  ↓ ngx-dropzone-wrapper (500 KB chunks)
StorageController  POST /storage/chunk_upload  (one chunk per request)
                   POST /storage/chunks_done   (finalize & persist)
  ↓ file UUID / id returned
Domain service  (VacationFileService, ApplicationFileService, …)
  ↓ POST /vacation_files  { vacationId, fileId }   — deferred link
Domain entity updated in backend

Download:
  GET /storage/download/{hash}  →  Blob  →  URL.createObjectURL  →  <a download>
```

---

## Services

### `FileService` — `src/app/services/file.service.ts`

Thin wrapper over the raw storage endpoints.

| Method | Endpoint | Description |
|--------|----------|-------------|
| `uploadFile(file, access)` | `POST storage/upload` | Single-file upload via FormData |
| `download(hash)` | `GET storage/download/{hash}` | Blob response (public) |
| `downloadPrivate(hash)` | `GET storage/download/{hash}` | Blob response (private/auth-required) |
| `delete(uuid)` | `DELETE storage/delete/{uuid}` | Remove file |
| `getThumb(h, w, hash)` | `GET storage/thumbnail/{h}/{w}/{hash}` | Base64 thumbnail |

### `ProfileDocumentsService` — `src/app/services/profile-documents.service.ts`

Employee identity documents (passport, driving licence, certificates).

- Endpoint: `profile_documents`
- CRUD: `list()`, `get(id)`, `create()`, `update()`, `delete()`

### `VacationFileService` — `src/app/services/vacation-file.service.ts`

Vacation request attachments with **deferred linking**.

- Endpoint: `vacation_files`, `vacation_files/setVacationId/{vacationId}`
- `filesToLinkWithVacationId` — local queue; files uploaded before the vacation is saved
- `setVacationId(id)` — flushes the queue by calling `setVacationId` on the backend
- `hasFiles` EventEmitter — notifies parent components of pending uploads
- CRUD: `create()`, `read()`, `list(vacationId)`, `delete()`, `deleteIds()`

### `ApplicationFileService` — `src/app/services/application-file.service.ts`

HR application form attachments with **deferred linking**.

- Endpoint: `application_files`, `application_files/setApplicationId/{applicationId}`
- `filesToLinkWithApplicationId` — local queue for pre-save uploads
- `getFilesCount()` — used by the `requiredFiles` async validator
- CRUD: `create()`, `read()`, `list(applicationId, componentId)`, `delete()`, `deleteIds()`

### `ApplicationPropertyFileService` — `src/app/modules/setting/services/application-property-file.service.ts`

Template/example files on HR form property definitions (admin-only).

- Endpoint: `application-property-files`
- Operations: `get(propertyId)`, `create()`, `delete()`

---

## Upload Implementation (ngx-dropzone-wrapper)

All upload components share the same chunked upload configuration:

```typescript
// Typical Dropzone config
{
  url: `${API_URL}/storage/chunk_upload`,
  chunking: true,
  chunkSize: 500_000,           // 500 KB
  maxFilesize: 100,             // 100 MB
  parallelUploads: 20,          // varies: 20–100
  parallelChunkUploads: true,   // varies by component
  acceptedFiles: '.jpeg,.jpg,.png,.heic,.pdf,.odt,.doc,.docx,.xlsx,.xls,.bdoc,.asice',
  headers: { Authorization: `Bearer ${localStorage.getItem('token')}` },
  withCredentials: true,
}
```

After all chunks are sent, `chunksUploaded` fires and the component posts to `chunks_done`:

```typescript
const data = new FormData();
data.append('dzuuid', file.upload.uuid);
data.append('filename', file.name);
data.append('dztotalchunkcount', String(file.upload.totalChunkCount));
data.append('dztotalfilesize', String(file.size));
data.append('access', 'private');   // 'public' for logos/settings
this.http.post(`${API_URL}/storage/chunks_done`, data).subscribe(…);
```

---

## Download Implementation

```typescript
this.fileService.downloadPrivate(file.uuid).subscribe(blob => {
    const url = URL.createObjectURL(new Blob([blob], { type: file.fileType }));
    const a = document.createElement('a');
    a.href = url;
    a.download = file.fileName;
    a.click();
    URL.revokeObjectURL(url);
});
```

---

## File Components by Feature Module

| Component path | Max files | Parallel uploads | Used in |
|---------------|-----------|-----------------|---------|
| `pages/vacations/vacation-form/vacation-files/` | 10 | 20 | Vacation request form |
| `pages/planned-vacations/planned-vacation-form/planned-vacation-files/` | 10 | 20 | Planned vacation form |
| `pages/profile/profile-documents/profile-document-form/` | 1 | — | Employee identity documents |
| `pages/profile/profile-educations/profile-education-form/` | 1 | — | Education certificates |
| `pages/applications/application/` | 10 | 100 | HR form submissions |
| `pages/issues/issue/` | 1 | — | HR issue/correction requests |
| `modules/candidate/pages/manage/candidates/candidate-documents/` | 10 | 100 | Recruitment — CVs, contracts |
| `modules/survey/pages/manage-surveys/manage-survey/` | 3 | 100 | Survey response attachments |
| `modules/setting/pages/application-settings/app-property-form/` | 10 | 100 | Form property template files |
| `modules/setting/pages/logo-settings/` | 1 | — | Company logo (access: public) |

---

## Models / Interfaces

### `File` — `src/app/models/file.ts`

Core file record returned by the backend.

```typescript
{
  id: number;
  uuid: string;         // used in DELETE and thumbnail URLs
  hash: string;         // used in GET / download URLs
  fileName: string;
  fileType: string;     // MIME type
  fileSize: number;     // bytes
  path: string;
  access: string;       // 'public' | 'private'
  key: string;
}
```

### `FileBase64` — `src/app/models/file-base64.ts`

Thumbnail response.

```typescript
{ id: string; data: string; }  // data = base64 data URI
```

### `VacationFile` — `src/app/models/vacation-file.ts`

```typescript
{ id: string; vacationId: string | null; fileId: string; key: string; file: File; }
```

### `ApplicationFile` — `src/app/models/application-file.ts`

```typescript
{ id: string; applicationId: string | null; componentId: string; fileId: string; key: string; file: File; }
```

### `Document` — `src/app/models/document.ts`

Profile document metadata (no binary content).

```typescript
{ id: string; type: string; number: string; dateOfIssue: string; validityDate: string; issuer: string; fileId: string; }
```

### `ProfileDocumentResponse`

Extends `Document` with display fields returned from the list endpoint:

```typescript
{ …Document; typeKey: string; fileType: string; fileSize: number; fileName: string; key: string; status: string; comment: string; }
```

### `ApplicationPropertyFile` — `src/app/modules/setting/models/application-property-file.ts`

```typescript
{ id: string; propertyId: string; fileId: string; name: string; size: number; }
```

---

## Utilities

### `FileSizePipe` — `src/app/modules/pipes/file-size.pipe.ts`

Converts raw byte count to human-readable string (B / KB / MB / GB / TB / PB). Default precision: 0 decimals for bytes, 1 for KB/MB, 2 for GB+.

### `requiredFiles` validator — `src/app/validators/required-files.ts`

Async form validator. Combines already-confirmed files from the backend with pending (deferred) local uploads to enforce a minimum file count on form submission.

---

## Deferred Linking Pattern

Some entities (vacation, application) may not exist yet when the user uploads files. The pattern:

1. User uploads file → backend returns `{ fileId, uuid }` → service appends to `filesToLinkWith*Id[]`
2. Parent form saves the entity → entity ID received
3. Service calls `setVacationId(id)` or `setApplicationId(id)` → backend links all queued `fileId`s to the entity
4. `hasFiles` EventEmitter signals pending uploads to parent components so they can conditionally block submission or show a warning

---

## API Endpoints Summary

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `storage/upload` | Single file (logos, small items) |
| `POST` | `storage/chunk_upload` | One 500 KB chunk |
| `POST` | `storage/chunks_done` | Finalize chunked upload |
| `GET` | `storage/download/{hash}` | Download blob |
| `GET` | `storage/thumbnail/{h}/{w}/{hash}` | Base64 thumbnail |
| `DELETE` | `storage/delete/{uuid}` | Remove file |
| `GET/POST/PUT/DELETE` | `profile_documents[/{id}]` | Profile document CRUD |
| `GET/POST/DELETE` | `vacation_files[/{id}]` | Vacation file CRUD |
| `POST` | `vacation_files/setVacationId/{vacationId}` | Deferred link |
| `GET/POST/DELETE` | `application_files[/{id}]` | Application file CRUD |
| `GET` | `application_files/getFilesCount` | Count for validator |
| `POST` | `application_files/setApplicationId/{applicationId}` | Deferred link |
| `GET/POST/DELETE` | `application-property-files` | Template file CRUD |
