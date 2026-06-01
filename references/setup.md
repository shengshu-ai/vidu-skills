# Vidu CLI Setup

Use this reference only for installation, credential setup, missing CLI errors, or external-user onboarding.

## Requirements

- Node.js 14 or newer
- npm
- Vidu API token in `VIDU_TOKEN`

## Install

```bash
npm install -g vidu-cli@latest
```

The package installs the `vidu-cli` binary and downloads the platform runtime during postinstall.

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

## Verify

```bash
vidu-cli --help
vidu-cli task submit --help
vidu-cli element create --help
vidu-cli task tts --help
```

Every API-backed command prints one JSON object to stdout. Success includes `ok: true`; CLI/API failures include `ok: false` and an `error` object. Copy error fields exactly when reporting failures.

## Common Setup Failures

- `vidu-cli: command not found`: install with npm or fix `PATH`.
- Missing token: set `VIDU_TOKEN` in the execution environment.
- Wrong region: set `VIDU_BASE_URL` to the user account's region.
- Unknown command or flag: run `vidu-cli <subcommand> --help`; update `SKILL.md` if the CLI changed.
