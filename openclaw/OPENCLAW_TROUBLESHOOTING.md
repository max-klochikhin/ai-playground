# OpenClaw Mac Mini — Troubleshooting & Setup Notes

> Created: 2026-03-07  
> Environment: Mac mini, macOS, OpenClaw v2026.3.7

---

## Summary

After installing OpenClaw on the Mac mini, the gateway LaunchAgent was repeatedly
crashing (exit code 1) and the Telegram bot was not responding to messages.
This document describes each issue found, the root cause, and the fix applied.

---

## Issue 1: Gateway blocked from starting — `gateway.mode` not set

### Symptom
The LaunchAgent `ai.openclaw.gateway` was in a crash-loop visible via:

```
launchctl list | grep ai.openclaw
# showing non-zero exit code
```

`/Users/max/.openclaw/logs/gateway.err.log` was full of:

```
Gateway start blocked: set gateway.mode=local (current: unset) or pass --allow-unconfigured.
```

### Root Cause
OpenClaw requires `gateway.mode` to be explicitly set in `~/.openclaw/openclaw.json`
before the service will start. It was left unset after the initial installation wizard.

### Fix Applied
1. Added `"mode": "local"` inside the `gateway` object in `/Users/max/.openclaw/openclaw.json`.
2. Added `--allow-unconfigured` flag to the ProgramArguments in the LaunchAgent plist
   as a safety net to prevent this crash-loop from happening again in the future.

```json
// ~/.openclaw/openclaw.json  — gateway section
"gateway": {
  "mode": "local",
  "auth": {
    "mode": "token",
    "token": "..."
  }
}
```

---

## Issue 2: Telegram bot not configured — token missing from OpenClaw config

### Symptom
Even after the gateway started successfully, messages sent to the Telegram bot
(`@open_claw_ai_assistant_bot`) received no response.

`openclaw channels list` showed:

```
Telegram default: not configured, token=none, enabled
```

### Root Cause
OpenClaw stores its own Telegram bot token in `~/.openclaw/openclaw.json`.
Even though a `TELEGRAM_BOT_TOKEN` existed in `/Users/max/openclaw/.env`,
OpenClaw does **not** automatically read from that legacy `.env` file at runtime
— the token must be explicitly registered via `openclaw channels add`.

### Fix Applied
Ran the following command to register the bot token in OpenClaw's config:

```bash
/opt/homebrew/opt/node/bin/node /Users/max/openclaw/openclaw.mjs \
  channels add --channel telegram --token 8756536016:AAHInF4IcHHzQEBg_NilP2cR0x4SDtQC0y4
```

This wrote `"botToken": "..."` into the `channels.telegram` section of
`~/.openclaw/openclaw.json`.

---

## Issue 3: User allowlist missing sender's Telegram ID

### Symptom
Bot started successfully but still no response to messages.

### Root Cause
The `channels.telegram.allowFrom` list in `openclaw.json` contained only one user ID
(`1297932849`) but the user sending messages had a different Telegram user ID
(`135208609`). OpenClaw uses a strict allowlist — messages from unlisted IDs are
silently dropped.

### Fix Applied
Added the second user ID to the allowlist in `/Users/max/.openclaw/openclaw.json`:

```json
"channels": {
  "telegram": {
    "enabled": true,
    "dmPolicy": "allowlist",
    "botToken": "...",
    "allowFrom": [
      1297932849,
      135208609
    ],
    ...
  }
}
```

---

## Issue 4: LaunchAgent missing WorkingDirectory — .env not loaded on boot

### Symptom
After a Mac restart, the service started without loading environment variables from
`/Users/max/openclaw/.env`, causing potential API key failures.

### Root Cause
LaunchAgents do not inherit a working directory, so relative `.env` file lookups
fail. Without `WorkingDirectory` set, the Node process could not find `.env`
relative to the OpenClaw project directory.

### Fix Applied
Added `WorkingDirectory` to `/Users/max/Library/LaunchAgents/ai.openclaw.gateway.plist`:

