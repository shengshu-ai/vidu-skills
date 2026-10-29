---
name: vidu-mcp
description: Generate and deliver Vidu images, videos, and speech through an already connected Vidu MCP server. Use for reference uploads, generation quotes, task recovery, downloads, reusable assets, and cloned voices through MCP.
metadata:
  compatibility_manifest: compatibility.json
---

# Vidu MCP Workflows

Use the connected Vidu MCP server and its discovered tools for Vidu business operations. Connection names and tool prefixes depend on the host. Keep the user's selected account and region; a failed request is not a reason to switch regions.

The connected server's tool descriptions and input Schema define the available models, parameter names, defaults, and limits. Read the relevant tool before calling it. This MCP workflow does not require `vidu-cli`, Node.js, local `VIDU_TOKEN`, or a bundled transfer script. Authentication belongs to the MCP connection; if it fails, have the user reconnect that account rather than asking for a Token in chat.

For local uploads or downloaded results, the agent also needs access to the selected files and a local HTTP or command execution tool. Use that host capability for the actual PUT/GET transfer. A remote MCP call cannot read a path on the user's computer.

## Check Compatibility First

Before starting a business workflow, read the local [compatibility manifest](compatibility.json) and call the connected server's read-only `get_server_info` tool. Its successful result must identify `data.name` as `vidu` and provide `data.version`.

The manifest is the only source of the MCP Skill's independent `skill_version` and supported server range. Accept only stable `X.Y.Z` versions, with three non-negative integer components and no leading zeroes except zero itself. Compare the components numerically: `mcp_server.min_version <= data.version < mcp_server.max_version_exclusive`. Do not infer this version from connector metadata, MCP protocol dates, the CLI version, or a model ID; prerelease, build-metadata, or prefixed strings do not pass this check.

If the manifest cannot be read, the tool is missing or fails, the identity/version is unavailable or invalid, or the server is outside the range, stop the business workflow before uploads, quotes, or task submissions. Report the actual issue and the manifest's required range without claiming compatibility was verified. Ask for an appropriate MCP server upgrade/reconnection or installation of a Skill release compatible with that server. Do not bypass the check by switching to CLI, another account, or another region. Recheck after the connection or either release changes.

Passing the range check permits this workflow to proceed; the discovered tool Schema still governs individual operations and parameters. A version declaration does not create missing tools or establish acceptance of an unsupported argument.

## Choose the Tools

| Goal | Tools / guidance |
|---|---|
| Verify server compatibility | `get_server_info`; compare its identity and stable version with the local compatibility manifest before business operations. |
| Generate an image or video | `quote_generation` → `generate`; use the current Schema's task/model combinations. |
| Upload reference media | `prepare_upload` → local HTTP PUT → `complete_upload` → `get_upload_status`; read [file transfer](references/files.md). |
| Generate speech or lip-sync | `list_official_voices` or `list_voices` to select a real voice; `quote_speech` → `text_to_speech`, or `quote_lip_sync` → `lip_sync`. |
| Compose a timeline | `compose`; timeline media URL fields must use ready upload URIs. |
| Extend or enhance a creation | The matching `quote_extend_video`, `quote_upscale_image`, or `quote_upscale_video`, then `extend_video`, `upscale_image`, or `upscale_video`. |
| Find, resume, cancel, or delete a task | `list_tasks`, `get_task`, `cancel_task`, `delete_task`; read [results and recovery](references/results.md). |
| Create a new task from historical inputs | `rerun_task`; it creates new paid work rather than resuming the original task. |
| Manage reference elements | `check_element_name`, `preprocess_element`, `create_element`, `get_element`, `list_elements`, `search_elements`, `update_element`, `delete_element`, `favorite_element`. |
| Find and deliver creations | `list_creations`, `favorite_creation`, `get_download_info`; read [file transfer](references/files.md). |
| Clone or manage a personal voice | `create_voice`, `get_voice`, `list_voices`, `list_voice_tasks`, `update_voice`; read [results and recovery](references/results.md). |
| Read balance, pass quota, or ledger | `get_credits`, `get_pass_status`, `list_credit_transactions`. |

## Generation Decisions

- Use the user's chosen model and the exact internal `model_version` supported by the task's Schema. Models have no automatic default. Do not substitute a newer model, product display name, CLI alias, or guessed public API model ID.
- The six basic task types are `text2image`, `reference2image`, `text2video`, `img2video`, `headtailimg2video`, and `character2video`. `img2image` is a CLI alias, not an MCP task type. Text-only types do not accept reference images.
- Put ready image/video/audio URIs into the corresponding arrays. `img2video` takes one image; head/tail takes two in order. Other reference counts and supported media depend on the model; read the Schema rather than applying one universal limit. Reusable elements use `materials: [{id, version}]` with real current values from element details.
- `audios` supplies reference audio. `with_audio` controls output sound; omission, `true`, and `false` are distinct choices. Image tasks omit `duration`, `transition`, and `with_audio`.
- Schedule values are `normal` (credits), `claw_pass` (pass), and `nopeak` (off-peak / 闲时). To use the server's automatic selection, omit the field; `auto` is not a valid value. Respect the user's choice and report quota/eligibility failures before changing models or billing.
- Where a matching quote tool exists, quote the intended settings before paid submission. Keep model, resolution, duration, sample count, schedule, and output audio choices consistent with the quote. Use the returned price, not a local price table. Apply the user's current authorization without repeatedly asking for an already authorized action.

Example parameters for an image quote, followed by generation when authorized:

```json
{"type":"text2image","model_version":"3.2_image_2_5_fast","text":"A cat walking beside a river","resolution":"1080p","aspect_ratio":"1:1","sample_count":1,"schedule_mode":"normal"}
```

This is an example, not a default model. Confirm the combination against the connected server's Schema. Reference-based generation first completes the upload workflow and supplies its returned ready URI.

## MCP-Specific Parameters

Use MCP parameter names instead of translating CLI flags literally. Current examples of important differences:

- TTS uses `segments` and a real `voice_id`; `volume` is an integer 0–100, default 100. `subtitle_enable` defaults to false and supports a single segment only. Do not add the CLI quote's `pitch` flag.
- Lip-sync requires a ready `video` plus exactly one of `text` or ready `audio`. Text mode also requires `voice_id`; lip-sync `volume` is an integer 0–10, not the TTS scale. The server derives duration, including for quotes.
- Video upscale uses exactly one completed `creation_id` or ready `video` URI. The server reads the full source duration; a local video must be uploaded before this quote. Image enhancement and video extension use creation IDs, not upload or task IDs.
- Compose accepts structured `timeline` JSON. Upload local or remote source files first, then use ready `ssupload` URIs in media URL fields. Optional output `width` and `height` must be provided together.
- Personal voice cloning consumes `source_uri` from a ready recording upload. `use_trial` defaults to false; a rejected trial does not authorize paid cloning.

Preserve IDs as strings; do not round numeric IDs through floating point. Use the response's `next_page_token` unchanged where present; numeric `page` starts at zero. Element update `labels` replaces the whole set: empty clears it, omission preserves it.

For completion, distinguish successful queries from successful tasks and retain the original identifiers. Read [results and recovery](references/results.md) when waiting, downloading, handling errors, or recovering after interruption.
