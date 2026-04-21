# Vidu Task Parameters Reference

Daily use: **`vidu-cli`** flags and the sections below.

---

## Task Support Matrix

| Task Type | CLI Command | Model Version | Duration | Resolution | Aspect Ratio | Transition | Images |
|-----------|----------|---------------|----------|------------|--------------|------------|--------|
| text2image | `text2image` | 3.1, 3.2_fast_m, 3.2_pro_m | 0 | 1080p, 2k, 4k | 4:3, 3:4, 1:1, 9:16, 16:9 | N/A | 0 |
| text2video | `text2video` | 3.0, 3.1, 3.2 | 3.0: 5s; 3.1: 2–8s; 3.2: 1–16s | 1080p | 16:9, 9:16, 1:1, 4:3, 3:4 | 3.2: pro/speed | 0 |
| img2video | `img2video` | 3.0, 3.1, 3.2 | 3.0: 5s; 3.1: 2–8s; 3.2: 1–16s | 1080p | from image (do not pass) | 3.0: creative/stable; 3.1+: pro/speed | exactly 1 |
| headtailimg2video | `headtailimg2video` | 3.0, 3.1, 3.2 | 3.0: 5s; 3.1: 2–8s; 3.2: 1–16s | 1080p | N/A | 3.0: creative/stable; 3.1+: pro/speed | exactly 2 |
| reference2image | `reference2image` | 3.1, 3.2_fast_m, 3.2_pro_m | 0 | 1080p, 2k, 4k | 4:3, 3:4, 1:1, 9:16, 16:9 | N/A | images + materials: 1–7 |
| character2video | `character2video` | 3.0, 3.1, 3.1_pro, 3.2 | 3.0: 5s; 3.1: 2–8s; 3.1_pro: -1/2–8s; 3.2: 1–16s | 1080p | 16:9, 9:16, 1:1, 4:3, 3:4 | 3.2: **pro/speed (required)** | images + materials: 1–7 |
| lip_sync | `task lip-sync` | N/A | auto (from text/audio) | 1080p | from video | N/A | 1 video + (text OR audio) |
| tts | `task tts` | N/A | auto (from text) | N/A | N/A | N/A | text + voice-id |

**Capability notes**

- **text-to-image**: Text only.
- **text-to-video**: Text only.
- **image-to-video**: One image + text; aspect ratio comes from the image.
- **head-tail-image-to-video**: Two images (start, end) + text.
- **reference2image** and **character2video** — same input rule: **image count + material count must be ≥ 1 and ≤ 7** (each `--image` and each `--material` counts toward the total). **Text prompt (`--prompt`) is required** for both (cannot omit or leave empty).
- **lip-sync**: Drive video mouth movement with text-to-speech or audio file. Two modes: **text mode** (TTS with voice selection) or **audio mode** (custom audio file). Video: MP4/MOV/AVI, ≤500MB. Audio: MP3/WAV/AAC/M4A, ≤100MB.
- **TTS**: Convert text to speech audio. Uses `task tts` command (not `task submit`). Requires `--prompt` and `--voice-id`. List voices with `task tts-voices`.
- **When is `element create` needed?** `element create` saves a reference for **future reuse** across multiple tasks. It is **not required** for one-off generation — just pass `--image` directly. Use `--material` (with `[@name]` in prompt) only when referencing a **previously saved** or **community** element. You may combine `--image` and `--material` as long as the total stays in 1–7.
- **Create References**: `vidu-cli element create --name ... --image ...` runs check → preprocess → create; returns element `id` and `version`.
- **List personal references**: `vidu-cli element list [--keyword kw]`.
- **Search community references**: `vidu-cli element search --keyword "..."`.

---

## CLI commands (overview)

### Upload image

```bash
vidu-cli upload <image_path>
```

- Detects dimensions; compresses if larger than 10MB.
- Returns: `upload_id`, `ssupload_uri`.

### Submit task

