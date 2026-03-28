# /feishu:access — Manage Feishu channel access

Manage who can reach you via the Feishu bot. Reads and writes
`~/.claude/channels/feishu/access.json` directly. Changes take effect on the
next inbound message (the server re-reads the file each time). Restart is
never required.

## Usage

```
/feishu:access [subcommand] [args]
```

Run with no arguments to print current state.

---

## Subcommands

### (no args) — print current state

Print `dmPolicy`, the `allowFrom` list, all `groups`, and any `pending`
pairings with their expiry times.

```
/feishu:access
```

Example output:
```
dmPolicy: pairing
allowFrom: [ou_abc123, ou_def456]
p2pChats: {ou_abc123: oc_xxx, ou_def456: oc_yyy}
groups: {}
pending: {a1b2c3: {senderId: ou_zzz, expiresAt: 2026-03-28T15:00:00Z}}
ackReaction: EYES
replyToMode: first
textChunkLimit: 4000
chunkMode: length
```

---

### pair \<code\> — approve a pairing request

Approve the pending pairing code. Moves the sender's `open_id` into
`allowFrom`, records their `p2pChats` mapping, removes the pending entry, and
drops a file at `~/.claude/channels/feishu/approved/<open_id>` so the server
sends a confirmation message to the user on Feishu.

```
/feishu:access pair a1b2c3
```

**Steps to implement:**

1. Read `~/.claude/channels/feishu/access.json`
2. Find the entry in `pending` whose key matches `<code>` (case-insensitive)
3. If not found or expired (`expiresAt < Date.now()`): print error, stop
4. Extract `senderId` and `chatId` from the pending entry
5. Add `senderId` to `allowFrom` (dedup)
6. Set `p2pChats[senderId] = chatId`
7. Delete `pending[code]`
8. Save `access.json`
9. `mkdir -p ~/.claude/channels/feishu/approved/`
10. Write an empty file at `~/.claude/channels/feishu/approved/<senderId>`
    (the server polls this dir and sends the confirmation on Feishu)
11. Print: `Paired: <senderId> added to allowFrom`

---

### deny \<code\> — discard a pending code

Remove the pending entry without notifying the sender.

```
/feishu:access deny a1b2c3
```

**Steps:**

1. Read `access.json`
2. Find the pending entry by code (case-insensitive)
3. If not found: print "not found"
4. Delete `pending[code]`, save
5. Print: `Denied: code a1b2c3 discarded`

---

### allow \<open_id\> — add user directly

Add an `open_id` to `allowFrom` without going through pairing.

```
/feishu:access allow ou_abc123def456
```

**Steps:**

1. Read `access.json`
2. If already in `allowFrom`: print "already allowed"
3. Append to `allowFrom`, save
4. Print: `Allowed: ou_abc123def456 added`

---

### remove \<open_id\> — remove from allowlist

Remove an `open_id` from `allowFrom` and from `p2pChats`.

```
/feishu:access remove ou_abc123def456
```

**Steps:**

1. Read `access.json`
2. Filter `allowFrom` to remove the id
3. Delete `p2pChats[open_id]` if present
4. Save
5. Print: `Removed: ou_abc123def456`

---

### policy \<pairing|allowlist|disabled\> — set DM policy

```
/feishu:access policy allowlist
```

Valid values: `pairing`, `allowlist`, `disabled`.

**Steps:**

1. Validate value is one of the three
2. Read `access.json`, set `dmPolicy`, save
3. Print: `dmPolicy set to: allowlist`

---

### group add \<chat_id\> [--no-mention] [--allow id1,id2] — enable a group

```
/feishu:access group add oc_1234567890abcdef
/feishu:access group add oc_1234567890abcdef --no-mention
/feishu:access group add oc_1234567890abcdef --allow ou_abc123,ou_def456
```

Feishu group chat IDs start with `oc_`.

**Steps:**

1. Parse flags: `--no-mention` sets `requireMention: false`; `--allow id1,id2`
   sets `allowFrom` to that list
