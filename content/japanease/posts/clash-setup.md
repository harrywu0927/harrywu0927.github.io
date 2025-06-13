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

echo "設定をダウンロード中..."
if curl -fsSL "$SUB_URL" -o "$TMP_FILE"; then
    cp "$CONFIG_FILE" "$BACKUP_FILE"
    mv "$TMP_FILE" "$CONFIG_FILE"
    echo "更新成功、Clash を再起動します..."
    systemctl restart clash
else
    echo "ダウンロード失敗、旧設定を復元します"
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

## Web UI の設定
事前にビルドされた UI を直接取得し、現在のディレクトリ（/opt/clash/ui）に保存します:
```bash
git clone https://github.com/metacubex/metacubexd.git -b gh-pages ./ui
```

### config.yaml の編集
`config.yaml` ファイルで、`external-ui` フィールドを編集します:
```yaml
external-controller: 127.0.0.1:9090 # ローカルネットワークへのアクセスを許可する場合は 0.0.0.0:9090 にバインド
external-ui: ui
secret: 「yourpassword」  # 任意のパスワードを設定可能
```

Clash を再起動：
```bash
sudo systemctl restart clash
```

ブラウザでアクセス：http://localhost:9090/ui/

UI の更新を取得：
```bash
sudo git -C /opt/clash/ui pull -r
```

### 自動更新サブスクリプションスクリプトの更新
サブスクリプションを更新するたびに現在の `config.yaml` が上書きされるため、`update-clash.sh` に以下のコードを追加する必要があります：
{{< highlight bash "linenos=inline, hl_lines= 9-10 18-30" >}}
#!/bin/bash

CLASH_DIR="/opt/clash"
SUB_URL="[あなたのサブスクリプションリンク]"
CONFIG_FILE="$CLASH_DIR/config.yaml"
BACKUP_FILE=「$CLASH_DIR/config.yaml.bak」
TMP_FILE="$CLASH_DIR/config.tmp.yaml"

# Web UI の更新を取得
git -C /opt/clash/ui pull -r

echo 「Clash 設定のサブスクリプションをダウンロード中...」
if curl -fsSL 「$SUB_URL」 -o 「$TMP_FILE」; then
    # バックアップを作成
    cp 「$CONFIG_FILE」 「$BACKUP_FILE」
    mv 「$TMP_FILE」 「$CONFIG_FILE」

    # 設定を確認し追加
    sed -i 『0,/^[[:space:]]*.*external-ui:/s|^\([[:space:]]*\).*external-ui:.*|\1external-ui: ui|』 「$CONFIG_FILE」

    sed -i "0,/^[[:space:]]*.*external-controller:/s|^\([[:space:]]*\). *external-controller:.*|\1external-controller: 『0.0.0.0:9090』|「 」$CONFIG_FILE"


    if grep -q 「^secret:」 「$CONFIG_FILE」; then
        # 既存の secret を置き換え
        sed -i 's/^secret:. */secret: 「mypasswd」/' 「$CONFIG_FILE」
    else
        # 存在しない場合は追加
        echo 『secret: 「mypasswd」』 >> 「$CONFIG_FILE」
    fi

    echo 「設定更新成功、Clash サービスを再起動中...」
    sudo systemctl restart clash
else
    echo 「ダウンロード失敗、旧設定を復元」
fi
{{< /highlight >}}
