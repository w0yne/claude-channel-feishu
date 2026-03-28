# Claude Channel Feishu

A [Claude Code](https://claude.ai/code) channel plugin that bridges [Feishu (飞书)](https://www.feishu.cn/) bot messages into your Claude Code session.

**WebSocket long-connection mode** — no public IP, no reverse proxy needed.

Built on the Channels protocol introduced in Claude Code v2.1.80.

[English](#features) · [中文](#中文)

## Architecture

```
Feishu user sends message
 │
 ▼
Feishu server ──(WebSocket long-connection push)──► Local Channel Server
 │                                                   (MCP subprocess)
 │                                            notifications/claude/channel
 ▼
 Claude Code session
 │                    reply tool
 ▼
 Channel Server → Feishu API
 │
 ▼
 Message delivered to user
```

## Features

Full feature parity with the official [Telegram channel plugin](https://github.com/anthropics/claude-plugins-official/tree/main/external_plugins/telegram):

- **Access control** — pairing flow, allowlists, per-group policies, mention detection
- **Text messages** — send and receive with auto-chunking at 4000 chars
- **Images** — auto-download inbound photos, send images as attachments
- **Files & documents** — deferred download via `download_attachment` tool
- **Audio & video** — forwarded as attachment metadata
- **Rich text (post)** — extracted to plain text for Claude
- **Reactions** — add emoji reactions to messages
- **Edit messages** — update previously sent messages
- **Reply threading** — thread replies under specific messages
- **Permission relay** — approve/deny Claude Code permission requests from Feishu
- **Static mode** — pin access config at boot for locked-down environments

## Prerequisites

| Requirement | Version / Notes |
| --- | --- |
| OS | macOS or Linux |
| Bun | Latest (runtime) |
| Claude Code | v2.1.80+ with a claude.ai account |
| Feishu app | Self-built internal enterprise app on [Feishu Open Platform](https://open.feishu.cn/app) |

## Quick Setup — 3 Steps

### Step 1 — Install

```bash
git clone https://github.com/w0yne/claude-channel-feishu.git
cd claude-channel-feishu
bun install
```

### Step 2 — Configure credentials

```bash
mkdir -p ~/.claude/channels/feishu
cat > ~/.claude/channels/feishu/.env <<EOF
FEISHU_APP_ID=cli_xxxxxxxxxxxxxxxx
FEISHU_APP_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
EOF
chmod 600 ~/.claude/channels/feishu/.env
```

Find your App ID and App Secret in the [Feishu Developer Console](https://open.feishu.cn/app) under **Credentials & Basic Info**.

Or run `/feishu:configure` inside Claude Code for an interactive setup.

### Step 3 — Launch

```bash
claude --channels plugin:feishu@w0yne/claude-channel-feishu
```

On success, Claude Code shows:
```
Listening for channel messages from: server:feishu
```

## Feishu Developer Console Setup

1. Go to [open.feishu.cn](https://open.feishu.cn/app) → select or create an app
2. **Enable Bot capability**: App features → Bot → Enable
3. **Add permissions** (Permission management):

| Permission | Description |
| --- | --- |
| `im:message` | Read messages sent to the bot |
| `im:message:send_as_bot` | Send messages as the bot |
| `im:chat` | Access chat/group information |
| `im:resource` | Download images and files from messages |
| `im:message:reaction` | Add reactions to messages (optional) |
| `im:message:patch` | Edit bot messages (optional) |

4. **Enable event subscription**: Event subscription → **Use long-connection** → Add event: `im.message.receive_v1`
5. **Publish** the app version (enterprise self-built apps require admin approval)

> **Tip:** Run `/feishu:setup` inside Claude Code for an interactive walkthrough.

## Usage

| Scenario | How to use |
| --- | --- |
| Direct message | Find the bot in Feishu and send a text message |
| Group chat | Add the bot to a group, then @mention it |

Messages are pushed in real-time to your current Claude Code session. Claude replies automatically and the response is sent back via the bot.

## Tools

| Tool | Purpose |
| --- | --- |
| `reply` | Send to a chat. Takes `chat_id` + `text`, optionally `reply_to` for threading and `files` for attachments. Images send as image messages; other types as file messages. Auto-chunks text at 4000 chars. |
| `react` | Add an emoji reaction to a message. Uses Feishu emoji names (e.g. `THUMBSUP`, `OK`, `SMILE`). |
| `edit_message` | Edit a bot-sent message. Useful for progress updates. Edits don't trigger push notifications. |
| `download_attachment` | Download file/image from an inbound message to local inbox. Returns local path for `Read`. |

## Skills

| Command | Description |
| --- | --- |
| `/feishu:access` | Manage access — pair users, edit allowlists, configure groups and policies |
| `/feishu:configure` | Save credentials and check channel status |
| `/feishu:setup` | Interactive Feishu Developer Console setup guide |

## Access Control

See **[ACCESS.md](./ACCESS.md)** for full documentation on pairing, allowlists, group policies, mention detection, and delivery config.

Quick reference:
- Default policy: `pairing` (bot replies with a 6-char code, user approves in Claude Code)
- IDs are Feishu `open_id` values (e.g. `ou_xxxx`)
- Groups are opt-in with configurable mention requirements
- Switch to `allowlist` policy once all users are paired

## Limitations

- Rich text replies are text-only (no card messages yet)
- No typing indicator (Feishu Bot API doesn't support it)
- Stickers are forwarded as `(sticker)` — content not extracted
- Launched with development channel flags (pre-marketplace)

## Security

- **Never commit your App Secret** to a git repository
- Store credentials in `~/.claude/channels/feishu/.env` (already in `.gitignore`)
- The Channels protocol carries prompt injection risk — only use in trusted environments
- Access control prevents unauthorized users from reaching your Claude Code session

## License

Apache 2.0

---

## 中文

将飞书机器人消息桥接到 Claude Code 会话的 Channel 插件。

WebSocket 长连接模式 — 无需公网 IP，无需内网穿透。

### 功能

与官方 Telegram 插件完全对标：

- **访问控制** — 配对流程、白名单、群组策略、@提及检测
- **文本消息** — 收发，自动分片（4000 字符）
- **图片** — 自动下载收到的图片，支持发送图片附件
- **文件/文档** — 通过 `download_attachment` 工具延迟下载
- **音频/视频** — 作为附件元数据转发
- **富文本** — 提取为纯文本供 Claude 处理
- **表情回应** — 给消息添加 emoji 回应
- **编辑消息** — 更新已发送的消息
- **回复串联** — 在指定消息下创建回复线程
- **权限中继** — 从飞书批准/拒绝 Claude Code 权限请求

### 快速开始

```bash
# 1. 克隆安装
git clone https://github.com/w0yne/claude-channel-feishu.git
cd claude-channel-feishu && bun install

# 2. 配置凭据
mkdir -p ~/.claude/channels/feishu
cat > ~/.claude/channels/feishu/.env <<EOF
FEISHU_APP_ID=cli_xxx
FEISHU_APP_SECRET=xxx
EOF
chmod 600 ~/.claude/channels/feishu/.env

# 3. 启动
claude --channels plugin:feishu@w0yne/claude-channel-feishu
```

### 飞书开发者控制台配置

1. 访问 [open.feishu.cn](https://open.feishu.cn/app) → 创建/选择应用
2. 开启**机器人**能力
3. 添加权限：`im:message`、`im:message:send_as_bot`、`im:chat`、`im:resource`
4. 事件订阅 → 使用**长连接**接收事件 → 添加 `im.message.receive_v1`
5. 发布应用版本

在 Claude Code 中运行 `/feishu:setup` 可获得交互式配置引导。
