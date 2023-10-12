# Docker Desktopを使用せずに、Windows上でDocker環境を用意する

## はじめに

GPUを利用するためにはバージョン21H2以上が必要

WSL上のLinuxディストリビューションにDocker Engineをインストールすることでタイトルが実現する

- [参考](https://dev.to/_nicolas_louis_/how-to-run-docker-on-windows-without-docker-desktop-hik)

## Windows Subsystem for Linux(WSL)のインストール

[公式](https://docs.microsoft.com/ja-jp/windows/wsl/install-manual)の手順通りに行う

1. Windows Subsystem for Linux(WSL)の有効化

    管理者権限で起動して以下を実行

    ```bash
    # Windows Subsystem for Linuxの有効化
    dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart

    # 仮想マシンプラットフォームのオプションコンポーネントを有効化
    dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
    ```

    コントロールパネルから有効化することもできる

    - コントロールパネル -> プログラム -> プログラムと機能 -> Windowsの機能の有効化と無効化を選択し、Linux用Windowsサブシステム、仮想マシンプラットフォームにチェックを入れる。

  終わったらPCを再起動

1. Linuxカーネル更新プログラム パッケージをダウンロード＆インストール

    https://docs.microsoft.com/ja-jp/windows/wsl/install-manual#step-4---download-the-linux-kernel-update-package

1. WSL2を既定のバージョンに設定

    ```bash
    wsl --set-default-version 2
    ```

1. Linuxディストリビューションをインストール

    Microsoft Storeから適当なディストリビューションをインストール。初回はユーザー名などの設定があるため、適当に設定する。

    ### Storeからインストールできない場合

    [ディストリビューションをダウンロード](https://docs.microsoft.com/ja-jp/windows/wsl/install-manual#downloading-distributions)して、ローカルインストールする。ダウンロード後以下を実行([ダブルクリックでもインストールできるようになった？](https://docs.microsoft.com/ja-jp/windows/msix/app-installer/troubleshoot-appinstaller-issues))。

    ```bash
    dism.exe /Online /Add-ProvisionedAppxPackage /PackagePath:[ダウンロードしたファイルのパス] /SkipLicense
    ```

    例えばUbuntu 20.04をインストールした場合、Windowsキーを押して「Ubuntu 20.04」と検索するとアプリが表示されるはずなので、実際に起動するかを確かめる。

    コマンドでインストールできなかった場合は[こちら](https://phst.hateblo.jp/entry/2020/01/18/000000)を参考にインストールする

1. 起動したディストリビューションにプロキシ設定(__プロキシないならスキップ__)

    DNSとプロキシを環境に合わせて設定

    <details><summary>Ubuntu 20.04での設定例</summary><div>

    
    - resolv.confの上書きを無効化

        /etc/wsl.confを作成して以下を記述

        ```conf
        [network]
        generateResolvConf = false
        ```

    - /etc/resolv.confの編集

        もともとはシンボリックリンクなのでまずはunlink。unlinkせずリンク先の本体を編集してもOK

        ```bash
        sudo unlink /etc/resolv.conf
        sudo vi /etc/resolv.conf
        ```

        以下を追記

        ```conf
        nameserver xxx.xxx.xxx.xxx  #優先DNS
        nameserver yyy.yyy.yyy.yyy  #代替DNS
        domain aaa.bbb.co.jp
        ```

    - /etc/apt/apt.conf.d/02proxy

        以下を追記。なければ新規作成

        ```conf
        Acquire::http::Proxy "http://xxxxx:port";
        Acquire::https::Proxy "http://xxxxx:port";
        ```

    - /etc/profile.d/proxy.shの作成

        以下を記述して保存

        ```sh
        export http_proxy=http://xxxxx:port/
        export https_proxy=http://xxxxx:port/
        ```

    - その他のプログラムごとの設定

        [ここ](https://lambdalisue.hatenablog.com/entry/2013/06/25/140630)を参考に

    </div></details>

  設定を終えたらwslを再起動

## WSL2に用意したディストリビューション上にDockerをインストール

LinuxディストリビューションはUbuntu 20.04で確認。

1. Linux系OSおける[通常の手順](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)と同様にインストール。

    必要パッケージのインストール

    ```bash
    sudo apt update && sudo apt install \
        ca-certificates \
        curl \
        gnupg \
        lsb-release
    ```

    docker公式のGPGキーの追加

    ```bash
    sudo mkdir -p /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
    ```

    最新の安定版のリポジトリを設定

    ```bash
    echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    ```

    docker engine (& docker compose)のインストール
    ```bash
    sudo apt update && sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin
    ```

    特にエラーが出なければインストール自体はこれで完了

1. 一般ユーザーへの権限付与

    ```bash
    sudo usermod -aG docker $USER
    ```

1. Docker Deamonの起動

    ```bash
    sudo /etc/init.d/docker start
    ```

1. dockerにプロキシの設定

    ここもプロキシなければスキップ。/etc/default/dockerに以下を追記。再起動後に反映される。

    ```config
    export http_proxy="http://xxxxx:port"
    export https_proxy="http://xxxxx:port"
    ```

1. dockerコマンドをsudoなしで実行できるようにする

    もともとsudoなしで実行できるのであればこの手順は不要。以下のコマンドを順に実行

    ```bash
    # dockerグループがなければ作る
    sudo groupadd docker

    # 現行ユーザをdockerグループに所属させる
    sudo gpasswd -a $USER docker
    ```

1. 自動起動の設定

    このままだとWSLを再起動するたびにDocker Deamonを手動で起動する必要があるが、以下により自動で起動するように設定できる

    sudoersに以下を追記(ユーザ名はLinuxディストリビューションに設定したものへ変更)

    ```bashrc
    ユーザ名 ALL=NOPASSWD: /usr/sbin/service docker start, /usr/sbin/service docker stop, /usr/sbin/service docker restart
    ```

    .bashrc(zshなら.zshrc)に以下を追記

    ```bashrc
    # start docker deamon
    if test $(service docker status | awk '{print $4}') = 'not'; then
        sudo service docker start
    fi
    ```

    ### Ubuntu 22.04での注意点

    ```bash
    sudo service docker start
    ```

    と実行し、

    ```bash
    * Starting Docker: docker                        [ OK ]
    ```

    と出ているのに、いざdocker関連のコマンドを打つと

    ```bash
    docker: Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?. See 'docker run --help'.
    ```

    と出て、実行できない場合がある。そのときは以下を入力し、iptables-legacyモードに変更すると解決する

    ```bash
    sudo update-alternatives --config iptables
    ```

これにより、今後はWSL上でdockerコマンドを使用できるようになる。Windowsのパスはそのままでは使用できないため、WSLから見たパスで置き換える必要がある点に注意。

## NVIDIA Container Toolkitのインストール(NVIDIA製GPUを使用したい場合)

公式ガイドの[Setting up NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#setting-up-nvidia-container-toolkit)に従ってインストール

1. リポジトリとGPGキーの設定

    ```bash
    distribution=$(. /etc/os-release;echo $ID$VERSION_ID) \
        && curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
        && curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | \
            sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
            sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
    ```

1. nvidia-docker2のインストール

    ```bash
    sudo apt update && sudo apt install -y nvidia-docker2
    ```

インストールが終わったらdocker engineを再起動

## おまけ: VS Code Remote Container

Docker Desktopが使用できなくなって地味に影響を受けるのがVS CodeのRemote Container拡張機能。

ただ、こちらは設定で

> Remote > Containers: Execute In WSL

にチェックを入れるだけ、もしくはsettings.jsonに以下を追加で解決する。

```json
"remote.containers.executeInWSL": true,
```

これでこれまで通り、リモートエクスプローラー -> Containersから接続したいコンテナを選択できる。

別マシン上でたてたコンテナに接続したい際も、VS CodeのDocker拡張機能をインストールして以前と同様settings.jsonに以下を追記するだけ。

```json
"docker.host": "ssh://リモート接続先のユーザ名@ipアドレス",
```
