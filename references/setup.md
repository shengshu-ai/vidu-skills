# Vidu CLI Setup

Use this reference only for installation, credential setup, missing CLI errors, or external-user onboarding.

## Requirements

- `vidu-cli` **v0.2.36 or newer** (skill 1.5.0 is aligned with v0.2.36, commit `6bb4b07`)
- Node.js 14 or newer
- npm
- Vidu API token in `VIDU_TOKEN` (see [Obtaining a token](#obtaining-a-token))

## Install

```bash
npm install -g vidu-cli@latest
```

The package installs the `vidu-cli` binary and downloads the platform runtime during postinstall.

Check `vidu-cli --version` on first use. If it is older than v0.2.36, upgrade with the install command above and check again. This skill does not maintain legacy fallback command flows. Keep the current default JSON format for agents; `vidu-cli --format table ...` is optional for human inspection.

## Environment

```bash
export VIDU_TOKEN="your_token"
# Optional, mainland China default:
export VIDU_BASE_URL="https://service.vidu.cn"
# Optional, overseas:
export VIDU_BASE_URL="https://service.vidu.com"
# Optional debugging:
export VIDU_DEBUG=1
```

### Obtaining a token

Mainland China: log in at https://www.vidu.cn, open the account menu from the avatar in the lower-left corner, then copy the token from **Vidu Token**.

Overseas: log in at https://www.vidu.com, open the account menu from the avatar in the left sidebar, then copy the token from **Vidu Token**.

Screenshots for both regions: [README.md](../README.md#get-your-vidu-api-token).

## Verify

```bash
vidu-cli --version
vidu-cli --help
vidu-cli task submit --help
vidu-cli task wait --help
vidu-cli task upscale --help
vidu-cli element --help
vidu-cli creation --help
vidu-cli voice --help
vidu-cli quota --help
vidu-cli task tts --help
```

These version/help checks do not submit work. Normal runtime API results print one JSON object to stdout: success includes `ok: true`; runtime failures include `ok: false` and an `error` object. Argument parsing failures instead print text to stderr and exit 2; help/version output is also text. Waiting progress is on stderr. Inspect task `state` separately from `ok`; see [errors and retry](errors_and_retry.md).

Use the host's available command execution tool; no batch orchestration tool is required. Prefer argv where available, or correctly quoted shell commands and prompt/timeline files. Normal task waits use polling; personal voice waits use SSE.

`ffprobe` is optional: video upscale first tries the built-in MP4 duration reader, then falls back to it for other readable video formats. If both fail, follow the returned metadata error. It is not a mandatory dependency for all skill workflows.

## Common Setup Failures

- `vidu-cli: command not found`: install with npm or fix `PATH`.
- CLI version below v0.2.36: upgrade; do not try new commands against the old binary.
- Missing token: set `VIDU_TOKEN` in the execution environment; see [Obtaining a token](#obtaining-a-token).
- Wrong region: set `VIDU_BASE_URL` to the user account's region.
- Unknown command or flag: check the installed version and scoped `--help`; do not guess replacement flags.