```bash
vidu-cli task submit \
  --type <task_type> \
  --prompt "text prompt" \
  [--image <path|url|ssupload_uri>] \
  [--material "name:id:version"] \
  --duration <seconds> \
  --model-version <version> \
  [--aspect-ratio <ratio>] \
  [--transition <mode>] \
  [--resolution <res>] \
  [--sample-count <n>] \
  [--codec <codec>] \
  [--movement-amplitude <amp>] \
  [--schedule-mode <mode>]
```

`--image` and `--material` may be repeated where applicable.

### Query task

```bash
vidu-cli task get <task_id> [--output/-o <dir>]
```

Returns: `task_id`, `state`, `type`, `model`; on failed: `err_code`, `err_msg`. **Note**: `ok` may be `true` even when `state` is `failed` — always check `state` to determine success.

`--output <dir>` (optional): when state is `success`, downloads all media files to `<dir>` and returns `downloaded_files` (list of local paths). If state is not `success`, returns `download_skipped: "task not ready"`.

### Search community references

```bash
vidu-cli element search --keyword "keyword" [--pagesz 20]
```

### Query claw-pass status

```bash
vidu-cli quota pass
```

Returns: `has_pass` (bool), `daily_quota_seconds`, `used_seconds`, `remain_seconds`, `package_id`, `tier`.

| Field | Type | Description |
|-------|------|-------------|
| `has_pass` | boolean | Whether the user has an active claw-pass |
| `package_id` | string | Package identifier |
| `tier` | string | Subscription tier |
| `daily_quota_seconds` | integer | Total daily quota in seconds |
| `used_seconds` | integer | Seconds consumed today |
| `remain_seconds` | integer | Remaining seconds today |
| `cycle_start_at` | datetime | Billing cycle start time |
| `cycle_end_at` | datetime | Billing cycle end time |
| `next_refresh_at` | datetime | Next quota refresh time |
| `refresh_timezone` | string | Refresh timezone (e.g. "Asia/Shanghai") |

### Query credit balance

```bash
vidu-cli quota credit
```

| Field | Type | Description |
|-------|------|-------------|
| `credits` | integer | Current available credits |
| `credits_expire_today` | integer | Credits expiring today |
| `credits_expire_monthly` | integer | Credits expiring this month |
| `credits_permanent` | integer | Permanent (non-expiring) credits |
| `concurrency` | integer | Concurrent task limit |
| `credits_free` | integer | Free credits |
| `credits_subscribed` | integer | Credits from subscription |
| `credits_purchased` | integer | Purchased credits |
| `credit_sub_expires_at` | string (datetime) | Subscription expiration time |

---

## Task types (`--type`)

| Value | Meaning |
|-------|---------|
| `text2image` | Text to image |
| `text2video` | Text to video |
| `img2video` | Image to video |
| `headtailimg2video` | Head and tail frames to video |
| `reference2image` | Reference to image |
| `character2video` | Reference to video |

**Note**: `lip_sync` uses a different command: `vidu-cli task lip-sync` (not `task submit --type lip_sync`).

---

## CLI settings (`task submit`)

### `--model-version` (required)

| Value | Maps to | Notes |
|-------|---------|-------|
| `3.0` | Q1 | |
| `3.1` | Q2 | text2image, reference2image, 2–8s video models |
| `3.2` | Q3 | 1–16s video models |
| `3.1_pro` | Q2 pro | `character2video` only |
| `3.2_fast_m` | Q3 fast | text2image / reference2image only (2k/4k) |
| `3.2_pro_m` | Q3 pro | text2image / reference2image only (2k/4k) |

### `--duration`

- **text2image**, **reference2image**: `0` (required — these are image tasks with no duration; passing any other value causes a validation error)
- **text2video**, **img2video**, **headtailimg2video**, **character2video**: valid ranges depend on `model_version` (see matrix above; e.g. 3.1 often 2–8s, 3.2 often 1–16s)

### `--resolution` (optional; default 1080p where applicable)

- **text2image**: `1080p` (3.1); `2k` / `4k` with 3.2_fast_m / 3.2_pro_m
- **reference2image**: `1080p`, `2k`, `4k`
- **Video tasks**: `1080p` only

### `--aspect-ratio` (optional; task-dependent)

