# Errors, Waiting, and Retry Decisions

Use with `vidu-cli v0.2.36+`. Keep the default JSON format for automation; table mode is for human inspection. Read stdout, stderr, and the exit code separately.

## Output Channels and Envelopes

| Result | stdout | stderr / exit status |
|---|---|---|
| Help/version | Human-readable text | Exit 0; not a JSON API response. |
| CLI argument parsing failure | May be empty | Text with usage, available values/commands, and suggestions when applicable; exit 2. |
| Runtime success | One JSON object with `ok: true` | Progress/debug output may appear on stderr; exit 0. |
| Runtime/API/network/wait failure | One JSON object with `ok: false` and `error` | Nonzero exit, normally 1. |

Typical submit success:

```json
{"ok":true,"task_id":"task_from_response"}
```

Runtime errors have `error.type` and `error.message`, plus available context: `http_status`, `code`, `step`, `fields`, `trace_id`, `cause`, `suggestion`. Success responses may also contain `trace_id`. These fields are optional; do not invent missing values or assume every command returns a trace ID.

Example waiting timeout:

```json
{"ok":false,"error":{"type":"timeout_error","message":"Task wait timed out; the background task continues","step":"wait_task","fields":{"task_id":"task_from_response","state":"processing"}}}
```

| Error type | Meaning / next step |
|---|---|
| `client_error` | Local input, file, credential, or validation error; correct the reported issue. |
| `http_error` | API returned a failure response; preserve available status, code, fields, cause, and suggestion. |
| `network_error` | Connection or transport failure; whether retry is safe depends on the operation. |
| `parse_error` | Missing/malformed response or metadata; a submit may still have reached the server. |
| `task_error` | A creator task reached a failed or canceled outcome while waiting. |
| `timeout_error` | A creator task's local waiting deadline expired; background work can continue. |
| `voice_failed` | Personal voice cloning failed; available status details are in the error context. |
| `timeout` | A personal voice's local waiting deadline expired; server cloning can continue. |

Report returned error fields faithfully, including actionable `suggestion` and useful IDs. Do not infer undocumented causes from a status code alone. Do not expose tokens or reconstruct signed download URLs in reports.

## Creator Task States

Typical progression: `created` → `queueing` → `preparation` → `scheduling` → `processing` → `success` / `failed` / `canceled`. Only the last three are terminal.

`task get` is a successful read even when the remote task failed: it can return `ok: true`, `state: "failed"`, `err_code`, and `err_msg`. Inspect `state`, not only `ok`. Report available `err_code` / `err_msg`; do not present failed output as completed media.

`task wait` and submissions with `--wait` wait for completion. Failed/canceled waits return `ok: false` with `task_error` and task context in `error.fields`. If the wait error does not include the server's failure code, query `task get TASK_ID` for the available `err_code` / `err_msg` rather than inventing it.

Personal cloned voices have their own `created`, `processing`, `success`, `failed` states. Use `voice get/wait`, not `task get/wait`, with the returned personal `voice_id`. A failed status snapshot/wait returns `voice_failed`; its waiting deadline error uses `timeout`, not the creator wait's `timeout_error`. See [voices](voices.md).

## Waiting and Recovery

- Default creator wait timeout: 1800 seconds; default polling: 5 seconds. `task wait` accepts a custom positive `--poll-interval`. Submission `--timeout`/`--output` require `--wait`.
- Progress on stderr does not mean failure. The final envelope is on stdout.
- A timeout or interruption ends local waiting only. It does not cancel the server task or voice clone.
- Preserve IDs from the initial submission and available `error.fields.task_id` / `voice_id`; resume with the same ID. Do not create another task because waiting stopped.
- Voice waits use SSE and have no poll-interval option. Downloads after creator success can be retried through `get` or `wait` for the same task.

```bash
vidu-cli task wait "$TASK_ID" --timeout 1800 --output ./results
vidu-cli task get "$TASK_ID" --details
vidu-cli voice wait "$VOICE_ID" --timeout 1800
```

## Retry by Operation

| Operation / failure | Action |
|---|---|
| Read-only query/quote/status or downloading an existing task | Bounded retries are appropriate for transient network/5xx failures, e.g. up to 3 attempts with 1s/2s/4s backoff. Stop and report persistent failures. |
| Submission, rerun, extension, upscale, voice clone, or another write with ambiguous transport/5xx/parse failure | Do not blindly repeat. It may already have created work or consumed credits. Preserve all returned IDs and inspect task history/details or personal voice lists to establish the outcome. |
| Known task/voice ID but wait failed or timed out | Continue querying/waiting for that ID. |
| Input/authentication/permission rejection | Fix the specific returned issue; unchanged repeated requests are not a remedy. |
| Quota/eligibility rejection | Explain the selected mode's failure and alternatives; do not silently change billing or model. |
| Trial clone rejection | Do not retry as paid without authorization. |

If a write outcome cannot be established, report the uncertainty and the available identifiers instead of claiming failure proves nothing was created. Upload or read errors should be diagnosed at the reported step; restarting the whole generation is not a general retry strategy.

## Common Corrections

| Reported issue | Correction |
|---|---|
| `--output` or `--timeout` requires `--wait` on a submission | Add `--wait` when completion is needed, or submit without those options and later use `task wait/get`. |
| Unknown `task download` | Use `task get TASK_ID --output DIR` or `task wait TASK_ID --output DIR`. |
| Invalid material shape | Pass one `--material "NAME:ELEMENT_ID:VERSION"` per element; do not join multiple materials into one argument. |
| Missing TTS voice | Select an exact official voice from `task tts-voices` or a ready personal ID from `voice list`; use `--voice-id` (alias `--voice`). |
| `task submit --type tts` | Use the separate `task tts` command. |
| Multi-segment TTS with subtitles | Pass `--subtitle-enable false`, or use a single segment if subtitles are required. |
| File output with multiple artifacts | Use a directory, especially for TTS audio plus subtitle JSON. |
| No task/voice ID in a submit response | Inspect the actual stdout/stderr and history/list before considering another submission. |
| Unknown flag, command, model, or value | Read the CLI's suggestion and scoped `--help`; confirm CLI ≥v0.2.36. |

### Pass and Off-Peak Rejections

`ClawPassExplicitModeRequired` is a submit-time error in `error.code`, not a remote task state. It can occur when pass mode was automatically selected but the daily quota is exhausted. Do not automatically resubmit with `--schedule-mode normal`; that changes billing. Explain waiting for refresh or choosing credit mode, and follow the user's authorization.

`nopeak` is **off-peak mode (闲时模式)**. Eligibility, remaining off-peak quota, and task/model restrictions are server-validated. Do not infer off-peak availability from `quota pass`, or invent a universal off-peak quota/error code. Q4 accepts off-peak subject to server rules; Drama/Wan 3 reject it and only support credit mode. See [parameters](parameters.md).
