# Vidu Skills

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

## 安装

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
  --duration 5

# 文字转语音
vidu-cli task tts \
  --prompt "你好，欢迎使用 Vidu" \
  --voice-id zh_female_1

# 列出可用声音
vidu-cli task tts-voices
```

## 命令速查

| 命令 | 用途 |
|------|------|
| `vidu-cli upload <image>` | 上传图片，返回 `upload_id` |
| `vidu-cli task submit --type ... --prompt ...` | 提交任务，返回 `task_id` |
| `vidu-cli task get <task_id> [--output dir]` | 查询任务状态，可下载结果 |
| `vidu-cli task compose --timeline <json>` | 视频合成，返回 `task_id` |
| `vidu-cli task lip-sync --video ... --text ...` | 口型同步（TTS 模式） |
| `vidu-cli task lip-sync --video ... --audio ...` | 口型同步（音频模式） |
| `vidu-cli task tts --prompt ... --voice-id ...` | 文字转语音 |
| `vidu-cli element create --name ... --image ...` | 创建参考素材 |
| `vidu-cli element list` | 列出个人素材 |
| `vidu-cli element search --keyword ...` | 搜索社区素材 |

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
