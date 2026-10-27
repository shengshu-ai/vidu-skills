# Task History, Delivery, and Post-Processing

Use with `vidu-cli v0.2.36+`. Generation settings and schedule rules are in [parameters](parameters.md); error recovery is in [errors and retry](errors_and_retry.md).

## IDs and History

`TASK_ID` below must be a real response task ID. `CREATION_ID` must be a positive numeric creation ID returned by `task get --details` or `creation list`, not a task ID. Never assume one task has only one creation.

```bash
vidu-cli task list --state success --type text2video --pagesz 20
vidu-cli task list --state success --state failed --pagesz 20
vidu-cli task list --page-token "$NEXT_PAGE_TOKEN" --pagesz 20
vidu-cli task get "$TASK_ID" --details
vidu-cli --format table task list
```

`task list` supports repeated `--state`, `--type`, and `--scene`; it uses a continuation token from the response with `--page-token`, not a numeric page. `--pagesz` is 1–100, default 20. History can contain types that cannot be submitted or rerun by this CLI; a history type is not proof of a supported generation command.

`task get` is one read. It returns `task_id`, `state`, `type`, `model`; failed tasks also include `err_code` and `err_msg`. `--details` adds raw `input`, `settings`, and `creations` with normalized `creation_id` values. Use details to inspect original references, model settings, and outputs before rerunning or post-processing.

## Wait and Download

Without `--wait`, a creator command returns its task ID immediately. `submit`, `tts`, `lip-sync`, `compose`, `rerun`, `extend`, `upscale-image`, and `upscale` support:

```text
--wait [--timeout SECONDS] [--output PATH]
```

On these submissions, `--timeout` and `--output` / `-o` require `--wait`. The default timeout is 1800 seconds and polling interval is 5 seconds. To choose a polling interval for an existing task, use `task wait`:

```bash
vidu-cli task submit --type text2video --prompt "A quiet street at dawn" \
  --model-version 3.2 --duration 5 --resolution 1080p --wait --output ./results

# Resume using the response ID, including after a local timeout.
vidu-cli task wait "$TASK_ID" --timeout 1800 --poll-interval 5 --output ./results

# Query once, downloading only if already successful.
vidu-cli task get "$TASK_ID" --output ./results
```

Both timeout and poll interval must be positive. Progress goes to stderr; the final JSON envelope goes to stdout. `success`, `failed`, and `canceled` are terminal. Failed/canceled waits and timeouts return a nonzero exit code. A plain `get` may have `ok: true` while reporting a failed task: always inspect `state`.

Waiting timeout/interruption ends only local waiting, not the remote task. Preserve IDs from the submission or available `error.fields`; resume with that ID. For batches, submit all tasks, record IDs, then wait separately rather than blocking each submission in sequence. Do not resubmit simply because waiting stopped.

### Output Paths and Subtitles

`get --output`, `wait --output`, and waiting submissions use the same download path rules:

- An existing directory, or a path without a filename extension, is a directory. Downloaded media paths are returned in `downloaded_files`.
- A file path such as `./result.mp4` is allowed for a single artifact only. Multiple samples, multiple creations, or audio plus subtitle JSON require a directory. Parents are created as needed.
- If `subtitle_uri` is present, downloads also include preprocessed `{task_id}_subtitle.json`. Use a directory for TTS with subtitles enabled.
- `task get --output` on an unfinished task returns `download_skipped: "task not ready"`; it does not wait. A failed download can be retried through the same task ID without creating another task.

```bash
# Only for a task with exactly one downloadable artifact.
vidu-cli task get "$TASK_ID" --output ./result.mp4
# Use a directory when the artifact count is unknown or subtitles are present.
vidu-cli task get "$TASK_ID" --output ./results
```

There is no `task download` subcommand.

## Cancel and Delete

```bash
vidu-cli task cancel "$TASK_ID"
vidu-cli task delete "$TASK_ID"
```

Cancellation applies before `processing` and does not support `clip_compose`; permissions and state restrictions are enforced by the server. Deletion is a task history soft delete; `queueing` and `processing` tasks cannot be deleted. A wait timeout is not cancellation, and deleting history is not a download or a new task.

## Rerun Historical Inputs

