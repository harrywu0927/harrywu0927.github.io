+++
date = '2025-06-10T15:39:33+08:00'
draft = false
title = 'Deploying Clash.Meta on Ubuntu as a System Service with Auto Subscription Updates'
+++

## Download the Clash Core
Clash is a general-purpose proxy core developed in Go that supports multiple proxy protocols (e.g., vmess, vless, trojan, shadowsocks, socks, http, etc.).
It does not include a GUI and only runs the core logic. It must be used with a configuration file or a control panel (e.g., Clash Verge, Meta UI, Yacd).

The original and Premium versions of Clash have been abandoned. The former Meta version has now been renamed to [mihomo](https://github.com/MetaCubeX/mihomo/tree/Alpha).

```bash
# Download the latest Clash.Meta version (Linux amd64)
wget https://github.com/MetaCubeX/mihomo/releases/download/v1.19.10/mihomo-linux-amd64-v1.19.10.gz -O clash.meta.gz
gunzip clash.meta.gz
chmod +x clash.meta
```

## Prepare config.yaml

You need a subscription URL (usually provided by your proxy service provider), and a tool to convert it to a Clash config file.

- Method A: Manual Download  
    1. Your provider usually offers a downloadable Clash YAML config file.  
    2. Download and name it `config.yaml`.

For example, using SpeedCat’s converted Clash subscription:
```bash
wget -O config.yaml [your-subscription-url]
```

- Method B: Use [Subconverter](https://github.com/tindy2013/subconverter/blob/master/README-cn.md)
Generate a Clash config file in this format:
```bash
https://subconverter.yourdomain.com/sub?target=clash&url=your-subscription-url&udp=true
```

{{< details summary="Why convert configurations?" >}}
> Because each proxy client (Clash, Surge, Shadowrocket, V2RayN, etc.) uses a different config format, and raw subscriptions (e.g., ss://, vmess://, etc.) need to be converted for compatibility.
{{< /details >}}

## Run clash.meta
```bash
./clash.meta
```
You may encounter an error on first run:
```text
ERRO can't initial GeoIP: can't download MMDB: context deadline exceeded
FATA Parse config error: rules[...] [GEOIP,CN,DIRECT] error: can't download MMDB: context deadline exceeded
```
This occurs when Clash fails to download the GeoIP file (e.g., MaxMind Country.mmdb) due to network issues.

> GEOIP rules match IPs based on country codes, e.g., `GEOIP,CN,DIRECT`.

Solutions:
- Option 1: Manually download using a reachable CDN or proxy:
```bash
wget https://cdn.jsdelivr.net/gh/Dreamacro/maxmind-geoip@release/Country.mmdb -O Country.mmdb
```
Place it in the same directory as `config.yaml`.

- Option 2: Temporarily comment out all GEOIP rule lines.

Once Clash starts and you have network access, restore the GEOIP lines.

When Clash runs successfully, you should see logs like:
```
INFO[...] Start initial configuration in progress
INFO[...] Initial configuration complete, total time: XXms
```

## Test the Proxy
### Set Global System Proxy (Ubuntu 22)
```bash
sudo vi /etc/environment
```

Add:
```
http_proxy="http://127.0.0.1:7890"
https_proxy="http://127.0.0.1:7890"
```

Apply changes:
```bash
source /etc/environment
```

### Test Connectivity
```bash
curl --proxy http://127.0.0.1:7890 https://api.ipify.org
```

## Run as a System Service
### Create Clash User
```bash
sudo useradd -r -s /usr/sbin/nologin clash
sudo chown -R clash:clash /opt/clash
```

{{< details summary="Why a dedicated user?" >}}
- Principle of Least Privilege
- Prevent accidental interference
{{< /details >}}

### Create systemd Service File
```bash
sudo vi /etc/systemd/system/clash.service
```
Content:
```ini
[Unit]
Description=Clash Meta Service
After=network.target

[Service]
Type=simple
User=clash
WorkingDirectory=/opt/clash
ExecStart=/opt/clash/clash.meta -d .
Restart=on-failure
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
```

### Copy Runtime Files
```bash
sudo cp * /opt/clash
```

### Start Service
```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable clash
sudo systemctl start clash
```

### Check Logs
```bash
sudo systemctl status clash
journalctl -u clash -f
```

## Setup Auto Subscription Updates

### Create Update Script
```bash
sudo vi /opt/clash/update-clash.sh
```
Content:
```bash
#!/bin/bash
CLASH_DIR="/opt/clash"
SUB_URL="[your-subscription-url]"
CONFIG_FILE="$CLASH_DIR/config.yaml"
BACKUP_FILE="$CLASH_DIR/config.yaml.bak"
TMP_FILE="$CLASH_DIR/config.tmp.yaml"

cp "$CONFIG_FILE" "$BACKUP_FILE"
echo "Downloading new config..."
if curl -fsSL "$SUB_URL" -o "$TMP_FILE"; then
    mv "$TMP_FILE" "$CONFIG_FILE"
    echo "Update successful. Restarting Clash..."
    systemctl restart clash
else
    echo "Download failed. Restoring previous config..."
    mv "$BACKUP_FILE" "$CONFIG_FILE"
fi
```

Make it executable:
```bash
sudo chmod +x /opt/clash/update-clash.sh
```

### Create Cron Job
```bash
sudo crontab -e
```

Add:
```cron
0 3 * * * /opt/clash/update-clash.sh >> /opt/clash/update.log 2>&1
```
