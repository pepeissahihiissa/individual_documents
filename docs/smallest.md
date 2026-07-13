以下、素のUbuntu 26.04 + Windows で Nextcloud AIO + EuroOffice を動かす最小手順です。

---

## 0. 前提

- Ubuntu 26.04 LTS（VM or 実機）
- Windows 11（ブラウザアクセス用）
- 同一LAN内
- 外部公開不要

---

## 1. Ubuntu初期セットアップ

```
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget ca-certificates
```

## 2. Dockerインストール

```
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
```
→ ログアウト＆再ログイン

## 3. Caddyインストール

```
sudo apt install -y debian-keyring debian-archive-keyring
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update && sudo apt install -y caddy
```

## 4. Caddyfile設定

```
# /etc/caddy/Caddyfile
cloud.home.arpa {
    reverse_proxy http://localhost:11000
}
```

```
sudo systemctl enable --now caddy
```

## 5. AIO起動

```
sudo docker run \
  --sig-proxy=false \
  --name nextcloud-aio-mastercontainer \
  --restart always \
  --publish 8080:8080 \
  --env APACHE_PORT=11000 \
  --env APACHE_IP_BINDING=0.0.0.0 \
  --env SKIP_DOMAIN_VALIDATION=true \
  --volume nextcloud_aio_mastercontainer:/mnt/docker-aio-config \
  --volume /var/run/docker.sock:/var/run/docker.sock:ro \
  nextcloud/all-in-one:latest
```

## 6. AIO初期設定

1. http://localhost:8080 にアクセス
2. 初期パスワードを入力
3. ドメイン: `cloud.home.arpa`
4. EuroOffice を有効化 → Save & Apply
5. コンテナが全て Up になるまで待機

## 7. CA証明書をWindowsへ配布

```
sudo cp /var/lib/caddy/.local/share/caddy/pki/authorities/local/root.crt ~/lab-config/
```
→ root.crt をWindowsにコピー → ダブルクリック → 信頼されたルート証明機関にインストール

## 7'. CA証明書をLinuxに登録

Caddyの内部CA証明書をシステムCAとしてコピー
```
sudo cp /var/lib/caddy/.local/share/caddy/pki/authorities/local/root.crt /usr/local/share/ca-certificates/caddy-root.crt
```
CAデータベース更新
```
sudo update-ca-certificates
```
確認
```
curl https://cloud.home.arpa
```

## 8. EuroOffice設定: StorageUrl指定

```
sudo docker exec --user www-data nextcloud-aio-nextcloud \
  php occ config:app:set rich_commands storage_url \
  --value 'http://nextcloud-aio-apache:11000/'
```

## 9. 動作確認

Windowsブラウザ → `https://cloud.home.arpa` にアクセス
→ Nextcloudログイン → xlsxファイル作成 → 編集画面表示確認

---

## 最小手順のまとめ（たったこれだけ）

| # | やること | コマンドor操作 |
|---|---|---|
| 1 | Docker+Caddyインストール | `apt install` |
| 2 | Caddyfile書く | `/etc/caddy/Caddyfile` |
| 3 | AIO起動 | `docker run` (SKIP_DOMAIN_VALIDATION=true) |
| 4 | AIO管理画面で EuroOffice 有効化 | ブラウザ操作 |
| 5 | root.crtをWindowsにインストール | コピー→ダブルクリック |
| 6 | StorageUrl設定 | `occ config:app:set storage_url` |
| 7 | ブラウザで確認 | `https://cloud.home.arpa` |

ホストCaddy＋EuroOfficeで証明書エラーにハマる場合、**手順8（StorageUrl）が抜けている**ことが原因です。これだけで解決します。