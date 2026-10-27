# Vidu Skills

[中文文档](./README.zh.md)

Generate and deliver Vidu videos, images, and speech, and manage tasks, reference assets, creations, and cloned voices through `vidu-cli`.

**Skill 1.5.0 requires vidu-cli v0.2.36 or newer**, aligned with v0.2.36 (`6bb4b07`). It uses the host's available command execution tool; no batch orchestration dependency is required.

## Capabilities

| Feature | Description |
|---|---|
| Text-to-Image / Reference-to-Image | Includes `3.2_image_2_5_fast` and `3.2_image_2_5_pro`; 1080p/2k/4k |
| Text-to-Video | Legacy models, Q3.1 Drama, and Wan 3; up to 30s depending on model |
| Image / Head-Tail-to-Video | Single image or ordered start/end frames, with model-specific support |
| Reference-to-Video | Images/materials and supported audio/video inputs; Q4, Drama, and Wan 3 have distinct limits, including audio-only workflows |
| TTS / Lip-Sync | Official shared voice pool; personal cloned voices can be used for TTS |
| Video Compose | Multi-track video/audio/subtitle/effect timeline export |
| Task Management | History, details, wait/download, cancellation, and history deletion |
| Reuse / Post-Processing | Historical rerun, video extension, image enhancement, and video upscale |
| Reference Elements / Creations | Create/manage references; search/favorite community elements; list/favorite creations |
| Personal Voice Cloning | Create, list, inspect, wait for, and update cloned voices; explicit trial option |
| Quotes / Account | Generation/TTS/lip-sync/upscale credit estimates, pass quota, credit balance, and transactions |

Output audio is controlled by `--with-audio` / `--with-audio=false`, separately from reference `--audio`. Schedule modes are `nopeak` (**off-peak mode / 闲时模式**), `claw_pass` (pass mode), and `normal` (credit mode). Q4 defaults to credits, also accepts off-peak subject to server rules, and rejects pass mode; Drama/Wan 3 only support credits. See the [model and parameter reference](references/parameters.md) before selecting settings. New models do not automatically replace the user's selected model.

## Install as Agent Skill

```bash
# Via npx skills (recommended, supports Claude Code, Cursor, Copilot, and 40+ agents)
npx skills add shengshu-ai/vidu-skills

# Via ClawHub (OpenClaw ecosystem)
clawhub install github:shengshu-ai/vidu-skills
```

## Install vidu-cli

Requires Node.js ≥ 14 and CLI **v0.2.36+**:

```bash
npm install -g vidu-cli@latest
vidu-cli --version
```

