# /feishu:configure — Configure Feishu channel credentials

Save your Feishu app credentials so the channel server can authenticate.

## What this skill does

1. Prompts for (or accepts) `FEISHU_APP_ID` and `FEISHU_APP_SECRET`
2. Writes them to `~/.claude/channels/feishu/.env`
3. Sets file permissions to `600` (owner-read-only)
4. Optionally sets `FEISHU_ACCESS_MODE=static` for read-only access config

## Usage

```
/feishu:configure
```

Run with no arguments to enter interactive mode. Or pass values directly:

```
/feishu:configure cli_xxxxxxxx xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

---

## Implementation steps

1. **Determine credentials:**
   - If arguments are provided: first arg = `FEISHU_APP_ID`, second = `FEISHU_APP_SECRET`
   - Otherwise: prompt the user to paste their App ID and App Secret
   - App IDs look like `cli_xxxxxxxxxxxxxxxx` (starts with `cli_`)
   - App Secrets are 32+ character hex strings

2. **Validate:**
   - App ID must be non-empty
   - App Secret must be non-empty
   - Warn (but don't block) if App ID doesn't start with `cli_`

3. **Write the .env file:**
   ```
   mkdir -p ~/.claude/channels/feishu
   ```
   Write to `~/.claude/channels/feishu/.env`:
   ```
   FEISHU_APP_ID=cli_xxxxxxxx
   FEISHU_APP_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   ```
   Set permissions: `chmod 600 ~/.claude/channels/feishu/.env`

4. **Print confirmation:**
   ```
   Feishu credentials saved to ~/.claude/channels/feishu/.env
   App ID: cli_xxxxxxxx

   To start the channel: bun server.ts (or Claude Code will start it automatically)
   Run /feishu:setup if you haven't configured your Feishu app yet.
   ```

5. **Check for existing access.json** — if it doesn't exist, suggest running
   `/feishu:access policy pairing` or note that pairing is the default.

---

## Security notes

- Never print the full App Secret to the terminal — only confirm it was saved
- The `.env` file is chmod 600 so other users on the same machine can't read it
- If credentials already exist in the `.env`, ask before overwriting

---

## Static mode (optional)

If the user wants to lock access config at startup (prevents runtime pairing):

```
/feishu:configure --static
```

Adds `FEISHU_ACCESS_MODE=static` to the `.env` file. In this mode:
- `dmPolicy: pairing` is downgraded to `allowlist` at startup
- Access config is snapshotted at boot and never re-read
- Useful for production deployments with fixed access lists
