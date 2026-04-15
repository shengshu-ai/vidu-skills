# Errors and Retry Strategy (vidu-cli)

CLI JSON shapes: **parameters.md** (canonical). This file focuses on **task lifecycle** and **retry** behavior.

---

## Task state lifecycle

Typical order: `created` → `queueing` → `preparation` → `scheduling` → `processing` → **`success`** or **`failed`** (or `canceled`).

Terminal states: **`success`**, **`failed`**, **`canceled`**. Do not treat other states as final.

---

## When `state` is `failed`

- **`err_code`**: Server-defined code (in `vidu-cli task get` response when available).
- **`err_msg`**: Human-readable message when present.

Tell the user the task failed and pass through **`err_code`** / **`err_msg`**. Do not return media URLs.

---

## Polling results

There is **no built-in wait** in the CLI: `vidu-cli task submit` returns `task_id`; the caller runs **`vidu-cli task get <task_id>`** when needed.

- **`vidu-cli task get`** is read-only and safe to repeat (poll until terminal state or timeout).
- **`vidu-cli task get <task_id> --output <dir>`** downloads media files on success.

---

## Network and client errors (CLI `ok: false`)

- **4xx on submit**: Inspect `error.type`, `code`, `message`; fix parameters or token. Do not retry the same invalid body without changes.
- **5xx or connection error on submit**: Retry with backoff (e.g. 1s, 2s, 4s), limited attempts (e.g. up to 3). If still failing, report the error to the user.
- **Upload / network errors** during image handling: Same idea — bounded retries, then surface **`error`** fields verbatim.
- **Timeouts on `task get`**: Retry once or twice; if still failing, report that the result could not be fetched and suggest trying again later.

---

## Summary for agents

- Treat only **`success`** / **`failed`** / **`canceled`** as terminal (for task `state`).
- On **`failed`**: return **`err_code`** / **`err_msg`**, no media link.
- **No built-in wait**: poll with **`vidu-cli task get`** (or SSE) until done.
- On transport or ambiguous failures: limited backoff retries, then report **`error`** from stdout exactly as **`parameters.md`** describes.
