# パーフェクトWSL2環境構築

```powershell
$ wsl --version
WSL バージョン: 2.1.5.0
カーネル バージョン: 5.15.146.1-2
WSLg バージョン: 1.0.60
MSRDC バージョン: 1.2.5105
Direct3D バージョン: 1.611.1-81528511
DXCore バージョン: 10.0.25131.1002-220531-1700.rs-onecore-base2-hyp
Windows バージョン: 10.0.22631.3593
```

## Dドライブへの移行

容量バク食いするような用途の際は推奨

エクスポート：
```bash
# wsl --export <Distribution Name> <FileName>
wsl --export Ubuntu-24.04 C:\tmp\ubuntu.tar
```

元の登録解除
```bash
# wsl --unregister <Distribution Name>
wsl --unregister Ubuntu-24.04
```

インポート：
```bash
# wsl --import <Distribution Name> <InstallLocation> <FileName>
wsl --import Ubuntu-24.04 D:\wsl C:\tmp\ubuntu.tar
```

移行後はデフォルトでrootになっているため、ユーザ名を変更。

```conf
# /etc/wsl.conf
[user]
default=user-name
```

## dockerのインストール

通常のubuntuと同様の手順でインストール

- https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

# install
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

sudoなしでdockerコマンドを使用できるようにする(実行後、ターミナルの再読込が必要)

```bash
sudo usermod -aG docker $USER
```

dockerデーモンの自動起動設定(wsl.confのsystemd=true)

```bash
sudo systemctl enable docker
```

<details><summary>systemd=falseの場合は以下を参考に</summary><div>

sudoersに以下を追記

```
ユーザ名 ALL=NOPASSWD: /usr/sbin/service docker start, /usr/sbin/service docker stop, /usr/sbin/service docker restart
```

.bashrc(zshなら.zshrc)に以下を追記

```bash
# start docker deamon
if test $(service docker status | awk '{print $4}') = 'not'; then
    sudo service docker start
fi
```

</div></details>

## NVIDIA Container Toolkitのインストール

GPU使うために必要

- https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html

```bash
# Configure the production repository:
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# Optionally, configure the repository to use experimental packages:
sudo sed -i -e '/experimental/ s/^#//g' /etc/apt/sources.list.d/nvidia-container-toolkit.list

# install
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
```

dockerでgpuを使用する際はインストール後にdockerデーモンの再起動をしておく
```bash
sudo systemctl restart docker
```

## その他

pythonパスの設定

```bash
sudo ln -s /usr/bin/python3 /usr/bin/python
```

zellijのインストール

```bash
# Rust and Cargo
curl https://sh.rustup.rs -sSf | sh

# after reload terminal,
cargo install --locked zellij
```
