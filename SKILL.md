---
name: vidu-skills
description: Generate video and images by calling the official Vidu API via vidu CLI. Use when the user wants text-to-image, text-to-video, image-to-video, head-tail-image-to-video, reference-to-image, reference-to-video, lip-sync, text-to-speech, video-compose, Create References, or to submit or check Vidu tasks. Requires VIDU_TOKEN and optional VIDU_BASE_URL.
version: 1.4.14
homepage: https://www.vidu.cn/
primaryEnv: VIDU_TOKEN
metadata: {"openclaw":{"requires":{"bins":["node","npm","vidu-cli"],"env":["VIDU_TOKEN"]},"primaryEnv":"VIDU_TOKEN","install":[{"id":"vidu-cli","kind":"node","package":"vidu-cli","bins":["vidu-cli"],"label":"Install vidu-cli via npm (requires Node.js >=14; postinstall downloads a platform binary from GitHub)"}]}}
---

# Vidu Video and Image Generation Skill

Generate AI videos and images with Vidu via `vidu-cli`: text-to-image, text-to-video, image-to-video, start/end-frame video, reference-based generation, reusable elements, TTS, lip-sync, and compose.

## Execution Model

All execution uses the `vidu-cli` CLI. Use CLI flags, not raw JSON request bodies. Use argv arrays; do not wrap commands in shell strings.

Vidu creator tasks are async: submit returns `task_id`; poll with `vidu-cli task get <task_id>` and download with `vidu-cli task get <task_id> -o <dir>`.

Environment:

- `VIDU_TOKEN` (required): Vidu API token.
- `VIDU_BASE_URL` (optional): default `https://service.vidu.cn`; use `https://service.vidu.com` for overseas.
- `VIDU_DEBUG=1` (optional): print full response body to stderr for debugging.

For installation, token setup, and first-run checks, read `references/setup.md` only when the user is setting up the skill or the CLI is missing.

## Decision Table

| User intent | Use this command shape | Notes |
|---|---|---|
| Create reusable subject/character from existing image | `vidu-cli element create --name ... --image ... [--description ...] [--style ...]` | Returns `id`, `version`; no `duration`. |
| Generate reference image from prompt/images | `vidu-cli task submit --type reference2image --prompt ... --image ... --duration 0 --model-version ... --resolution ...` | Use when no reusable element exists yet. |
| Generate character/reference video | `vidu-cli task submit --type character2video --prompt ... (--image ...\|--material name:id:version) --duration ... --model-version ... --resolution 1080p` | Image/material total must be 1-7. |
| Generate narration/audio | `vidu-cli task tts --prompt ... --voice-id ... [--speed ...]` | Use `--voice-id`, never `--voice`. |
| Compose final video | `vidu-cli task compose --timeline <json> ...` | Read `references/compose.md` first. |
| Query/download task | `vidu-cli task get <task_id> [-o <dir>]` | `--output` only belongs on `task get`. |

## Do Not Invent

Never use these:

- `vidu-cli reference ...`
- `vidu-cli reference create-image-reference`
- `vidu-cli task submit --type tts`
- `vidu-cli task tts --voice ...`
- `vidu-cli task submit --output ...`
- `vidu-cli task download`
- `vidu-cli task submit --material "a:id:ver,b:id:ver"`
- `sh -c "vidu-cli ..."`

## Minimal Rules

- Every async creator command (`task submit`, `task tts`, `task compose`, `task lip-sync`) must go through `provider_batch`, even for one item.
- Use direct argv arrays; put every flag and value in separate argv entries.
- Repeat `--image`, `--material`, `--audio`, and `--video` once per item. Do not comma-join.
- `reference2image` and `character2video` require non-empty prompt plus 1-7 total references (`--image` + `--material`).
- Image tasks use `--duration 0`; reusable `element create` has no `duration`.
- For `character2video` with `3.2_a`, duration must be an explicit `4-15`; for other models, check `references/parameters.md`.
- TTS subtitle output is enabled by default for single `--prompt` or `--prompt-path`; use `--subtitle-enable false` to disable it. Multi-segment `--text` mode currently requires `--subtitle-enable false`.
- Download successful media with `vidu-cli task get <task_id> --output <dir>`; when `subtitle_uri` is present, this also downloads subtitle JSON.
- Report CLI/API errors from JSON fields exactly; do not infer hidden causes.

## Common Shapes

```bash
vidu-cli element create --name "角色A" --image /path/a.png --description "..." --style "..."

vidu-cli task submit --type reference2image --prompt "..." --image /path/a.png --duration 0 --model-version 3.2_image_2 --resolution 1080p

vidu-cli task submit --type character2video --prompt "[@角色A] walks into office" --material "角色A:ID:VERSION" --duration 5 --model-version 3.2_a --resolution 1080p

vidu-cli task tts --prompt "旁白文本" --voice-id Chinese_Mandarin_Gentleman --speed 1.3
```

## Load References Only When Needed

- `references/parameters.md`: exact model matrix, durations, flags, examples, validation details.
- `references/compose.md`: required before building compose timeline JSON.
- `references/errors_and_retry.md`: lifecycle states, retry behavior, polling edge cases.
- `references/setup.md`: installation, environment variables, first-run checks, and external-user setup notes.

If local CLI help conflicts with this file, trust `vidu-cli <subcommand> --help` and update this skill afterward.
