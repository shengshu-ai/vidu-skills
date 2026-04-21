---
name: vidu-skills
description: Generate video and images by calling the official Vidu API via vidu CLI. Use when the user wants text-to-image, text-to-video, image-to-video, head-tail-image-to-video, reference-to-image, reference-to-video, lip-sync, text-to-speech, video-compose, Create References, or to submit or check Vidu tasks. Requires VIDU_TOKEN and optional VIDU_BASE_URL.
version: 1.4.2
homepage: https://www.vidu.cn/
primaryEnv: VIDU_TOKEN
metadata: {"openclaw":{"requires":{"bins":["node","npm","vidu-cli"],"env":["VIDU_TOKEN"]},"primaryEnv":"VIDU_TOKEN","install":[{"id":"vidu-cli","kind":"node","package":"vidu-cli","bins":["vidu-cli"],"label":"Install vidu-cli via npm (requires Node.js >=14; postinstall downloads a platform binary from GitHub)"}]}}
---

# Vidu Video and Image Generation Skill

Generate AI videos and images with Vidu via `vidu-cli` — text-to-image, text-to-video, image-to-video, start-end frame, reference-based generation, and material elements, up to 1080p/2K/4K.

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
| `vidu-cli task get <task_id> [--output/-o <dir>]` | Query task → `state`, `type`, `model`; use `--output` to download media on success |
| `vidu-cli task compose --timeline <json> [--width N --height N]` | Compose video from timeline → `task_id`. Query with `task get`. **MUST read references/compose.md before building the timeline JSON — do not guess the schema.** |
| `vidu-cli task lip-sync --video <path> --text <text> [options]` | Lip-sync with text-to-speech → `task_id`. Supports `--schedule-mode` (auto-detected if omitted). |
| `vidu-cli task lip-sync --video <path> --audio <path>` | Lip-sync with audio file → `task_id`. Supports `--schedule-mode` (auto-detected if omitted). |
| `vidu-cli task lip-sync-voices` | List available lip-sync voices (~86, Chinese/English/Cantonese/Cartoon etc.) |
| `vidu-cli task tts --prompt ... --voice-id ...` | Text-to-speech → `task_id`. Supports `--schedule-mode` (auto-detected if omitted). |
| `vidu-cli task tts-voices` | List available TTS voices (300+, 20+ languages) |
| `vidu-cli task cost --type ... --model-version ... --duration ...` | Query task credit cost (estimate before submitting) |
| `vidu-cli quota pass` | Query claw-pass daily quota status |
| `vidu-cli quota credit` | Query user credit balance |
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

- **text-to-image** — Text-only image generation
- **text-to-video** — Text-only video generation
- **image-to-video** — One image + text → video
- **head-tail-image-to-video** — Start + end frames + text
- **reference-to-image** — **Images + materials: 1–7** total; **text prompt required**; can be images-only, materials-only, or mixed; images-only needs no `element create`
- **reference-to-video** — Same rule: **1–7** total; **text prompt required**
- **lip-sync** — Drive video mouth movement with text-to-speech or audio file
- **text-to-speech** — Convert text to speech audio via `task tts`
- **video-compose** — Compose multi-track timeline (video/audio/subtitle/effect) into a single exported video via `task compose`
- **create-references** — `element create` (single command)
- **search-community-references** — `element search`
- **query-task** — `task get [--output <dir>]`

---

## Setup

1. `npm install -g vidu-cli@latest` (requires Node.js >=14; postinstall auto-downloads the platform binary)
2. Obtain `VIDU_TOKEN` (e.g. Vidu console).
3. Set `VIDU_TOKEN` environment variable (required); set `VIDU_BASE_URL` if not using default region.
4. Verify: `vidu-cli task submit --help`

---

## Data usage and privacy (summary)

Content you send (prompts, images, task settings) goes to Vidu’s API. Confirm this meets your privacy and IP needs. Prefer least-privilege tokens for testing. Terms: https://www.vidu.com/terms (overseas), https://www.vidu.cn/terms (mainland China).

