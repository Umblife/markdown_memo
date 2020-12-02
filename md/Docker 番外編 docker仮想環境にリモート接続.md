# リモートマシン上のDockerコンテナにクライアントのVSCodeから多段sshで接続

[part2](Docker%20part2.%20docker仮想環境にリモート接続.md)ではVSCodeの[Remote Container](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)機能を使いコンテナにアクセスする設定の説明をした。こちらでは[Remote SSH](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh)機能のみでVSCodeからコンテナへアクセスする説明をする。基本的にpart2での方法で問題ないが、例えばDockerコンテナが起動しているホストマシン自体に多段sshをしなければ接続できない場合などで、こちらの設定を使う必要がある。

## はじめに

基本的な前提はpart2と同じ。最低限はあちらを参考に設定しておく。今回はコンテナに対しssh接続をするため、コンテナにsshサーバーが設定されていなければならない。そのため以下のステップに分けて解説する。

1. sshdを導入したDockerイメージの作成
2. ssh-agentに秘密鍵の登録
3. コンテナに公開鍵の登録
4. 多段sshの設定

---

## ①sshサーバーをインストールしたDockerイメージの作成

設定している事自体はシンプルで、サーバーを立ててからssh接続できるまでの設定をDockerfileの中に記述すればよい。作成したいDockerイメージのDockerfileの中に以下の記述を追加する。

[公式ガイド](https://docs.docker.jp/engine/examples/running_ssh_service.html)

```dockerfile
# ssh-serverのインストール
RUN apt-get update && apt-get install -y openssh-server
# sshd用のディレクトリ作成
RUN mkdir /var/run/sshd
# rootパスワード(この場合emptypasswd)の設定。ただし次の設定でパスワードでは接続禁止
RUN echo 'root:emptypasswd' | chpasswd
# rootでのログイン許可。ただしパスワードでの接続は禁止
RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
# ポートの変更とそのポートの開放。ポート変更は別にしなくてもいい
RUN sed -i 's/#Port 22/Port 20022/' /etc/ssh/sshd_config
EXPOSE 20022
# サーバーの起動。
CMD ["/usr/sbin/sshd", "-D"]
```

これにより作成されたイメージをもとにしたコンテナはssh接続が可能になる。ただし、設定でパスワードでのログインは禁止しているため、コンテナに公開鍵を渡す必要がある。パスワードでもログインしたい場合は以下を参考に上の設定を変更する。

```dockerfile
# rootでのログイン許可。ただしパスワードでの接続は禁止
RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
# rootでのログイン許可。パスワードでも接続可能
RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
```

当然ながらパスワードログインの際のパスワードは以下の行で設定されているものであるため、このパスワードを知っている人であれば誰でもssh接続できるコンテナとなってしまう。すべての設定が終わり目的のDockerコンテナに公開鍵認証でssh接続できるようになった後は、rootユーザーでのパスワードログインは禁止するよう設定を変えることをオススメする

```dockerfile
RUN echo 'root:emptypasswd' | chpasswd
```

---

## ②ssh-agentに秘密鍵の登録

クライアントでの設定。してなかったら登録

```bash
ssh-add -K [秘密鍵のパス]
```

---

## ③コンテナに公開鍵の登録

①で作成したイメージを元にコンテナを立ち上げる([part1](Docker%20part1.%20dockerで仮想環境の用意.md)参照)。あとは普通の公開鍵登録と同じ。ただ、コンテナは基本(今回の例でも勿論)rootユーザーであるため若干パスが違う。注意するのは、ここで登録するのはクライアントの公開鍵である点

```bash
# まずは作成したコンテナへ接続
docker exec -it [コンテナ名] bash
# 以下のディレクトリに移動。なかったら作成
cd /root/.ssh/
# authorized_keysファイルの作成。まず無いとは思うけど既にファイルあったらこのコマンドは無視
touch /root/.ssh/authorized_keys
```

あとはこのauthorized_keysにvimなりcatコマンド使うなりで公開鍵を登録

```bash
# vimで開いたらコピペなり何なり。
vi /root/.ssh/authorized_keys
# catコマンドで。登録したい公開鍵は予めコンテナ内にどうにかして送る
cat [登録したい公開鍵ファイル] >> /root/.ssh/authorized_keys
```

結構忘れがちなのはauthorized_keysの権限設定。これを600にしておかないとssh接続の際にグダる

```bash
chmod 600 /root/.ssh/authorized_keys
```

ここまでダラダラと設定方法を書いたが、実は①のDockerfileの設定の際に以下の1行入れるだけで良かったりする。作成したDockerイメージもコンテナも、自分しか使わないのであればこっちの方がずっと楽。イメージが共有されるにしても、一応「公開」鍵で誰にバレようと問題ないわけだし

```dockerfile
COPY [コピー元公開鍵のパス] /root/.ssh/authorized_keys
```

---

## ④多段sshの設定

ここもクライアントでの設定。例としてClient -> Host1 -> Host2 -> Host2上のDockerコンテナと多段sshすることを考える。まずは~/.ssh/configを開く。多分大体は以下のような感じのはず

```configs
Host Host1
  HostName xxx.xxx.xx.xx
  User Hoge1

Host Host2
  HostName yyy.yyy.yy.yy
  User Hoge2
  Port 22222
```

ここでClientからHost1を踏み台にHost2に多段sshをする場合、以下のような設定になる。ただし、Clientの公開鍵をHost2に登録しておく必要がある

```configs
Host Host1
  HostName xxx.xxx.xx.xx
  User Hoge1

Host Host2
  HostName yyy.yyy.yy.yy
  User Hoge2
  Port 22222
  ProxyCommand ssh -W %h:%p Host1
```

多段sshでコンテナまで接続する際も同じように設定すればよい。Dockerコンテナのアドレスは以下のコマンドで調べられる

```bash
# 以下のコマンドで出てきた中でIPAddressという項目がそれ
docker inspect [コンテナ名]
```

仮にDockerコンテナのアドレスがzzz.zzz.zz.zzだった場合以下のように設定

```configs
Host Host1
  HostName xxx.xxx.xx.xx
  User Hoge1

Host Host2
  HostName yyy.yyy.yy.yy
  User Hoge2
  Port 22222
  ProxyCommand ssh -W %h:%p Host1

Host Host2-DockerContainer
  HostName zzz.zzz.zz.zz
  User root
  Port 20022
  ProxyCommand ssh -W %h:%p Host2
```

これで多段sshの設定は終了。接続テストをしてみよう

---

これらすべての設定が終われば、VSCodeのRemote-SSH機能を使うことでコンテナへ接続できる
