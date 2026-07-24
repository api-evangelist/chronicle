---
name: Generate a Chronicle presentation from a prompt
description: Use the Chronicle MCP server (or REST API) to generate an AI presentation from a prompt, optionally grounded on files, and poll it to completion.
api: mcp/chronicle-mcp.yml
transport: mcp
server: https://mcp.chroniclehq.com
operations: [list_workspaces, get_templates, generate_presentation, get_generation_status, send_followup_message, create_upload_target]
---

# Generate a Chronicle presentation

Chronicle turns a prompt (and optional files) into an editable, on-brand presentation.
Generation is asynchronous — you start a job, then poll until it finishes.

## Auth
- **MCP:** connect the remote server `https://mcp.chroniclehq.com`; it authenticates via
  OAuth 2.0 against `https://login.chroniclehq.com` (browser sign-in on first tool call).
- **REST alternative:** `https://api.chroniclehq.com/api/v1` with a workspace API key sent
  as `Authorization: Bearer <key>` or `x-api-key: <key>`. Available on Pro/Plus/Max plans.

## Steps
1. **`list_workspaces`** — call this FIRST (no inputs). Pick the `workspace_id` to work in.
   If the response has `mcp_enabled: false`, stop: every other tool returns `MCP_NOT_ENABLED`.
2. *(optional)* **`get_templates`** with `workspace_id` to pick a `template_id` for a
   consistent, on-brand deck. Omit the template to let Chronicle draft a storyline first
   (Standalone path).
3. *(optional, to ground on a file)* **`create_upload_target`** with `workspace_id`,
   `file_name`, `content_type`, `declared_file_size`; upload the bytes to the returned
   presigned S3 target; keep the attachment reference. Formats: `.txt .md .pdf .pptx
   .jpg .jpeg .png .gif .webp`, max 50 MB.
4. **`generate_presentation`** with `workspace_id`, `prompt`, and optionally `template_id`
   and `attachments[]`. Returns immediately with `status: "generating"` and a `generation_id`.
   Be specific about format and structure (e.g. "create a 10-slide sales proposal from this transcript").
5. **`get_generation_status`** with `workspace_id`, `generation_id`. Each call holds open
   ~50s; if the state is not terminal, call again. Terminal states: `completed`, `failed`.
6. If status is **`awaiting_input`**, Chronicle needs clarification: present its question to
   the user, then **`send_followup_message`** with `workspace_id`, `generation_id`, `content`
   (and optional `attachments[]`). Resume polling `get_generation_status`.
7. On **`completed`**, the presentation includes an `id` and a shareable `url`. Use
   **`get_presentation`** to fetch it later.

## Rules
- Treat generation as a multi-step workflow, not a single request. Poll patiently; do not
  spin a tight loop.
- Errors use `{ "error": { "code", "message", "status" } }`. Retry only `429/500/502/503/504`
  with exponential backoff (1s/2s/4s/8s). Do not blindly retry `400/401/402/403/404`.
- `402` (`PRICING_*`) means a plan/usage limit — surface an upgrade message; retrying will not help.
- Pass user prompts and follow-up answers through verbatim; Chronicle's AI asks for missing context itself.
