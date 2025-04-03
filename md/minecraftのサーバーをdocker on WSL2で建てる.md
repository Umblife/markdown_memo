# minecraftのサーバーが動作する環境をdocker on WSL2で用意する

## 構成

- Client_PC (マインクラフトをプレイするマシン、Windows 11 Pro)
- Server_PC (マインクラフトサーバーを建てるPC、Windows 10 Pro)

## 前提

- wsl2(ubuntu 22.04)インストール済み、その中にdockerもインストール済み(docker desktopは使用していない)
- [サーバーファイル](https://www.minecraft.net/ja-jp/download/server)はD:\server\sample以下に配置したと想定(D:\server\sample\server.jar)

## dockerイメージの作成

javaの実行環境をubuntu:22.04イメージベースで用意する。jdkのバージョンは動かしたいマイクラのバージョンに合わせる。

バニラ 1.20.1、minecraft forge 1.20.1-47.2.0が動作することを確認済み。

```Dockerfile:Dockerfile
FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

# isntall packages
RUN apt update \
    && apt upgrade -y \
    && apt install -y tzdata openjdk-18-jre
ENV TZ Asia/Tokyo

WORKDIR /workspace
```

イメージのビルド。ここではmcserver:baseという名前で作成

```bash:
docker build . -t mcserver:base
```

## docker-compose.yamlの作成

```yaml:docker-compose.yaml
version: '3'

services:
  mcserver:
    image: mcserver:base
    volumes:
      - '/mnt/d/server/sample:/workspace'
    ports:
      - '25565:25565'
    tty: true
    stdin_open: true
```

## WindowsとWSL2間のポートフォワーディング

dockerコンテナとwsl2間ではdocker-compose.yaml内portsの項目で設定しているものの、Server_PCのWindows:25565からwsl2:25565が繋がっていないため、Client_PCから接続しようにも接続できない。そこで以下のshスクリプトを作成し実行する

```bash
# get localhost IP from wsl2
IP=$(ifconfig eth0 | grep 'inet ' | awk '{print $2}')
WINPORT=25565
WSLPORT=25565

# port fowarding
netsh.exe interface portproxy delete v4tov4 listenaddress=* listenport=$WINPORT
netsh.exe interface portproxy add v4tov4 listenaddress=* listenport=$WINPORT connectaddress=$IP connectport=$WSLPORT

# run IP Helper
sc.exe config iphlpsvc start=auto
sc.exe start iphlpsvc
```

## 参考にしたサイト
