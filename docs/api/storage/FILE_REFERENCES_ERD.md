# File References ERD

Entity-relationship diagram for all database tables that reference the central `files` table.

> **Note on join key:** All `file_id` columns in domain tables store the `files.uuid` value (varchar),
> **not** `files.id` (bigserial). No foreign-key constraints are declared from domain tables to `files`;
> the link is enforced at the application layer. Only `candidate_files` and `survey_answer_files`
> have declared FK constraints back to their parent domain tables.

Diagram source: [`FILE_REFERENCES_ERD.mmd`](./FILE_REFERENCES_ERD.mmd)

<!-- Render with: VS Code "Mermaid Preview" extension, IntelliJ "Mermaid" plugin, or any Mermaid-aware tool -->
![FILE_REFERENCES_ERD](./FILE_REFERENCES_ERD.mmd)

## Summary table

| Table | Type | References files via | Parent entity |
|---|---|---|---|
| `profile_documents` | embedded field | `file_id → files.uuid` | `profiles` |
| `profile_educations` | embedded field | `file_id → files.uuid` | `profiles` |
| `profile_health_inspections` | embedded field | `file_id → files.uuid` | `profiles` |
| `applications` | embedded field | `file_id → files.uuid` | — |
| `vacation_files` | join table | `file_id → files.uuid` | `vacations` |
| `scheduled_vacation_files` | join table | `file_id → files.uuid` | `scheduled_vacations` |
| `application_files` | join table | `file_id → files.uuid` | `applications` + component |
| `application_property_files` | join table | `file_id → files.uuid` | `application_properties` |
| `application_creator_files` | join table | `file_id → files.uuid` | `applications` |
| `candidate_files` | join table | `file_id → files.uuid` | `candidates` |
| `survey_answer_files` | join table | `file_id → files.uuid` | `survey_answers` |
