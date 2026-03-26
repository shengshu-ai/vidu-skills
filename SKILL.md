---
name: vidu-skills
description: Generate video and images by calling the official Vidu API via vidu CLI. Use when the user wants text-to-image (文生图), text-to-video (文生视频), image-to-video (图生视频), head-tail-image-to-video (首尾帧生视频), reference-to-image (参考生图), reference-to-video (参考生视频), Create References (创建参考资料), or to submit or check Vidu tasks. Requires VIDU_TOKEN and optional VIDU_BASE_URL.
compatibility: Requires vidu-cli >= 0.2.0 (install via `cargo install vidu-cli`). Set VIDU_TOKEN in the environment; VIDU_BASE_URL optional (default https://service.vidu.cn).
version: 1.3.1
url: https://www.vidu.cn/
secrets:
  - VIDU_TOKEN
dependencies:
  - cargo
---

# Vidu Video and Image Generation Skill (Vidu 音视频/图像生成技能)

Generate AI videos and images with Vidu (生数) via `vidu-cli` — text-to-image, text-to-video, image-to-video, start-end frame, reference-based generation, and material elements, up to 1080p/2K/4K.

## Execution model: use vidu CLI

**All execution is done via the `vidu-cli` CLI tool.** Parameters are CLI flags (not raw JSON bodies).

**Environment variables**

- `VIDU_TOKEN` (required) — Vidu API token
- `VIDU_BASE_URL` (optional) — Default `https://service.vidu.cn` (mainland China); use `https://service.vidu.com` for overseas
- `VIDU_DEBUG` (optional) — Set to `1` to print full response body to stderr for debugging

**Stdout contract**

- Every command prints **one line of JSON** to stdout.
- Success: `{"ok": true, "trace_id": "...", ...}` — exit code `0`
- Failure: `{"ok": false, "error": {"type": "...", "http_status": ..., "code": "...", "message": "..."}}` — exit code `1`
- **`trace_id`** appears on API-backed responses for support/debugging.
- **CRITICAL: Never guess why an error happened. Copy fields from `error` exactly.** Full shapes and edge cases: **references/parameters.md**.

**Error `type` values**

- `http_error` — API 4xx/5xx (`http_status`, `code`, `message`)
- `network_error` — Connection failure or timeout
- `parse_error` — Response is not valid JSON
- `client_error` — Local issues (missing token, bad path, validation)

**Main commands**

| Command | Purpose |
|---------|---------|
| `vidu-cli upload <image_path>` | Upload image → `upload_id`, `ssupload_uri` |
| `vidu-cli task submit --type ... --prompt ... [options]` | Submit task → `task_id`. `--image`: local path, URL, or `ssupload:?id=...` (auto-upload). |
| `vidu-cli task get <task_id>` | Query task → `state`, `media_urls` when successful |
| `vidu-cli task sse <task_id>` | Stream SSE state events |
| `vidu-cli element create --name ... --image ... [--description ...] [--style ...]` | Create reference element (check → preprocess → create). Returns `id`, `version`. |
| `vidu-cli element check --name ...` | Check name availability |
| `vidu-cli element list [--keyword kw]` | List personal elements |
| `vidu-cli element search --keyword kw` | Search community elements |

**Smart image handling** (`task submit --image`, `element create --image`)

- Local path → auto-upload (auto-compress when file is larger than 10MB)
- `http(s):` URL → download then upload
- `ssupload:?id=...` → use as-is

---

## Key Capabilities

- **text-to-image (文生图)** — Text-only image generation
- **text-to-video (文生视频)** — Text-only video generation
- **image-to-video (图生视频)** — One image + text → video
- **head-tail-image-to-video (首尾帧生视频)** — Start + end frames + text
- **reference-to-image (参考生图)** — **Images + materials: 1–7** total; **text prompt required**; can be images-only, materials-only, or mixed; images-only needs no `element create`
- **reference-to-video (参考生视频)** — Same rule: **1–7** total; **text prompt required**
- **Create References (创建主体)** — `element create` (single command)
- **Search Community References (搜索社区主体库)** — `element search`
- **Query task (查询任务)** — `task get` / `task sse`

---

## Setup

1. `cargo install vidu-cli`
2. Obtain `VIDU_TOKEN` (e.g. Vidu console).
3. `export VIDU_TOKEN="..."` — required; `export VIDU_BASE_URL=...` if not using default region.
4. Verify: `vidu-cli task submit --help`

---

## Data usage and privacy (summary)

Content you send (prompts, images, task settings) goes to Vidu’s API. Confirm this meets your privacy and IP needs. Prefer least-privilege tokens for testing. Terms: https://www.vidu.com/terms (overseas), https://www.vidu.cn/terms (mainland China).

---

## Async workflow (short)

- Vidu generation is **asynchronous**: `task submit` → **`task_id`** → poll **`task get <task_id>`** until terminal state.
- **Model nicknames**: Q1 → `3.0`, Q2 → `3.1`, Q3 → `3.2` (see **references/parameters.md** for per-task allowed versions).
- Task-type summaries, **task support matrix**, **copy-paste CLI examples**, **prompt tips**, and **element create/list/search** details are in **references/parameters.md**.
- Task lifecycle, retries, and polling guidance: **references/errors_and_retry.md**.

---

## Implementation guide

1. Pick capability → map to `--type` and options using **references/parameters.md** (matrix + validation).
2. Prepare inputs: for **reference2image** / **character2video**, `--image` and/or `--material` so **combined count is 1–7**; optional `[@name]` in prompt per **references/parameters.md**.
3. `vidu-cli task submit ...` → store `task_id` and `trace_id`.
4. `vidu-cli task get <task_id>` until `success` or `failed` (or use `task sse` if appropriate).
5. On success return `media_urls`; on task failure return `err_code` / `err_msg`; on CLI `ok: false` return `error` fields verbatim.

---

## Output to the user

- After **submit**: return **`task_id`** and **`trace_id`**; state that processing is in progress.
- After **query**: if `state` is success, return **`media_urls`**; if failed, return **`err_code`** and **`err_msg`** exactly (note: response may still have `ok: true` while `state` is `failed`).
- On **CLI failure** (`ok: false`): report `error.type`, `http_status`, `code`, `message` exactly — **do not infer causes**.

---

## References (bundled)

| File | Contents |
|------|----------|
| **references/parameters.md** | Task matrix, CLI flags, examples, prompt tips, validation |
| **references/errors_and_retry.md** | States, retries, polling |

---

## Fallback (no Rust / Cargo)

If `cargo` / `vidu-cli` cannot be installed, this skill cannot run. Require **vidu-cli ≥ 0.2.0** and point users to **references/parameters.md** for parameter details.