```xml
<key>WorkingDirectory</key>
<string>/Users/max/openclaw</string>
```

---

## Issue 5: Gemini API limits reached — Shared account interference

### Symptom
OpenClaw was frequently hitting Gemini API rate limits (RPM/TPM) because the API key was shared with another system.

### Root Cause
Google AI Studio enforces free-tier limits per account. Using one key for multiple active systems caused them to compete for the same 15 RPM / 1500 RPD budget.

### Fix Applied (2026-03-07)
1. **Created Dedicated Account**: Logged into a new Google account (`avneet.everett.2021@gmail.com`).
2. **Generated New API Key**: Created a fresh key in AI Studio.
3. **Verified Key**: Verified the key works via a standalone `curl` request to the Gemini API.
4. **Isolated Config**: Updated the `GEMINI_API_KEY` in `/Users/max/openclaw/.env`. This decouples OpenClaw from the other system's limits.

---

## Issue 6: Migration to Google Vertex AI — A Chain of 8 Issues

> Date: 2026-04-04  
> Goal: Switch the agent from `google/gemini-flash-latest` to `google-vertex/gemini-3-flash-preview`  
> Credentials used: OAuth2 refresh_token from the neighboring Telegram bot `kleinanzeigen-bot`

---

### Issue 6.1 — `No API key found for provider "google-vertex"`

**Symptom:**
```
Agent failed before reply: No API key found for provider "google-vertex".
Auth store: /Users/max/.openclaw/agents/main/agent/auth-profiles.json
```

**Root Cause:**  
The `auth-profiles.json` file was in an outdated flat format (`{"google": {...}}`), whereas OpenClaw expects a wrapped format: `{"version": 1, "profiles": {...}}`. During loading, the `coerceAuthStore` function saw the unrecognized format and returned an empty store.

**Fix Applied:**  
Rewrote `/Users/max/.openclaw/agents/main/agent/auth-profiles.json` using the correct structure:

```json
{
  "version": 1,
  "profiles": {
    "google": {
      "default": { "apiKey": "AIzaSy..." }
    },
    "google-vertex": {
      "default": { "apiKey": "AIzaSy..." }
    }
  }
}
```

---

### Issue 6.2 — `Vertex AI requires a project ID`

**Symptom:**
```
Vertex AI requires a project ID. Set GOOGLE_CLOUD_PROJECT/GCLOUD_PROJECT or pass project in options.
```

**Root Cause:**  
LaunchAgent starts the daemon without environment variables from the `.env` file. `GOOGLE_CLOUD_PROJECT` was not passed to the process.

**Fix Applied:**  
Added to `/Users/max/Library/LaunchAgents/ai.openclaw.gateway.plist`:

```xml
<key>GOOGLE_CLOUD_PROJECT</key>
<string>gen-lang-client-0431347096</string>
```

---

### Issue 6.3 — `Vertex AI requires a location`

**Symptom:**
```
Vertex AI requires a location. Set GOOGLE_CLOUD_LOCATION or pass location in options.
```

**Fix Applied:**  
Added to the plist:

```xml
<key>GOOGLE_CLOUD_LOCATION</key>
<string>global</string>
```

---

### Issue 6.4 — `Could not load the default credentials`

**Symptom:**
```
Could not load the default credentials. Browse to https://cloud.google.com/docs/authentication/...
```

**Root Cause:**  
Vertex AI requires OAuth2 credentials. The `gcloud_credentials.json` file from the neighboring bot (`kleinanzeigen-bot`) contains a `client_id`, `client_secret`, and `refresh_token` of type `authorized_user` — these must be utilized.

**Fix Applied:**  
Added to the plist:

```xml
<key>GOOGLE_APPLICATION_CREDENTIALS</key>
<string>/Users/max/kleinanzeigen_bot/tools/telegram_llm_bot/config/gcloud_credentials.json</string>
```

---

### Issue 6.5 — `Project/location and API key are mutually exclusive`

**Symptom:**
```
Project/location and API key are mutually exclusive in the client initializer.
```

