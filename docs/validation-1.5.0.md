# Validation Record: vidu-skills 1.5.0

Date: 2026-10-27 (Asia/Shanghai).

## Baseline and Changes

- CLI baseline: local `vidu-cli v0.2.36`, commit `6bb4b07`; built binary reports `vidu-cli v0.2.36`.
- Skill baseline: `1.4.15`, repository commit `14c0441`; updated version: `1.5.0`, minimum CLI: v0.2.36.
- Changed the skill entrypoint, both READMEs, and existing parameter/setup/error references; added [task workflows](../references/task_workflows.md), [assets](../references/assets.md), and [personal voices](../references/voices.md).
- Compose delivery/mode guidance was updated; its timeline schema and existing timeline examples were retained.
- No CLI source changes, production submissions, real credential use, or publishing were part of this validation.

## Local CLI Regression Results

Run in the CLI checkout at the baseline above, using its existing tests:

```text
cargo test --offline --bin vidu-cli
cargo test --offline --test cli_errors_e2e
cargo test --offline --test feature_commands_e2e
```

| Suite | Passed | Failed |
|---|---|---|
| CLI unit tests | 202 | 0 |
| Argument/error end-to-end tests | 18 | 0 |
| Local HTTP/SSE Mock contracts | 41 | 0 |
| Total | 261 | 0 |

The initial Mock run could not bind `127.0.0.1` under the restricted sandbox (`Operation not permitted`). The suite was rerun with local loopback allowed and all 41 tests passed. These tests set a loopback base URL and a fake token; they do not require a production account.

## Documentation Checks

- Enumerated six command groups and 42 public operations from 48 scoped CLI help pages (excluding the generated `help` operation). Both READMEs cover every operation.
- Checked 127 shell command snippets against scoped help and supported aliases. Response variables were replaced with typed fixture values without evaluating shell input; one non-executable usage template was excluded. Only help requests were executed, with no API submissions.
- Help checks establish command/option syntax, not backend acceptance or all cross-option dependencies. The existing unit/Mock suites and source review supply those behavioral checks.
- Parsed all Markdown documents; checked local link targets, images, heading fragments, and balanced code fences.
- Parsed YAML frontmatter, verified version `1.5.0`, and compared `name`, `homepage`, `primaryEnv`, and the entire OpenClaw metadata object with the previous revision: preserved.
- Checked removal of stale batch-tool dependencies, mandatory manual polling, voice-alias prohibitions, universal reference limits, and unconditional submission retries. Chinese mode terminology consistently uses `nopeak` = 闲时模式, `claw_pass` = 通行证模式, `normal` = 积分模式.
- `git diff --check` passed.

### Bundled Skill Validator Compatibility

The bundled generic `quick_validate.py` rejects the existing top-level packaging extensions `homepage`, `primaryEnv`, and `version`; these fields were already present and are intentionally retained by this update. This is not a full-package strict-validator pass.

The validator passed against an isolated temporary copy containing the core frontmatter and the unchanged new skill body, with only those three extensions omitted. The actual package retained all extensions; its YAML and metadata preservation were checked separately. No validator or unrelated configuration was modified.

## Scenario Review

| Request | Verified guidance / contract |
|---|---|
| Generate a Wan 3 video with sound | Matching `--with-audio` on quote/submit; 5s/720p/16:9 defaults; server price; normal credit mode. |
| Drama audio-only reference | `character2video`, no image/material requirement; local 4–30s audio without video; default output settings. With video, the documented 2–30s local bound still leaves non-edit minimums to the server. |
| Q4 audio-only / off-peak generation | No forced image/material input when audio is present; correct per-type duration range; `nopeak` is off-peak mode and server-validated; `claw_pass` rejected. |
| Find an old task and rerun with a revised prompt | Inspect history/details first; preserve assets; supported model overrides; new task and billing; no blind resubmission. |
| Quote and upscale a local video | Cost query does not upload; source duration is read automatically; submit uses the same source/resolution; remote sources require full duration. |
| Clone a voice and use it in TTS | Local 10–300s, ≤20 MiB recording; upload review before clone; SSE wait; ready personal voice ID; no automatic trial-to-paid fallback. |
| Wait, recover from timeout, and download | Same task/voice ID after timeout; task and voice error types kept distinct; single-file vs multi-artifact/subtitle directory rules. |
| Manage references and creations | Element ID/version vs creation ID; label replacement/clearing; cursor vs numeric pagination; internal share-ID resolution. |

The maintained entrypoint is [SKILL.md](../SKILL.md); model-specific limits and billing selection are in [parameters](../references/parameters.md).