- **text2image**, **reference2image**: `4:3`, `3:4`, `1:1`, `9:16`, `16:9`
- **text2video**, **character2video**: `16:9`, `9:16`, `1:1`, `4:3`, `3:4`
- **img2video**: do not pass (derived from image)
- **headtailimg2video**: do not pass

### `--transition` (optional; video)

- **text2video** (3.2 only): `pro`, `speed`
- **text2video** (3.1): do not pass
- **img2video**, **headtailimg2video**: `pro`, `speed` (3.0: creative/stable per matrix)
- **character2video** (3.2): `pro`, `speed` (**required** for model version 3.2; omitting causes a validation error)
- **character2video** (3.0, 3.1, 3.1_pro): do not pass
- **reference2image**, **text2image**: do not pass

### Other flags (when supported by CLI)

- `sample_count` / `--sample-count`: default 1
- `schedule_mode` / `--schedule-mode`: `claw_pass` (daily quota) or `normal` (credits). Optional — if omitted, auto-detected by querying claw-pass status: uses `claw_pass` when user has an active pass, otherwise `normal`.
- `codec` / `--codec`: default `h265`
- `use_trial`: if exposed by CLI
- `movement_amplitude` / `--movement-amplitude`: e.g. `auto`

---

## CLI vs raw JSON (background)

Use **`vidu-cli` flags only** — do not hand-craft request bodies or invent extra parameters.

| CLI | Role |
|-----|------|
| `--prompt` | Text prompt (respect length limits, e.g. up to 4096 chars) |
| `--image` | Images (paths, URLs, or `ssupload` URIs after upload) |
| `--material` | Reference material ids |
| `input.enhance` / recaption | **No CLI flag for task submit** — handled internally by `vidu-cli` (do not invent a flag). For `task cost`, use `--enhance` (default true) to estimate with/without enhance. |

---

## CLI examples

### 1. text-to-video

```bash
vidu-cli task submit \
  --type text2video \
  --prompt "A cat walks in the snow at sunset" \
  --duration 5 \
  --model-version 3.2 \
  --aspect-ratio 16:9 \
  --transition pro \
  --resolution 1080p
```

Response: `{"ok": true, "task_id": "...", "trace_id": "..."}`

### 2. text-to-image

```bash
vidu-cli task submit \
  --type text2image \
  --prompt "A beautiful sunset over the ocean" \
  --duration 0 \
  --model-version 3.1 \
  --resolution 2k
```

### 3. image-to-video

```bash
vidu-cli task submit \
  --type img2video \
  --prompt "The cat starts running" \
  --image /path/to/image.jpg \
  --duration 5 \
  --model-version 3.2 \
  --resolution 1080p
```

`--image` accepts local path, URL, or `ssupload:?id=...`.

### 4. head-tail-image-to-video

```bash
vidu-cli task submit \
  --type headtailimg2video \
  --prompt "Smooth transition between scenes" \
  --image start.jpg \
  --image end.jpg \
  --duration 5 \
  --model-version 3.2 \
  --resolution 1080p
```

### 5. reference-to-video

**With a saved reference (subject)** — `[@name]` matches `--material` (count toward the 1–7 total):

```bash
vidu-cli task submit \
  --type character2video \
  --prompt "[@aliya] walks in the garden" \
  --material "aliya:3073530415201165:1765430214" \
  --duration 5 \
  --model-version 3.2 \
  --aspect-ratio 16:9 \
  --transition pro \
  --resolution 1080p
```

**Reference + extra image(s)** — mixed `--material` and `--image` (repeat either flag; combined count ≤ 7):

```bash
vidu-cli task submit \
  --type character2video \
  --prompt "[@aliya] walks in the garden" \
  --material "aliya:3073530415201165:1765430214" \
  --image /path/to/auxiliary.jpg \
  --duration 5 \
  --model-version 3.2 \
  --aspect-ratio 16:9 \
  --transition speed \
  --resolution 1080p
```

**Images only (no subject / no `element create`)** — text + `--image`; repeat `--image` for multiple images (total with any materials ≤ 7):

