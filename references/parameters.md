# Vidu Generation Parameters

Aligned with `vidu-cli v0.2.36`. Use this reference for `task submit`, TTS, lip-sync, schedule modes, quotes, and quota. For lifecycle, downloads, and post-processing, read [task workflows](task_workflows.md); for reusable elements, read [assets](assets.md).

## Contents

- [Task and model matrix](#task-and-model-matrix)
- [Reference inputs](#reference-inputs)
- [Submit options and examples](#submit-options-and-examples)
- [Schedule modes](#schedule-modes)
- [Text-to-speech and lip-sync](#text-to-speech-and-lip-sync)
- [Quotes and quota](#quotes-and-quota)

## Task and Model Matrix

`--model-version` (alias `--model`) is required. `--type` accepts six canonical types: `text2image`, `reference2image`, `text2video`, `img2video`, `headtailimg2video`, `character2video`. `img2image` is an alias for `reference2image` on submit and cost commands. TTS, lip-sync, compose, extension, and upscaling have their own subcommands.

| Model version | Supported submit types | Output duration | Resolution |
|---|---|---|---|
| `3.1`, `3.2_fast_m`, `3.2_pro_m`, `3.2_image_2`, `3.2_image_2_5_fast`, `3.2_image_2_5_pro` | `text2image`, `reference2image` | Not needed; normalized to 0 | 1080p, 2k, 4k |
| `3.0` | All four video types | 5s | 1080p |
| `3.1` | All four video types | 2–8s | 1080p |
| `3.1_pro` | `character2video` only | -1 (auto) or 2–8s | 1080p |
| `3.2` | All four video types | 1–16s | 1080p |
| `3.2_a` | All four video types | 4–15s, explicit; no -1 | 720p, 1080p |
| `3.4` (Q4) | `img2video` | 3–16s | 540p, 720p, 1080p, 2k, 4k |
| `3.4` (Q4) | `character2video` | 1–16s | 540p, 720p, 1080p, 2k, 4k |
| `viduq3.1-drama-std` (Q3.1 Drama) | `text2video`, `character2video` | 4–30s, default 5s | 480p, **720p default**, 1080p |
| `viduq2-drama-std` (Wan 3) | All four video types | 2–30s, default 5s | 480p, **720p default**, 1080p |

The four video types are `text2video`, `img2video`, `headtailimg2video`, and `character2video`. Q4 does not support text-to-video or head/tail submission; Drama does not support image-to-video or head/tail submission. Use their supported reference workflow instead.

Defaults for **submit**:

- Drama/Wan 3 default to 5 seconds, 720p, 16:9; these options can be omitted.
- Other video models, including Q4, require an explicit duration and resolution. If the user has no resolution preference, keep the established 1080p choice when supported.
- Image models require resolution; duration may be omitted and is normalized to 0 by the CLI.

### Aspect Ratio and Transition

The common aspect ratios are `16:9`, `9:16`, `1:1`, `4:3`, `3:4`. For legacy `img2video` / `headtailimg2video`, derive the format from the images and omit aspect ratio. Wan 3 additionally accepts `21:9` and `adaptive`, including image and head/tail modes; its default is 16:9.

| Task/model | `--transition` |
|---|---|
| `img2video` / `headtailimg2video` + `3.0` | Optional `creative` / `stable` |
| `img2video` / `headtailimg2video` + `3.1` / `3.2` | **Required** `pro` / `speed` |
| `img2video` / `headtailimg2video` + `3.2_a` | Optional `pro` / `speed` |
| `text2video` + `3.2` | Optional `pro` / `speed` |
| `character2video` + `3.2` | **Required** `pro` / `speed` |
| Other combinations, including Q4, Drama, Wan 3 and images | Omit |

Established model nicknames: Q1 = `3.0`, Q2 = `3.1`, Q2 Pro = `3.1_pro`, Q3 = `3.2`, Q3-A / Omni Video Pro / 全能Video Pro = `3.2_a`, 全能Q3 Fast/Pro = `3.2_fast_m` / `3.2_pro_m`, 全能Image 2 = `3.2_image_2`. Preserve a user's explicit selection; new models do not replace it automatically.

## Reference Inputs

Provide a nonempty text prompt with reference generation. Use one flag per input. Materials use `--material "NAME:ELEMENT_ID:VERSION"`; an optional `[@NAME]` in the prompt identifies that character. Obtain IDs/versions from `element create/get/list/search/favorites`; see [assets](assets.md). Creating an element is useful for reuse, not required for a one-off image input.

| Task/model | Images/materials | Audio | Video |
|---|---|---|---|
| `text2image` / `text2video` | Text only | None | None |
| `img2video` | Exactly 1 image | None | None |
| `headtailimg2video` | Exactly 2 images, head then tail | None | None |
| `reference2image` | 1–7 images + materials total | None | None |
| Legacy `character2video` | Use 1–7 images + materials for image/material reference workflows | `3.2_a` only: max 3 | `3.2_a` only: max 3 |
| Q4 `character2video` | Max 7 images + materials; may omit when audio is supplied | Max 3; audio-only references supported | None |
| Drama `character2video` | Max 9 images; no materials | Max 10 | Max 3 |
| Wan 3 `character2video` | Max 10 images; no materials | Max 5 | Max 5 |

Drama and Wan 3 require at least one image, video, or audio reference. Wan 3 allows at most 20 media inputs total. Do not reuse the legacy image/material minimum for Q4 audio-only, Drama, or Wan 3. Wan 3 image/head-tail modes cannot mix in audio or video references. Drama `text2video` accepts text only; the reference workflow is `character2video`.

### Paths, URLs, and Uploads

- `--image` accepts a local path, HTTP/HTTPS URL (downloaded then uploaded), or existing `ssupload:?id=...`. Local images are auto-uploaded; the shared uploader compresses images over 10 MiB.
- Reference `--audio` and `--video` accept local files or existing `ssupload:?id=...`; **HTTP/HTTPS reference audio/video URLs are rejected**. Video upscale and compose have different URL rules; see their references.
- `vidu-cli upload ./photo.jpg` uploads a local image and returns `upload_id` and `ssupload_uri`. It is not a general video/audio upload command.
- Local reference media is checked before image uploads. Existing uploads pass through; the server checks metadata and full collections, including mixed local/uploaded inputs. Do not claim local validation proves server acceptance.
- v0.2.36 reads and uploads reference video dimensions/duration automatically, including its HEVC track-dimension fallback. No manual metadata flags are needed. If metadata cannot be read, report the CLI's actual corrective suggestion.

### Local Reference Audio Limits

All reference audio files are WAV/MP3, at most 15 MiB each.

| Model | Duration per audio | Total audio duration |
|---|---|---|
| `3.2_a` | 2–15s | ≤15s |
| Q4 | 1–16s | No legacy 15s total limit; max 3 clips |
| Drama, without reference video | 4–30s | ≤30s |
| Drama, with any reference video | 2–30s | ≤30s |
| Wan 3 | 1–15s | ≤15s |

With Drama video references, the server determines the task type and still requires ≥4s audio for non-edit tasks. The CLI exposes no explicit Drama edit/extend selector. Do not infer acceptance of 2–4s audio merely from local preflight passing.

### Local Reference Video Limits

| Model | File / dimensions | Duration / frame rate |
|---|---|---|
| `3.2_a` | MP4/MOV, ≤50 MiB; each side 300–60000px, ratio 0.4–2.5, total pixels 409600–2086876 | Total ≤15s |
| Drama | Same format, size, and geometry as `3.2_a` | Each 4–30s, total ≤30s; average 24–60 fps |
| Wan 3 | MP4/MOV, ≤100 MiB; sides 240–4096px, ratio 1:8–8:1 | Each 1–15s, total ≤15s; average ≥16 fps; input video total + output duration ≤30s |

Wan 3 local images must be JPEG/PNG/BMP/WEBP, nonempty and ≤20 MiB, with sides 240–8000px and ratio 1:8–8:1. Limits apply to source files even when the shared uploader later compresses them. Existing `ssupload` metadata is validated by the server.

## Submit Options and Examples

| Option | Behavior |
|---|---|
| `--prompt TEXT` | Inline prompt; takes precedence over a supplied prompt file. |
| `--prompt-path PATH` / `--prompt-file PATH` | UTF-8 file ≤1 MiB; one trailing newline is removed. Supported by submit, TTS, and rerun. |
| `--sample-count N` | Default 1; positive integer. Wan 3 supports 1–4. |
| `--codec h264\|h265` | Default h265; submit may fall back to h264 when ffprobe is unavailable. |
| `--movement-amplitude VALUE` | Default `auto`. |
| `--with-audio` | Enable generated output audio on supported video models. |
| `--with-audio=false` | Disable generated output audio. The false value uses `=`. |
| `--schedule-mode MODE` | See the mode table below; `auto` means omit the flag, not a valid value. |
| `--wait [--timeout SECONDS] [--output PATH]` | Wait for a terminal state and optionally download; see [task workflows](task_workflows.md). |

`--with-audio` is exposed on submit, cost, and rerun. `--audio` only supplies reference audio and does not automatically enable output audio. Omitting the output switch preserves existing behavior; actual model capability and price come from the server. Image tasks reject the output audio switch. Rerun preserves the historical audio choice unless overridden; `--with-audio=false` also clears historical lip-sync settings.

```bash
# Existing video model, with its required settings.
vidu-cli task submit --type text2video --prompt "A cat walking in rain" \
  --model-version 3.2 --duration 5 --resolution 1080p --wait --output ./results

# Image-to-video: transition and resolution are explicit for 3.2.
vidu-cli task submit --type img2video --prompt "Slow camera push-in" \
  --image ./scene.jpg --model-version 3.2 --duration 5 --resolution 1080p \
  --transition pro --wait --output ./results

# Head/tail ordering matters.
vidu-cli task submit --type headtailimg2video --prompt "A smooth transition" \
  --image ./start.jpg --image ./end.jpg --model-version 3.2 \
  --duration 5 --resolution 1080p --transition pro --wait --output ./results

# New image models support both text2image and reference2image.
vidu-cli task submit --type text2image --prompt "A poster with clear lettering" \
  --model-version 3.2_image_2_5_pro --resolution 2k --wait --output ./images
vidu-cli task submit --type reference2image --prompt "Keep the subject, change the background" \
  --image ./subject.jpg --model-version 3.2_image_2_5_fast --resolution 1080p

# Q4 audio-only reference: a local 1–16s WAV/MP3 clip.
vidu-cli task submit --type character2video --prompt "A scene inspired by the sound" \
  --model-version 3.4 --audio ./reference.wav --duration 5 --resolution 1080p

# Drama audio-only reference: a local 4–30s WAV/MP3 clip.
vidu-cli task submit --type character2video --prompt "A dramatic scene with dialogue" \
  --model-version viduq3.1-drama-std --audio ./dialogue.wav --wait --output ./results

# Wan 3 text-to-video with generated sound, using its 5s/720p/16:9 defaults.
vidu-cli task submit --type text2video --prompt "Waves breaking on a beach" \
  --model-version viduq2-drama-std --with-audio --wait --output ./results

# Wan 3 multimodal references; input video total + output must fit within 30s.
vidu-cli task submit --type character2video --prompt "Follow the subject, camera and soundtrack" \
  --model-version viduq2-drama-std --image ./subject.jpg \
  --video ./reference.mp4 --audio ./soundtrack.wav --with-audio
```

Use one `--material "NAME:ELEMENT_ID:VERSION"` per element for models that accept materials. Do not comma-join or space-join multiple materials into one value. CLI-managed enhance/recaption fields do not require invented submit flags.

## Schedule Modes

| Value | Meaning | Selection |
|---|---|---|
| `nopeak` | Off-peak mode (**闲时模式**) | Explicit selection; server validates eligibility, remaining quota, and task/model restrictions. |
| `claw_pass` | Pass mode (**通行证模式**) | Uses pass quota. |
| `normal` | Credit mode (**积分模式**) | Uses credits. |

Q4 defaults to `normal`, allows `nopeak` subject to server rules, and rejects `claw_pass`. Drama and Wan 3 default to and only accept `normal`. Other generation models, TTS, lip-sync, and compose auto-detect an active pass when the flag is omitted; otherwise they use `normal`. They do not automatically choose `nopeak`. Video upscale has fixed `normal` billing and no schedule flag.

Do not automatically change the user's mode after a quota or eligibility rejection. `ClawPassExplicitModeRequired` is a submit error: explain the exhausted pass quota and the alternatives; use credit mode only when authorized. `quota pass` reports pass quota, not off-peak quota.

## Text-to-Speech and Lip-Sync

### TTS

Use `task tts`, not `task submit --type tts`. List official voices with `task tts-voices`; `task lip-sync-voices` returns the same official pool. Select an exact returned ID. Personal voices come from `voice list` / `voice create`; read [voices](voices.md).

| Option | Required / default / limits |
|---|---|
| `--prompt`, `--prompt-path` (alias `--prompt-file`), or `--text` | One input mode required. Inline prompt wins over file; prompt/file conflict with `--text`. |
| `--text TEXT` | Repeat for up to 20 segments, each nonempty and ≤2000 characters; single prompt has the same per-segment limit. |
| `--voice-id ID` (alias `--voice`) | Required; use an official returned ID or ready personal `voice_...` ID. |
| `--speed` | Default 1.0; 0.5–2.0. |
| `--volume` | Default 80; integer 0–100. |
| `--emotion TEXT` | Optional; ≤100 characters, free text such as `happy` or `calm`, not a fixed CLI enum. Repeat values pair by position with text segments; do not supply more emotions than segments. |
| `--language-boost VALUE` | Optional; `Chinese`, `English`, `auto`, and other values listed by `task tts --help`. |
| `--subtitle-enable` / `--subtitle-enabled` | Default true; supported for prompt/file or a single `--text`. Multi-segment text requires `--subtitle-enable false`. |
| `--schedule-mode` | Optional; see schedule modes. |
| `--wait`, `--timeout`, `--output` | Standard task submission wait/download options. |

```bash
vidu-cli task tts-voices
# Set OFFICIAL_VOICE_ID to an exact ID returned above.
vidu-cli task tts --prompt-file ./narration.txt --voice-id "$OFFICIAL_VOICE_ID" \
  --speed 1.0 --emotion calm --wait --output ./audio

vidu-cli task tts --text "Welcome to the story." --text "An adventure begins." \
  --emotion calm --emotion happy --voice-id "$OFFICIAL_VOICE_ID" \
  --subtitle-enable false --wait --output ./audio
```

When `subtitle_uri` is available, directory downloads include audio and preprocessed `{task_id}_subtitle.json`. Read text from files with `--prompt-path` / `--prompt-file`; avoid shell command substitution.

### Lip-Sync

Choose exactly one of text mode or audio mode:

```bash
# Set OFFICIAL_VOICE_ID from task lip-sync-voices (or task tts-voices).
vidu-cli task lip-sync --video ./clip.mp4 --text "Hello, welcome to our story!" \
  --voice-id "$OFFICIAL_VOICE_ID" --speed 1.0 --volume 1.0 --wait --output ./results
vidu-cli task lip-sync --video ./clip.mp4 --audio ./speech.mp3 --wait --output ./results
```

- Video: local MP4/MOV/AVI, ≤500 MiB. Audio mode: local MP3/WAV/AAC/M4A, ≤100 MiB.
- Text: Chinese 2–1000 characters or English 4–2000; estimated speech duration must also be at least 2 seconds. Duration is inferred from text or audio metadata, not supplied as a submit duration flag.
- Text mode: `--voice-id` / `--voice` defaults to `English_Aussie_Bloke`; speed 0.5–2.0, volume 0.5–2.0 or 0 for server default (default 0). TTS uses a different volume scale.
- Also accepts `--enhance` (enabled by default), `--codec` (default h265), `--schedule-mode`, and the standard wait/download options. Inspect local help for exact option syntax.

## Quotes and Quota

Use the matching cost command; it queries a quote rather than submitting a task. Preserve the intended model, duration, resolution, aspect ratio, transition, sample count, codec, schedule, and output audio choice wherever exposed. Quote responses do not guarantee a later submission will pass all input checks or have the same available balance.

```bash
# Same settings and audio choice as the Wan 3 generation example.
vidu-cli task cost --type text2video --model-version viduq2-drama-std --with-audio
vidu-cli task cost --type text2video --model-version viduq3.1-drama-std
vidu-cli task cost --type text2video --model-version 3.2 --duration 5 --resolution 1080p
vidu-cli task tts-cost --text "Hello, welcome to our story!" \
  --voice-id "$OFFICIAL_VOICE_ID" --speed 1.0 --volume 80 --pitch 0
vidu-cli task lip-sync-cost --duration 5 --voice-id "$OFFICIAL_VOICE_ID"
vidu-cli quota pass
vidu-cli quota credit
vidu-cli quota records --page 0 --pagesz 20
```

| Command | Options / result |
|---|---|
| `task cost` | Same six types/model aliases as submit; no media input flags. Legacy video duration is required, image duration is ignored; resolution defaults to 1080p for legacy models. Drama/Wan 3 use the same 5s/720p/16:9 defaults as submit. Accepts aspect ratio, transition, sample count (default 1), codec (default h265), schedule mode, and output audio. |
| `task tts-cost` | Requires `--text` and `--voice-id` / `--voice`; speed 1.0, pitch 0, volume 80 by default; optional schedule mode. `--pitch` belongs to this quote command, not `task tts`. |
| `task lip-sync-cost` | Requires `--duration`; voice defaults to `English_Aussie_Bloke`, speed 1.0, volume 0, codec h265; optional schedule mode. |
| `task upscale-cost` | Same sources/resolution/duration rules as video upscale; no upload or task creation. See [task workflows](task_workflows.md). |
| `quota pass` | `has_pass`, `package_id`, `tier`, daily/used/remaining seconds and cycle/refresh timestamps when available. |
| `quota credit` | `credits`, expiry buckets, concurrency, credit source buckets, and subscription expiry when available. |
| `quota records` | Zero-based numeric `--page` (default 0), `--pagesz` 1–100 (default 20); returns `records` and `total`, not a cursor. |

Shared quote fields include `cost_credits`, `original_cost_credits`, `current_credits`, `can_submit`, and `claw_pass_quota` when applicable. Use the server's returned price; do not maintain a local price table. Quote commands do not accept submission `--wait`, `--timeout`, or `--output`.

For JSON envelopes, stderr parsing errors, optional `trace_id`/`suggestion`, terminal task states, and safe retry decisions, read [errors and retry](errors_and_retry.md).
