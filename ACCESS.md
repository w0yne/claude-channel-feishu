# Feishu — Access & Delivery

A Feishu bot is reachable by any user in your organization who finds it. Without a gate, those messages flow straight into your assistant session. The access model described here decides who gets through.

By default, a DM from an unknown sender triggers **pairing**: the bot replies with a 6-character code and drops the message. You run `/feishu:access pair <code>` from your assistant session to approve them. Once approved, their messages pass through.

All state lives in `~/.claude/channels/feishu/access.json`. The `/feishu:access` skill commands edit this file; the server re-reads it on every inbound message, so changes take effect without a restart. Set `FEISHU_ACCESS_MODE=static` to pin config to what was on disk at boot (pairing is unavailable in static mode since it requires runtime writes).

## At a glance

| | |
| --- | --- |
| Default policy | `pairing` |
| Sender ID | `open_id` (e.g. `ou_xxxx`) |
| Group key | Group chat ID (e.g. `oc_xxxx`) |
| `ackReaction` | Any Feishu emoji name (e.g. `THUMBSUP`, `EYES`, `OK`) |
| Config file | `~/.claude/channels/feishu/access.json` |

## DM policies

`dmPolicy` controls how DMs from senders not on the allowlist are handled.

| Policy | Behavior |
| --- | --- |
| `pairing` (default) | Reply with a pairing code, drop the message. Approve with `/feishu:access pair <code>`. |
| `allowlist` | Drop silently. No reply. Useful when you don't want the bot to reveal itself to unknown users. |
| `disabled` | Drop everything, including allowlisted users and groups. |

```
/feishu:access policy allowlist
```

## User IDs

Feishu identifies users by **open_id** — a string like `ou_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`. These are app-specific (different apps see different open_ids for the same user), stable, and never shown in the Feishu UI.

Pairing captures the open_id automatically. To find one manually:
- Have the user send a DM to your bot — their open_id appears in server logs
- Use the Feishu API Explorer at open.feishu.cn

```
/feishu:access allow ou_abc123def456
/feishu:access remove ou_abc123def456
```

## Groups

Groups are off by default. Opt each one in individually.

```
/feishu:access group add oc_1234567890abcdef
```

Feishu group chat IDs start with `oc_`. To find a group's chat_id:
- Add your bot to the group and send any message
- Check the server logs — the event data includes `chat_id`

With the default `requireMention: true`, the bot responds only when @mentioned in the group. Unlike Telegram, Feishu bots receive all group messages by default (no server-side privacy filtering), so `requireMention` is enforced in code.

Pass `--no-mention` to process every message in the group, or `--allow id1,id2` to restrict which members can trigger the bot.

```
/feishu:access group add oc_1234567890abcdef --no-mention
/feishu:access group add oc_1234567890abcdef --allow ou_abc123,ou_def456
/feishu:access group rm oc_1234567890abcdef
```

## Mention detection

In groups with `requireMention: true`, any of the following triggers the bot:

- A structured @mention of the bot (bot's open_id appears in message.mentions)
- A match against any regex in `mentionPatterns`

```
/feishu:access set mentionPatterns '["^hey claude\\b", "\\bassistant\\b"]'
```

Note: Unlike Telegram, "reply to bot's message" does not automatically count as a mention in Feishu — use `mentionPatterns` if you want that behavior.

## p2pChats mapping

Because Feishu p2p (DM) chat IDs differ from user open_ids, the server maintains a `p2pChats` mapping: `open_id → chat_id`. This is populated automatically during pairing or first contact, and is used to send outbound messages to allowed users.

You rarely need to edit this directly, but it's visible in `/feishu:access` output.

## Delivery

Configure outbound behavior with `/feishu:access set <key> <value>`.

**`ackReaction`** reacts to inbound messages on receipt. Feishu supports any standard emoji by name. Common names:

> `THUMBSUP` `THUMBSDOWN` `OK` `SMILE` `EYES` `HEART` `FIRE` `CLAP` `THINK` `HUNDRED` `CHECK` `CROSS`

```
/feishu:access set ackReaction EYES
/feishu:access set ackReaction ""
```

**`replyToMode`** controls threading on chunked replies. When a long response is split, `first` (default) threads only the first chunk under the inbound message; `all` threads every chunk; `off` sends all chunks standalone.

**`textChunkLimit`** sets the split threshold. Feishu rejects messages over ~4096 characters; the server's cap is 4000 for safety.

**`chunkMode`** chooses the split strategy: `length` cuts exactly at the limit; `newline` prefers paragraph boundaries.

## Skill reference

| Command | Effect |
| --- | --- |
| `/feishu:access` | Print current state: policy, allowlist, pending pairings, enabled groups. |
| `/feishu:access pair a1b2c3` | Approve pairing code `a1b2c3`. Adds the sender to `allowFrom` and sends a confirmation on Feishu. |
| `/feishu:access deny a1b2c3` | Discard a pending code. The sender is not notified. |
| `/feishu:access allow ou_abc123` | Add a user open_id directly. |
| `/feishu:access remove ou_abc123` | Remove from the allowlist. |
| `/feishu:access policy allowlist` | Set `dmPolicy`. Values: `pairing`, `allowlist`, `disabled`. |
| `/feishu:access group add oc_xxx` | Enable a group. Default: requireMention=true. |
| `/feishu:access group add oc_xxx --no-mention` | Enable group, respond to all messages. |
| `/feishu:access group add oc_xxx --allow ou_a,ou_b` | Enable group, restrict to specific senders. |
| `/feishu:access group rm oc_xxx` | Disable a group. |
| `/feishu:access set ackReaction EYES` | Set ack reaction emoji name. |
| `/feishu:access set replyToMode all` | Thread all reply chunks. |
| `/feishu:access set textChunkLimit 2000` | Set split threshold. |
| `/feishu:access set chunkMode newline` | Split at paragraph boundaries. |
| `/feishu:access set mentionPatterns '["^hey"]'` | Set mention regexes. |

## Config file

`~/.claude/channels/feishu/access.json`. Absent file is equivalent to `pairing` policy with empty lists, so the first DM triggers pairing.

```jsonc
{
  // Handling for DMs from senders not in allowFrom.
  "dmPolicy": "pairing",

  // open_ids allowed to DM.
  "allowFrom": ["ou_abc123def456"],

  // open_id → p2p chat_id (for sending outbound to allowed users).
  "p2pChats": {
    "ou_abc123def456": "oc_p2pchatid"
  },

  // Groups the bot is active in. Empty object = DM-only.
  "groups": {
    "oc_groupchatid": {
      // true: respond only to @mentions (enforced in code, not by Feishu servers).
      "requireMention": true,
      // Restrict triggers to these senders. Empty = any member.
      "allowFrom": []
    }
  },

  // Case-insensitive regexes that count as a mention.
  "mentionPatterns": ["^hey claude\\b"],

  // Feishu emoji name. Empty string disables.
  "ackReaction": "EYES",

  // Threading on chunked replies: first | all | off
  "replyToMode": "first",

  // Split threshold. Feishu rejects > 4096.
  "textChunkLimit": 4000,

  // length = cut at limit. newline = prefer paragraph boundaries.
  "chunkMode": "length"
}
```
