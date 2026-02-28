# OpenClaw Full Root Access Setup Guide

> Complete guide to installing and configuring OpenClaw as a root-privileged AI gateway with Telegram command execution on Ubuntu/Debian servers.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Install Node.js via NVM](#2-install-nodejs-via-nvm)
3. [Install OpenClaw](#3-install-openclaw)
4. [Initial Setup (Interactive Wizard)](#4-initial-setup-interactive-wizard)
5. [Configure Telegram Bot](#5-configure-telegram-bot)
6. [Configure OpenClaw for Full Exec Access](#6-configure-openclaw-for-full-exec-access)
7. [Set Up Exec Approvals](#7-set-up-exec-approvals)
8. [Enable Passwordless Sudo](#8-enable-passwordless-sudo)
9. [Stop User-Level Services](#9-stop-user-level-services)
10. [Create Root System-Level Services](#10-create-root-system-level-services)
11. [Fix Extension Ownership](#11-fix-extension-ownership)
12. [Enable and Start Services](#12-enable-and-start-services)
13. [Verify Everything Works](#13-verify-everything-works)
14. [Post-Setup: Activate Exec in Telegram](#14-post-setup-activate-exec-in-telegram)
15. [Maintenance and Troubleshooting](#15-maintenance-and-troubleshooting)
16. [Security Considerations](#16-security-considerations)
17. [Quick Reference](#17-quick-reference)

---

## 1. Prerequisites

- **OS:** Ubuntu 24.04 LTS (or any systemd-based Linux)
- **User:** A non-root user with sudo access (referred to as `$USER` throughout)
- **Network:** Internet access for downloading packages
- **Telegram:** A Telegram bot token (from [@BotFather](https://t.me/BotFather))
- **AI Provider:** API access to an LLM provider (e.g., OpenAI, Anthropic, or local Ollama)

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install essential build tools
sudo apt install -y curl git build-essential
```

---

## 2. Install Node.js via NVM

OpenClaw requires Node.js v24+.

```bash
# Install NVM
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.4/install.sh | bash

# Reload shell
source ~/.bashrc

# Install Node.js v24
nvm install 24
nvm use 24
nvm alias default 24

# Verify
node --version   # Should show v24.x.x
npm --version
```

---

## 3. Install OpenClaw

```bash
# Install OpenClaw globally
npm install -g openclaw

# Verify installation
openclaw --version
```

---

## 4. Initial Setup (Interactive Wizard)

Run the setup wizard to create the initial configuration:

```bash
openclaw doctor
```

This creates:
- `~/.openclaw/openclaw.json` — main configuration
- `~/.openclaw/workspace/` — agent workspace directory
- `~/.openclaw/extensions/` — plugin extensions directory

The wizard will guide you through:
- Choosing an AI provider and model
- Setting up authentication
- Configuring basic gateway settings

After the wizard, OpenClaw typically installs user-level systemd services. **We will replace these with root-level services later.**

---

## 5. Configure Telegram Bot

### 5.1 Create a Bot via BotFather

1. Open Telegram and message [@BotFather](https://t.me/BotFather)
2. Send `/newbot` and follow the prompts
3. Save the **bot token** (format: `123456789:AAG...`)
4. Send `/setcommands` to BotFather and set the commands for your bot (optional)

### 5.2 Get Your Telegram User ID

1. Message [@userinfobot](https://t.me/userinfobot) on Telegram
2. It will reply with your **user ID** (a number like `6987875725`)

---

## 6. Configure OpenClaw for Full Exec Access

Edit `~/.openclaw/openclaw.json` with the full configuration. Replace the placeholder values with your own:

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "YOUR_PROVIDER/YOUR_MODEL"
      },
      "models": {
        "YOUR_PROVIDER/YOUR_MODEL": {}
      },
      "workspace": "/home/YOUR_USER/.openclaw/workspace",
      "userTimezone": "YOUR_TIMEZONE"
    }
  },
  "tools": {
    "profile": "full",
    "allow": [
      "exec",
      "process",
      "read",
      "write",
      "edit",
      "group:runtime",
      "group:fs",
      "group:sessions",
      "group:memory"
    ],
    "elevated": {
      "enabled": true,
      "allowFrom": {
        "telegram": ["*"]
      }
    },
    "web": {
      "search": {
        "enabled": false
      }
    },
    "exec": {
      "host": "gateway",
      "security": "full",
      "ask": "off",
      "timeoutSec": 1800
    }
  },
  "commands": {
    "native": "auto",
    "nativeSkills": "auto",
    "restart": true,
    "ownerDisplay": "raw"
  },
  "session": {
    "dmScope": "per-channel-peer"
  },
  "channels": {
    "telegram": {
      "enabled": true,
      "dmPolicy": "allowlist",
      "botToken": "YOUR_BOT_TOKEN",
      "allowFrom": [
        "YOUR_TELEGRAM_USER_ID"
      ],
      "groupPolicy": "allowlist",
      "streaming": "off"
    }
  },
  "gateway": {
    "port": 18789,
    "mode": "local",
    "bind": "loopback",
    "auth": {
      "mode": "token",
      "token": "GENERATE_A_RANDOM_TOKEN"
    },
    "tailscale": {
      "mode": "off",
      "resetOnExit": false
    }
  },
  "plugins": {
    "allow": [
      "telegram"
    ],
    "entries": {
      "telegram": {
        "enabled": true
      }
    }
  }
}
```

### Key Configuration Explained

| Config Path | Value | Purpose |
|---|---|---|
| `tools.profile` | `"full"` | Enable all tool categories |
| `tools.allow` | `["exec", "process", ...]` | Explicitly allow exec and filesystem tools |
| `tools.elevated.enabled` | `true` | Enable elevated (root) execution mode |
| `tools.elevated.allowFrom.telegram` | `["*"]` | Allow all permitted Telegram users to use elevated exec |
| `tools.exec.host` | `"gateway"` | Execute commands on the gateway host machine |
| `tools.exec.security` | `"full"` | No command restrictions (allow everything) |
| `tools.exec.ask` | `"off"` | Never prompt for approval before executing |
| `tools.exec.timeoutSec` | `1800` | 30-minute command timeout |
| `channels.telegram.dmPolicy` | `"allowlist"` | Only allow listed Telegram user IDs |
| `channels.telegram.allowFrom` | `["YOUR_ID"]` | Your Telegram user ID |

### Generate a Random Gateway Token

```bash
openssl rand -base64 32 | tr -d '/+=' | head -c 44
```

---

## 7. Set Up Exec Approvals

Create the exec approvals file with a wildcard pattern to auto-approve all commands:

```bash
cat > ~/.openclaw/exec-approvals.json << 'EOF'
{
  "version": 1,
  "socket": {},
  "defaults": {},
  "agents": {
    "*": {
      "allowlist": [
        {
          "pattern": "**"
        }
      ]
    }
  }
}
EOF
```

The `"pattern": "**"` wildcard means all commands are pre-approved for all agents.

---

## 8. Enable Passwordless Sudo

The service user needs passwordless sudo so systemd can manage root services:

```bash
sudo bash -c 'echo "YOUR_USERNAME ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/YOUR_USERNAME-nopasswd'
sudo chmod 440 /etc/sudoers.d/YOUR_USERNAME-nopasswd

# Verify
sudo -n whoami   # Should print: root
```

---

## 9. Stop User-Level Services

If OpenClaw's wizard already created user-level services, stop and disable them:

```bash
# Stop running services
systemctl --user stop openclaw-gateway.service openclaw-node.service 2>/dev/null

# Disable auto-start
systemctl --user disable openclaw-gateway.service openclaw-node.service 2>/dev/null

# Verify they're stopped
systemctl --user status openclaw-gateway.service 2>&1 | grep Active
systemctl --user status openclaw-node.service 2>&1 | grep Active
```

---

## 10. Create Root System-Level Services

### 10.1 Determine Paths

Find your Node.js and OpenClaw paths:

```bash
# Node.js binary
NODE_BIN=$(which node)
echo "Node binary: $NODE_BIN"

# OpenClaw installation
OPENCLAW_DIR=$(npm root -g)/openclaw
echo "OpenClaw dir: $OPENCLAW_DIR"

# Your home directory
echo "Home: $HOME"
```

### 10.2 Create Gateway Service

```bash
# Replace paths below with your actual paths from step 10.1
sudo tee /etc/systemd/system/openclaw-gateway.service > /dev/null << 'EOF'
[Unit]
Description=OpenClaw Gateway - Root
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/home/YOUR_USER/.nvm/versions/node/v24.14.0/bin/node /home/YOUR_USER/.nvm/versions/node/v24.14.0/lib/node_modules/openclaw/dist/entry.js gateway --port 18789
Restart=always
RestartSec=5
KillMode=process
User=root
Group=root
Environment=HOME=/home/YOUR_USER
Environment=TMPDIR=/tmp
Environment=PATH=/home/YOUR_USER/.nvm/versions/node/v24.14.0/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
Environment=OPENCLAW_GATEWAY_PORT=18789
Environment=OPENCLAW_GATEWAY_TOKEN=YOUR_GATEWAY_TOKEN
Environment=OPENCLAW_SYSTEMD_UNIT=openclaw-gateway.service
Environment=OPENCLAW_SERVICE_MARKER=openclaw
Environment=OPENCLAW_SERVICE_KIND=gateway

[Install]
WantedBy=multi-user.target
EOF
```

### 10.3 Create Node Service

```bash
sudo tee /etc/systemd/system/openclaw-node.service > /dev/null << 'EOF'
[Unit]
Description=OpenClaw Node Host - Root
After=network-online.target openclaw-gateway.service
Wants=network-online.target
Requires=openclaw-gateway.service

[Service]
Type=simple
ExecStart=/home/YOUR_USER/.nvm/versions/node/v24.14.0/bin/node /home/YOUR_USER/.nvm/versions/node/v24.14.0/lib/node_modules/openclaw/dist/index.js node run --host 127.0.0.1 --port 18789
Restart=always
RestartSec=5
KillMode=process
User=root
Group=root
Environment=HOME=/home/YOUR_USER
Environment=TMPDIR=/tmp
Environment=PATH=/home/YOUR_USER/.nvm/versions/node/v24.14.0/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
Environment=OPENCLAW_LAUNCHD_LABEL=ai.openclaw.node
Environment=OPENCLAW_SYSTEMD_UNIT=openclaw-node
Environment=OPENCLAW_LOG_PREFIX=node
Environment=OPENCLAW_SERVICE_MARKER=openclaw
Environment=OPENCLAW_SERVICE_KIND=node

[Install]
WantedBy=multi-user.target
EOF
```

### Critical Notes on Service Configuration

- **`User=root` and `Group=root`** — This is what gives OpenClaw root privileges
- **`Environment=HOME=/home/YOUR_USER`** — Must point to the user home where `~/.openclaw/` config lives (NOT `/root`)
- **`Requires=openclaw-gateway.service`** on the node service ensures the gateway starts first
- **`WantedBy=multi-user.target`** ensures services start on boot (system-level, not user-level)
- **PATH must include the nvm node binary directory** so the service can find `node`

---

## 11. Fix Extension Ownership

When running as root, OpenClaw rejects extensions owned by non-root users ("suspicious ownership" security check). Fix this:

```bash
sudo chown -R root:root ~/.openclaw/extensions/
```

If you install new extensions later, re-run this command.

---

## 12. Enable and Start Services

```bash
# Reload systemd to pick up new service files
sudo systemctl daemon-reload

# Enable services to start on boot
sudo systemctl enable openclaw-gateway.service openclaw-node.service

# Start gateway first, wait for it to initialize
sudo systemctl start openclaw-gateway.service
sleep 5

# Then start node service
sudo systemctl start openclaw-node.service
sleep 3

# Check status
sudo systemctl status openclaw-gateway.service openclaw-node.service
```

Both services should show `Active: active (running)`.

---

## 13. Verify Everything Works

### 13.1 Check Processes Run as Root

```bash
ps aux | grep openclaw | grep -v grep
```

Expected output — all processes owned by `root`:

```
root   12345  ...  openclaw
root   12346  ...  openclaw-gateway
root   12347  ...  openclaw-node
```

### 13.2 Check Logs for Errors

```bash
# Gateway logs
sudo journalctl -u openclaw-gateway.service --no-pager -n 30

# Node logs
sudo journalctl -u openclaw-node.service --no-pager -n 30
```

Look for:
- `listening on ws://127.0.0.1:18789` — gateway is accepting connections
- `[telegram] [default] starting provider` — Telegram bot is connected
- No `Invalid config` or `Config invalid` errors

### 13.3 Check Tools Configuration

```bash
openclaw config get tools
```

Should show `"profile": "full"`, `"security": "full"`, `"ask": "off"`.

### 13.4 Check Sandbox/Elevated Status

```bash
openclaw sandbox explain
```

Look for:
- `exec` in the **allow** list
- `Elevated: enabled: true`

---

## 14. Post-Setup: Activate Exec in Telegram

**Important:** After making config changes, existing Telegram sessions use cached tool lists. You must start a fresh session:

1. Open your Telegram bot chat
2. Send `/reset` or `/new` to start a new session
3. Test by asking the bot to run a command, e.g.: "Run `whoami` and `id` on the server"

The bot should now execute commands as root and return output.

---

## 15. Maintenance and Troubleshooting

### Restart Services

```bash
sudo systemctl restart openclaw-gateway.service
sleep 5
sudo systemctl restart openclaw-node.service
```

### View Live Logs

```bash
# Follow gateway logs
sudo journalctl -u openclaw-gateway.service -f

# Follow node logs
sudo journalctl -u openclaw-node.service -f
```

### Common Errors and Fixes

| Error | Cause | Fix |
|---|---|---|
| `Config invalid` | Invalid JSON or unrecognized config key | Check `openclaw.json` syntax; run `openclaw doctor --fix` |
| `plugin not found: ...` | Plugin directory not owned by root | `sudo chown -R root:root ~/.openclaw/extensions/` |
| `suspicious ownership` | Extension files owned by non-root user | Same as above |
| `connect ECONNREFUSED 127.0.0.1:18789` | Node started before gateway was ready | Restart node: `sudo systemctl restart openclaw-node.service` |
| `agents.defaults: Unrecognized key: "tools"` | `tools` is not valid inside `agents.defaults` | Move tools config to top-level `tools` section only |
| Bot says "no exec tool" | Stale Telegram session | Send `/reset` in Telegram to start fresh session |

### Update OpenClaw

```bash
# Update the package
npm update -g openclaw

# Fix extension ownership after update
sudo chown -R root:root ~/.openclaw/extensions/

# Update version in service files if needed, then restart
sudo systemctl daemon-reload
sudo systemctl restart openclaw-gateway.service
sleep 5
sudo systemctl restart openclaw-node.service
```

---

## 16. Security Considerations

This setup gives the Telegram bot **unrestricted root access** to the server. Understand the risks:

### What This Means

- Any user in the `channels.telegram.allowFrom` list can execute **any command as root**
- This includes: installing/removing software, reading/writing any file, managing services, network configuration, etc.

### Recommended Safeguards

1. **Restrict Telegram users strictly** — Only add your own Telegram user ID to `allowFrom`. Never use `"*"` in `channels.telegram.allowFrom`.

2. **Use allowlist DM policy** — Keep `"dmPolicy": "allowlist"` to reject messages from unknown users.

3. **Monitor logs** — Regularly check `sudo journalctl -u openclaw-gateway.service` for unexpected activity.

4. **Firewall the gateway port** — The gateway binds to loopback (`127.0.0.1`) by default, which is correct. Never expose port 18789 to the internet.

5. **Secure bot token** — If your bot token is compromised, anyone could interact with the bot. Regenerate via BotFather if needed.

6. **Consider per-sender restrictions for groups** — If using group chats:
   ```json
   "channels": {
     "telegram": {
       "groups": {
         "GROUP_CHAT_ID": {
           "toolsBySender": {
             "id:YOUR_USER_ID": {
               "allow": ["exec", "process", "group:runtime"]
             }
           }
         }
       }
     }
   }
   ```

---

## 17. Quick Reference

### File Locations

| File | Purpose |
|---|---|
| `~/.openclaw/openclaw.json` | Main configuration |
| `~/.openclaw/exec-approvals.json` | Command approval whitelist |
| `~/.openclaw/extensions/` | Plugin extensions |
| `~/.openclaw/workspace/` | Agent workspace |
| `/etc/systemd/system/openclaw-gateway.service` | Root gateway service |
| `/etc/systemd/system/openclaw-node.service` | Root node service |
| `/etc/sudoers.d/YOUR_USER-nopasswd` | Passwordless sudo config |
| `/tmp/openclaw-0/openclaw-*.log` | Runtime log files |

### Essential Commands

```bash
# Service management
sudo systemctl start openclaw-gateway.service
sudo systemctl start openclaw-node.service
sudo systemctl stop openclaw-gateway.service openclaw-node.service
sudo systemctl restart openclaw-gateway.service
sudo systemctl status openclaw-gateway.service openclaw-node.service

# Logs
sudo journalctl -u openclaw-gateway.service -f
sudo journalctl -u openclaw-node.service -f

# Config inspection
openclaw config get tools
openclaw sandbox explain
openclaw status --all

# Telegram session reset (send in chat)
/reset
/new
```

### Minimal Config for Full Root Exec via Telegram

The absolute minimum config keys required for root exec via Telegram:

```json
{
  "tools": {
    "profile": "full",
    "elevated": {
      "enabled": true,
      "allowFrom": { "telegram": ["*"] }
    },
    "exec": {
      "host": "gateway",
      "security": "full",
      "ask": "off"
    }
  },
  "channels": {
    "telegram": {
      "enabled": true,
      "dmPolicy": "allowlist",
      "botToken": "YOUR_TOKEN",
      "allowFrom": ["YOUR_USER_ID"]
    }
  }
}
```

Combined with:
- System-level systemd services running as `User=root`
- `exec-approvals.json` with wildcard `"pattern": "**"`
- Extension directories owned by root

---

*Guide based on OpenClaw v2026.2.26 running on Ubuntu 24.04 LTS with Node.js v24.*
