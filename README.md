# openhanako-nas-deploy

[![English](https://img.shields.io/badge/🌐_English-0077B5?style=for-the-badge&logo=github)](README.md) [![中文](https://img.shields.io/badge/🌐_中文-FF6F00?style=for-the-badge&logo=github)](README.zh-CN.md)

One-click HanaAgent Server deployment on Linux NAS (or any Linux server) — from zero to production.
Includes: Node.js environment setup, HanaAgent build & deploy, model provider configuration (DeepSeek / Ollama), user auth, firewall, external access, systemd auto-start.

> **HanaAgent**: [liliMozi/openhanako](https://github.com/liliMozi/openhanako) — Open-source desktop AI assistant
>
> **Companion repo**: For desktop client-side patches (NAS connection, CSP whitelist), see [`openhanako-nas-connect`](https://github.com/JohnnyClaudeCh/openhanako-nas-connect).

---

## Prerequisites

| Environment | Description | Required | Notes |
|-------------|-------------|----------|-------|
| Linux Server/NAS | Runs HanaAgent Server | Yes | Debian 12 / Ubuntu / fnOS |
| Node.js >= 18 | HanaAgent runtime | Yes | v18 / v20 / v22 |
| git | Clone HanaAgent source | Recommended | |
| Domain (optional) | External access | Optional | Alibaba Cloud DDNS |

---

## Quick Start

```bash
# 1. Clone this repo
git clone https://github.com/JohnnyClaudeCh/openhanako-nas-deploy.git
cd openhanako-nas-deploy

# 2. Edit configuration
cp config/models.json.template ~/.hanako-dev/models.json
cp config/config.yaml.template ~/.hanako-dev/agents/hanako/config.yaml

# 3. Run the one-click setup script
bash scripts/setup.sh
```

---

## Manual Deployment Steps

### Step 1: Environment Setup

```bash
# Debian / Ubuntu
sudo apt update
sudo apt install -y curl git nodejs npm

# Verify
node --version   # >= 18
npm --version
```

### Step 2: Deploy HanaAgent Server

```bash
# Clone HanaAgent (replace with your repo URL)
git clone https://github.com/liliMozi/openhanako.git /vol1/1000/Hanako
cd /vol1/1000/Hanako

# Install dependencies & build
npm install
npm run build:client

# First run (test)
npm run server
```

Visit `http://your-nas-ip:14500/desktop/` — you should see the HanaAgent interface. Press Ctrl+C to stop the test server.

### Step 3: Configure Models

HanaAgent needs at least one model provider. Choose one:

#### Option A: DeepSeek API (Recommended, zero hardware cost)

Create/edit `~/.hanako-dev/models.json`:

```json
{
    "providers": {
        "deepseek": {
            "baseUrl": "https://api.deepseek.com",
            "api": "openai",
            "apiKey": "sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
            "models": [
                {
                    "id": "deepseek-chat",
                    "name": "DeepSeek V3",
                    "input": ["text"],
                    "output": ["text"]
                }
            ]
        }
    }
}
```

#### Option B: Local Ollama

If you have a GPU machine running Ollama, add it as a provider:

```json
{
    "providers": {
        "ollama": {
            "baseUrl": "http://your-gpu-machine-ip:11434/v1",
            "api": "ollama",
            "apiKey": "ollama",
            "models": [
                {
                    "id": "qwen2.5:7b",
                    "name": "Qwen2.5 7B",
                    "input": ["text"],
                    "output": ["text"]
                }
            ]
        }
    }
}
```

### Step 4: Configure Agent

Edit `~/.hanako-dev/agents/hanako/config.yaml`:

```yaml
name: my-agent
version: "1.0"
system_prompt: "You are a helpful AI assistant"
models:
  chat: deepseek-chat
  utility: deepseek-chat
  utility_large: deepseek-chat
```

### Step 5: Network Configuration

Edit `~/.hanako-dev/server-network.json`:

```json
{
  "mode": "lan",
  "listenHost": "0.0.0.0",
  "listenPort": 14500
}
```

### Step 6: Start the Service

```bash
# Foreground (testing)
cd /vol1/1000/Hanako && npm run server

# Or use systemd auto-start
sudo cp scripts/hanako.service /etc/systemd/system/
sudo systemctl enable hanako
sudo systemctl start hanako
sudo systemctl status hanako
```

### Step 7: Firewall & Port Forwarding

```bash
# firewall-cmd (CentOS/RHEL)
sudo firewall-cmd --add-port=14500/tcp --permanent
sudo firewall-cmd --reload

# ufw (Debian/Ubuntu)
sudo ufw allow 14500/tcp comment 'HanaAgent Server'
```

For external access, add a port forwarding rule on your router: external port 14500 → NAS internal IP:14500.

### Step 8: Create Admin Account

After starting HanaAgent Server for the first time, open the Web UI in your browser. You'll be guided through admin account creation.

---

## Verification

| Check | Method | Expected |
|-------|--------|----------|
| Service running | `sudo systemctl status hanako` | active (running) |
| Web UI | Browser to `http://NAS_IP:14500/desktop/` | HanaAgent interface |
| External access | Browser from outside network to `http://domain:14500/desktop/` | HanaAgent interface |
| Auto-restart | `sudo systemctl restart hanako` | Restarts cleanly |

---

## Configuration Reference

| Local Path | NAS Path | Description |
|-----------|---------|-------------|
| `config/server-network.json` | `~/.hanako-dev/server-network.json` | Network mode, port |
| `config/models.json.template` | `~/.hanako-dev/models.json` | Model provider config |
| `config/config.yaml.template` | `~/.hanako-dev/agents/hanako/config.yaml` | Agent runtime config |

---

## Logs & Debugging

```bash
sudo journalctl -u hanako -n 50 -f   # systemd logs
tail -f /tmp/hanako.log              # Application logs
tail -f ~/.hanako-dev/logs/*.log     # HanaAgent logs
sudo systemctl restart hanako        # Restart
```

---

## Updating HanaAgent

```bash
cd /vol1/1000/Hanako
git pull
npm install
npm run build:client
sudo systemctl restart hanako
```

> Updates may overwrite the whitelist patch in `mobile-static.ts`.
> If settings page returns 404 after update, re-run `patch_static.js` from `openhanako-nas-connect`.

---

## FAQ

### Q: Settings page returns 404

HanaAgent's default whitelist doesn't include `settings.html`. Run `patch_static.js` to fix this. See `openhanako-nas-connect` for details.

### Q: Desktop client can't connect to remote NAS

Electron's CSP security policy blocks cross-origin connections. You need to patch `connection-csp.js` inside `app.asar`. See `openhanako-nas-connect` for instructions.

### Q: Forgot Web UI login password

Stop the service, delete `~/.hanako-dev/local-user-auth.json`, restart, and re-register.

### Q: Complete uninstall

```bash
sudo systemctl stop hanako
sudo systemctl disable hanako
sudo rm /etc/systemd/system/hanako.service
rm -rf /vol1/1000/Hanako          # Remove code
rm -rf ~/.hanako-dev              # Remove config
```
