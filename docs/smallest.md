以下、素のUbuntu 26.04 + Windows で Nextcloud AIO + EuroOffice を動かす最小手順です。

---

## 0. 前提

- Ubuntu 24.04 LTS（VM or 実機）
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
office.home.arpa {
    tls internal
    reverse_proxy http://localhost:11000
}
```

## 5. hosts設定

```
vi /etc/hosts
192.168.11.2 office.home.arpa
```

## 6. caddy再起動

```
sudo systemctl restart caddy
systemctl status caddy
```

## 7. AIO起動

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

## 8. AIO初期設定

1. http://localhost:8080 にアクセス
2. 初期パスワードを入力
3. ドメイン: `office.home.arpa`
4. Nextcloud Hub 26 Springインストールにチェック
5. EuroOffice を有効化 → Save & Apply
6. コンテナが全て Up になるまで待機

## 9. CA証明書をLinuxに登録

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
curl https://office.home.arpa
```
## 9'. CA証明書をWindowsへ配布(Windowsから使用する場合)

→ root.crt をWindowsにコピー → ダブルクリック → 信頼されたルート証明機関にインストール

## 10. CA証明書をBrave（ブラウザ）にインポート
```
sudo cp /var/lib/caddy/.local/share/caddy/pki/authorities/local/root.crt ~/Downloads/
sudo chmod 644 ~/Downloads/root.crt
```

Braveの証明書マネージャを開いて、信頼できる証明書のインポートを押して、証明書を読み込む

## 11. EuroOffice設定

Nextcloudに管理者でログインする
右上のメニューから管理者設定を選択する

サーバーから内部リクエストに使用されるNextcloud Officeアドレス：
http://nextcloud-aio-eurooffice/
Nextcloud Officeから内部リクエストに利用されるサーバーアドレス：
http://nextcloud-aio-apache:11000/

## 12. 動作確認

Windowsブラウザ → `https://office.home.arpa` にアクセス
→ Nextcloudログイン → xlsxファイル作成 → 編集画面表示確認

---