---

## Async workflow (short)

- Vidu generation is **asynchronous**: `task submit` → **`task_id`** → poll **`task get <task_id>`** until terminal state.
- **Model nicknames**: Q1 → `3.0`, Q2 → `3.1`, Q3 → `3.2`. Additional variants exist: `3.1_pro`, `3.2_fast_m`, `3.2_pro_m` — see **references/parameters.md** for the complete per-task model version list.
- Task-type summaries, **task support matrix**, **copy-paste CLI examples**, **prompt tips**, and **element create/list/search** details are in **references/parameters.md**.
- Task lifecycle, retries, and polling guidance: **references/errors_and_retry.md**.

---

## Implementation guide

### For task submit (generation tasks)

1. Pick capability → map to `--type` and options using **references/parameters.md** (matrix + validation).
2. Prepare inputs: for **reference2image** / **character2video**, `--image` and/or `--material` so **combined count is 1–7**; optional `[@name]` in prompt per **references/parameters.md**.
3. `vidu-cli task submit ...` → store `task_id` and `trace_id`.
   - **schedule-mode auto-detection**: if `--schedule-mode` is omitted, CLI queries claw-pass status and uses `claw_pass` when user has an active pass, otherwise `normal`. If submit fails with `ClawPassExplicitModeRequired`, tell the user their daily claw-pass quota is exhausted. Do not retry automatically.
4. `vidu-cli task get <task_id>` until `success` or `failed`; use `--output <dir>` to download media on success.
5. On success return `downloaded_files` (if `--output` used) or prompt user to re-run with `--output`; on task failure return `err_code` / `err_msg`; on CLI `ok: false` return `error` fields verbatim.

### For task compose (video composition)

**CRITICAL: Before constructing the `--timeline` JSON, you MUST read **references/compose.md** first.** The timeline has a specific JSON schema with exact field names, nesting structure, and media_url rules. Do NOT guess the structure — always refer to compose.md for the complete schema, supported fields, and examples.

1. Read **references/compose.md** to understand the timeline JSON schema, media_url rules, and limits.
2. Build the timeline JSON following the exact structure: `video_tracks[].video_track_clips[]`, `audio_tracks[].audio_track_clips[]`, `subtitle_tracks[].subtitle_track_clips[]`, `effect_tracks[].effect_track_items[]`.
3. For `media_url`: use `ssupload:?id=xxx`, http URL, or local file path (auto-uploaded by CLI).
4. For `file_url` (subtitles): use `ssupload:?id=xxx`, http URL, or local .srt file path.
5. `vidu-cli task compose --timeline <file_or_json> [--width N --height N]` → returns `task_id`.
6. `vidu-cli task get <task_id>` to poll status, same as other tasks.

---

## Output to the user

- After **submit**: return **`task_id`** and **`trace_id`**; state that processing is in progress.
- After **query**: if `state` is success, return **`downloaded_files`** (if `--output` was used) or the `task_id` with a note to re-run with `--output <dir>` to download; if failed, return **`err_code`** and **`err_msg`** exactly (note: response may still have `ok: true` while `state` is `failed`).
- On **CLI failure** (`ok: false`): report `error.type`, `http_status`, `code`, `message` exactly — **do not infer causes**.

---

## References (bundled)

| File | Contents |
|------|----------|
| **references/parameters.md** | Task matrix, CLI flags, examples, prompt tips, validation |
| **references/errors_and_retry.md** | States, retries, polling |
| **references/compose.md** | Timeline schema, media_url rules, clip compose examples |

---

## Fallback (no Node.js / npm)

If `node` / `npm` / `vidu-cli` cannot be installed, this skill cannot run. Require **vidu-cli latest** (via `npm install -g vidu-cli@latest`, Node.js >=14) and point users to **references/parameters.md** for parameter details.
