# REST API Naming Conventions

This document defines the REST API naming conventions for the project.
All new and modified endpoints **must** follow these rules.

Reference: [RESTful API Resource Naming](https://restfulapi.net/resource-naming/)

## URL Structure

```
/{resource}                     → collection
/{resource}/{id}                → single resource
/{resource}/{id}/{sub-resource} → nested collection
```

## Rules

1. **Use nouns, not verbs.** Resources represent things, not actions.
   - `GET /settings/leave` — not `GET /getLeaveSettings`
   - `PUT /settings/leave` — not `POST /updateLeaveSettings`

2. **Use plural nouns** for collections.
   - `/settings`, `/profiles`, `/vacations`

3. **Use kebab-case** for multi-word resource names.
   - `/vacation-types` (if standalone), `/leave/types` (as sub-resource)

4. **Use HTTP methods** to indicate the action.
   | Method | Meaning |
   |--------|---------|
   | `GET` | Read resource(s) |
   | `POST` | Create a new resource |
   | `PUT` | Full update / upsert |
   | `PATCH` | Partial update |
   | `DELETE` | Remove resource |

5. **Hierarchical nesting** for domain-scoped resources.
   - `/settings/leave` — leave settings under the settings domain
   - `/settings/leave/types` — leave types under leave settings
   - `/settings/leave/types/{id}` — single leave type

6. **No trailing slashes.**
   - `/settings/leave` — not `/settings/leave/`

7. **Use query parameters** for filtering, sorting, pagination.
   - `GET /vacations?status=CONFIRMED&page=0&size=20`

8. **Path variables** for resource identifiers.
   - `GET /settings/leave/types/{id}` — UUID path variable

9. **Standard HTTP status codes.**
   | Code | Usage |
   |------|-------|
   | `200` | Successful read or update |
   | `201` | Successful create |
   | `204` | Successful delete (no body) |
   | `400` | Validation error |
   | `401` | Unauthenticated |
   | `403` | Forbidden (insufficient permissions) |
   | `404` | Resource not found |

## Settings Domain Pattern

All settings follow the `/settings/{domain}` pattern:

| Domain | Base URL | Status |
|--------|----------|--------|
| Core/General | `/settings` | Active |
| Leave | `/settings/leave` | Active |
| Absence | `/settings/absence` | Planned |
| Request | `/settings/request` | Planned |

Each domain controller lives in its own package under `ee.andevis.api.settings.{domain}`.
