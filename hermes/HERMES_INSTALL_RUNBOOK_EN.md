# Runbook: Migrating Mac mini from OpenClaw to Hermes

> Goal: install Hermes on the Mac mini, reuse **existing Vertex credentials** and the **OpenClaw Telegram bot** (`@open_claw_ai_assistant_bot`), then remove OpenClaw.  
> All commands below run **on the Mac mini** unless stated otherwise. You can execute them locally over SSH.

---

## Short answer: when to remove OpenClaw

| Stage | What to do with OpenClaw | Why |
|-------|--------------------------|-----|
| Before installing Hermes | **Stop the gateway only**, leave config in place | You need the bot token and Vertex paths from `~/.openclaw/` |
| After `hermes chat`, before Telegram | **Fully stop / uninstall the gateway / daemon** | One bot token can only be used by one gateway |
| After Hermes works in Telegram | **Remove OpenClaw** | Rollback is no longer needed |

**Do not remove OpenClaw entirely until `hermes chat` works.**  
**Do not connect Hermes to Telegram while the OpenClaw gateway is still running.**

Recommended sequence:

```text
Stop OpenClaw gateway
→ Install Hermes
→ Vertex (reuse existing creds)
→ hermes chat
→ Stop/uninstall OpenClaw gateway
→ Hermes Telegram (same bot)
→ Verify
→ Remove OpenClaw completely
```

---

## 0. Context (from the current OpenClaw setup)

| Parameter | Value |
|-----------|-------|
| Host | `crazy-home.keenetic.name:2022` |
| User | `Max` |
| SSH | `ssh -p 2022 -i /Users/max.klochikhin/projects/kleinanzeigen-bot/tools/virtual_machines/mac_mini/ssh-keys/id_ed25519_macmini Max@crazy-home.keenetic.name` |
| Telegram bot | `@open_claw_ai_assistant_bot` |
| Telegram user ID (primary) | `1297932849` |
| OpenClaw config | `~/.openclaw/openclaw.json` |
| OpenClaw LaunchAgent | `~/Library/LaunchAgents/ai.openclaw.gateway.plist` |
| OpenClaw logs | `~/.openclaw/logs/gateway.log` |
| Vertex project | `gen-lang-client-0431347096` |
| Vertex region | `global` |
| Vertex credentials (already in place) | `/Users/max/kleinanzeigen_bot/tools/telegram_llm_bot/config/gcloud_credentials.json` |
| OpenClaw project path | `/Users/max/openclaw` |

Do **not** create new GCP keys or a service account — reuse what already works with OpenClaw.

---

## 1. Preparation: backup and stop the OpenClaw gateway

### 1.1 SSH into the Mac mini

```bash
ssh -p 2022 -i /Users/max.klochikhin/projects/kleinanzeigen-bot/tools/virtual_machines/mac_mini/ssh-keys/id_ed25519_macmini Max@crazy-home.keenetic.name
```

### 1.2 Save OpenClaw secrets (for reuse in Hermes)

```bash
mkdir -p ~/hermes-migration-backup
cp ~/.openclaw/openclaw.json ~/hermes-migration-backup/
cp ~/openclaw/.env ~/hermes-migration-backup/ 2>/dev/null || true
cp ~/Library/LaunchAgents/ai.openclaw.gateway.plist ~/hermes-migration-backup/ 2>/dev/null || true
```

Extract the Telegram bot token:

```bash
# token is in openclaw.json → channels.telegram.botToken
python3 - <<'PY'
import json, pathlib
p = pathlib.Path.home() / ".openclaw/openclaw.json"
data = json.loads(p.read_text())
token = data.get("channels", {}).get("telegram", {}).get("botToken", "")
print("TELEGRAM_BOT_TOKEN found:" if token else "TELEGRAM_BOT_TOKEN missing", bool(token))
PY
```

Save the token in a password manager or temporarily in `~/hermes-migration-backup/telegram-token.txt` with mode `600`.

### 1.3 Stop the OpenClaw gateway (but do not delete the project)

