# Candidate Signing Module — Dokobit Integration

## Overview

The candidate signing module handles the full lifecycle of employment contract signing via the **Dokobit Gateway** (digital signature service). After a candidate completes their profile and uploads required documents, HR initiates a multi-party signing workflow in Dokobit. Once all parties have signed, the final `.asice` container is stored locally and the signed contract is forwarded to the external HR/payroll system via Kafka.

---

## Key Classes

| Class | Package | Responsibility |
|---|---|---|
| `DokobitService` | `ee.andevis.api.auth.service` | All Dokobit Gateway API calls (file upload/delete, container create/store) |
| `CandidateContractSignerService` | `ee.andevis.api.candidate.service` | Signing workflow — tracks per-signer status, finalises contract when all have signed |
| `CandidateFileService` | `ee.andevis.api.candidate.service` | Manages files attached to a candidate; uploads to Dokobit and deletes on removal |
| `CandidateService` | `ee.andevis.api.candidate.service` | Candidate lifecycle; triggers container creation on status transition |

---

## Data Model

### `candidates` table (relevant signing fields)

| Column | Type | Description |
|---|---|---|
| `container_token` | `varchar` | Dokobit Gateway token for the signing container (set at container creation) |
| `container_name` | `varchar` | Display name for the container (set by HR at signing initiation) |
| `container_id` | `uuid` | UUID of the locally stored signed `.asice` file (set after all parties sign) |
| `status` | `enum` | Candidate lifecycle status (see [Status Flow](#status-flow)) |

### `candidate_contract_signers` table

| Column | Type | Description |
|---|---|---|
| `candidate_id` | `uuid` | Parent candidate |
| `role` | `varchar` | `"guest"` (candidate) or `"personal"` (HR/company representative) |
| `first_name`, `last_name`, `identity_code` | `varchar` | Signer identity (sent to Dokobit) |
| `token` | `varchar` | Dokobit per-signer signing token (returned by container creation) |
| `is_signed` | `boolean` | Whether this signer has completed signing |
| `signed_at` | `timestamp` | Timestamp of signing |

### `candidate_files` table

| Column | Type | Description |
|---|---|---|
| `candidate_id` | `uuid` | Parent candidate |
| `token` | `varchar` | Dokobit file token (returned on upload to Gateway) |
| `file_id` | `uuid` | Reference to local `file_documents` record |

---

## Status Flow

```
AWAITING_CONTRACT
      │
      │  HR adds signers + files, initiates signing
      ▼
CANDIDATE_SIGNING  ──── DokobitService.createCandidateContainer()
      │                 container_token set on Candidate
      │  candidate (guest role) signs
      ▼
HR_SIGNING
      │  HR representative (personal role) signs
      │  all signers signed → DokobitService.storeContainerFile()
      ▼
SIGNED             ──── container_id set (local file UUID)
                        Kafka message sent (candidate_contracts)
```

Status transitions are handled in `CandidateService.updateStatus()` (container creation) and `CandidateContractSignerService.update()` (finalisation).

---

## Dokobit Gateway API Calls

All calls go to `appProperty.auth.dokobit.gatewayUrl` with `?access_token=<gatewayToken>`.

### Upload file — `DokobitService.createFile(UUID fileId)`

Uploads a local file to the Dokobit Gateway before container creation.

```
POST /api/file/upload.json?access_token=…
Content-Type: multipart/form-data

file[name]=<fileName>
file[digest]=<sha256>
file[content]=<base64>
```

Returns `DokobitFileResponse` with a `token` stored in `candidate_files.token`.

### Delete file — `DokobitService.deleteFile(String token)`

Removes an uploaded file from the Gateway. Called by `CandidateFileService.delete()` when a candidate file is removed before signing.

```
POST /api/file/{token}/delete.json?access_token=…
```

Returns `DokobitFileResponse`; `status == "ok"` indicates success.

### Create signing container — `DokobitService.createCandidateContainer(UUID candidateId)`

Creates an ASiC-E signing container with all uploaded files and all registered signers.

```
POST /api/signing/create.json?access_token=…
Content-Type: multipart/form-data

type=asice
name=<containerName>
signers[0][id]=<signerId>
signers[0][name]=<firstName>
signers[0][surname]=<lastName>
signers[0][code]=<identityCode>
files[0][token]=<fileToken>
…
```

On success (`status == "ok"`):
- `candidate.containerToken` is set to the returned container token.
- Each signer's Dokobit signing token is saved to `candidate_contract_signers.token`.

### Retrieve signed container — `DokobitService.storeContainerFile(String token, String fileName)`

Polls container status and downloads the completed `.asice` file once all parties have signed.

```
GET /api/signing/{token}/status.json?access_token=…
```

When `status == "completed"`, the file URL is downloaded and stored locally via `FileService.store()`. Returns the UUID of the new `FileDocument`, which is saved to `candidate.containerId`.

---

## Signing Workflow (Step by Step)

1. **Status transition** → `CANDIDATE_SIGNING`:
   `CandidateService.updateStatus()` calls `DokobitService.createCandidateContainer()`. The candidate's `containerToken` is persisted.

2. **Candidate signs** (`role = "guest"`):
   `CandidateContractSignerService.update(signerId)` marks the signer as signed. Status advances to `HR_SIGNING`; HR is notified.

3. **HR representative signs** (`role = "personal"`):
   Same `update()` call. When `totalSigned == totalSigners`:
   - `DokobitService.storeContainerFile()` downloads and stores the final `.asice`.
   - `candidate.containerId` is set to the local file UUID.
   - `candidate.status` → `SIGNED`.
   - Notification email is sent (`CandidateNotificationService.sendSigned()`).
   - Signed contract is published to Kafka topic `candidate_contracts` (`CandidateContractSignerService.sendContract()`).

---

## Kafka Contract Message

After signing completes, `sendContract()` publishes a `Message<CandidateContractMessage>` with:

| Field | Source |
|---|---|
| `identityCode` | `candidate.identityCode` |
| `registrationNumber` | `company.registrationNumber` |
| `candidateBackupId` | `candidate.externalId` |
| `file` | base64-encoded `.asice` from local storage |
| `fileName` | `fileDocument.fileName` |

Message type: `candidate_contracts`, action: `create`.

---

## Configuration

```yaml
auth:
  dokobit:
    url: "https://developers.dokobit.com"          # Auth API (Mobile-ID / Smart-ID)
    token: "<api-token>"
    gateway-url: "https://gateway.dokobit.com"     # Signing Gateway API
    gateway-token: "<gateway-token>"
    smart-id: true
    mobile-id: false
    signature: false
```

The `signature` flag controls whether the Dokobit signing flow is enabled at runtime (consumed via `GET /api/properties`).

---

## Dokobit API documentations

Direct link to deletion API
https://gateway-sandbox.dokobit.com/api/doc?_gl=1*poqzwp*_gcl_aw*R0NMLjE3NzQ1MTE1NTguQ2p3S0NBandzcFBPQmhCOUVpd0FURmJpNUVsM3VqaS1pN2JuQ1BnUXBqSmhVeWhBREVZeS1DbzB1eVlFRUUyQVdnYTFkem41OUZEZU1Sb0NOV3dRQXZEX0J3RQ..*_gcl_au*MTczNjY0NzA3Ni4xNzc0NTExNTU4

---
## Pending: Container Cleanup After Signing (VIT-4096)

After `storeContainerFile()` succeeds, the signing container **remains in Dokobit Gateway**. VIT-4096 will add automatic deletion of the container once signing is complete:

- New method `DokobitService.deleteContainer(String containerToken)` — expected endpoint: `POST /api/signing/{token}/delete.json`.
- Called asynchronously in `CandidateContractSignerService.update()` after `candidate.status` is set to `SIGNED`.
- Must be non-blocking: a deletion failure must not affect the signing outcome.
- 404 responses treated as idempotent success (container already deleted).
- All attempts (success and failure) are logged with `containerId`, timestamp, and result.
