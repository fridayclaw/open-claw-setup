# OpenClaw Setup Instructions (Ubuntu/Debian)  
## Goal  
Set up OpenClaw using the OpenAI API (fast, reliable, costs money)  

## Prerequisites  
- **User:** `$USER` should be configured to a valid username.  
- **Gateway Port:** Adjust `${GATEWAY_PORT}` as needed (default: 18789)  

### 0. Variables (Edit Once)  
```bash  
USER=friday  
GATEWAY_PORT=18789  
```  

### 1. Create User and Grant Sudo Privileges  
```bash  
sudo adduser $USER  
sudo usermod -aG sudo $USER  
```  

### 2. Configure Basic Firewall (UFW)  
```bash  
sudo apt-get update  
sudo apt-get install -y ufw  
sudo ufw allow OpenSSH  
sudo ufw allow ${GATEWAY_PORT}/tcp  
sudo ufw --force enable  
sudo ufw status  
```  

### 3. Install Required Dependencies  
```bash  
sudo apt-get install -y curl ca-certificates git jq  
```  

### 4. Install OpenClaw  
```bash  
# Official Install Script  
curl -fsSL https://openclaw.ai/install.sh | sh  
openclaw --version  
```  

### 5. Set OpenAI Key for Daemon  
```bash  
sudo -iu $USER mkdir -p ~/.openclaw  
chmod 700 ~/.openclaw  
cat > ~/.openclaw/.env <<'ENV'  
OPENAI_API_KEY=PASTE_YOUR_OPENAI_API_KEY_HERE  
ENV  
chmod 600 ~/.openclaw/.env  
```  

### 6. Write OpenClaw Configuration (OpenAI Provider)  
```bash  
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
```  

### 7. Systemd Service Setup  
```bash  
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
```  

### 8. Helpful Commands  
#### Tail Logs  
```bash  
sudo journalctl -u openclaw -f  
```  
#### Restart OpenClaw  
```bash  
sudo systemctl restart openclaw  
```  
#### Stop OpenClaw  
```bash  
sudo systemctl stop openclaw  
```  

### 9. Preventing Restart Loops  
```bash  
sudo mkdir -p /etc/systemd/system/openclaw.service.d  
sudo tee /etc/systemd/system/openclaw.service.d/no-restart.conf >/dev/null <<'CONF'  
[Service]  
Restart=no  
CONF  
```  

### 10. Web UI Access  
#### OPTION A (Recommended): SSH Tunnel (No Public Exposure)  
```bash  
# On your laptop:  
ssh -L 18789:127.0.0.1:18789 friday@<VPS_IP>  
# Then open: http://127.0.0.1:18789  
```  

Feel free to reach out if you need further assistance or have questions!