```bash
vidu-cli task get "$TASK_ID" --details
vidu-cli task rerun "$TASK_ID" --prompt "Make the scene rainy" --wait --output ./results
vidu-cli task rerun "$TASK_ID" --prompt-file ./revised.txt \
  --model-version viduq2-drama-std --duration 5 --resolution 720p \
  --with-audio=false --wait --output ./results
```

`rerun` creates a **new task with its own billing**. It supports only the six original submit types: `text2image`, `reference2image`, `text2video`, `img2video`, `headtailimg2video`, `character2video`. It preserves reference assets and does not guarantee reproduction with the same seed. It cannot rerun TTS, compose, or lip-sync.

Overrides: `--prompt`, `--prompt-path` / `--prompt-file`, `--model-version` / `--model`, `--duration` (video tasks only), `--resolution`, and `--with-audio[=false]`, plus standard wait/download options. There are no rerun flags for replacing reference files or directly selecting a schedule mode.

When the model actually changes, the CLI normalizes incompatible transition and schedule settings to the target model's accepted values/defaults, and lets the server select the vendor model again. It keeps an accepted transition; otherwise it uses `stable` for 3.0 image/head-tail, `speed` where 3.1/3.2 require it, or removes transition for targets that do not support it. Q4, Drama, and Wan 3 default to `normal`. Original style is preserved; switching to 3.0 with no style uses `general`. Historical duration/resolution remain unless overridden, so choose explicit compatible values when switching models.

Omitting the output audio switch preserves historical choices; `--with-audio=false` also clears historical lip-sync. Do not apply that switch to an image rerun.

## Extend Video or Enhance an Image

```bash
vidu-cli task extend --creation-id "$CREATION_ID" --duration 4 \
  --transition extendspeed --wait --output ./results
vidu-cli task upscale-image --creation-id "$IMAGE_CREATION_ID" \
  --resolution 2k --wait --output ./images
```

`extend` requires an accessible completed **1080p video creation**. It creates an extension using model 3.1 at 1080p, adds 1–7 seconds, and accepts `extendspeed` (default) or `extendpro`. Package eligibility and total duration remain server-validated. It does not accept a raw video path.

`upscale-image` requires an image creation ID, defaults to 2k, and accepts 720p, 1080p, 2k, 4k. It is distinct from video `upscale`.

## Video Upscale and Quote

`task upscale` accepts exactly one source: `--creation-id` **or** `--video`. `--resolution` is required and accepts 720p, 1080p, 2k, 4k. An existing source creation need not be 1080p.

| Source | Duration handling |
|---|---|
| Accessible completed video creation | Read from creation metadata; `--duration` cannot override it. |
| Nonempty local MP4/MOV/AVI, ≤500 MiB, with a video track | Read by built-in MP4 parser, falling back to local `ffprobe`; `--duration` cannot override it. |
| HTTP/HTTPS URL or `ssupload:?id=...` | `--duration` is required, a positive integer describing the **full source**, not a trim length. No remote duration probe is done. |

Automatically read durations are rounded to integer seconds. Every resulting duration must be in 1–2147483647. If both built-in parsing and ffprobe fail, report the metadata error; do not invent a duration for local files. Quote and submit use the same source and duration rules.

```bash
# Query first; quotes do not upload local files or create tasks.
vidu-cli task upscale-cost --video ./input.mp4 --resolution 4k
vidu-cli task upscale-cost --creation-id "$CREATION_ID" --resolution 2k
vidu-cli task upscale-cost --video 'ssupload:?id=123' --duration 8 --resolution 2k

# Process when the user has requested upscaling.
vidu-cli task upscale --video ./input.mp4 --resolution 4k --wait --output ./results
vidu-cli task upscale --creation-id "$CREATION_ID" --resolution 2k --wait --output ./results
vidu-cli task upscale --video 'https://example.com/input.mp4' --duration 8 --resolution 2k
```

The upload ID and URL above are placeholders; use real accessible media. Local submissions auto-upload. Quotes may read source metadata but never upload it; `upscale-cost` accepts no `--wait`, `--timeout`, or `--output`. It returns server-provided `cost_credits`, `current_credits`, `can_submit`, and `original_cost_credits`; `can_submit` is only a balance check at quote time.

Video upscale uses `normal` credit mode, with provider selection by the server; there is no model or schedule flag. Submission is not automatically retried after ambiguous transport/server failures. Save its task ID and use `task wait` to continue after a timeout.