**Root Cause:**  
The `@google/genai` SDK does not allow passing `project`/`location` and `apiKey` simultaneously when `vertexai: true` is set.

**Fix Applied:**  
Patched the file `pi-ai/dist/providers/google-vertex.js` — the `createClient` function now does not pass the `apiKey` when `project`/`location` is present:

```javascript
function createClient(model, project, location, optionsHeaders) {
    const httpOptions = {};
    const accessToken = process.env.GOOGLE_ACCESS_TOKEN;
    if (accessToken) {
        httpOptions.headers = { 'Authorization': 'Bearer ' + accessToken };
    }
    return new GoogleGenAI({
        vertexai: true,
        project,
        location,
        apiVersion: API_VERSION,
        httpOptions: Object.keys(httpOptions).length ? httpOptions : undefined,
    });
}
```

---

### Issue 6.6 — `API keys are not supported by this API. Expected OAuth2 access token` (HTTP 401)

**Symptom:**
```json
{"error": {"code": 401, "message": "API keys are not supported by this API.
Expected OAuth2 access token...", "status": "UNAUTHENTICATED"}}
```

**Root Cause:**  
The Vertex AI endpoint (`aiplatform.googleapis.com`) only accepts OAuth2 Bearer tokens. Keys of type `AIzaSy...` (Google AI Studio) do not work here.

**Fix Applied:**  
1. Created a script `/Users/max/openclaw/scripts/refresh-vertex-token.sh`, which:
   - Reads `client_id`, `client_secret`, and `refresh_token` from `gcloud_credentials.json`
   - Exchanges the `refresh_token` for a fresh `access_token` via `https://oauth2.googleapis.com/token`
   - Writes the token into the plist as `GOOGLE_ACCESS_TOKEN`
   - Restarts the daemon
2. The token is passed in the HTTP header `Authorization: Bearer <token>` via `httpOptions.headers` inside `createClient`.

---

### Issue 6.7 — `Cannot convert undefined or null to object` (first version)

**Symptom:**
```
error=Cannot convert undefined or null to object
```

**Root Cause:**  
Even with the Bearer token in `httpOptions.headers`, the `@google/genai` SDK still invoked `this.googleAuth.getRequestHeaders(url)` (ADC) inside `addGoogleAuthHeaders`. The ADC function did not check if `Authorization` was already set, causing it to fail with this error.

**Attempted Fix (incorrect file):**  
Patched `@google/genai/dist/node/index.cjs` to add an early-return:
```javascript
async addGoogleAuthHeaders(headers, url) {
    if (headers.get('Authorization') || headers.get('authorization')) return;
    // ...
}
```
However, Node.js loaded the ES module version (`index.mjs`) instead of CJS. The patch had no effect.

---

### Issue 6.8 — `Cannot convert undefined or null to object` (final)

**Symptom:**  
The same error, despite the patch on `index.cjs`.

**Diagnostics:**  
Stack trace obtained by adding a try/catch in `google-vertex.js`:

```
at NodeAuth.addGoogleAuthHeaders
  (file:///.../node_modules/@google/genai/dist/node/index.mjs:18402:29)
at async ApiClient.getHeadersInternal
  (file:///.../node_modules/@google/genai/dist/node/index.mjs:12283:9)
```

**Key Discovery:**  
The file is loaded as `index.mjs` (ES module), not `index.cjs`. The previous patch was applied to the wrong file.

**Final Fix Applied:**  
Patched `/Users/max/openclaw/node_modules/.pnpm/@google+genai@1.43.0/node_modules/@google/genai/dist/node/index.mjs`:

```javascript
// Around line ~18395
async addGoogleAuthHeaders(headers, url) {
    // PATCH: skip ADC if Bearer token is already set
    if (headers.get('Authorization') || headers.get('authorization')) {
        return;
    }
    if (this.googleAuth === undefined) {
        throw new Error('Trying to set google-auth headers but googleAuth is unset');
    }
    const authHeaders = await this.googleAuth.getRequestHeaders(url);
    // ...
}
```

