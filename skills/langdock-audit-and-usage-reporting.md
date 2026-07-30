---
name: Audit and usage reporting on a Langdock workspace
description: Page the Langdock workspace audit trail and export usage and cost data by agent, user, project, model, and API key.
api: openapi/langdock-openapi-original.yml
operations:
  - listAuditLogs
  - POST /export/assistants
  - POST /export/users
  - POST /export/projects
  - POST /export/models
  - POST /export/api-keys
generated: '2026-07-19'
method: generated
source: openapi/langdock-openapi-original.yml + https://docs.langdock.com/en/developer/audit-logs-api/intro-to-audit-logs-api
---

# Audit and usage reporting on a Langdock workspace

Two related governance surfaces: the audit trail (who did what, when, from where) and
the usage exports (consumption and cost). Use these to feed a SIEM or a chargeback
report.

## Audit logs

`listAuditLogs` — `GET /audit-logs/{workspace_id}`.

- Requires an API key with the **`AUDIT_LOG_API`** scope.
- The `workspace_id` in the path must match the key's workspace, or you get `403`.
- Cursor-paginated: pass `cursor` and `limit`, read items from `data[]`, follow
  `next_cursor` until it is null.
- Filter with `from`, `to`, `entity_type`, `actor_id`.

Note the casing split: the audit-log API is **snake_case** (`next_cursor`, `actor_id`,
`created_at`) while the Skills and Agents APIs are camelCase. Do not share a
deserializer across them.

Each entry carries:

- `action` — dot notation `{entity}.{operation}`
- `entity_type` — PascalCase model name (e.g. `User`), plus `entity_id`
- `actor_id`, `actor_type`, `actor_name` — the actor may be a user, an API key, or SCIM
- `ip_address`, `user_agent`
- `changes` — structured diff with `before` and `after` objects
- `snapshot` — full entity snapshot or extra event metadata

For incremental sync, page forward with `from` set to the last successfully processed
`created_at` rather than storing cursors, since cursors are opaque and not guaranteed
durable.

## Usage exports

Five sibling endpoints, all `POST`, all returning CSV or JSON:

| Endpoint | Exports |
|---|---|
| `POST /export/assistants` | agent usage — message counts, active users, trends |
| `POST /export/users` | user activity — subject to workspace privacy settings |
| `POST /export/projects` | project activity — involved users, resource consumption |
| `POST /export/models` | model usage — request counts, BYOK token consumption |
| `POST /export/api-keys` | usage and cost by API key |

Handle these two cases explicitly, because both are normal:

- `400 Bad Request - Export too large or invalid date range` — narrow the window and
  page through in chunks rather than retrying the same range
- `404 No data found for the selected period` — an empty period is not an error
  condition; treat it as zero rows

User-level export is **subject to privacy settings**, so a workspace may legitimately
return less than you expect. Do not treat missing user rows as a failure.

## Error handling

The export endpoints use their own flat envelope `{"error": "...", "message": "..."}`,
not the `{"type":"error","error":{...}}` union used elsewhere. Branch accordingly — see
`errors/langdock-problem-types.yml`.
