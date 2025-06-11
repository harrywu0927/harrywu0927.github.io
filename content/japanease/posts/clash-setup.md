+++
date = '2025-06-10T15:39:33+08:00'
draft = false
title = 'Ubuntu で Clash.Meta をデプロイし、システムサービス化とサブスクリプション自動更新を設定する'
+++

## Clash コアのダウンロード
Clash は Go 言語で開発された汎用プロキシクライアントのコアであり、vmess、vless、trojan、shadowsocks、socks、http など複数のプロトコルに対応しています。
GUI は付属せず、コアのロジックだけを実行するため、設定ファイルや Clash Verge、Meta UI、Yacd などのコントロールパネルと併用して使用します。

Clash のオリジナル版および Premium 版はすでに開発停止しており、元の Meta 版は現在 [mihomo](https://github.com/MetaCubeX/mihomo/tree/Alpha) に改名されています。

```bash
# Clash.Meta の最新バージョンをダウンロード（Linux amd64）
wget https://github.com/MetaCubeX/mihomo/releases/download/v1.19.10/mihomo-linux-amd64-v1.19.10.gz -O clash.meta.gz
gunzip clash.meta.gz
chmod +x clash.meta
```

## 設定ファイル config.yaml を準備

通常、プロキシサービス（通称「空港」）から提供されるサブスクリプションリンクを使用して Clash 用設定ファイルに変換する必要があります。

- 方法 A：手動ダウンロード
    1. 多くのサービスでは Clash 用 YAML 設定ファイルのダウンロードリンクが提供されます。
    2. ダウンロード後、`config.yaml` にリネームします。

例：SpeedCat 空港の変換済みサブスクリプションを利用：
```bash
wget -O config.yaml [あなたのサブスクリプションURL]
```

- 方法 B：[Subconverter](https://github.com/tindy2013/subconverter/blob/master/README-cn.md) を使用
以下のような形式で生成可能：
```bash
https://subconverter.yourdomain.com/sub?target=clash&url=あなたのリンク&udp=true
```

{{< details summary="なぜ変換が必要？" >}}
> 各プロキシクライアント（Clash、Surge、Shadowrocket、V2RayN など）は異なる設定形式を使用します。サブスクリプションリンクで提供される元のデータ（ss://、vmess:// 等）は一般形式のため、クライアントに合った形式に変換する必要があります。
{{< /details >}}

## clash.meta を実行
```bash
./clash.meta
```

初回実行時に以下のようなエラーが出る場合があります：
```text
ERRO can't initial GeoIP: can't download MMDB: context deadline exceeded
FATA Parse config error: rules[...] [GEOIP,CN,DIRECT] error: can't download MMDB: context deadline exceeded
```

これは Clash が GeoIP データ（MaxMind の Country.mmdb など）のダウンロードに失敗したためです。

> GEOIP は国コードに基づき IP を振り分けます。例：`GEOIP,CN,DIRECT` は中国の IP に直接接続を指示します。

対処方法：
- 方法1：CDN またはプロキシ経由で手動ダウンロード
```bash
wget https://cdn.jsdelivr.net/gh/Dreamacro/maxmind-geoip@release/Country.mmdb -O Country.mmdb
```
`config.yaml` と同じディレクトリに配置してください。

- 方法2：一時的に GEOIP 行をすべてコメントアウト

初回実行時にネットワークアクセスがなければ GeoIP ダウンロードができません。
まずコメントアウトして起動し、プロキシが動作したら再度有効化して再実行すれば OK です。

正常に起動すると以下のようなログが表示されます：
```
INFO[...] 初期設定が完了しました、所要時間: XXms
```

## プロキシのテスト
### グローバルシステムプロキシを設定（Ubuntu 22）
```bash
sudo vi /etc/environment
```

以下を追加：
```
http_proxy="http://127.0.0.1:7890"
https_proxy="http://127.0.0.1:7890"
```

反映：
```bash
source /etc/environment
```

### 動作確認
```bash
curl --proxy http://127.0.0.1:7890 https://api.ipify.org
```

## システムサービス化
### clash ユーザーを作成
```bash
sudo useradd -r -s /usr/sbin/nologin clash
sudo chown -R clash:clash /opt/clash
```

{{< details summary="専用ユーザーを作成する理由" >}}
- 最小権限の原則（必要な最小限の権限だけを付与）
- 他ユーザーや root との干渉を避ける
{{< /details >}}

### systemd サービスファイルを作成
```bash
sudo vi /etc/systemd/system/clash.service
```
以下を入力：
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

### 実行ファイルを配置
```bash
sudo cp * /opt/clash
```

### サービス起動
```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable clash
sudo systemctl start clash
```

### ログ確認
```bash
sudo systemctl status clash
journalctl -u clash -f
```

## サブスクリプションの自動更新

### 更新スクリプト作成
```bash
sudo vi /opt/clash/update-clash.sh
```
以下の内容を追加：
```bash
#!/bin/bash
CLASH_DIR="/opt/clash"
SUB_URL="[あなたのサブスクリプションURL]"
CONFIG_FILE="$CLASH_DIR/config.yaml"
BACKUP_FILE="$CLASH_DIR/config.yaml.bak"
TMP_FILE="$CLASH_DIR/config.tmp.yaml"

cp "$CONFIG_FILE" "$BACKUP_FILE"
echo "設定をダウンロード中..."
if curl -fsSL "$SUB_URL" -o "$TMP_FILE"; then
    mv "$TMP_FILE" "$CONFIG_FILE"
    echo "更新成功、Clash を再起動します..."
    systemctl restart clash
else
    echo "ダウンロード失敗、旧設定を復元します"
    mv "$BACKUP_FILE" "$CONFIG_FILE"
fi
```

実行権限を付与：
```bash
sudo chmod +x /opt/clash/update-clash.sh
```

### Cron に登録
```bash
sudo crontab -e
```

以下を追加（毎日午前3時に更新）：
```cron
0 3 * * * /opt/clash/update-clash.sh >> /opt/clash/update.log 2>&1
```
