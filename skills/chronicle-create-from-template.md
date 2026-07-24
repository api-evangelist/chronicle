---
name: Create a Chronicle presentation from a template
description: Create a new Chronicle presentation directly from a template (no AI generation), then fetch or retitle it.
api: mcp/chronicle-mcp.yml
transport: mcp
server: https://mcp.chroniclehq.com
operations: [list_workspaces, get_templates, create_presentation, get_presentation]
---

# Create a Chronicle presentation from a template

This is the deterministic, no-AI path: pick a template and instantiate it. Good for
consistent, on-brand output where you do not need generation.

## Auth
Same as the generation skill — MCP (`https://mcp.chroniclehq.com`, OAuth via
`login.chroniclehq.com`) or the REST API (`https://api.chroniclehq.com/api/v1` with a
workspace API key in `Authorization: Bearer` or `x-api-key`).

## Steps
1. **`list_workspaces`** (no inputs) — pick `workspace_id`. Bail if `mcp_enabled: false`.
2. **`get_templates`** with `workspace_id` — choose a `template_id`. Templates include
   published public ones plus those available to the workspace.
3. **`create_presentation`** with `workspace_id`, `template_id`, and optionally
   `selected_section_ids` to include only specific sections. No AI is involved; this
   returns a presentation `id` and shareable `url` synchronously.
4. *(optional)* **`get_presentation`** with `workspace_id`, `presentation_id` to fetch it.
5. *(REST only)* Retitle with `PATCH /presentations/:id` — in v1 only `title` is editable;
   any other content change requires a fresh generation.

## Rules
- Errors use `{ "error": { "code", "message", "status" } }`. `TEMPLATE_NOT_FOUND` /
  `DOCUMENT_NOT_FOUND` (404) mean the id is unknown or outside your workspace.
- `403 INSUFFICIENT_PRIVILEGES` means the key/token is bound to a different workspace than
  the resource. Each key is workspace-scoped.