```bash
vidu-cli task submit \
  --type character2video \
  --prompt "The character turns and walks toward the camera" \
  --image /path/to/ref_sheet.jpg \
  --duration 5 \
  --model-version 3.2 \
  --aspect-ratio 16:9 \
  --transition pro \
  --resolution 1080p
```

### 6. reference-to-image

Same limits as **character2video**: **images + materials between 1 and 7**, **non-empty `--prompt` required**.

**With a saved reference:**

```bash
vidu-cli task submit \
  --type reference2image \
  --prompt "[@aliya] portrait in watercolor style" \
  --material "aliya:3073530415201165:1765430214" \
  --duration 0 \
  --model-version 3.1 \
  --aspect-ratio 16:9 \
  --resolution 2k
```

**Reference + image(s)** — optional extra `--image` lines (combined count ≤ 7):

```bash
vidu-cli task submit \
  --type reference2image \
  --prompt "[@aliya] portrait in watercolor style" \
  --material "aliya:3073530415201165:1765430214" \
  --image /path/to/auxiliary.jpg \
  --duration 0 \
  --model-version 3.1 \
  --aspect-ratio 16:9 \
  --resolution 2k
```

**Images only (no subject)** — still must pass `--prompt`:

```bash
vidu-cli task submit \
  --type reference2image \
  --prompt "Portrait in watercolor style, soft lighting" \
  --image /path/to/ref.jpg \
  --duration 0 \
  --model-version 3.1 \
  --aspect-ratio 16:9 \
  --resolution 2k
```

### 7. lip-sync

**Text mode (TTS with voice selection)**:

```bash
vidu-cli task lip-sync \
  --video /path/to/video.mp4 \
  --text "Hello, welcome to Vidu!" \
  --voice-id English_Aussie_Bloke \
  --speed 1.0 \
  --volume 1.0 \
  --codec h265
```

**Audio mode (custom audio file)**:

```bash
vidu-cli task lip-sync \
  --video /path/to/video.mp4 \
  --audio /path/to/audio.mp3 \
  --codec h265
```

**Constraints**:
- Video: MP4/MOV/AVI, ≤500MB
- Audio: MP3/WAV/AAC/M4A, ≤100MB
- Text: Chinese 2–1000 chars, English 4–2000 chars
- `--text` and `--audio` are mutually exclusive (use one or the other)
- `--speed`: 0.5–2.0 (default 1.0, text mode only)
- `--volume`: 0.1–2.0, or 0 for server default (text mode only)
- `--voice-id`: default `English_Aussie_Bloke` (90+ voices available, see voice list below). Voice IDs from `lip-sync-voices` only; do not use `tts-voices` IDs here — using wrong pool causes a validation error.
- `--schedule-mode`: `claw_pass` (use daily quota) or `normal` (use credits). Optional — auto-detected from claw-pass status if omitted.
- Duration is auto-calculated from text length or audio file

**Available voice IDs** (partial list, 90+ total):
- English: `English_Aussie_Bloke`, `English_Trustworthy_Man`, `English_Graceful_Lady`, `English_Whispering_girl`, `English_Diligent_Man`, `English_Gentle-voiced_man`
- Chinese: `male-qn-qingse`, `male-qn-jingying`, `male-qn-badao`, `male-qn-daxuesheng`, `female-shaonv`, `female-yujie`, `female-chengshu`, `female-tianmei`
- Premium: `male-qn-qingse-jingpin`, `female-shaonv-jingpin`, etc.
- Cartoon: `clever_boy`, `cute_boy`, `lovely_girl`, `cartoon_pig`
- Cantonese: `Cantonese_ProfessionalHost（F)`, `Cantonese_GentleLady`, `Cantonese_ProfessionalHost（M)`, `Cantonese_PlayfulMan`

To get the complete list of all 90+ voice IDs:

```bash
vidu-cli task lip-sync-voices
```

Returns: `{"ok": true, "count": 90+, "voice_ids": [...]}`

### 8. TTS (Text-to-Speech)

```bash
vidu-cli task tts \
  --prompt "text" \
  --voice-id "Chinese (Mandarin)_Reliable_Executive" \
  --speed 1.0 \
  --volume 80 \
  --emotion "happy" \
  --language-boost "Chinese"
```

