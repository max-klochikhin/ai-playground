# 🚀 Master Plan: OpenClaw Deployment on Mac mini (Remote via SSH)

This document serves as the source of truth for the deployment process ON THE REMOTE MAC MINI. All commands are executed from the local laptop via SSH.

## 📍 Current Status (Status Quo)

- **Remote Host**: Mac mini (Apple Silicon) at `crazy-home.keenetic.name:2022`
- **User**: `Max`
- **Remote Project Path**: `/Users/max/openclaw`
- **Telegram User ID**: `1297932849` (Numeric ID from @userinfobot)
- **Kleinanzeigen ID**: `135208609` (Used only for website-related tasks, NOT Telegram auth)
- **GitHub**: [max-klochikhin/openclaw](https://github.com/max-klochikhin/openclaw)
- **Gemini API Level**: Tier 2 (Upgraded/Active)
- **Gemini Key**: `AIzaSyD...Gqsb46xLGRDDA` (Last confirmed active April 2026)
- **Local Proxy folder**: `/Users/max.klochikhin/projects/openclaw` (Documentation and tracking).

## 📋 Implementation Roadmap

### Step 1: Environment Preparation (DONE)

1. **Node.js**: Installed v25+ via brew on Mac mini.
2. **PM2**: Installed globally via npm on Mac mini.
3. **pnpm**: Installed via brew on Mac mini.

### Step 2: Configuration (DONE)

OpenClaw primarily uses `~/.openclaw/openclaw.json` for configuration.

1. **Wizard/Doctor**: Run `pnpm start setup` and `pnpm start doctor --fix` to initialize.
2. **Telegram Auth (DONE)**:
   - Config path: `channels.telegram.allowFrom`
   - Command: `pnpm start config set channels.telegram.allowFrom "[1297932849]"` (Correct Telegram ID).
3. **LLM Provider (UPDATED 2026-09-08)**:
   - All model I/O goes through LiteLLM: `http://134.98.138.255:4000`
   - Default model: `litellm/gemini-3-flash-preview`
   - LaunchAgent uses `LITELLM_API_KEY` (same master key as the Kleinanzeigen bot). Vertex / Gemini env vars removed.
   - Details: [`LITELLM.md`](LITELLM.md), example config: [`mac-mini/openclaw.json.example`](mac-mini/openclaw.json.example)

### Step 3: Remote Installation (DONE)

1. Install dependencies: `pnpm install`
2. Build the project: `pnpm build`

### Step 4: Launch & Persistence (DONE)

1. **Native Daemon**: Instead of PM2, use the built-in daemon: `pnpm start daemon install`.
2. This creates a LaunchAgent at `~/Library/LaunchAgents/ai.openclaw.gateway.plist`.
3. Logs are available at `~/.openclaw/logs/gateway.log`.

### Step 5: Optimization

1. Prevent sleep: `sudo pmset -a preventsleep 1`

---

## 📜 Installation Log (Remote)

| Step | Action                | Remote Command                          | Result | Rollback Command                       |
| :--- | :-------------------- | :-------------------------------------- | :----- | :------------------------------------- |
| 1    | Install Node/pnpm     | `brew install node pnpm`                | Done   | `brew uninstall node pnpm`             |
| 2    | Install PM2           | `npm install -g pm2`                    | Done   | `npm uninstall -g pm2`                 |
| 3    | Install dependencies  | `pnpm install`                          | Done   | `rm -rf ~/openclaw/node_modules`       |
| 4    | Build project         | `pnpm build`                            | Done   | `rm -rf ~/openclaw/dist`               |
| 5    | Create .env           | `echo ... > .env`                       | Done   | `rm ~/openclaw/.env`                   |
| 6    | Create Workspace      | `mkdir -p /Users/max/projects`          | Done   | `rmdir /Users/max/projects`            |
| 7    | Initialize Config     | `pnpm start setup`                      | Done   | `rm -rf ~/.openclaw`                   |
| 8    | Fix Doctor Issues     | `pnpm start doctor --fix`               | Done   | N/A                                    |
| 9    | Set Telegram Access   | `pnpm start config set ...`             | Done   | Reset config                           |
| 10   | Install Daemon        | `pnpm start daemon install`             | Done   | `launchctl unload ...`                 |
| 11   | Gemini Config         | `models set google/gemini-flash-latest` | Done   | Reset models                           |
| 12   | Antigravity.app       | Copy to /Applications                   | Done   | `rm -rf /Applications/Antigravity.app` |
| 13   | Correct Telegram ID   | Set ID `1297932849`                     | Done   | Back to `135208609`                    |
| 14   | Verify Bot Connection | Send "test" message                     | Done   | N/A                                    |

---

## 🛠 Troubleshooting & Notes

- **Agent-Specific Auth Profiles**: Each agent (e.g. `main`) can have its own auth store at `~/.openclaw/agents/<id>/agent/auth-profiles.json`. This **overrides** both global `.env` and `openclaw.json` settings. If you get a "No API key found" or "invalid key" error for a specific agent, check and update this JSON file.
- **LiteLLM (2026-09-08)**: OpenClaw no longer calls Vertex from the Mac mini. The proxy on `134.98.138.255:4000` talks to Vertex. See [`LITELLM.md`](LITELLM.md).
- **Node.js OAuth Requirement**: When using standard OAuth credentials (`authorized_user` type), the Node.js `google-auth-library` strictly requires the `"token_uri": "https://oauth2.googleapis.com/token"` key to be present in `gcloud_credentials.json`. Without it, token refresh will crash with `Cannot convert undefined or null to object`.

---

## 🧹 Cleanup (Rollback)

To completely remove OpenClaw from your Mac mini:

1. **Uninstall Daemon**: `ssh ... "cd ~/openclaw && pnpm start daemon uninstall"`
2. **Remove Config**: `ssh ... "rm -rf ~/.openclaw"`
3. **Remove Project**: `ssh ... "rm -rf ~/openclaw"` (WARNING: backup .env if needed)
4. **Remove Dependencies**: `ssh ... "npm uninstall -g pm2 && brew uninstall node pnpm"` (If you want to remove tools too)

## � Future Ideas / Backlog

- **Direct Antigravity CLI**: Install and use Antigravity CLI directly on the Mac mini to manage the project without local SSH overhead.
- **Enhanced Remote Dev**: Fully configure the remote environment for direct development (IDE, terminal optimizations).

## �📎 Links and Commands

- **SSH Command**:
  `ssh -p 2022 -i /Users/max.klochikhin/projects/kleinanzeigen-bot/tools/virtual_machines/mac_mini/ssh-keys/id_ed25519_macmini Max@crazy-home.keenetic.name`
