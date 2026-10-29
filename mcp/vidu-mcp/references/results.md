# Results, IDs, and Recovery

## Read the Result at the Correct Layer

Successful business tools return `{ "ok": true, "data": { ... } }`. Business failures return `{ "ok": false, "error": { "code", "message", "http_status", "retryable", "trace_id" }, "resources"?: { ... } }` and set MCP `isError=true`. Protocol or input-Schema errors may instead come directly from the MCP client/SDK. These are not CLI stdout/stderr or process exit codes.

For `get_task`, `ok: true` means the query succeeded, not that its task succeeded. Inspect `data.state`: `success`, `failed`, and `canceled` are terminal; other states require continued waiting. Report the actual returned failure details. A quote is an estimate at that time, not a guarantee that later submission will pass permissions, model constraints, or balance checks.

## Keep Identifiers Separate

| Identifier | Response source | Later use |
|---|---|---|
| Task ID | A creator tool such as `generate`: `data.id`; or task history | `get_task`, cancel, delete history, rerun |
| Creation ID | `get_task`: `data.creations[].id`; or `list_creations` | `get_download_info`, enhancement, extension, creation favorite |
| Original upload ID | `prepare_upload`: `data.upload_id` | `complete_upload`, `get_upload_status`, preparation recovery |
| Ready upload URI | `get_upload_status` after `data.state=ready`: `data.uri` | Media references; do not substitute its derived ID for the original upload ID |
| Element ID and version | `get_element` or element list/search/create results | Generation `materials` references; use the actual current version |
| Personal voice ID | `create_voice` or `list_voices` | `get_voice`, `update_voice`, and TTS once ready |

Use ID strings from actual responses. Official voice IDs come from `list_official_voices` and do not identify personal clone records. A one-off image reference does not need a reusable element; upload it and pass the ready URI directly.

## Waiting and Ambiguous Writes

Save identifiers as soon as a submission returns. MCP tools use short requests: poll the appropriate status tool at a reasonable interval instead of adding CLI `--wait`, `--output`, or SSE options. Personal clone creation progress can be inspected with `list_voice_tasks` and `get_voice`; use the real returned status rather than treating a successful create request as a ready voice.

A timeout or interrupted local wait does not cancel the server task or voice clone. Continue querying the original ID after reconnecting. `rerun_task` creates new work and new billing; it is not a recovery query or a network retry. Cancel and delete are separate operations governed by the task's current state and permissions.

If task creation, compose, rerun, extension, upscale, or voice cloning times out or returns an ambiguous network/5xx failure, it may already have taken effect. Preserve returned IDs and resources, inspect recent task/voice records using the original account, and compare the available time/type/input details. If the result cannot be established, explain the uncertainty and obtain authorization before creating another job. Do not silently retry the write, change models or billing modes, or convert a failed voice trial to paid cloning.

Read-only queries and quotes can be retried with bounded backoff for transient network/5xx failures. Parameter, authentication, permission, and quota rejections require correcting the reported issue; unchanged repeated calls are not a remedy. Follow the server's Schema and corrective error details rather than guessing unsupported models or fields.

For interrupted uploads and download retries, use the original upload or creation identifiers as described in [file transfer](files.md). Report only IDs, states, and useful non-sensitive error details; omit Tokens and complete signed URLs.