| Parameter | Required | Default | Range | Description |
|-----------|----------|---------|-------|-------------|
| `--prompt` | Yes | - | 1-2000 chars | Text to convert to speech |
| `--voice-id` | Yes | - | See tts-voices | Voice ID. Voice IDs from `tts-voices` only; do not use `lip-sync-voices` IDs here — using wrong pool causes a validation error. |
| `--speed` | No | 1.0 | 0.5-2.0 | Speed multiplier (values outside range cause validation error) |
| `--volume` | No | 80 | 0-100 | Volume level (values outside range cause validation error) |
| `--emotion` | No | - | Any text | Emotion hint |
| `--language-boost` | No | - | Chinese, English, auto | Enhance specific language recognition |
| `--schedule-mode` | No | auto | claw_pass, normal | Schedule mode: `claw_pass` (use daily quota) or `normal` (use credits). Auto-detected from claw-pass status if omitted. |

List available voices: `vidu-cli task tts-voices` (grouped by language with count)

Returns task_id — query result with `task get <task_id>`.

### 9. Query task credit cost

```bash
# Video generation cost
vidu-cli task cost \
  --type text2video \
  --model-version 3.2 \
  --duration 5 \
  --resolution 1080p

# Lip-sync cost
vidu-cli task cost \
  --type lip_sync \
  --model-version 3.2 \
  --duration 5

# TTS cost
vidu-cli task cost \
  --type tts \
  --model-version 3.2 \
  --duration 10
```

Supports all task types including video generation, lip-sync, and TTS. Uses the same parameters as `task submit` (minus prompt/images/materials). Query before submitting to check cost and eligibility.

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `--type` | Yes | - | Task type |
| `--model-version` | Yes | - | Model version |
| `--duration` | Yes | - | Duration in seconds |
| `--resolution` | No | 1080p | Resolution |
| `--aspect-ratio` | No | - | Aspect ratio |
| `--transition` | No | - | Transition style |
| `--sample-count` | No | 1 | Sample count |
| `--codec` | No | h265 | Codec |
| `--enhance` | No | true | Whether to enable enhance/recaption |
| `--schedule-mode` | No | auto | Schedule mode. Auto-detected from claw-pass status if omitted. |

**Response fields:**

| Field | Type | Description |
|-------|------|-------------|
| `cost_credits` | integer | Credits this task will cost |
| `can_submit` | boolean | Whether the user can submit this task |
| `current_credits` | integer | Current available credits |
| `original_cost_credits` | integer | Original price before discount |
| `claw_pass_quota` | object | Claw-pass quota details (when applicable) |

**`claw_pass_quota` fields (when present):**

| Field | Type | Description |
|-------|------|-------------|
| `has_pass` | boolean | Whether the user has an active claw-pass |
| `package_id` | string | Package identifier |
| `tier` | string | Subscription tier |
| `daily_quota_seconds` | integer | Total daily quota in seconds |
| `used_seconds` | integer | Seconds consumed today |
| `remain_seconds` | integer | Remaining seconds today |
| `cycle_start_at` | datetime | Billing cycle start time |
| `cycle_end_at` | datetime | Billing cycle end time |
| `next_refresh_at` | datetime | Next quota refresh time |
| `refresh_timezone` | string | Refresh timezone (e.g. "Asia/Shanghai") |

### 10. Query task result

```bash
vidu-cli task get "$TASK_ID"
```

- `state`: `success` | `failed` | `processing` | ...
- Response always includes: `task_id`, `state`, `type`, `model`
- **failed**: `err_code`, `err_msg` — note `ok` may still be `true` with `state: failed`
- **processing**: poll again later

Download media on success:

```bash
vidu-cli task get "$TASK_ID" --output ./downloads
```

- Downloads all media files to `./downloads/` as `{task_id}_{index}.mp4` (index is 0-based: `{task_id}_0.mp4`, `{task_id}_1.mp4`, etc.)
- Returns `downloaded_files` (list of local paths)
- If not yet `success`: returns `download_skipped: "task not ready"`