```bash
cd ~/openclaw
pnpm start daemon stop 2>/dev/null || true
pnpm start daemon uninstall 2>/dev/null || true
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/ai.openclaw.gateway.plist 2>/dev/null || true
launchctl list | grep -i openclaw || echo "OpenClaw gateway stopped"
```

Verification: message the bot in Telegram — **there should be no reply**. That is expected at this stage.

---

## 2. Install Hermes

### 2.1 CLI install

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
source ~/.zshrc
hermes --help
hermes doctor
```

If you get `hermes: command not found`:

```bash
export PATH="$HOME/.local/bin:$PATH"
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
```

---

## 3. Vertex AI — reuse existing credentials

Hermes can use the same `gcloud_credentials.json` that OpenClaw already uses.

### 3.1 Create `~/.hermes/.env`

```bash
mkdir -p ~/.hermes
cat >> ~/.hermes/.env <<'EOF'
# Vertex — reuse existing OpenClaw / kleinanzeigen credentials
GOOGLE_APPLICATION_CREDENTIALS=/Users/max/kleinanzeigen_bot/tools/telegram_llm_bot/config/gcloud_credentials.json
VERTEX_PROJECT_ID=gen-lang-client-0431347096
VERTEX_REGION=global

# Telegram — reuse OpenClaw bot + allowlist
TELEGRAM_ALLOWED_USERS=1297932849
EOF
chmod 600 ~/.hermes/.env
```

If the credentials are `authorized_user` OAuth (as they are now), Hermes should refresh the access token automatically.  
The OpenClaw `refresh-vertex-token.sh` script and cron job are **not needed** for Hermes.

> Later, if you want to simplify production setup, you can migrate to a service account JSON. That is **not required** now.

### 3.2 Choose a model

```bash
hermes model
```

Select:

```text
More providers → Google Vertex AI → project → region global → google/gemini-3-flash-preview
```

---

## 4. CLI verification (mandatory checkpoint)

```bash
hermes doctor
hermes chat
```

Send a simple test: `Hello, reply with one word: ok`.

**If chat does not work — do not move on to Telegram and do not delete the OpenClaw backup.**

Common issues:

| Symptom | What to check |
|---------|---------------|
| Vertex auth error | path to `gcloud_credentials.json`, presence of `"token_uri"` in JSON |
| wrong region | `VERTEX_REGION=global` |
| model not found | `google/` prefix on the model name |
| quota | shared GCP project with kleinanzeigen-bot |

---

## 5. Tool security (before Telegram)

```bash
hermes tools
```

Minimum for a persistent Telegram agent:

- do **not** disable command approval
- restrict dangerous tools if you do not need them immediately
- keep credentials only in `~/.hermes/.env` with mode `600`

`~/.hermes/.env` should already include:

```bash
TELEGRAM_ALLOWED_USERS=1297932849
```

Without an allowlist, the Hermes gateway denies everyone by default.

---

## 6. Telegram — reuse the OpenClaw bot

### 6.1 Confirm the OpenClaw gateway is fully stopped

```bash
launchctl list | grep -i openclaw || echo "OK: OpenClaw not running"
ps aux | grep -i openclaw | grep -v grep || echo "OK: no openclaw process"
```

### 6.2 Configure the Hermes gateway

```bash
hermes gateway setup
```

- Platform: **Telegram**
- Bot token: **the same one** used by OpenClaw (`@open_claw_ai_assistant_bot`)

### 6.3 Foreground test first

```bash
hermes gateway
```

In another terminal, or after Ctrl+C — if foreground works:

```bash
hermes gateway install
hermes gateway start
hermes gateway status
tail -f ~/.hermes/logs/gateway.log
```

### 6.4 Telegram verification

Message `@open_claw_ai_assistant_bot` — Hermes should reply.

---

## 7. Autostart and Mac mini stability

```bash
hermes gateway install
hermes gateway start
hermes gateway status
sudo pmset -a preventsleep 1
```

After reboot:

```bash
hermes gateway status
tail -n 50 ~/.hermes/logs/gateway.log
```

Then send another test message in Telegram.

---

## 8. Full OpenClaw removal (only after Hermes works)

Run this **only when**:

- [ ] `hermes chat` works
- [ ] Telegram replies through Hermes
- [ ] `hermes gateway status` = running
- [ ] reboot test passed (recommended)

### 8.1 Stop and remove the OpenClaw daemon

```bash
cd ~/openclaw
pnpm start daemon stop 2>/dev/null || true
pnpm start daemon uninstall 2>/dev/null || true
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/ai.openclaw.gateway.plist 2>/dev/null || true
rm -f ~/Library/LaunchAgents/ai.openclaw.gateway.plist
```

### 8.2 Remove the OpenClaw cron refresh job (if present)

```bash
crontab -l | grep -v refresh-vertex-token.sh | crontab -
```

### 8.3 Remove config and project

```bash
rm -rf ~/.openclaw
rm -rf ~/openclaw
```

> You can keep the backup in `~/hermes-migration-backup/` for a few more days.

### 8.4 What you can leave untouched

- `gcloud_credentials.json` — **keep it**, Hermes uses it
- Node/pnpm/brew — the Hermes installer pulls its own dependencies; global cleanup is optional

---

## 9. After migration (optional)

Once the base agent is stable:

```text
memory/skills → SOUL.md → subagents
```

Commands:

```bash
hermes tools
hermes config get
hermes config set
```

---

## 10. Rollback (if Hermes does not work)

```bash
hermes gateway stop
hermes gateway uninstall 2>/dev/null || true

