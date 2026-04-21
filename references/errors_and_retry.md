# Errors and Retry Strategy (vidu-cli)

CLI JSON shapes: **parameters.md** (canonical). This file focuses on **task lifecycle** and **retry** behavior.

---

## Task state lifecycle

Typical order: `created` → `queueing` → `preparation` → `scheduling` → `processing` → **`success`** or **`failed`** (or `canceled`).

Terminal states: **`success`**, **`failed`**, **`canceled`**. Do not treat other states as final.

---

## When `state` is `failed`

- **`err_code`**: Server-defined code (in `vidu-cli task get` response when available). Common codes include `INVALID_PROMPT_LENGTH`, `INSUFFICIENT_CREDITS`, `CONTENT_MODERATION`, `TASK_TIMEOUT`. Always report the exact code — do not interpret or translate.
- **`err_msg`**: Human-readable message when present.

Tell the user the task failed and pass through **`err_code`** / **`err_msg`**. Do not return media URLs.

### `ClawPassExplicitModeRequired`

Returned when `--schedule-mode` is omitted (auto-detected as `claw_pass`) but the user's daily claw-pass quota is exhausted. **Do not retry** — inform the user that their daily quota is used up. They can either wait for the next refresh or re-submit with `--schedule-mode normal` to use credits instead.

---

## Polling results

The CLI does **not** block waiting for task completion: `vidu-cli task submit` returns `task_id` immediately; the caller must poll with **`vidu-cli task get <task_id>`** in a loop until a terminal state is reached.

**Recommended polling strategy**: poll every **3–5 seconds** for active tasks. For tasks expected to take longer (e.g. 16s video generation), start with a 5s interval and increase to 10s after the first minute.

- **`vidu-cli task get`** is read-only and safe to repeat (poll until terminal state or timeout).
- **`vidu-cli task get <task_id> --output <dir>`** downloads media files on success.

---

## Network and client errors (CLI `ok: false`)

- **4xx on submit**: Inspect `error.type`, `code`, `message`; fix parameters or token. **Do not retry** — 4xx errors indicate invalid input or authentication issues that won't resolve by retrying.
- **5xx or connection error on submit**: Retry with backoff (e.g. 1s, 2s, 4s), limited attempts (e.g. up to 3). If still failing, report the error to the user.
- **Upload / network errors** during image handling: Same idea — bounded retries, then surface **`error`** fields verbatim.
- **Timeouts on `task get`**: Retry once or twice with a 30-second timeout per request; if still failing, report that the result could not be fetched and suggest trying again later.

---

## Summary for agents

- Treat only **`success`** / **`failed`** / **`canceled`** as terminal (for task `state`).
- On **`failed`**: return **`err_code`** / **`err_msg`**, no media link.
- **No built-in wait**: the CLI returns immediately after submit; poll with **`vidu-cli task get`** every 3–5 seconds until done.
- On transport or ambiguous failures: limited backoff retries, then report **`error`** from stdout exactly as **`parameters.md`** describes.