### 11. Image upload (optional)

```bash
vidu-cli upload /path/to/image.jpg
```

Returns: `upload_id`, `ssupload_uri`. Usually unnecessary — `task submit --image` and `element create --image` accept paths and URLs directly.

---

## Material elements (references)

### Create reference element

One command performs name check → preprocess (AI description/style) → create:

```bash
vidu-cli element create \
  --name "my_character" \
  --image image1.jpg \
  --image image2.jpg
```

With custom description and style:

```bash
vidu-cli element create \
  --name "my_character" \
  --image image1.jpg \
  --description "A young woman with long black hair" \
  --style "realistic"
```

**Constraints**

- 1–3 images required (fewer than 1 or more than 3 causes a validation error)
- `--image`: local path, URL, or `ssupload` URI
- `--description`: optional, 1–1280 chars (AI default if omitted)
- `--style`: optional, max 64 chars (AI default if omitted)
- Name must be unique (checked automatically)

Returns: `id`, `version` (for `--material` / `[@name]` usage).

### List personal elements

```bash
vidu-cli element list --keyword "keyword"
```

Example shape: `elements: [{ id, version, name, ... }], next_page_token`.

### Search community elements

```bash
vidu-cli element search --keyword "tiger" --pagesz 20
```

Present results with `id`, `version`, `name`, `description`, `category` when available.

---

## Prompt tips

- **text-to-image**: Subject, style, lighting, composition.
- **text-to-video**: Scene + action; optional camera language (e.g. slow pan left, close-up tracking shot).
- **image-to-video**: Describe motion or change, not only static description.
- **head-tail-image-to-video**: Similar frames → smoother transition; very different frames → stronger morph.
- **reference-to-image / reference-to-video**: **reference2image** and **character2video** both require a **non-empty text prompt**. **Image count + material count** must be **≥ 1 and ≤ 7**. You may use images only, materials only, or a mix; without `element create` when using images only. With a saved or community reference, use `[@reference_name]` and matching `--material`.
- **lip-sync**: Choose text mode for TTS with voice selection, or audio mode for custom audio. Text mode auto-calculates duration from text length (Chinese: ~5 chars/sec, English: ~10 chars/sec). Audio mode reads duration from audio file metadata. Video must contain visible face/mouth for best results.
- **TTS**: Use `tts-voices` to find the right voice ID. `--language-boost` helps when mixing languages. `--emotion` provides a hint for expressive speech.

---

## CLI stdout errors (canonical)

All commands return one JSON line on stdout.

**Success (submit)**

```json
{"ok": true, "task_id": "...", "trace_id": "..."}
```

**Failure (CLI / transport / HTTP)**

```json
{"ok": false, "error": {"type": "client_error|http_error|network_error|parse_error", "message": "...", "http_status": 422, "code": "invalid_param"}}
```

**Never guess** — report `error` fields exactly. Retry behavior: **errors_and_retry.md**.

---

## Validation rules

- `type` must match a supported task.
- `model_version` must be allowed for that task (see matrix).
- `duration` must fit model + task.
- `resolution` must be supported for the task type.
- Do not pass `aspect_ratio` for **img2video** / **headtailimg2video** when disallowed.
- Do not pass `transition` for reference tasks or **text2image**.
- For **reference2image** and **character2video**: **image count + material count** in **1–7** (inclusive); **non-empty text prompt required**. Violating either constraint causes a validation error.
- API-level `input.enhance` / recaption: **no manual CLI field for task submit** — rely on `vidu-cli` defaults. `task cost` accepts `--enhance` flag.
- `--schedule-mode`: valid values are `claw_pass` and `normal`. If omitted, auto-detected by querying claw-pass status.

### `--material` format

The `--material` flag uses the format `name:id:version` where:
- `name`: the element name (matches `[@name]` in prompt)
- `id`: the element ID returned by `element create` or `element list/search`
- `version`: the element version

Example: `--material "aliya:3073530415201165:1765430214"` pairs with `[@aliya]` in the prompt text. The `[@name]` in the prompt is optional — it helps the model associate the reference with specific parts of the prompt, but generation works without it.
