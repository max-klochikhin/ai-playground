# OpenClaw on Mac mini via LiteLLM

Applied 2026-09-08 on the Mac mini (`ai.openclaw.gateway`).

OpenClaw no longer calls Vertex / Gemini directly. All model I/O goes through the same LiteLLM proxy as the Kleinanzeigen Telegram bot.

| Item | Value |
|---|---|
| Host | Mac mini, user `Max` |
| Live config | `~/.openclaw/openclaw.json` |
| LaunchAgent | `~/Library/LaunchAgents/ai.openclaw.gateway.plist` |
| Primary model | `litellm/gemini-3-flash-preview` |
| Proxy | `http://134.98.138.255:4000` |
| Also registered | `litellm/gemini-2.5-flash`, `litellm/gemini-2.0-flash` |

Sanitized copy of the live config: [`mac-mini/openclaw.json.example`](mac-mini/openclaw.json.example).

## Why

Vertex on OpenClaw was failing with `401 UNAUTHENTICATED` / `ACCESS_TOKEN_TYPE_UNSUPPORTED`. The bot ADC file was removed from the Kleinanzeigen tree (`tools/telegram_llm_bot/config/gcloud_credentials.json`). OpenClaw still pointed at that path.

## What changed on the Mac mini

1. Backup: `~/.openclaw/openclaw.json.bak.pre-litellm.20260908_175720`
2. Backup: `~/Library/LaunchAgents/ai.openclaw.gateway.plist.bak.pre-litellm.20260908_175720`
3. Added `models.providers.litellm` (OpenAI-compatible, `api: openai-completions`)
4. Set `agents.defaults.model.primary` to `litellm/gemini-3-flash-preview`
5. Removed from the LaunchAgent: `GEMINI_API_KEY`, `GOOGLE_ACCESS_TOKEN`, `GOOGLE_APPLICATION_CREDENTIALS`, `GOOGLE_CLOUD_PROJECT`, `GOOGLE_CLOUD_LOCATION`
6. Added `LITELLM_API_KEY` to the LaunchAgent (same master key as the Telegram bot `OPENAI_API_KEY`)
7. Wrote the master key into `openclaw.json` `models.providers.litellm.apiKey` — OpenClaw 2026.3.7 did **not** expand `${LITELLM_API_KEY}` from the process env (`missing env var "LITELLM_API_KEY"`). Keep the real key only on the Mac mini file (mode `600`). The example in git uses the placeholder.

Restart after env/plist edits:

```bash
UID_NUM=$(id -u)
PLIST="$HOME/Library/LaunchAgents/ai.openclaw.gateway.plist"
launchctl bootout "gui/${UID_NUM}/ai.openclaw.gateway"
launchctl bootstrap "gui/${UID_NUM}" "$PLIST"
```

`kickstart` alone is not enough for new LaunchAgent env.

## Verified

Local one-turn (Mac mini):

```bash
/opt/homebrew/opt/node/bin/node /Users/max/openclaw/dist/index.js \
  agent --agent main --message "Reply with exactly: litellm-openclaw-ok"
```

Result: `litellm-openclaw-ok`

Gateway log: `agent model: litellm/gemini-3-flash-preview`

LiteLLM journal (UTC = CEST−2), source IP of the Mac mini `188.192.84.7`:

```text
15:58:23  POST /chat/completions      200
15:58:25  POST /v1/chat/completions   200
15:58:28  POST /chat/completions      200
```

OpenClaw hits both `/chat/completions` and `/v1/chat/completions`. Keep `baseUrl` without a trailing path, or with `/v1` only if you confirm the client does not double-prefix.

## Rollback

```bash
cp ~/.openclaw/openclaw.json.bak.pre-litellm.20260908_175720 ~/.openclaw/openclaw.json
cp ~/Library/LaunchAgents/ai.openclaw.gateway.plist.bak.pre-litellm.20260908_175720 \
   ~/Library/LaunchAgents/ai.openclaw.gateway.plist
# then bootout + bootstrap as above
```

That restores Vertex. It will fail again until ADC / a valid Google token is back.