2. Read `access.json`
3. Set `groups[chat_id] = { requireMention, allowFrom }`
4. Save
5. Print: `Group oc_xxx enabled (requireMention: true, allowFrom: [])`

**Note:** Feishu bots receive all group messages by default (no privacy mode
equivalent to Telegram). `requireMention: true` (default) means the bot only
responds when @mentioned. With `--no-mention` it responds to every message.

---

### group rm \<chat_id\> — disable a group

```
/feishu:access group rm oc_1234567890abcdef
```

**Steps:**

1. Read `access.json`
2. Delete `groups[chat_id]`, save
3. Print: `Group oc_xxx disabled`

---

### set \<key\> \<value\> — set a config value

```
/feishu:access set ackReaction EYES
/feishu:access set ackReaction ""
/feishu:access set replyToMode all
/feishu:access set textChunkLimit 2000
/feishu:access set chunkMode newline
/feishu:access set mentionPatterns '["^hey claude\\b", "\\bassistant\\b"]'
```

Supported keys and their types:

| Key | Type | Notes |
| --- | --- | --- |
| `ackReaction` | string | Emoji name (e.g. `THUMBSUP`, `EYES`). Empty string `""` disables. |
| `replyToMode` | `off\|first\|all` | Threading on chunked replies |
| `textChunkLimit` | number | Max chars per message. Capped at 4000. |
| `chunkMode` | `length\|newline` | Split strategy |
| `mentionPatterns` | JSON array of strings | Case-insensitive regexes that count as a mention |

**Steps:**

1. Validate key is one of the supported keys
2. Parse value (JSON for mentionPatterns, number for textChunkLimit)
3. Read `access.json`, set the key, save
4. Print: `Set ackReaction = EYES`

---

## Config file reference

`~/.claude/channels/feishu/access.json`

```jsonc
{
  // Handling for DMs from senders not in allowFrom.
  "dmPolicy": "pairing",

  // open_ids allowed to DM.
  "allowFrom": ["ou_abc123def456"],

  // open_id → p2p chat_id mapping (for sending to allowed users).
  "p2pChats": {
    "ou_abc123def456": "oc_p2pchatid"
  },

  // Groups the bot is active in. Empty object = DM-only.
  "groups": {
    "oc_groupchatid": {
      // true: respond only to @mentions.
      "requireMention": true,
      // Restrict triggers to these senders. Empty = any member.
      "allowFrom": []
    }
  },

  // Case-insensitive regexes that count as a mention.
  "mentionPatterns": ["^hey claude\\b"],

  // Emoji name for ack reaction. Empty string disables.
  "ackReaction": "EYES",

  // Threading on chunked replies: first | all | off
  "replyToMode": "first",

  // Split threshold. Feishu rejects > 4096.
  "textChunkLimit": 4000,

  // length = cut at limit. newline = prefer paragraph boundaries.
  "chunkMode": "length"
}
```

---

## Quick reference

| Command | Effect |
| --- | --- |
| `/feishu:access` | Print current state |
| `/feishu:access pair a1b2c3` | Approve pairing code `a1b2c3` |
| `/feishu:access deny a1b2c3` | Discard pending code |
| `/feishu:access allow ou_abc123` | Add open_id directly |
| `/feishu:access remove ou_abc123` | Remove from allowlist |
| `/feishu:access policy allowlist` | Set `dmPolicy` |
| `/feishu:access group add oc_xxx` | Enable group (with @mention required) |
| `/feishu:access group add oc_xxx --no-mention` | Enable group (any message) |
| `/feishu:access group add oc_xxx --allow ou_a,ou_b` | Enable group, restrict senders |
| `/feishu:access group rm oc_xxx` | Disable group |
| `/feishu:access set ackReaction EYES` | Set ack reaction emoji name |
| `/feishu:access set replyToMode all` | Thread all reply chunks |
| `/feishu:access set textChunkLimit 2000` | Set chunk size |
| `/feishu:access set chunkMode newline` | Split at paragraph boundaries |
| `/feishu:access set mentionPatterns '["^hey"]'` | Set mention regexes |
