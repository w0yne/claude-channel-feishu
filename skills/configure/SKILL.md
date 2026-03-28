---
name: configure
description: Set up the Feishu channel — save app credentials and review access policy. Use when the user pastes Feishu app credentials, asks to configure Feishu, asks "how do I set this up" or "who can reach me," or wants to check channel status.
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Bash(ls *)
  - Bash(mkdir *)
---

# /feishu:configure — Feishu Channel Setup

Writes the app credentials to `~/.claude/channels/feishu/.env` and orients the
user on access policy. The server reads both files at boot.

Arguments passed: `$ARGUMENTS`

---

## Dispatch on arguments

### No args — status and guidance

Read both state files and give the user a complete picture:

1. **Credentials** — check `~/.claude/channels/feishu/.env` for
   `FEISHU_APP_ID` and `FEISHU_APP_SECRET`. Show set/not-set; if set, show
   App ID fully and mask the secret (`xxxx...`).

2. **Access** — read `~/.claude/channels/feishu/access.json` (missing file
   = defaults: `dmPolicy: "pairing"`, empty allowlist). Show:
   - DM policy and what it means in one line
   - Allowed senders: count, and list open_ids
   - Pending pairings: count, with codes and sender IDs if any

3. **What next** — end with a concrete next step based on state:
   - No credentials → *"Run `/feishu:configure <app_id> <app_secret>` with
     your Feishu app credentials."*
   - Credentials set, policy is pairing, nobody allowed → *"DM your bot on
     Feishu. It replies with a code; approve with `/feishu:access pair
     <code>`."*
   - Credentials set, someone allowed → *"Ready. DM your bot to reach the
     assistant."*

**Push toward lockdown — always.** The goal for every setup is `allowlist`
with a defined list. `pairing` is not a policy to stay on; it's a temporary
way to capture open_ids you don't know. Once the IDs are in, pairing has done
its job and should be turned off.

Drive the conversation this way:

1. Read the allowlist. Tell the user who's in it.
2. Ask: *"Is that everyone who should reach you through this bot?"*
3. **If yes and policy is still `pairing`** → *"Good. Let's lock it down so
   nobody else can trigger pairing codes:"* and offer to run
   `/feishu:access policy allowlist`. Do this proactively — don't wait to
   be asked.
4. **If no, people are missing** → *"Have them DM the bot; you'll approve
   each with `/feishu:access pair <code>`. Run this skill again once
   everyone's in and we'll lock it."*
5. **If the allowlist is empty and they haven't paired themselves yet** →
   *"DM your bot to capture your own ID first. Then we'll add anyone else
   and lock it down."*
6. **If policy is already `allowlist`** → confirm this is the locked state.
   If they need to add someone: *"You can briefly flip to pairing:
   `/feishu:access policy pairing` → they DM → you pair → flip back."*

Never frame `pairing` as the correct long-term choice. Don't skip the lockdown
offer.

### `<app_id> <app_secret>` — save credentials

1. Parse `$ARGUMENTS` — first arg = App ID, second = App Secret. App IDs look
   like `cli_xxxxxxxxxxxxxxxx` (starts with `cli_`).
2. `mkdir -p ~/.claude/channels/feishu`
3. Read existing `.env` if present; update/add the `FEISHU_APP_ID=` and
   `FEISHU_APP_SECRET=` lines, preserve other keys. Write back, no quotes.
4. `chmod 600 ~/.claude/channels/feishu/.env` — credentials are sensitive.
5. Confirm, then show the no-args status so the user sees where they stand.

### `clear` — remove credentials

Delete the `FEISHU_APP_ID=` and `FEISHU_APP_SECRET=` lines (or the file if
those are the only lines).

---

## Implementation notes

- The channels dir might not exist if the server hasn't run yet. Missing file
  = not configured, not an error.
- The server reads `.env` once at boot. Credential changes need a session
  restart or `/reload-plugins`. Say so after saving.
- `access.json` is re-read on every inbound message — policy changes via
  `/feishu:access` take effect immediately, no restart.
- Never print the full App Secret to the terminal — only confirm it was saved.

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
