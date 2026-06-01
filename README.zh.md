# Vidu Skills

[English](./README.md)

通过 `vidu-cli` 调用 Vidu 官方 API，生成 AI 视频和图像。

## 功能

| 能力 | 说明 |
|------|------|
| 文生图 | 纯文本生成图像，支持 1080p/2K/4K |
| 文生视频 | 纯文本生成视频，最长 16s |
| 图生视频 | 单张图片 + 文本生成视频 |
| 首尾帧生视频 | 起始帧 + 结束帧 + 文本生成视频 |
| 参考生图 | 1–7 张图片/素材 + 文本生成图像 |
| 参考生视频 | 1–7 张图片/素材 + 文本生成视频 |
| 口型同步 | 用 TTS 或音频文件驱动视频口型 |
| 文字转语音 | 文本转语音，支持 300+ 声音、20+ 语言 |
| 视频合成 | 多轨时间线（视频/音频/字幕/特效）合成导出 |
| 创建素材 | 创建个人参考素材元素 |
| 搜索社区素材 | 搜索社区共享的参考素材 |
| 查询配额/积分 | 查询 claw-pass 每日配额和用户积分余额 |
| 预估消耗 | 提交前查询任务积分消耗（视频/图片/TTS/口型同步） |

## 作为 Agent Skill 安装

```bash
# 通过 npx skills（推荐，支持 Claude Code、Cursor、Copilot 等 40+ agents）
npx skills add shengshu-ai/vidu-skills

# 通过 ClawHub（OpenClaw 生态）
clawhub install github:shengshu-ai/vidu-skills
```

## 安装 vidu-cli

需要 Node.js ≥ 14：

```bash
npm install -g vidu-cli@latest
```

## 环境变量

| 变量 | 必填 | 说明 |
|------|------|------|
| `VIDU_TOKEN` | 是 | Vidu API Token |
| `VIDU_BASE_URL` | 否 | 默认 `https://service.vidu.cn`（中国大陆）；海外用 `https://service.vidu.com` |
| `VIDU_DEBUG` | 否 | 设为 `1` 可打印完整响应体到 stderr |

## 数据使用与隐私

本 Skill 会把用户提供的数据发送到 Vidu 服务器：

- 文本提示词会发送到 Vidu API。
- 本地图片、视频、音频文件作为任务输入使用时，会上传到 Vidu API 服务器（`service.vidu.cn` 或 `service.vidu.com`）。
- 任务参数（如设置、模型版本、时长、分辨率等）会发送到 Vidu API。

使用本 Skill 前，请确认将相关内容发送给 Vidu 符合你的隐私和知识产权要求。数据处理遵循 Vidu 官方政策。

安全建议：

- 如有可能，创建权限范围受限的 Token。
- 初次测试时避免使用生产环境或高权限 Token。
- 查看 Vidu 服务条款和隐私政策。

Vidu 条款与隐私：

- 海外：https://www.vidu.com/terms
- 中国大陆：https://www.vidu.cn/terms

## 快速上手

```bash
# 文生视频
vidu-cli task submit \
  --type text2video \
  --prompt "一只猫在草地上奔跑" \
  --model-version 3.2 \
  --duration 5 \
  --resolution 1080p

# 查询任务（轮询直到 success/failed）
vidu-cli task get <task_id>

# 下载结果
vidu-cli task get <task_id> --output ./output
```

```bash
# 图生视频
vidu-cli task submit \
  --type img2video \
  --prompt "镜头缓慢推进" \
  --image ./photo.jpg \
  --model-version 3.2 \
  --duration 5 \
  --transition pro

# 文字转语音
vidu-cli task tts \
  --prompt "你好，欢迎使用 Vidu" \
  --voice-id zh_female_1 \
  --subtitle-enable

# 列出可用声音
vidu-cli task tts-voices
```

## 命令速查

| 命令 | 用途 |
|------|------|
| `vidu-cli upload <image>` | 上传图片，返回 `upload_id` |
| `vidu-cli task submit --type ... [--prompt ... | --prompt-path ...]` | 提交任务，返回 `task_id` |
| `vidu-cli task get <task_id> [--output dir]` | 查询任务状态，可下载结果；存在 `subtitle_uri` 时同时下载字幕 JSON |
| `vidu-cli task compose --timeline <json>` | 视频合成，返回 `task_id` |
| `vidu-cli task lip-sync --video ... --text ...` | 口型同步（TTS 模式） |
| `vidu-cli task lip-sync --video ... --audio ...` | 口型同步（音频模式） |
| `vidu-cli task tts [--prompt ... | --prompt-path ... | --text ...] --voice-id ... [--subtitle-enable false]` | 文字转语音；单段输入默认启用字幕 |
| `vidu-cli element create --name ... --image ... [--description ...] [--style ...]` | 创建参考素材 |
| `vidu-cli element check --name ...` | 检查素材名称可用性 |
| `vidu-cli element list [--keyword kw]` | 列出个人素材 |
| `vidu-cli element search --keyword ...` | 搜索社区素材 |
| `vidu-cli task lip-sync-voices` | 列出口型同步可用声音（~86 种） |
| `vidu-cli task tts-voices` | 列出 TTS 可用声音（300+，20+ 语言） |
| `vidu-cli task cost --type ... --model-version ... --duration ...` | 预估视频/图片任务积分消耗 |
| `vidu-cli task tts-cost --text ... --voice-id ...` | 预估 TTS 任务积分消耗 |
| `vidu-cli task lip-sync-cost --duration ... --voice-id ...` | 预估口型同步任务积分消耗 |
| `vidu-cli quota pass` | 查询 claw-pass 每日配额 |
| `vidu-cli quota credit` | 查询用户积分余额 |

TTS 字幕：单段 `--prompt` 模式默认启用字幕输出。使用 `--subtitle-enable false` 可关闭字幕输出；多段 `--text` 模式当前需要显式加 `--subtitle-enable false`。使用 `vidu-cli task get <task_id> --output <dir>` 下载结果；存在 `subtitle_uri` 时会下载生成音频，并把字幕预处理后保存为 `{task_id}_subtitle.json`。

## 任务状态

任务状态流转：`created` → `queueing` → `preparation` → `scheduling` → `processing` → **`success`** / **`failed`** / `canceled`

终态为 `success`、`failed`、`canceled`，其余状态继续轮询。

## 参考文档

| 文件 | 内容 |
|------|------|
| `references/parameters.md` | 任务矩阵、CLI 参数、示例、提示词技巧 |
| `references/errors_and_retry.md` | 任务状态、重试策略 |
| `references/compose.md` | 视频合成时间线 JSON schema |

## 链接

- 官网：https://www.vidu.cn/
- npm：https://www.npmjs.com/package/vidu-cli
