+++
date = '2025-06-10T15:39:33+08:00'
draft = false
title = 'Ubuntu 部署 Clash.Meta 并设置为系统服务与自动订阅更新'
+++

## 下载 Clash 内核
Clash 是一个基于 Go 开发的通用代理客户端核心，支持多种代理协议（如 vmess, vless, trojan, shadowsocks, socks, http 等）。
它不带 GUI，只负责核心逻辑运行，需要配合配置文件或控制面板（如 Clash Verge、Meta UI、Yacd）来使用。

Clash 原版和 Premium 版已跑路，原 Meta 版现已更名为 [mihomo](https://github.com/MetaCubeX/mihomo/tree/Alpha).

```bash
# 下载 Clash.Meta 最新版本（Linux amd64）
wget https://github.com/MetaCubeX/mihomo/releases/download/v1.19.10/mihomo-linux-amd64-v1.19.10.gz -O clash.meta.gz
gunzip clash.meta.gz
chmod +x clash.meta
```

## 准备配置文件 config.yaml

你需要一个订阅链接（通常由机场提供），使用工具将其转换为 Clash 配置文件.

- 方式 A：手动下载配置
	1.	机场后台一般提供 Clash 的 YAML 配置文件下载.
	2.	下载后命名为 config.yaml.

我这里使用 SpeedCat 机场的 Clash 订阅链接，它已完成订阅转换，只需直接下载即可：
```bash
wget -O config.yaml [你的订阅地址]
```

- 方式 B：使用 [Subconverter](https://github.com/tindy2013/subconverter/blob/master/README-cn.md)
你可以用如下格式生成配置文件：
```bash
https://subconverter.yourdomain.com/sub?target=clash&url=你的订阅链接&udp=true
```

{{< details summary="为什么要用配置转换？" >}}
>使用配置转换（例如 Subconverter 或机场提供的转换服务）的原因主要是：不同代理客户端（如 Clash、Surge、Shadowrocket、V2RayN 等）使用的配置格式不同，而订阅链接提供的数据格式通常是通用或原始的（如 ss://, vmess://, trojan:// 等），需要转换成目标客户端支持的格式才能正常使用。
{{< /details >}}

## 运行 clash.meta
```bash
./clash.meta
```
初次尝试运行 clash.meta 时可能会报错：
```text
ERRO can't initial GeoIP: can't download MMDB: context deadline exceeded
FATA Parse config error: rules[...] [GEOIP,CN,直接连接] error: can't download MMDB: context deadline exceeded
```
通常是因为 Clash 在尝试在线下载 GeoIP 数据文件（通常是 MaxMind 的 Country.mmdb）时，由于网络问题（如连接超时、被墙等）导致失败，从而导致配置中 GEOIP 规则无法生效，程序直接退出.

> GEOIP会匹配符合国家代码的 IP 地址，例如 `GEOIP,cn,DIRECT `就会让所有中国的 IP 地址直连.

解决方案：
- 方法一：使用国内可访问的 CDN 或可以访问外网的代理手动下载：
```bash
wget https://cdn.jsdelivr.net/gh/Dreamacro/maxmind-geoip@release/Country.mmdb -O Country.mmdb
```
将其放入 Clash 的配置目录中（与 config.yaml 同目录）.

- 方法二：暂时注释掉所有 GEOIP 规则行

如果你第一次运行 Clash 时本身没有代理环境，那么它就无法联网下载 GeoIP 文件.

可以先：
1. 暂时注释掉所有 GEOIP 规则行（比如：GEOIP,CN,直接连接），启动 Clash；
2. Clash 成功运行后，你已具备代理环境；
3. 然后再恢复 GEOIP 规则，重新运行 Clash，即可让它联网下载 mmdb 成功.

再次运行 `clash.meta` ，如果出现形如下面的输出，说明可以正常代理：
```
INFO[2025-06-10T15:08:01.720945365+08:00] Start initial configuration in progress
INFO[2025-06-10T15:08:01.723899622+08:00] Geodata Loader mode: memconservative
INFO[2025-06-10T15:08:01.723923124+08:00] Geosite Matcher implementation: succinct
INFO[2025-06-10T15:08:01.738022930+08:00] Initial configuration complete, total time: 16ms
```

## 测试代理
### 设置全局系统代理
本文的配置环境为 Ubuntu 22，其他版本的代理配置方式可能不同.
```bash
sudo vi /etc/environment
```
Clash 默认的 Http、Https 端口为 7890，socks 端口为 7891，混合端口为 7892.

向 `/etc/environment`添加：
```
http_proxy="http://127.0.0.1:7890"
https_proxy="http://127.0.0.1:7890"
# ...
```

使配置生效：
```bash
source /etc/environment
```

### 测试是否生效
```bash
curl --proxy http://127.0.0.1:7890 https://api.ipify.org
```
应该返回代理出口的 IP .

## 配置为系统服务
### 创建 clash 用户
```bash
sudo useradd -r -s /usr/sbin/nologin clash
sudo chown -R clash:clash /opt/clash
```
{{< details summary="为什么建议创建单独的用户？" >}}
1. 最小权限原则（Principle of Least Privilege）
    -	Clash 不需要管理员权限来运行；
    -	用 root 启动 Clash 会让它拥有系统级权限，一旦程序存在漏洞或被劫持，将可能危及整个系统安全；
    - 专用用户权限受限，即使发生安全问题，影响范围也受限于该用户的目录和资源。

2. 避免误操作或冲突
    - 系统服务应该与普通用户隔离；
    - 使用专用 clash 用户，避免和你日常账户下的文件、权限、环境变量等相互影响；
    - 多用户环境下，防止其他用户意外终止 Clash 或篡改配置。

{{< /details >}}

### 创建 Systemd 服务文件
```bash
sudo vi /etc/systemd/system/clash.service
```
填入以下内容
```bash
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

### 拷贝运行文件
```bash
# 在当前 Clash 工作目录下
sudo cp * /opt/clash
```

### 启动服务
```bash
sudo systemctl daemon-reexec      # 推荐重载 systemd
sudo systemctl daemon-reload      # 重新加载服务配置
sudo systemctl enable clash       # 开机自启
sudo systemctl start clash        # 启动 Clash 服务
```

### 查看运行状态和日志
```bash
# 查看服务状态
sudo systemctl status clash
# 查看运行日志（实时）
journalctl -u clash -f
```

## 配置订阅自动更新

### 创建更新脚本
```bash
sudo vi /opt/clash/update-clash.sh
```
填入以下内容
```bash
#!/bin/bash

CLASH_DIR="/opt/clash"
SUB_URL="[你的订阅链接]"
CONFIG_FILE="$CLASH_DIR/config.yaml"
BACKUP_FILE="$CLASH_DIR/config.yaml.bak"
TMP_FILE="$CLASH_DIR/config.tmp.yaml"

cp "$CONFIG_FILE" "$BACKUP_FILE"

echo "下载 Clash 配置订阅中..."
if curl -fsSL "$SUB_URL" -o "$TMP_FILE"; then
    mv "$TMP_FILE" "$CONFIG_FILE"
    echo "配置更新成功，重新启动 Clash 服务..."
    systemctl restart clash
else
    echo "下载失败，恢复旧配置"
    mv "$BACKUP_FILE" "$CONFIG_FILE"
fi
```
赋予可执行权限
```bash
sudo chmod +x /opt/clash/update-clash.sh
```

### 创建定时任务
```bash
sudo crontab -e
```

添加以下行（每天 3 点更新）
```cron
0 3 * * * /opt/clash/update-clash.sh >> /opt/clash/update.log 2>&1
```
