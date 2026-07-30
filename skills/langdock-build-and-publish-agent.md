---
name: Build and publish a Langdock agent
description: Create a Langdock agent, attach files and knowledge, update it, and publish the draft as a new version using the Agents API.
api: openapi/langdock-openapi-original.yml
operations:
  - createAgentV2
  - getAgentV2
  - updateAgentV2
  - publishAgentV2
  - uploadAttachment
  - deleteAttachment
generated: '2026-07-19'
method: generated
source: openapi/langdock-openapi-original.yml + https://docs.langdock.com/en/developer/agents-api/agent-create
---

# Build and publish a Langdock agent

Agents are the central entity in Langdock: a model plus instructions plus attached
capabilities, resources, and integrations. This skill walks the create → configure →
publish loop.

## Before you start

- Base URL `https://api.langdock.com`, auth `Authorization: Bearer sk-ld-...`.
- The API key must carry the `AGENT_API` scope, or you get
  `403 Insufficient permissions - requires AGENT_API scope`.
- Use the **Agents** API (`/agent/v1`). The `/assistant/v1` family is deprecated — see
  `lifecycle/langdock-lifecycle.yml` and the migration guide at
  https://docs.langdock.com/en/developer/assistants-api/assistant-to-agent-migration.

## 1. Pick a model

Call `GET /agent/v1/models` to list models available to the workspace. Use a model id
from that response — do not hardcode a vendor model name, because availability is
workspace- and region-dependent.

## 2. Upload any attachments first

Call `uploadAttachment` (`POST /attachment/v1/upload`) with multipart form data for each
file the agent should carry. Keep the returned attachment UUIDs. A `400 No file
provided` means the multipart part is missing or misnamed.

To remove one later, call `deleteAttachment` (`DELETE /attachment/v1/delete`). A `403`
means the API key does not have access to that attachment.

## 3. Create the agent

Call `createAgentV2` (`POST /agent/v1/create`). Meaningful fields:

- `name`, `description`, `instruction` — the system prompt
- `model`, `temperature`
- `emojiIcon`, `conversationStarters[]`
- `inputType` — use `STRUCTURED` if you supply `inputFields[]`
- `webSearchEnabled`, `imageGenerationEnabled`, `canvasEnabled`, `extendedThinking`
  (extended thinking is only supported on some models)
- `knowledgeFolderIds[]`, `attachments[]`, `actions[]`

Returns `201`. There is no idempotency key on this API — if a create times out, call
`getAgentV2` before retrying so you do not create a duplicate.

## 4. Update, then publish

`updateAgentV2` (`PATCH /agent/v1/update`) writes to the **draft**. Changes are not live
until you publish.

`publishAgentV2` (`POST /agent/v1/publish`) promotes the draft to a new version. Handle:

- `409 No draft changes to publish` — nothing changed; this is a no-op, not a failure
- `403` — the key lacks edit access, the agent is in a different workspace, or the
  resource is a project

Read back with `getAgentV2` (`GET /agent/v1/get?agentId=...`).

## 5. Talk to the agent

`POST /agent/v1/chat/completions` returns a **Vercel AI SDK compatible** response
format, and supports streaming. It is not the OpenAI chat-completions shape — for that,
use `/openai/{region}/v1/chat/completions` instead.

## Sharing an agent with a key

An agent must be explicitly shared with an API key before that key can query it. See
https://docs.langdock.com/en/developer/agents-api/agent-api-guide. The same requirement
governs the Langdock Agent MCP Server (`mcp/langdock-mcp.yml`).

## Error handling

Vendor discriminated union, not problem+json: `{"type":"error","error":{"type":"...",
"message":"..."}}`. See `errors/langdock-problem-types.yml`.
