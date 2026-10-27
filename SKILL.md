---
name: vidu-skills
description: Generate and deliver Vidu videos, images, and speech through vidu-cli; manage generation tasks, reference assets, creations, personal cloned voices, and credit quotes. Use for Vidu generation, lip-sync, timeline composition, video extension or upscaling, and related Vidu account workflows. Requires VIDU_TOKEN and vidu-cli v0.2.36 or newer.
version: 1.5.0
homepage: https://www.vidu.cn/
primaryEnv: VIDU_TOKEN
metadata: {"openclaw":{"requires":{"bins":["node","npm","vidu-cli"],"env":["VIDU_TOKEN"]},"primaryEnv":"VIDU_TOKEN","install":[{"id":"vidu-cli","kind":"node","package":"vidu-cli","bins":["vidu-cli"],"label":"Install vidu-cli via npm (requires Node.js >=14; postinstall downloads a platform binary from GitHub)"}]}}
---

# Vidu Generation and Asset Workflows

Use `vidu-cli` **v0.2.36 or newer**. This skill is aligned with v0.2.36 (`6bb4b07`). Check `vidu-cli --version` on first use; ask users of older versions to upgrade rather than inventing compatibility fallbacks.

## Execution Model

Call the CLI with the host's command execution tool. Prefer argv arrays when supported; if the host accepts shell commands, quote arguments correctly and use prompt/timeline files for complex content. No batch orchestration tool is required. Use CLI flags, not raw API request bodies; compose timeline JSON is an input supported by the CLI.

Keep the default JSON output for agents. `--format table` is for human inspection. Parse exit status, stdout, and stderr separately: argument errors are text on stderr; runtime failures normally return `ok: false` JSON. See [errors and retry](references/errors_and_retry.md).

Environment: `VIDU_TOKEN` is required. `VIDU_BASE_URL` defaults to `https://service.vidu.cn`; overseas accounts use `https://service.vidu.com`. Optional `VIDU_DEBUG=1` prints response bodies to stderr. Read [setup](references/setup.md) for installation or credential problems.

## Choose a Workflow

| User intent | Command / reference |
|---|---|
| Generate images or videos | `task submit --type ... --model-version ...`; read [parameters](references/parameters.md) for the task/model matrix, reference limits, and defaults. |
| Generate speech or lip-sync | `task tts` or `task lip-sync`; read [parameters](references/parameters.md). Select official voice IDs from `task tts-voices` / `task lip-sync-voices`. |
| Compose video/audio/subtitle/effect tracks | `task compose --timeline ...`; read [compose](references/compose.md) before building a timeline. |
| Find, inspect, wait for, or download a task | `task list`, `task get --details`, `task wait`; read [task workflows](references/task_workflows.md). |
| Cancel/delete a task, rerun, extend, or enhance a creation | `task cancel/delete/rerun/extend/upscale-image/upscale`; read [task workflows](references/task_workflows.md). |
| Create/manage reference elements or browse/favorite creations | `element ...`, `creation ...`; read [assets](references/assets.md). |
| Clone/manage a personal voice and use it for TTS | `voice create/list/get/update/wait`; read [voices](references/voices.md). |
| Check quota, credits, transactions, or a quote | `quota pass/credit/records`, `task cost/tts-cost/lip-sync-cost`; read [parameters](references/parameters.md). Video upscale quotes use `task upscale-cost`, documented in [task workflows](references/task_workflows.md). |
| Upload a reusable image | `upload <local_image_path>` returns `upload_id` and `ssupload_uri`; image input flags can also upload automatically. |

## Core Rules

- Respect the user's selected model. Do not switch defaults solely because a newer model is available. Read the model matrix before adding model-specific settings.
- Repeat `--image`, `--material`, `--audio`, and `--video` per input. Each material is one `name:id:version` argument; never comma-join materials.
- Input requirements are model-specific. Q4 supports audio-only references; Drama and Wan 3 accept multimodal references with different limits. Do not apply the legacy image/material count to every reference task.
- `--audio` supplies reference audio; `--with-audio` enables output audio and `--with-audio=false` disables it on supported video models. Omission preserves existing behavior. Quote and submit with the same audio selection.
- Schedule modes: `nopeak` = **off-peak mode (闲时模式)**, `claw_pass` = **pass mode (通行证模式)**, `normal` = **credit mode (积分模式)**. Availability and off-peak eligibility/quota depend on the model and server. Do not automatically change an explicitly selected mode after rejection.
- Image tasks need no duration; `element create` has no duration option. Drama/Wan 3 have video defaults; older submit models still require explicit duration/resolution as applicable.
- `--prompt-file` aliases `--prompt-path`; `--model` aliases `--model-version`; `--voice` aliases `--voice-id` where supported. Prefer full names in examples.
- Task IDs, numeric creation IDs, element IDs/versions, and personal `voice_...` IDs come from responses and are not interchangeable.
- Generation returns a `task_id` by default. When the user needs a finished artifact, prefer `--wait --output ./results` for a single task. For a batch, submit each task, save all IDs, then wait individually.
- On task submissions, `--timeout` and `--output` require `--wait`. Use a directory for multiple artifacts or audio plus subtitles; a file path is allowed for a single artifact.
- Waiting timeout/interruption does not cancel the server task. Continue with the original ID. Never blindly resubmit after an ambiguous submission failure, or change a failed voice trial into a paid clone.

## Common Shapes

```bash
# The user's chosen model; legacy required settings remain explicit.
vidu-cli task submit --type text2video --prompt "A cat walking through rain" \
  --model-version 3.2 --duration 5 --resolution 1080p --wait --output ./results

# Explicit request for a Wan 3 video with generated audio.
vidu-cli task submit --type text2video --prompt "Waves breaking on a beach" \
  --model-version viduq2-drama-std --with-audio --wait --output ./results

# Resume an existing task; TASK_ID must come from a response.
vidu-cli task wait "$TASK_ID" --timeout 1800 --output ./results
```

There is no `task download`, top-level `reference`, `voice delete`, or `task submit --type tts` command. Use only the supported workflows above. If the installed CLI differs, inspect the relevant `--help`; do not guess flags or silently change the user's model or billing choice.
