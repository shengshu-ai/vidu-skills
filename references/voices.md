# Personal Cloned Voices

Use with `vidu-cli v0.2.36+` when the user requests voice cloning, personal voice management, or TTS with a cloned voice. Official voice selection and general speech settings are in [parameters](parameters.md).

## Official vs Personal Voices

| Source | Command | ID use |
|---|---|---|
| Official static voice pool | `task tts-voices` or `task lip-sync-voices` (same pool) | Choose an exact returned official ID. |
| Personal cloned voices | `voice create`, `voice list` | Use the returned `voice_...` ID for status/update and TTS after cloning succeeds. |

Do not invent IDs or pass an official static ID to `voice get/wait/update`. A voice ID is not a task ID; cloning waits use `voice wait`, not `task wait`. The CLI has no `voice delete` command.

## Create and Wait

```bash
# Default cloning is paid. Only run when cloning is requested.
vidu-cli voice create --audio ./voice-sample.wav --name "Story narrator" --wait --timeout 1800

# Use this alternative only when a trial is requested; do not run both creates.
vidu-cli voice create --audio ./voice-sample.wav --name "Trial narrator" --use-trial
```

The recording must be a nonempty **local regular file**, 10–300 seconds and at most 20 MiB, with valid sample rate/channels and readable audio metadata. The CLI checks decoded container properties rather than trusting the filename; WAV/MP3 are convenient choices. URLs, existing `ssupload` URIs, and caller-supplied durations are not accepted as the recording input.

Upload completion performs synchronous review; cloning is submitted with the accepted upload URI only after that review passes. The optional name is supplied at creation; `--description` is not a create flag and can be set later with update.

Without `--wait`, create returns the `voice_id` after upload/review and clone submission. With `--wait`, it continues waiting for the clone. `--timeout` requires `--wait`, defaults to 1800 seconds, and controls clone waiting, not the preceding upload/review. Voice creation has no `--output` or `--poll-interval` option.

`--use-trial` explicitly requests trial billing. A failed trial is not permission to retry as paid. A transport/server failure during clone submission may have created a paid clone: do not blindly issue another create. Preserve any returned voice/upload ID and inspect existing voices first. See [errors and retry](errors_and_retry.md).

## List, Inspect, and Update

```bash
vidu-cli voice list --pagesz 20
vidu-cli voice list --page-token "$NEXT_PAGE_TOKEN" --pagesz 20
vidu-cli voice get "$VOICE_ID"
vidu-cli voice wait "$VOICE_ID" --timeout 1800
vidu-cli voice update "$VOICE_ID" --name "New narrator" --description "For story narration"
```

Set `VOICE_ID` to the personal `voice_...` ID from a response. `list` uses a returned continuation token via `--page-token`; `--pagesz` is 1–100, default 20, and no numeric page option exists.

`get` reads a status snapshot from the server's SSE stream; `wait` continues over SSE until ready/failed or the deadline. There is **no polling interval flag**. Clone states are `created`, `processing`, `success`, `failed`; the CLI exposes a normalized `state` alongside available server status data. Failed snapshots/waits return `ok: false` with error type `voice_failed`; voice waiting timeout uses type `timeout`. Preserve `error.fields.voice_id` and available status details. A waiting timeout/interruption does not cancel the clone; continue with the original voice ID.

`update` requires at least one of `--name` or `--description`. It updates metadata rather than creating another clone. Voice metadata operations and status access are subject to ownership/permissions.

## Use a Ready Voice for TTS

```bash
# VOICE_ID comes from create/list; wait until state is success.
vidu-cli voice wait "$VOICE_ID" --timeout 1800
vidu-cli task tts --prompt "Welcome to the next chapter of our story." \
  --voice-id "$VOICE_ID" --wait --output ./audio
```

TTS returns its own creator `task_id`, separate from `voice_id`. Single-segment TTS has subtitles enabled by default, so use a directory for audio plus subtitle JSON. For multi-segment text, pass `--subtitle-enable false`. Do not assume clone completion also completed a TTS generation.
