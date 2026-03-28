# /feishu:setup — Interactive Feishu app setup guide

Walk through creating and configuring a Feishu bot app from scratch.

## What this skill does

Guides you through the Feishu Developer Console to create a bot with the right
capabilities and permissions, then saves the credentials via `/feishu:configure`.

---

## Step-by-step guide

### Step 1: Create an app

1. Go to [open.feishu.cn](https://open.feishu.cn/app) (or
   [open.larksuite.com](https://open.larksuite.com/app) for Lark)
2. Click **Create app** → **Custom app**
3. Enter a name (e.g. "Claude Code") and description
4. Click **Create**

You'll land on the app's overview page.

### Step 2: Get your credentials

1. In the left sidebar, click **Credentials & Basic Info**
2. Note your **App ID** (format: `cli_xxxxxxxxxxxxxxxx`)
3. Note your **App Secret** (click "View" to reveal)

Keep these — you'll need them for `/feishu:configure`.

### Step 3: Enable the bot capability

1. In the left sidebar, click **Add capabilities** (or **Capabilities**)
2. Find **Bot** and click **Enable**
3. Configure the bot name and avatar as desired
4. Click **Save**

### Step 4: Add permissions (scopes)

1. In the left sidebar, click **Permissions & Scopes**
2. Search for and add these permissions:

   | Permission | Purpose |
   | --- | --- |
   | `im:message` | Read messages sent to the bot |
   | `im:message:send_as_bot` | Send messages as the bot |
   | `im:message.file:download` | Download file/image attachments |
   | `im:message.reaction.create_and_delete` | Add emoji reactions to messages |

3. Click **Save** after adding all permissions

### Step 5: Subscribe to events

1. In the left sidebar, click **Event Subscriptions**
2. Under **Request URL**, select **Using long connection** (WebSocket mode)
   - This is the most important step — it means no public server URL is needed
   - The bot connects outbound to Feishu's servers
3. Click **Add event**
4. Search for and add: `im.message.receive_v1` (Receive messages)
5. Click **Save**

### Step 6: Enable WebSocket long-connection mode

Confirm that **Long connection** (长连接) is selected in Event Subscriptions,
not **Request URL**. This allows the bot to work without a public HTTPS endpoint.

### Step 7: Publish the app

1. In the left sidebar, click **App Release** (or **Version Management**)
2. Click **Create version**
3. Fill in the version notes
4. Click **Save and publish** (or submit for review if your organization requires it)

For self-built apps in your own organization, you may be able to publish directly.
For apps that need organization admin approval, submit for review and have an admin approve.

### Step 8: Add the bot to a chat

After publishing:
- For **DMs**: Users can search for your bot by name in Feishu and start a conversation
- For **group chats**: Add the bot to a group via the group settings → Members → Add bot

### Step 9: Save credentials

Run:
```
/feishu:configure
```

Paste your App ID and App Secret when prompted.

### Step 10: Test the connection

Start the server (Claude Code does this automatically when the plugin is installed):
```
bun server.ts
```

Or have Claude Code restart. Send a DM to your bot from Feishu — you should see
a pairing code response.

Then run:
```
/feishu:access pair <code>
```

---

## Troubleshooting

**Bot doesn't respond to DMs:**
- Check that `im.message.receive_v1` event is subscribed
- Check that the app is published and approved
- Check that credentials in `~/.claude/channels/feishu/.env` are correct
- Check server logs: `bun server.ts 2>&1`

**Bot doesn't respond in groups:**
- Add the bot to the group via group settings
- Run `/feishu:access group add <chat_id>` to enable the group
- Group chat IDs start with `oc_` — find yours by checking the server logs
  (dropped messages show the chat_id)

**How to find a group chat ID:**
- Temporarily add the bot to the group
- Send any message in the group
- Check the server logs — the chat_id appears in the event data

**Permission errors:**
- Ensure all four permissions are added AND the app is re-published after adding them
- Some permissions require re-approval after being added

**WebSocket connection issues:**
- Ensure you selected "long connection" in Event Subscriptions, not "Request URL"
- Check firewall rules — the server needs outbound HTTPS to Feishu's servers

---

## Finding user open_ids

Feishu user IDs (`open_id`) look like `ou_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`.

To find someone's open_id:
- Ask them to send a DM to the bot — their open_id appears in server logs
- Or use the Feishu API Explorer at open.feishu.cn to look up users by email

---

## Notes for Lark (international version)

If you're using Lark instead of Feishu (international version):
- Use [open.larksuite.com](https://open.larksuite.com/app) instead
- The SDK and API are identical — just different base URLs
- App IDs still start with `cli_`
