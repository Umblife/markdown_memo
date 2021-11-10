# ホストマシンで起動しているDockerコンテナにクライアントのVSCodeから接続

[part1](Docker%20part1.%20dockerで仮想環境の用意.md)ではホストマシンにDockerを用いて仮想環境を用意する手順を説明した。ここでは自分のPCなどのVisual Studio Codeから、リモートでDockerコンテナの起動しているホストマシンにアクセスして操作するための設定について解説する。

注意点として、ホストマシンへの接続が多段ssh(例: クライアント -> 踏み台サーバ -> ホストマシン)の場合ここで説明する方法では接続できない。多段sshを用いた設定方法は[こちら](Docker%20番外編%20dockerコンテナに多段ssh接続.md)。

## 前提

- クライアントからホストへのssh接続はできるようにしておく
- ホスト・クライアントともにDockerがインストールされており、どちらもローカルで利用できる状態にある

WindowsにDockerをインストールする際は注意する点がいくつかあるので、以下のサイトあたりを参考に。~~ちなみにWindows上ではGPUが使用できるコンテナは作成できない~~Windows 11でサポートされた模様。

- [WindowsでDocker環境を試してみる](https://qiita.com/fkooo/items/d2fddef9091b906675ca)
- [Docker for Windowsをインストール](https://ops.jig-saw.com/tech-cate/docker-for-windows-install)

---

## ①VSCodeに拡張機能のインストール

以下の拡張機能をクライアントのVSCodeにインストールする

- [Docker](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-docker)
- [Remote SSH](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh)
- [Remote Container](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)

---

## ②VSCodeの設定

以下のどちらか好きな方でクライアントマシンに設定する。

- SSHプロトコル(推奨)

  ssh-agentに秘密鍵の登録(不要かも？)

  ```bash
  ssh-add -K [鍵のパス]
  ```

  VSCodeのsettings.jsonに、以下の設定を必要な箇所を訂正して追加

  ```config
  {
    "docker.host": "ssh://[リモートでのユーザ名]@[リモートマシンのアドレス]"
    // "docker.host": "ssh://tanaka@192.168.10.10"   example
  }
  ```

- TCPプロトコルの使用

  VSCodeでsshのconfigを開き、ホストへの接続設定の部分にLocalForwardの項目を追記

  以下はポート23750番を使う場合の例

  ```bash
  # 例えばもともと以下のような設定だった場合
  Host hostexample1
    HostName 192.168.10.10
    IdentityFile C:/Users/yourname/.ssh/id_rsa
    User Tanaka
    Port 22

  # LocalForward 23750 /var/run/docker.sockを追記
  Host hostexample1
    HostName 192.168.10.10
    IdentityFile C:/Users/yourname/.ssh/id_rsa
    User Tanaka
    Port 22
    LocalForward 23750 /var/run/docker.sock
  ```

  VSCodeのsettings.jsonに以下を追記

  ```json
  {
    "docker.host": "tcp://[リモートマシンのアドレス]:23750"
  }
  ```

参考

- [公式ガイド](https://code.visualstudio.com/docs/remote/containers-advanced#_developing-inside-a-container-on-a-remote-docker-host)

- [チュートリアル](https://code.visualstudio.com/docs/remote/containers-tutorial)

---

## ③VSCodeから接続

1. VSCode左のリモートエクスプローラー
1. プルダウンリスト(多分デフォルトはSSH Targets)からContainersを選択
1. 目的のコンテナに接続

---
