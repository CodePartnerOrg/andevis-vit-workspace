# File Storage and Document Management — API

Backend file storage is a **local-filesystem-based** system with PostgreSQL tracking. Files are uploaded, validated, chunked, hashed, and associated with domain entities through join tables.

---

## Architecture Overview

```
Client
  ↓ multipart/form-data (single or chunked)
StorageController  (/storage/*)
  ↓
StorageService
  ├── MIME validation (whitelist)
  ├── SHA-256 hashing
  ├── EXIF extraction (ExifService)
  ├── Thumbnail generation (Thumbnailator)
  └── Filesystem write  →  ./data/storage/uploads/YYYY/M/D/{uuid}.ext
              ↓
        files (PostgreSQL table)
              ↓
  Domain join tables (vacation_files, profile_documents, …)
```

**Package:** `ee.andevis.api.storage`

**Key classes:**

| Class | Role |
|---|---|
| `StorageController` | REST entry point — auth-guarded (`isAuthenticated()`) |
| `StorageService` | Core upload/download/delete/thumbnail logic |
| `ExifService` | GPS, creation date, camera metadata extraction from images |
| `StorageProperties` | Binds `storage.path` from `application.yml` |

---

## REST Endpoints

Base path: `/storage`

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/upload` | Single-file upload. Params: `file` (MultipartFile), `access` (string) |
| `POST` | `/chunk_upload` | One chunk of a chunked upload (Dropzone.js format) |
| `POST` | `/chunks_done` | Merge chunks and persist final file |
| `GET` | `/{id}` | Stream file inline (image or document) |
| `GET` | `/download/{id}` | Download file as `Content-Disposition: attachment` |
| `GET` | `/thumbnail/{width}/{height}/{id}` | Return resized image as base64 data URI |
| `DELETE` | `/delete/{uuid}` | Soft-delete record + physical removal from filesystem |

### Chunked Upload Parameters (Dropzone.js)

| Parameter | Meaning |
|-----------|---------|
| `dzuuid` | Upload session UUID |
| `dzchunkindex` | Zero-based chunk index |
| `dztotalchunkcount` | Total number of chunks |
| `dztotalfilesize` | Total file size in bytes |
| `dzchunkbyteoffset` | Byte offset of this chunk |
| `dzchunksize` | Size of this chunk |
| `filename` | Original filename |

Chunks are stored as `{uploadDir}/{uuid}/{uuid}_{partIndex:05d}` and merged on `chunks_done`.

---

## Configuration

**`application.yml`**

```yaml
spring:
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 10MB

storage:
  path: "./data/storage"
```

**Filesystem layout:**

```
./data/storage/
  uploads/
    2026/3/4/
      550e8400-…-446655440000.pdf
  cache/
    thumb_200x200.jpg   # cached thumbnails
```

---

## Database Schema

### `files` table — central file record

```sql
CREATE TABLE public.files (
    id          bigserial PRIMARY KEY,
    uuid        varchar(255),   -- public identifier (used in URLs)
    file_name   varchar(255),   -- original filename
    file_type   varchar(255),   -- MIME type
    file_size   int8,           -- bytes
    path        varchar(255),   -- relative path under storage root (YYYY/M/D/)
    hash        varchar(255),   -- SHA-256 of file content
    access      varchar(255),   -- 'public' | 'private'
    deleted_at  timestamp       -- soft-delete timestamp (file also removed from disk)
);
```

### Domain join tables

| Table | Entity linked | Notable columns |
|-------|--------------|-----------------|
| `profile_documents` | Employee document | `profile_id`, `file_id`, type/number/dates, status, external sync columns |
| `vacation_files` | Vacation request | `vacation_id`, `file_id` |
| `scheduled_vacation_files` | Scheduled vacation | `scheduled_vacation_id`, `file_id` |
| `application_files` | HR form submission | `application_id`, `component_id`, `file_id` |
| `application_property_files` | Form property template | `application_property_id`, `file_id` |
| `application_creator_files` | Initiator attachment | `application_id`, `file_id` |
| `candidate_files` | Recruitment candidate | `candidate_id`, `file_id`, `token` (Dokobit signing token) |
| `survey_answer_files` | Survey response | `survey_answer_id`, `file_id` |

---

## MIME Type Whitelist

Uploads that do not match the whitelist are rejected and physically removed.

**Images:** `image/gif`, `image/jpeg`, `image/pjpeg`, `image/png`, `image/svg+xml`, `image/tiff`, `image/vnd.microsoft.icon`, `image/vnd.wap.wbmp`, `image/webp`, `image/heic`

**Documents:** `application/pdf`, `application/msword` (DOC), `application/vnd.ms-excel` (XLS), `application/zip`

**Office Open XML:** `application/x-tika-ooxml`, `application/vnd.openxmlformats-officedocument.wordprocessingml.document` (DOCX), `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` (XLSX)

**Other:** `application/vnd.oasis.opendocument.text` (ODT), `application/vnd.etsi.asic-e+zip` (BDOC/ASiC-E signed containers)

---

## Domain Services

Each feature module owns a service that wraps the join table and calls `StorageService` for raw file ops.

| Module | Service | Responsibilities |
|--------|---------|-----------------|
| `storage` | `StorageService` | Core upload/download/delete/thumbnail/hash |
| `profile` | `ProfileDocumentService` | Employee document CRUD; confirmation-required workflow; Kafka publish via `ProfileDocumentMessage` |
| `vacation` | `VacationFileRepository` | Link/unlink files to vacation requests |
| `scheduledvacation` | `ScheduledVacationFileRepository` | Link/unlink files to scheduled vacations |
| `application` | `ApplicationFileService` | Form attachment management; deferred linking (file before entity) |
| `application` | `ApplicationPropertyFileService` | Template/example files on form properties |
| `candidate` | `CandidateFileService` | CV/contract management; Dokobit signing integration |
| `survey` | `SurveyAnswerFileService` | Survey response attachments |
| `generator` | `FileManagerService` | XLSX report generation for vacation exports |

---

## Special Features

### Thumbnail generation
- Library: **Thumbnailator**
- Cached in `{storage.path}/cache/thumb_{w}x{h}.{ext}`
- Not supported for WebP and SVG (returned as-is or error)
- Endpoint: `GET /storage/thumbnail/{width}/{height}/{id}`

### EXIF extraction
- `ExifService` reads GPS lat/lon, creation date, camera make/model, orientation
- Metadata stored in `FileMetaInfo` and persisted alongside the file record

### Digital signatures (Candidate module)
- `CandidateFileService` stores a `token` per file referencing a **Dokobit** signing session
- Files can be sent for digital signing and retrieved as signed containers (ASiC-E)

### Kafka integration
- `ProfileDocumentService` publishes `ProfileDocumentMessage` events to Kafka after document changes so the Connector can sync to external HR/payroll systems

---

## Security

| Concern | Implementation |
|---------|---------------|
| Authentication | `@PreAuthorize("isAuthenticated()")` on all endpoints |
| File type | MIME whitelist enforced; rejected files deleted from disk |
| Integrity | SHA-256 hash stored per file |
| Identifiers | Files referenced by UUID (non-guessable), not sequential ID |
| Soft delete | `deleted_at` timestamp set; file physically removed from filesystem |
| Access levels | `access` column (`public`/`private`) stored; granular enforcement is per-domain service |