cd ~/openclaw   # if not deleted yet
pnpm start daemon install
pnpm start daemon start
```

Telegram will go back through OpenClaw. That is why **do not delete `~/openclaw` and `~/.openclaw` until Hermes succeeds**.

---

## 11. Checklist

### Phase A — preparation

- [ ] Backup `~/.openclaw/openclaw.json`
- [ ] Bot token saved
- [ ] OpenClaw gateway stopped

### Phase B — Hermes core

- [ ] `curl | bash` install
- [ ] `hermes doctor` OK
- [ ] `~/.hermes/.env` with Vertex + `TELEGRAM_ALLOWED_USERS`
- [ ] `hermes model` → Vertex / global / gemini-3-flash-preview
- [ ] `hermes chat` OK

### Phase C — Telegram handoff

- [ ] OpenClaw gateway not running
- [ ] `hermes gateway setup` with the same bot token
- [ ] Foreground test OK
- [ ] `hermes gateway install && start`
- [ ] Telegram replies

### Phase D — cleanup

- [ ] Reboot test OK
- [ ] OpenClaw daemon uninstalled
- [ ] `~/.openclaw` removed
- [ ] `~/openclaw` removed
- [ ] OpenClaw cron refresh removed

---

## 12. Quick cheat sheet

```bash
# 1) stop openclaw gateway only
cd ~/openclaw && pnpm start daemon uninstall

# 2) install hermes
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash && source ~/.zshrc

# 3) vertex reuse
# edit ~/.hermes/.env (GOOGLE_APPLICATION_CREDENTIALS, VERTEX_PROJECT_ID, VERTEX_REGION, TELEGRAM_ALLOWED_USERS)

# 4) test cli
hermes model && hermes chat && hermes doctor

# 5) telegram (same bot)
hermes gateway setup && hermes gateway && hermes gateway install && hermes gateway start

# 6) remove openclaw after success
rm -rf ~/.openclaw ~/openclaw
```

---

## Links

- [Hermes Installation](https://hermes-agent.nousresearch.com/docs/getting-started/installation)
- [Hermes Messaging Gateway](https://hermes-agent.nousresearch.com/docs/user-guide/messaging)
- [Hermes Security](https://hermes-agent.nousresearch.com/docs/user-guide/security)
- [Hermes Providers / Vertex](https://hermes-agent.nousresearch.com/docs/integrations/providers)
- Local OpenClaw docs: `../openclaw/OPENCLAW_MASTER_PLAN_EN.md`, `../openclaw/OPENCLAW_TROUBLESHOOTING.md`
