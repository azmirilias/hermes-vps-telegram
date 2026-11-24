# hermes-vps-telegram

A hardened setup guide for running **Hermes Agent on a Linux VPS with Telegram**
(long polling) and an OpenCode agent backend.

This is not a custom bot. Telegram is Hermes' native integration — the gateway
(`hermes gateway run`) connects to the Telegram Bot API and routes messages to the
agent. No webhook server, no inbound port, nothing exposed publicly.

```
Telegram ──getUpdates──▶ Hermes gateway (localhost only)
                             │  agent sessions
                             ▼
                        OpenCode (localhost only)
```

## What is configured

| Piece | How |
|---|---|
| Hermes gateway | `hermes gateway run` — agent sessions + platform adapters |
| Telegram | native platform adapter, long polling by default |
| OpenCode | separate service the agent shells out to, localhost only |
| Secrets | `~/.hermes/.env` (mode 600), never committed |

## Prerequisites

- A Linux VPS (Ubuntu LTS recommended)
- A Telegram bot token from [@BotFather](https://t.me/BotFather)
- Your numeric Telegram user ID (ask @userinfobot)

## Setup

### 1. Install Hermes

Follow the official install at https://hermes-agent.nousresearch.com/docs — this
installs the `hermes` CLI and the agent runtime under `~/.hermes/`.

### 2. Create the Telegram bot

1. Message [@BotFather](https://t.me/BotFather), send `/newbot`.
2. Give it a name and a username.
3. Copy the token BotFather returns.

### 3. Configure Telegram

Edit `~/.hermes/.env` (mode 600). Minimal Telegram section:

```env
TELEGRAM_BOT_TOKEN=<token-from-botfather>
TELEGRAM_ALLOWED_USERS=<your-telegram-user-id>
TELEGRAM_HOME_CHANNEL=<your-telegram-user-id>
TELEGRAM_HOME_CHANNEL_NAME=<label>
TELEGRAM_CRON_THREAD_ID=
```

- `TELEGRAM_ALLOWED_USERS` — comma-separated Telegram user IDs allowed to talk to the
  agent. **Set this to your own ID**; anything else is rejected.
- `TELEGRAM_HOME_CHANNEL` — default destination for scheduled output.
- `TELEGRAM_CRON_THREAD_ID` — optional forum/topic thread for cron output.

Long polling is the default and needs no inbound port. Webhook is optional and not
recommended for a personal setup.

### 4. Start the gateway

Run it under systemd with a **dedicated unprivileged user** — see
`deploy/hermes-gateway.service`. Verify it connected by checking the gateway state
shows `telegram: connected`.

### 5. OpenCode agent

```bash
sudo cp deploy/opencode.service /etc/systemd/system/
sudo mkdir -p /etc/opencode && sudo cp .env.opencode.example /etc/opencode/.env
sudo chmod 600 /etc/opencode/.env   # then fill in OPENCODE_API_KEY
sudo systemctl daemon-reload
sudo systemctl enable --now opencode
```

OpenCode must bind to localhost only; its API key lives in `/etc/opencode/.env`.

## Hardening

Apply these before exposing the box to anything beyond your own use:

- **Least privilege.** Run the gateway and OpenCode as a dedicated unprivileged
  `hermes` user, not root. The user needs no sudo, no shell, and no password.
- **No passwordless sudo to root.** The gateway never needs root; if a single command
  does, use a scoped sudoers rule for that one command, never `NOPASSWD: ALL`.
- **SSH.** Disable root login (`PermitRootLogin no`), key-only auth
  (`PasswordAuthentication no`), and a non-default port.
- **Firewall.** `ufw` allow only SSH + 80/443; drop everything else. The gateway and
  bot need **no** inbound ports — long polling is outbound only.
- **fail2ban** on SSH.
- **Telegram allowlist.** `TELEGRAM_ALLOWED_USERS` must be set; a bot with an empty
  allowlist accepts anyone who finds the username.
- **Secrets.** `~/.hermes/.env` and `/etc/opencode/.env` are mode 600 and gitignored.
  If a token leaks, `/revoke` it in BotFather immediately and rotate.
- **Update regularly.** Keep Hermes, OpenCode, and the OS patched.

## Repository hygiene

- This repo contains **no tokens, no private chat IDs, no internal paths, and no
  deployment specifics.** Everything sensitive lives in your `.env` files.
- `.env*` is gitignored except the `*.example` templates with empty placeholders.
- Re-check `git status` before every push so a stray `.env` or log never ships.

## Licence

MIT.