If the reported CLI version is older than v0.2.36, upgrade before using this skill. Legacy fallback command flows are not maintained.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `VIDU_TOKEN` | Yes | Vidu API token — obtain it at [vidu.cn](https://www.vidu.cn) (mainland China) or [vidu.com](https://www.vidu.com) (overseas); see [Get your Vidu API Token](#get-your-vidu-api-token) |
| `VIDU_BASE_URL` | No | Default `https://service.vidu.cn` (mainland China); use `https://service.vidu.com` for overseas |
| `VIDU_DEBUG` | No | Set to `1` to print full response body to stderr |

### Get your Vidu API Token

Mainland China — visit [vidu.cn](https://www.vidu.cn), log in, open the account menu from the avatar in the lower-left corner, then click **Vidu Token**:

![Get your Vidu Token on vidu.cn](./assets/vidu-token-cn.png)

Overseas — visit [vidu.com](https://www.vidu.com), log in, open the account menu from the avatar in the left sidebar, then click **Vidu Token**:

![Get your Vidu Token on vidu.com](./assets/vidu-token-global.png)

## Data Usage and Privacy

This skill sends user-provided data to Vidu's servers:

- Text prompts are sent to the Vidu API.
- Local images, videos, and audio files are uploaded to Vidu API servers (`service.vidu.cn` or `service.vidu.com`) when used as task inputs.
- Task parameters such as settings, model version, duration, and resolution are sent to the Vidu API.
- Recordings used for personal voice cloning are uploaded for review and cloning.

Before using this skill, confirm that sending your content to Vidu is acceptable for your privacy and intellectual property requirements. Data handling follows Vidu's official policy.

Security recommendations:

- Create a token with limited scope if possible.
- Avoid using production or privileged tokens for initial testing.
- Review Vidu's terms of service and privacy policy.

Vidu terms and privacy:

- Overseas: https://www.vidu.com/terms
- Mainland China: https://www.vidu.cn/terms

## Quick Start

```bash
# Generate with an established model, wait, and download.
vidu-cli task submit --type text2video --prompt "A cat running through a meadow" \
  --model-version 3.2 --duration 5 --resolution 1080p --wait --output ./output

# Inspect history, then continue using a task ID from the response.
vidu-cli task list --state success --pagesz 20
vidu-cli task get "$TASK_ID" --details
vidu-cli task wait "$TASK_ID" --timeout 1800 --output ./output

# Image-to-video needs both resolution and transition for 3.2.
vidu-cli task submit --type img2video --prompt "Slow camera push-in" \
  --image ./photo.jpg --model-version 3.2 --duration 5 --resolution 1080p \
  --transition pro --wait --output ./output

# Select an exact returned official voice ID; do not invent one.
vidu-cli task tts-voices
# Set OFFICIAL_VOICE_ID to the selected response ID.
vidu-cli task tts --prompt "Hello, welcome to Vidu and our next story." \
  --voice-id "$OFFICIAL_VOICE_ID" --wait --output ./audio
```

`TASK_ID`, voice IDs, and pagination tokens must come from responses. Numeric creation/label/upload IDs in examples are placeholders. Task IDs and creation IDs are not interchangeable. Scripts use the default JSON output; humans can use `vidu-cli --format table task list`.

### New Models and Output Audio

```bash
# Wan 3 quote and generation with the same output audio selection.
# Both use the model defaults: 5s, 720p, 16:9, normal credit mode.
vidu-cli task cost --type text2video --model-version viduq2-drama-std --with-audio
vidu-cli task submit --type text2video --prompt "Waves breaking on a beach" \
  --model-version viduq2-drama-std --with-audio --wait --output ./output

# Drama audio-only reference: local WAV/MP3, 4–30s and at most 15 MiB.
vidu-cli task submit --type character2video --prompt "A dramatic scene with dialogue" \
  --model-version viduq3.1-drama-std --audio ./dialogue.wav --wait --output ./output

# New image model; image tasks do not need --duration.
vidu-cli task submit --type text2image --prompt "A poster with clear lettering" \
  --model-version 3.2_image_2_5_pro --resolution 2k --wait --output ./images
```

Q4 (`3.4`) supports `img2video` (3–16s) and `character2video` (1–16s), with explicit duration/resolution. Drama is `viduq3.1-drama-std` (4–30s); Wan 3 is `viduq2-drama-std` (2–30s). Their input/reference rules differ; do not impose a universal 1–7 image/material limit. The legacy `3.2_a` model also supports 720p.

### Video Upscale and Personal Voices

```bash
# Quote without uploading the local video or creating a task.
vidu-cli task upscale-cost --video ./input.mp4 --resolution 4k
# When upscaling is requested, submit with the same source/settings.
vidu-cli task upscale --video ./input.mp4 --resolution 4k --wait --output ./output

# Local 10–300s recording, at most 20 MiB; cloning is paid by default.
vidu-cli voice create --audio ./voice-sample.wav --name "My narrator" --wait
# Set VOICE_ID from the create/list response; use the ready personal voice.
vidu-cli task tts --prompt "Welcome to the next chapter of our story." \
  --voice-id "$VOICE_ID" --wait --output ./audio
```

Upscale accepts an existing video `--creation-id` or `--video`, not both. Local/creation duration is inferred; a URL/`ssupload` source requires the full `--duration`. Voice trials require explicit `--use-trial` and do not fall back to paid creation. Details: [task workflows](references/task_workflows.md), [personal voices](references/voices.md).

## Command Reference

All commands below use the `vidu-cli` prefix. Creator submissions support `--wait [--timeout SECONDS] [--output PATH]`; timeout/output require wait. Alias support includes `--model`, `--voice`, `--prompt-file`, and `img2image`; prefer canonical names in examples.

| Command | Purpose |
|---|---|
| `vidu-cli upload IMAGE_PATH` | Upload a local image; returns `upload_id`, `ssupload_uri` |
| `vidu-cli task submit --type ... --model-version ...` | Submit image/video generation |
| `vidu-cli task list` | History with repeated state/type/scene filters and cursor pagination |
| `vidu-cli task get TASK_ID [--details] [-o PATH]` | Query once; optionally inspect input/settings/creations or download |
| `vidu-cli task wait TASK_ID` | Wait for completion; optional timeout, poll interval, download |
| `vidu-cli task cancel TASK_ID` | Cancel before processing, subject to server rules |
| `vidu-cli task delete TASK_ID` | Soft-delete eligible task history |
| `vidu-cli task rerun TASK_ID` | Create a new task from historical generation inputs |
| `vidu-cli task extend --creation-id ... --duration ...` | Extend an eligible completed 1080p video by 1–7s |
| `vidu-cli task upscale-image --creation-id ...` | Enhance an image creation, default 2k |
| `vidu-cli task upscale --creation-id ... --resolution ...` | Upscale a video; alternatively use `--video` |
| `vidu-cli task upscale-cost --video ... --resolution ...` | Quote video upscale without uploading or submitting |
| `vidu-cli task compose --timeline ...` | Export a multi-track timeline |
| `vidu-cli task lip-sync --video ... --text ...` | Text-driven lip-sync; alternatively use `--audio` |
| `vidu-cli task tts --prompt ... --voice-id ...` | Speech; alternatively use prompt file or repeated text segments |
| `vidu-cli task tts-voices` | List the official static voice pool |
| `vidu-cli task lip-sync-voices` | List the same official voice pool |
| `vidu-cli task cost --type ... --model-version ...` | Quote image/video generation |
| `vidu-cli task tts-cost --text ... --voice-id ...` | Quote TTS |
| `vidu-cli task lip-sync-cost --duration ...` | Quote lip-sync |
| `vidu-cli element check --name ...` | Check element name availability |
| `vidu-cli element preprocess --name ... --image ...` | Preprocess reference images |
| `vidu-cli element create --name ... --image ...` | Create a reusable element from 1–3 images |
| `vidu-cli element get ELEMENT_ID` | Get the latest visible element |
| `vidu-cli element update ELEMENT_ID ...` | Change name, replace/clear labels, or recaption |
| `vidu-cli element delete ELEMENT_ID` | Delete a personal element |
| `vidu-cli element list` | Personal elements; cursor or numeric-page compatibility |
| `vidu-cli element search --keyword ...` | Community search, optional category/topic filters |
| `vidu-cli element favorites` | Cursor list of favorite community elements |
| `vidu-cli element favorite ELEMENT_ID` | Favorite an element; share ID resolved internally |
| `vidu-cli element unfavorite ELEMENT_ID` | Unfavorite an element |
| `vidu-cli creation list [--favorites-only]` | Personal/favorite creations, numeric pages |
| `vidu-cli creation favorite CREATION_ID` | Favorite an accessible creation |
| `vidu-cli creation unfavorite CREATION_ID` | Unfavorite a creation |
| `vidu-cli voice create --audio ... [--use-trial]` | Clone a personal voice, optional wait |
| `vidu-cli voice list` | Cursor list of personal voices |
| `vidu-cli voice get VOICE_ID` | Read a clone status snapshot |
| `vidu-cli voice update VOICE_ID ...` | Update name/description |
| `vidu-cli voice wait VOICE_ID` | Wait over SSE; no polling interval option |
| `vidu-cli quota pass` | Pass quota; not off-peak quota |
| `vidu-cli quota credit` | Credit balance |
| `vidu-cli quota records` | Credit transactions, zero-based numeric pages |

## Task Delivery and Errors

Terminal task states are `success`, `failed`, `canceled`; other states continue waiting. A successful `task get` can report a failed task, so inspect `state` separately from `ok`. Waiting progress goes to stderr and final runtime JSON to stdout; argument parsing errors are text on stderr with exit code 2.

Use a directory for multiple artifacts or TTS audio plus subtitle JSON (`{task_id}_subtitle.json` when available). A single artifact can use a file path. Multi-segment TTS requires `--subtitle-enable false`; prompt/file or a single text segment supports subtitles by default.

A wait timeout/interruption does not cancel the server task or clone. Continue with the original task/voice ID. Do not blindly resubmit ambiguous writes, switch rejected off-peak/pass requests to credits, or turn a failed trial into a paid clone. See [errors and retry](references/errors_and_retry.md).

## Reference Docs

| File | Contents |
|---|---|
| [Setup](references/setup.md) | Install, version requirement, credentials, environment |
| [Parameters](references/parameters.md) | Models, media limits, generation/TTS/lip-sync flags, modes, quotes, quota |
| [Task workflows](references/task_workflows.md) | History, delivery, rerun, extension, image/video upscale |
| [Assets](references/assets.md) | Elements, community favorites, creations, IDs and pagination |
| [Personal voices](references/voices.md) | Clone, inspect, wait, update, trial, and TTS use |
| [Compose](references/compose.md) | Timeline JSON schema and export |
| [Errors and retry](references/errors_and_retry.md) | Output channels, states, error context, safe recovery |

## Links

- Homepage (mainland China): https://www.vidu.cn/
- Homepage (overseas): https://www.vidu.com/
- npm: https://www.npmjs.com/package/vidu-cli
