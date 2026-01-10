# FileMaker Server 22.0.2 を Docker + Ubuntu 24.04 で構築する手順

## 2025-11-02

<https://support.claris.com/s/answerview?anum=000043656&language=en_US>

※以下、公式ページに記載のない補足・試行錯誤で必要だった手順には【独自補足】を付けています。

## ✅ スタート地点

```bash
docker pull ubuntu:24.04
```

---

## ⚙️ Docker Desktop 設定メモ（macOS）【独自補足】

Settings → General → Virtual Machine Options の "Choose file sharing implementation for your containers" は、デフォルト値の `VirtioFS` のままではインストールが失敗するため、必ず `gRPC FUSE` に切り替えること。

---

## 📁 ディレクトリ構成（例: `~/dockerenv/anglers-fms`）

```text
anglers-fms/
├── Dockerfile
├── tmp/
│   └── Assisted Install.txt
│   └── filemaker-server-22.0.2.204-arm64.deb
│   └── LicenseCert.fmcert
├── Database/       ← データ永続化先 インストール後に作成される
```

---

## 🧱 Dockerfile（FMS 22.0.2 専用 ubuntu のインストラーにふくまれている DockerFile を流用 ）

```Dockerfile
FROM ubuntu:24.04

# update all software download sources
RUN DEBIAN_FRONTEND=noninteractive      \
    apt update
 
# upgrade all installed software
# packages
RUN DEBIAN_FRONTEND=noninteractive      \
    apt full-upgrade                 -y

# update all software download sources
RUN DEBIAN_FRONTEND=noninteractive      \
    apt update

# install filemaker server dependencies
RUN DEBIAN_FRONTEND=noninteractive      \
    apt install                      -y \
        acl                             \
        apt-utils                       \
        apache2-bin                     \
        apache2-utils                   \
        ca-certificates                 \
        curl                            \
        expect                          \
        fonts-baekmuk                   \
        fonts-liberation2               \
        fonts-noto                      \
        fonts-takao                     \
        fonts-wqy-zenhei                \
        logrotate                       \
        lsb-release                     \
        net-tools                       \
        nginx                           \
        openssl                         \
        policycoreutils                 \
        sysstat                         \
        unzip                           \
        ufw                             \
        zip                             
 
# install user management
RUN DEBIAN_FRONTEND=noninteractive      \
    apt install                      -y \
        init
 
# clean up installations
RUN DEBIAN_FRONTEND=noninteractive      \
    apt --fix-broken install         -y
RUN DEBIAN_FRONTEND=noninteractive      \
    apt autoremove                   -y
RUN DEBIAN_FRONTEND=noninteractive      \
    apt clean                        -y

# COPY . /install/
 
# document the ports that may be
# published when filemaker server
# is installed
EXPOSE 80
EXPOSE 443
EXPOSE 2399
EXPOSE 5003
 
# when containers run, start this
# command as root to initialize
# user management
USER root
CMD ["/sbin/init"]

```

---

## 🔨 Dockerイメージビルド

```bash
docker build -t anglers-fms:prep .
```

---

## 🌐 ポート公開方針（重要）

- 通常運用は HTTPS（443）前提のため **80 は公開しない**
- 80 は **FileMaker Server 内蔵 Let’s Encrypt（HTTP-01）** を使う場合のみ必要

## 🚀 セットアップ用コンテナ起動

```bash
docker run \
  --detach \
  --hostname anglers-fms \
  --name anglers-fms \
  --privileged \
  --publish 443:443 \
  --publish 2399:2399 \
  --publish 5003:5003 \
  --volume ~/dockerenv/anglers-fms/tmp:/install \
  --volume ~/dockerenv/anglers-fms/Database:"/opt/FileMaker/FileMaker Server/Data" \
  anglers-fms:prep
```

※ Let’s Encrypt を使う場合のみ、上記に `--publish 80:80` を追加する。

---

## 🛠 セットアップ作業（コンテナ内）

-シェルに入る

```bash
docker exec -it anglers-fms /bin/bash
```

-パッケージ情報更新【独自補足】

```bash
apt-get update
```

-タイムゾーン設定【独自補足】

```bash
timedatectl set-timezone Asia/Tokyo
```

-インストーラ配置ディレクトリへ

```bash
cd /install
```

-サイレントインストール（Assisted Install 使用）

```bash
FM_ASSISTED_INSTALL=/install apt install /install/filemaker-server-22.0.2.204-arm64.deb
```

~~-不要な nginx を停止・自動起動無効化【独自補足】~~

```bash
systemctl stop nginx
systemctl disable nginx.service
```

↑ これは、FMSインストール時に実行されているので不要

---

## 📦 インストール済コンテナをイメージ化して保存

-セットアップ済コンテナをイメージ化（スナップショット保存）

```bash
docker commit anglers-fms anglers-fms:final
```

- 現在のコンテナ内状態（インストール済み、設定済み）を `anglers-fms:final` イメージとして保存。

-セットアップ用コンテナを停止

```bash
docker stop anglers-fms
```

- セットアップ専用コンテナを安全に停止。

-不要になったセットアップ用コンテナを削除

```bash
docker container rm anglers-fms
```

- 再利用しないセットアップ用コンテナを削除し、最終イメージから運用用コンテナを新規起動する前提にする。

---

## 🔁 運用用に再起動

```bash
docker run \
  --detach \
  --hostname anglers-fms \
  --name anglers-fms \
  --privileged \
  --publish 443:443 \
  --publish 2399:2399 \
  --publish 5003:5003 \
  --volume ~/dockerenv/anglers-fms/Database:"/opt/FileMaker/FileMaker Server/Data" \
  anglers-fms:final
```

※ Let’s Encrypt を使う場合のみ、上記に `--publish 80:80` を追加する。

---

## ✅ 初期化と管理コンソール有効化（必要な場合のみ）

FileMaker Server 22.0.2 の Docker イメージでは、インストール完了時点でサービスと Admin Console が自動的に有効化されているため、通常は追加操作は不要。管理者パスワードをリセットしたいなど、手動操作が必要なケースのみ以下を実行する。

```bash
docker exec -it anglers-fms /bin/bash

/bin/systemctl restart fmshelper.service
/usr/bin/fmsadmin resetpw admin YourPassword
/usr/bin/fmsadmin enable adminconsole
```

- FileMaker Server 22.0.2 は systemd 管理のため `/opt/FileMaker/FileMaker Server/bin/start.sh` は存在しない。再起動や状態確認は `systemctl` を利用する。
- `fmsadmin` は `/usr/bin/fmsadmin` に配置され、PATH に通っている。

---

## 🌐 アクセス確認

<https://localhost/admin-console>

---

## 🎉 完了

これで FileMaker Server 22.0.2 が Docker 環境上で完全に起動・管理可能な状態になった。