After this patch, a direct API call successfully returned the response:
```
> Reply with just: OK
OK
DONE
```

---

### Automatic Token Refresh

OAuth2 access tokens expire in ~1 hour. To automate refresh:

**Script:** `/Users/max/openclaw/scripts/refresh-vertex-token.sh`

```bash
#!/bin/bash
# Updates GOOGLE_ACCESS_TOKEN in LaunchAgent plist and restarts the daemon
# Runs via cron every 45 minutes

CREDS='/Users/max/kleinanzeigen_bot/tools/telegram_llm_bot/config/gcloud_credentials.json'
PLIST='/Users/max/Library/LaunchAgents/ai.openclaw.gateway.plist'
LOG='/Users/max/.openclaw/logs/token-refresh.log'

CLIENT_ID=$(python3 -c "import json; d=json.load(open('$CREDS')); print(d['client_id'])")
CLIENT_SECRET=$(python3 -c "import json; d=json.load(open('$CREDS')); print(d['client_secret'])")
REFRESH_TOKEN=$(python3 -c "import json; d=json.load(open('$CREDS')); print(d['refresh_token'])")

RESPONSE=$(curl -s -X POST https://oauth2.googleapis.com/token \
  -d "client_id=${CLIENT_ID}&client_secret=${CLIENT_SECRET}&refresh_token=${REFRESH_TOKEN}&grant_type=refresh_token")

ACCESS_TOKEN=$(echo "$RESPONSE" | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('access_token',''))")

if [ -z "$ACCESS_TOKEN" ]; then
  echo "$(date): ERROR: failed to obtain token: $RESPONSE" >> "$LOG"
  exit 1
fi

# Update or add GOOGLE_ACCESS_TOKEN in the plist
python3 << PYEOF
import re, subprocess
plist_path = '$PLIST'
token = '$ACCESS_TOKEN'

with open(plist_path, 'r') as f:
    content = f.read()

if '<key>GOOGLE_ACCESS_TOKEN</key>' in content:
    # Update existing token
    content = re.sub(
        r'(<key>GOOGLE_ACCESS_TOKEN</key>\s*<string>)[^<]*(</string>)',
        f'\\g<1>{token}\\g<2>',
        content
    )
else:
    # Add new token
    old = '    <key>GEMINI_API_KEY</key>'
    new = f'    <key>GOOGLE_ACCESS_TOKEN</key>\n    <string>{token}</string>\n    <key>GEMINI_API_KEY</key>'
    content = content.replace(old, new, 1)

with open(plist_path, 'w') as f:
    f.write(content)
PYEOF

# Restart the daemon
launchctl unload "$PLIST" 2>/dev/null
sleep 1
launchctl load "$PLIST"

echo "$(date): Token updated and daemon restarted" >> "$LOG"
```

**Cron Job (every 45 minutes):**

```
*/45 * * * * /Users/max/openclaw/scripts/refresh-vertex-token.sh
```

Added via `crontab -e` for user `max`.

---

### Final Status after all fixes (2026-04-04)

| Component | Value |
|---|---|
| Agent model | `google-vertex/gemini-3-flash-preview` |
| Vertex project | `gen-lang-client-0431347096` |
| Vertex location | `global` |
| Credentials source | `kleinanzeigen-bot/config/gcloud_credentials.json` (authorized_user) |
| Token in plist | `GOOGLE_ACCESS_TOKEN` (updated by cron every 45 min) |
| Patched files | `pi-ai/dist/providers/google-vertex.js`, `@google/genai/dist/node/index.mjs` |
| Refresh script | `/Users/max/openclaw/scripts/refresh-vertex-token.sh` |
| Autostart | cron `*/45 * * * *` |

### Important Warning about node_modules Patches

Both patched files are inside `node_modules` and will be overwritten during package updates (`pnpm install`, `pnpm update`). After any dependency update, you must re-apply the patches:

1. **`pi-ai/dist/providers/google-vertex.js`** — `createClient` with `Authorization: Bearer` inject
2. **`@google/genai/dist/node/index.mjs`** — `addGoogleAuthHeaders` with early-return when token is present

The long-term solution is to wait for official `GOOGLE_ACCESS_TOKEN` support in `pi-ai` or create a fork with a persistent patch.

---

## Final State (as of 2026-04-04)

| Component | Value |
|---|---|
| LaunchAgent | `ai.openclaw.gateway` (loaded, RunAtLoad=true, KeepAlive=true) |
| Gateway port | `18789` |
| Bot username | `@open_claw_ai_assistant_bot` |
| Agent model | `google-vertex/gemini-3-flash-preview` |
| Vertex project | `gen-lang-client-0431347096` |
| Vertex location | `global` |
| OAuth2 credentials | `/Users/max/kleinanzeigen_bot/tools/telegram_llm_bot/config/gcloud_credentials.json` |
| Token refresh | cron `*/45 * * * *` → `/Users/max/openclaw/scripts/refresh-vertex-token.sh` |
| Config file | `/Users/max/.openclaw/openclaw.json` |
| Auth profiles | `/Users/max/.openclaw/agents/main/agent/auth-profiles.json` |
| Plist file | `/Users/max/Library/LaunchAgents/ai.openclaw.gateway.plist` |
| Logs (stdout) | `/Users/max/.openclaw/logs/gateway.log` |
| Logs (stderr) | `/Users/max/.openclaw/logs/gateway.err.log` |
| Detailed logs | `/tmp/openclaw/openclaw-YYYY-MM-DD.log` |
| Token refresh log | `/Users/max/.openclaw/logs/token-refresh.log` |

---

## Will it survive a Mac restart?

**Yes.** The LaunchAgent has:
- `RunAtLoad = true` — starts automatically when you log in
- `KeepAlive = true` — restarts automatically if it crashes
- `WorkingDirectory` configured — so `.env` is loaded correctly
- `gateway.mode = "local"` set in config — no longer blocked on startup
- `botToken` stored in config — Telegram channel is fully configured
- `GOOGLE_CLOUD_PROJECT`, `GOOGLE_CLOUD_LOCATION`, `GOOGLE_APPLICATION_CREDENTIALS`, `GOOGLE_ACCESS_TOKEN` — configured in plist

**The only thing that won't survive a reboot without cron:**  
The OAuth2 access token (`GOOGLE_ACCESS_TOKEN`) in the plist expires in 1 hour. The cron job `*/45 * * * *` updates it automatically. After a Mac restart, cron starts automatically with the system.

**To check status after a restart:**

```bash
launchctl list | grep ai.openclaw
# Expected: PID   0   ai.openclaw.gateway  (0 = no error)

tail -f /Users/max/.openclaw/logs/gateway.log
# Should show: [telegram] [default] starting provider (@open_claw_ai_assistant_bot)

tail -f /Users/max/.openclaw/logs/token-refresh.log
# Should show recent: Token updated and daemon restarted
```

**To restart the service manually:**

```bash
launchctl unload ~/Library/LaunchAgents/ai.openclaw.gateway.plist
launchctl load   ~/Library/LaunchAgents/ai.openclaw.gateway.plist
```

**To manually refresh the Vertex AI token:**

```bash
/Users/max/openclaw/scripts/refresh-vertex-token.sh
```

---

## Issue: Vertex 401 after ADC was removed (2026-09-08)

### Symptom
`gateway.err.log` showed `401 UNAUTHENTICATED` / `ACCESS_TOKEN_TYPE_UNSUPPORTED` on
`google-vertex/gemini-3-flash-preview`. LaunchAgent still pointed
`GOOGLE_APPLICATION_CREDENTIALS` at
`/Users/max/kleinanzeigen_bot/tools/telegram_llm_bot/config/gcloud_credentials.json`,
which no longer exists.

### Fix
Switched the gateway to LiteLLM. See [`LITELLM.md`](LITELLM.md). Do not restore the
Vertex env vars unless you put a valid ADC back.
