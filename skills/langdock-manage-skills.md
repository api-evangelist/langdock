---
name: Manage Langdock Skills
description: Create, find, update, import, and delete reusable Skill instruction packages in a Langdock workspace via the Skills API.
api: openapi/langdock-openapi-original.yml
operations:
  - listSkills
  - createSkill
  - getSkill
  - updateSkill
  - deleteSkill
  - importSkill
generated: '2026-07-19'
method: generated
source: openapi/langdock-openapi-original.yml + https://docs.langdock.com/api-endpoints/skills/skills-overview
---

# Manage Langdock Skills

A Skill is a named, reusable instruction package that agents in a Langdock workspace
can use. This skill covers the full lifecycle: list, create, read, update, import, delete.

## Before you start

- Base URL is `https://api.langdock.com`. On a dedicated deployment it is
  `https://<your-domain>/api/public`.
- Authenticate every call with `Authorization: Bearer sk-ld-...`. Keys are created in
  Workspace Settings → Products → API.
- The key needs the Skills scope. A `403 Missing required scope or access` means the
  key lacks the scope; `403 Missing required scope or create/editor/owner permission`
  means it lacks permission on that specific Skill.
- Never call this API from a browser — Langdock blocks browser-origin calls. Backend only.

## Find a Skill

Call `listSkills` (`GET /skills/v1`). It is cursor-paginated and filterable:

- `query` — free-text search
- `slug` — look up by stable slug
- `limit` — page size
- `cursor` — opaque cursor from the previous page

Read results from `skills[]` and follow `nextCursor` until it is absent. Do not assume
offset pagination; there is no page/offset parameter.

Then call `getSkill` (`GET /skills/v1/{skillId}`) for the full record.

## Create a Skill

Call `createSkill` (`POST /skills/v1`). A Skill carries:

- `name` — display name
- `slug` — stable identifier; must be unique in the workspace
- `description` — explains *when* the Skill should apply (this is what agent selection
  reads, so make it specific)
- `instructions` — the instruction body
- `integrationIds[]` — integrations the Skill needs

Returns `201`. A `409 Skill slug conflict` means the slug is taken — pick another
rather than retrying the same body. There is no idempotency-key mechanism on this API,
so a retry after a timeout may create a duplicate; check with `listSkills?slug=...`
before retrying.

## Import a Skill from SKILL.md or a zip

Call `importSkill` (`POST /skills/v1/import`) with multipart form data. This creates or
updates from a `SKILL.md` file or a zip archive, so it is the upsert path.

- `200` = updated an existing Skill, `201` = created a new one
- `409` = existing slug conflict or an ambiguous upsert slug — make the slug explicit
- `413` = the archive is too large; shrink it
- `400` = invalid form data or a malformed zip

## Update and delete

- `updateSkill` (`PATCH /skills/v1/{skillId}`) — partial update. `409` on slug conflict.
- `deleteSkill` (`DELETE /skills/v1/{skillId}`) — requires owner access on the Skill.

Both return `404 Skill not found` if the id is wrong or the Skill lives in another
workspace.

## Error handling

Errors come back as `{"type":"error","error":{"type":"...","message":"..."}}` — not
RFC 9457 problem+json. Branch on `error.type`:

- `invalid_request_error` (400) — fix the payload
- `authentication_error` (401) — bad or missing key
- `permission_error` (403) — missing scope or Skill permission
- `not_found_error` (404) — wrong id or wrong workspace
- `rate_limit_error` (429) — back off; default limits are 500 RPM / 60,000 TPM per model
- `api_error` (500) — retry with backoff, then check https://status.langdock.com/

See `errors/langdock-problem-types.yml` and `conventions/langdock-conventions.yml`.
