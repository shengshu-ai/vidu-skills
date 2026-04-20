# Vidu Skills

Generate AI videos and images via the official Vidu API using `vidu-cli`.

## Capabilities

| Feature | Description |
|---------|-------------|
| Text-to-Image | Generate images from text, up to 1080p/2K/4K |
| Text-to-Video | Generate videos from text, up to 16s |
| Image-to-Video | One image + text → video |
| Head-Tail-to-Video | Start frame + end frame + text → video |
| Reference-to-Image | 1–7 images/materials + text → image |
| Reference-to-Video | 1–7 images/materials + text → video |
| Lip Sync | Drive video mouth movement with TTS or audio file |
| Text-to-Speech | Convert text to speech, 300+ voices, 20+ languages |
| Video Compose | Multi-track timeline (video/audio/subtitle/effect) → exported video |
| Create References | Create personal reference material elements |

## Installation

Requires Node.js ≥ 14:

```bash
npm install -g vidu-cli@latest
```

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `VIDU_TOKEN` | Yes | Vidu API token |
| `VIDU_BASE_URL` | No | Default `https://service.vidu.cn` (mainland China); use `https://service.vidu.com` for overseas |
| `VIDU_DEBUG` | No | Set to `1` to print full response body to stderr |

## Quick Start

```bash
# Text-to-video
vidu-cli task submit \
  --type text2video \
  --prompt "A cat running through a meadow" \
  --model-version 3.2 \
  --duration 5 \
  --resolution 1080p

# Poll until success/failed
vidu-cli task get <task_id>

# Download result
vidu-cli task get <task_id> --output ./output
```

```bash
# Image-to-video
vidu-cli task submit \
  --type img2video \
  --prompt "slow camera push-in" \
  --image ./photo.jpg \
  --model-version 3.2 \
  --duration 5

# Text-to-speech
vidu-cli task tts \
  --prompt "Hello, welcome to Vidu" \
  --voice-id en_female_1

# List available voices
vidu-cli task tts-voices
```

## Command Reference

| Command | Purpose |
|---------|---------|
| `vidu-cli upload <image>` | Upload image, returns `upload_id` |
| `vidu-cli task submit --type ... --prompt ...` | Submit task, returns `task_id` |
| `vidu-cli task get <task_id> [--output dir]` | Query task status, optionally download result |
| `vidu-cli task compose --timeline <json>` | Video compose, returns `task_id` |
| `vidu-cli task lip-sync --video ... --text ...` | Lip sync (TTS mode) |
| `vidu-cli task lip-sync --video ... --audio ...` | Lip sync (audio file mode) |
| `vidu-cli task tts --prompt ... --voice-id ...` | Text-to-speech |
| `vidu-cli element create --name ... --image ...` | Create reference element |
| `vidu-cli element list` | List personal elements |
| `vidu-cli element search --keyword ...` | Search community elements |

## Task Lifecycle

`created` → `queueing` → `preparation` → `scheduling` → `processing` → **`success`** / **`failed`** / `canceled`

Terminal states are `success`, `failed`, and `canceled`. Keep polling for all other states.

## Reference Docs

| File | Contents |
|------|----------|
| `references/parameters.md` | Task matrix, CLI flags, examples, prompt tips |
| `references/errors_and_retry.md` | Task states, retry strategy |
| `references/compose.md` | Video compose timeline JSON schema |

## Links

- Homepage: https://www.vidu.cn/
- npm: https://www.npmjs.com/package/vidu-cli
