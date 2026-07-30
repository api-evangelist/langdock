---
name: Search and maintain Langdock knowledge bases
description: Upload, update, list, delete, and semantically search files in a Langdock knowledge folder shared with your API key.
api: openapi/langdock-openapi-original.yml
operations:
  - POST /knowledge/search
  - POST /knowledge/{folderId}
  - PATCH /knowledge/{folderId}
  - GET /knowledge/{folderId}/list
  - DELETE /knowledge/{folderId}/{attachmentId}
generated: '2026-07-19'
method: generated
source: openapi/langdock-openapi-original.yml + https://docs.langdock.com/en/developer/knowledge-folder-api/sharing
---

# Search and maintain Langdock knowledge bases

Knowledge folders are Langdock's retrieval surface: upload files, then run semantic
search across them. This is the API to use for RAG against workspace documents.

> Note: the knowledge-folder operations carry no `operationId` in the published
> OpenAPI, so they are addressed here by method + path.

## Before you start — sharing is mandatory

A knowledge folder must be **explicitly shared with your API key** before the key can
read or write it. This is separate from scopes. See
https://docs.langdock.com/en/developer/knowledge-folder-api/sharing.

Auth is `Authorization: Bearer sk-ld-...` against `https://api.langdock.com`.

## Semantic search

`POST /knowledge/search` searches across **all** folders shared with the API key — it is
not folder-scoped, so you do not pass a `folderId`. Use it as the retrieval step before
composing a prompt.

Because the search scope is defined by what is shared with the key, control retrieval
scope by issuing separate API keys with different folder shares rather than by
filtering in the request.

## List what is in a folder

`GET /knowledge/{folderId}/list` retrieves files in a folder, or details for a specific
file. Note this endpoint declares **no pagination parameters** — unlike `GET /skills/v1`
and the audit-log endpoint, which are cursor-paginated. Do not write cursor-following
logic here.

## Upload, update, delete

- `POST /knowledge/{folderId}` — upload a new file to the folder
- `PATCH /knowledge/{folderId}` — replace an existing file with a new version
- `DELETE /knowledge/{folderId}/{attachmentId}` — remove a file

Prefer `PATCH` over delete-then-upload when replacing a document, so the file identity
is preserved.

There is no idempotency-key mechanism. If an upload times out, list the folder before
retrying so you do not create a duplicate file.

## Wiring knowledge into an agent

An agent references folders through `knowledgeFolderIds[]` at create/update time — see
`skills/langdock-build-and-publish-agent.md`.

## Error handling

- `401` — invalid or missing API key
- `403` — the folder is not shared with this key
- `404` — folder or attachment not found, or it belongs to another workspace
- `429` — rate limited (default 500 RPM / 60,000 TPM per model)

Errors use the vendor envelope `{"type":"error","error":{"type":"...","message":"..."}}`.
See `errors/langdock-problem-types.yml` and `conventions/langdock-conventions.yml`.
