# Video Compose Reference

Compose a multi-track timeline into a single exported video using `vidu-cli v0.2.36+`. This reference retains the timeline schema; task delivery is described in [task workflows](task_workflows.md).

---

## CLI Commands

### Submit compose

```bash
vidu-cli task compose \
  --timeline <file_path_or_json_string> \
  [--width 1920] \
  [--height 1080] \
  [--schedule-mode <mode>] \
  [--wait [--timeout 1800] [--output ./results]]
```

Output width/height are optional; the server decides when omitted. The separate `--output` download path requires `--wait` on a compose submission, as does `--timeout`.

`--timeline` accepts either a JSON file path or an inline JSON string. The CLI auto-detects: if the argument starts with `{` or `[`, it is treated as inline JSON; otherwise as a file path. When using inline JSON in shell, wrap in single quotes to prevent shell interpretation: `--timeline '{"video_tracks":[...]}'`.

`--schedule-mode`: `nopeak` = off-peak mode (闲时模式), `claw_pass` = pass mode (通行证模式), `normal` = credit mode (积分模式). Off-peak eligibility, remaining quota, and task/model restrictions are server-validated. If omitted, an active pass selects `claw_pass`, otherwise `normal`; off-peak is not automatically selected. Do not automatically switch billing after rejection. See [parameters](parameters.md#schedule-modes).

Without `--wait`, returns `ok: true` with `task_id` and `trace_id` when available. Save the task ID. With `--wait`, progress goes to stderr and the final task result goes to stdout.

### Query compose job

```bash
vidu-cli task get "$TASK_ID"
vidu-cli task wait "$TASK_ID" --timeout 1800 --output ./results
```

Set `TASK_ID` from the compose response. `get` queries once; `wait` polls until a terminal state (default timeout 1800s, poll interval 5s), optionally downloading media. Timeout/interruption only stops local waiting; continue with the same task ID rather than submitting another compose job. See [errors and retry](errors_and_retry.md).

For a single requested export, use `vidu-cli task compose --timeline ./timeline.json --wait --output ./results`. JSON here is the CLI's timeline input, not a raw API request body.

---

## media_url Rules

The CLI automatically normalizes `media_url` and `file_url` fields in the timeline before sending:

**media_url** (video/image clips):

| Input format | Behavior |
|---|---|
| `ssupload:?id=xxx` | Pass through |
| `http(s)://...` | Pass through (treated as valid remote URL) |
| Local file path | Upload → `ssupload:?id=...` |

**file_url** (subtitle files, e.g. .srt):

| Input format | Behavior |
|---|---|
| `ssupload:?id=xxx` | Pass through |
| `http(s)://...` | Pass through |
| Local file path | Upload → `ssupload:?id=...` |

For local files and http URLs (image/video), the CLI validates before upload:
- File size: image ≤ 50MB, video ≤ 500MB
- Dimensions: width and height must be 128–4096 px, short side ≤ 2160 px

---

## Limits

- Each track type (video, audio, subtitle, effect) is limited to 100 tracks maximum.

---

## Timeline JSON Schema

Top-level structure:

```json
{
  "video_tracks": [],
  "audio_tracks": [],
  "subtitle_tracks": [],
  "effect_tracks": []
}
```

### VideoTrack

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Track type: set to `\"video\"` for standard video tracks |
| `main_track` | boolean | Whether this is the main track |
| `track_shorten_mode` | string | `AutoSpeed` — auto-accelerate when exceeding main track. Omit to use server default. |
| `track_expand_mode` | string | `AutoSpeed` — auto-decelerate when shorter than main track. Omit to use server default. |
| `video_track_clips` | VideoTrackClip[] | Clips on this track |

### VideoTrackClip

| Field | Type | Description |
|-------|------|-------------|
| `media_url` | string | Media reference (see media_url rules above) |
| `type` | string | `Video` (video file), `Image` (static image displayed for `duration` seconds), `GlobalImage` (background image spanning the entire timeline) |
| `x`, `y` | float | Position. [0-0.9999] = percentage of output dimensions, ≥2 = absolute pixels. Values in [1.0, 2.0) are reserved — do not use. |
| `width`, `height` | float | Size. Same rules as x/y: [0-0.9999] = percentage, ≥2 = pixels. Values in [1.0, 2.0) are reserved — do not use. |
| `adapt_mode` | string | `Contain`, `Cover`, `Fill` (default) |
| `in` | float | Entry point in source material (seconds) |
| `out` | float | Exit point in source material (seconds) |
| `duration` | float | Clip duration in seconds. **Required for `Image` and `GlobalImage` types** (specifies display time); optional for `Video` (derived from source if omitted). |
| `timeline_in` | float | Start position on timeline (**required**) |
| `timeline_out` | float | End position on timeline (**required**) |
| `speed` | float | Playback speed (0.1-100) |
| `opacity` | float | 0 (transparent) to 1 (opaque) |
| `effects` | Effect[] | Effects applied to this clip |

### AudioTrack

| Field | Type | Description |
|-------|------|-------------|
| `main_track` | boolean | Whether this is the main track |
| `track_shorten_mode` | string | `AutoSpeed` |
| `track_expand_mode` | string | `AutoSpeed` |
| `audio_track_clips` | AudioTrackClip[] | Clips on this track |

### AudioTrackClip

| Field | Type | Description |
|-------|------|-------------|
| `media_url` | string | Media reference |
| `type` | string | Default `Audio` for audio files; use `AI_TTS` for text-to-speech (when `AI_TTS`: `content` and `voice` are **required**, `media_url` must be omitted) |
| `content` | string | Text content (**required** for `AI_TTS` type; ignored for `Audio` type) |
| `voice` | string | Voice identifier (**required** for `AI_TTS` type; ignored for `Audio` type) |
| `in`, `out` | float | Trim points (seconds) |
| `timeline_in`, `timeline_out` | float | Timeline positioning (**required**) |
| `speed` | float | Playback speed (0.1-100) |
| `loop_mode` | boolean | If `true`, the audio loops to fill the duration between `timeline_in` and `timeline_out`. If `false` or omitted, audio plays once and stops. |
| `effects` | Effect[] | Effects applied to this clip |

### SubtitleTrack

| Field | Type | Description |
|-------|------|-------------|
| `subtitle_track_clips` | SubtitleTrackClip[] | Clips on this track |

### SubtitleTrackClip

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | `Subtitle` (external .srt/.ass file — requires `file_url`, `sub_type`) or `Text` (inline text — requires `content`) |
| `sub_type` | string | `srt` or `ass` (**required** when `type` is `Subtitle`; ignored for `Text`) |
| `file_url` | string | External subtitle file path or URL (**required** when `type` is `Subtitle`; ignored for `Text`) |
| `content` | string | Inline text content (**required** when `type` is `Text`; ignored for `Subtitle`) |
| `font` | string | Font name (default: SimSun) |
| `font_size` | int | 0-5000 |
| `font_color` | string | Hex `#RRGGBB` |
| `font_color_opacity` | float | 0-1 |
| `font_face` | object | `{ bold, italic, underline }` (booleans) |
| `x`, `y` | float | Position |
| `timeline_in`, `timeline_out` | float | Timeline positioning (**required**) |
| `alignment` | string | TopLeft, TopCenter, TopRight, CenterLeft, CenterCenter, CenterRight, BottomLeft, BottomCenter (default), BottomRight |
| `adapt_mode` | string | AutoWrap, AutoScale, AutoWrapAtSpaces |
| `spacing` | int | Character spacing (pixels) |
| `line_spacing` | int | Line spacing (pixels) |
| `angle` | float | Rotation degrees |

### EffectTrack

| Field | Type | Description |
|-------|------|-------------|
| `effect_track_items` | EffectTrackItem[] | Items on this track |

### EffectTrackItem

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | `VFX` or `Filter` |
| `sub_type` | string | Specific effect subtype |
| `timeline_in` | float | Start position (**required**) |
| `timeline_out` | float | End position (**required**) |
| `duration` | float | Effect duration |
| `x`, `y`, `width`, `height` | float | Effect region (for mosaic/blur) |

### Effect (on clips)

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | `Transition`, `VFX`, `Filter`, `Volume`, `AFade`, etc. |
| `sub_type` | string | Specific effect subtype |
| `timeline_in`, `timeline_out` | float | Effect time range |
| `duration` | float | Effect duration |
| `gain` | float | Volume/AEqualize gain |

---

## output_media_config

All fields are optional. Only provided fields are sent to the server.

| Field | Type | Description |
|-------|------|-------------|
| `width` | int | Output video width in pixels |
| `height` | int | Output video height in pixels |

---

## Examples

### Simple: two video clips concatenated

The example below shows three `media_url` formats: local file path (auto-uploaded by CLI), remote URL, and previously uploaded media (`ssupload:` URI).

```json
{
  "video_tracks": [
    {
      "main_track": true,
      "video_track_clips": [
        {
          "media_url": "/Users/me/videos/intro.mp4",
          "type": "Video",
          "timeline_in": 0,
          "timeline_out": 5
        },
        {
          "media_url": "https://example.com/video2.mp4",
          "type": "Video",
          "timeline_in": 5,
          "timeline_out": 10
        },
        {
          "media_url": "ssupload:?id=abc123def456",
          "type": "Video",
          "timeline_in": 10,
          "timeline_out": 15
        }
      ]
    }
  ]
}
```

```bash
vidu-cli task compose --timeline timeline.json
```

### With audio and subtitle

```json
{
  "video_tracks": [
    {
      "main_track": true,
      "video_track_clips": [
        {
          "media_url": "https://example.com/clip.mp4",
          "type": "Video",
          "timeline_in": 0,
          "timeline_out": 10,
          "speed": 1.0
        }
      ]
    }
  ],
  "audio_tracks": [
    {
      "audio_track_clips": [
        {
          "media_url": "/path/to/bgm.mp3",
          "timeline_in": 0,
          "timeline_out": 10,
          "loop_mode": true
        }
      ]
    }
  ],
  "subtitle_tracks": [
    {
      "subtitle_track_clips": [
        {
          "type": "Text",
          "content": "Hello World",
          "font_size": 48,
          "font_color": "#FFFFFF",
          "alignment": "BottomCenter",
          "timeline_in": 0,
          "timeline_out": 5
        }
      ]
    }
  ]
}
```

### Inline JSON

```bash
vidu-cli task compose --timeline '{"video_tracks":[{"main_track":true,"video_track_clips":[{"media_url":"/path/to/video.mp4","type":"Video","timeline_in":0,"timeline_out":5}]}]}'
```

