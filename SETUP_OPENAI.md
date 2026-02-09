# SETUP_OPENAI.md (Ubuntu/Debian)  
# Goal: OpenClaw using OpenAI API (fast, reliable, costs money)  
## 0) Variables (edit once)  
USER=friday  
GATEWAY_PORT=18789  
## 1) Create user + sudo  
sudo adduser $USER  
sudo usermod -aG sudo $USER  
## 2) Basic firewall (UFW)  
sudo apt-get update  
sudo apt-get install -y ufw  
sudo ufw allow OpenSSH  
sudo ufw allow ${GATEWAY_PORT}/tcp  
sudo ufw --force enable  
sudo ufw status  
## 3) Install dependencies  
sudo apt-get install -y curl ca-certificates git jq  
## 4) Install OpenClaw  
# Official install script  
curl -fsSL https://openclaw.ai/install.sh | sh  
openclaw --version  
## 5) Set OpenAI key for the daemon  
sudo -iu $USER mkdir -p ~/.openclaw  
chmod 700 ~/.openclaw  
# Put env vars where OpenClaw expects them (wizard uses this pattern too)  
cat > ~/.openclaw/.env <<'ENV'  
OPENAI_API_KEY=PASTE_YOUR_OPENAI_API_KEY_HERE  
ENV  
chmod 600 ~/.openclaw/.env  
## 6) Write OpenClaw config (OpenAI provider)  
# NOTE:  
# - ChatGPT $20/mo (Plus) is NOT the same as API credits.  
# - API uses OPENAI_API_KEY and bills per token.  
cat > ~/.openclaw/openclaw.json <<'JSON'  
{  
  "gateway": {  
    "mode": "local",  
    "port": 18789,  
    "bind": "127.0.0.1",  
    "auth": {  
      "mode": "token",  
      "token": "CHANGE_ME_LONG_RANDOM"  
    }  
  },  
  "agents": {  
    "defaults": {  
      "workspace": "/home/friday/.openclaw/workspace",  
      "model": "openai/gpt-4o-mini"  
    }  
  },  
  "models": {  
    "providers": {  
      "openai": {  
        "baseUrl": "https://api.openai.com/v1",  
        "apiKeyEnv": "OPENAI_API_KEY",  
        "api": "openai-completions",  
        "models": [  
          {  
            "id": "gpt-4o-mini",  
            "name": "gpt-4o-mini",  
            "reasoning": false,  
            "input": [  
              "text"  
            ],  
            "cost": {  
              "input": 0,  
              "output": 0,  
              "cacheRead": 0,  
              "cacheWrite": 0  
            },  
            "contextWindow": 128000,  
            "maxTokens": 2048  
          }  
        ]  
      }  
    }  
  }  
}  
JSON  
# Validate/fix  
openclaw doctor --fix  
## 7) Systemd service (system service, runs as friday)  
sudo tee /etc/systemd/system/openclaw.service >/dev/null <<'UNIT'  
[Unit]  
Description=OpenClaw AI Agent  
After=network-online.target  
Wants=network-online.target  
[Service]  
Type=simple  
User=friday  
WorkingDirectory=/home/friday  
Environment=HOME=/home/friday  
EnvironmentFile=/home/friday/.openclaw/.env  
ExecStart=/usr/local/bin/openclaw gateway start --foreground  
Restart=on-failure  
RestartSec=3  
[Install]  
WantedBy=multi-user.target  
UNIT  
sudo systemctl daemon-reload  
sudo systemctl enable --now openclaw  
sudo systemctl status openclaw --no-pager  
## 8) Helper commands  
# Tail logs  
sudo journalctl -u openclaw -f  
# Restart OpenClaw  
sudo systemctl restart openclaw  
# Stop OpenClaw  
sudo systemctl stop openclaw  
## 9) “Stop the restart loop” file (handy when config breaks)  
# This makes systemd NOT restart on failure until you revert it.  
sudo mkdir -p /etc/systemd/system/openclaw.service.d  
sudo tee /etc/systemd/system/openclaw.service.d/no-restart.conf >/dev/null <<'CONF'  
[Service]  
Restart=no  
CONF  
# Apply override + restart  
sudo systemctl daemon-reload  
sudo systemctl restart openclaw  
# To revert (back to Restart=on-failure):  
# sudo rm /etc/systemd/system/openclaw.service.d/no-restart.conf  
# sudo systemctl daemon-reload  
# sudo systemctl restart openclaw  
## 10) Web UI access  
# OPTION A (recommended): SSH tunnel (no public exposure)  
# On your laptop:  
# ssh -L 18789:127.0.0.1:18789 friday@<VPS_IP>  
# Then open: http://127.0.0.1:18789  
