---
layout: post
title: "Dockerを使ってCucumberテストを並列実行する"
date: 2013-09-10 22:15
comments: true
categories: 
---
![](/images/homepage-docker-logo.png)
## Dockerってなに？
[Docker](https://www.docker.io/) は一言で言えばLXCのラッパーです。Dockerを使うことですこし面倒なLXCをとても簡単に操作することができるようになります。それに加えて、Dockerは"Union File System"という機能があり、これのおかげでLXCコンテナのバージョン管理を_commit_や_push_など使い慣れたインターフェイスで操作することができます。まるでGitのLXC版のような感覚です。

LXCとは仮想化技術のひとつでホストマシンからは隔離されたコンテナと呼ばれる仮想マシンを実行することができます。ハイパーバイザー型の仮想化の代表であるXen Serverなどとは異なり、LXCは軽量な仮想マシンを作成することができます。とても軽量なので、通常は同じリソースでハイパーバイザー型の仮想マシンよりも多くのマシンを実行することができます。

この軽量とLXCの性質が生きてくるのは、多くのテストを実行することだと思います。

テストを実行する時、できるだけ各テストが他のテストに影響を与えないことに気を配っていると思います。これをするためには、いくつもの仮想マシンを立ち上げて、そこで各テストを実行するというやり方があると思います。ハイパーバイザー型の仮想マシンでもこの方法を実行できますが、ハイパーバイザー型のマシンは開始/停止に時間がかかるため、テスト全体の実行時間が増えてしまうという欠点があります。
しかし、LXCならこの問題を克服できます。なぜなら、LXCではマシンを実行するのは１つのプロセスを起動するのと同じくらい軽量だからです。

## Dockerをインストールしてみよう
DockerのインストールはUbuntu12.04を使っていればとても簡単です。

以下は[Dockerのサイトに](http://docs.docker.io/en/latest/installation/ubuntulinux/)に記載されていた手順です。

```bash
sudo apt-get update
sudo apt-get install linux-image-generic-lts-raring linux-headers-generic-lts-raring
sudo reboot
sudo sh -c "curl https://get.docker.io/gpg | apt-key add -"
sudo sh -c "echo deb https://get.docker.io/ubuntu docker main > /etc/apt/sources.list.d/docker.list"
sudo apt-get update
sudo apt-get install lxc-docker
```

これでコンテナをDockerから作成できるようになりました。

```bash
sudo docker run -i -t ubuntu /bin/bash
```

もし、Macを使っているならDockerをVagramtのUbuntuマシンで試すことができます。この場合のインストール方法は[ここ](http://docs.docker.io/en/latest/installation/vagrant/)にあります。
しかし、初めはUbuntuにインストールして試してみることをおすすめします。一度Vagrant上で動かしてみましたが、Ubuntuで動かしたよりもコンテナの実行速度が遅く感じられました。もしかしたら、すでに仮想化されているVagrant上で実行しているからかもしれません。

## 環境をセットアップする
上述したようにDockerはGitにとてもよく似たインターフェイスを備えています。Dockerのイメージを[Dockerのパブリックレポジトリ](https://index.docker.io/)からPullしてきてそれをベースにして自分の好きな変更を加えてCommitしたら、またそれをPushできます。
今回はCucumberのテストを実行するのでRubyの実行環境があるコンテナが必要になります。もちろん、自分でコンテナを作成することもできますが今回は私が作成したイメージを使いましょう。

```bash
# まずrootユーザになります
sudo -s

# イメージをPullします
docker pull kimh/ruby-base

# イメージでechoコマンドを実行してみましょう
docker run kimh/ruby-base echo "Running on Docker"
```

今度はコンテナにログインして、テスト用のアプリケーションをインストールしましょう。

```bash
# コンテナにログインします
docker run -i -t kimh/ruby-base /bin/bash
cd /git
git clone https://github.com/kimh/docker_demo
cd docker_demo/ci_app
bundle install
```

この時点で２つの比較的時間がかかる作業をしたので（イメージのPullとbundle installです）コンテナの変更を保存しましょう。それをするには、変更をCommitしてイメージに保存します。

```bash
# ここで exit とタイプしてはダメです！タイプするとマシンが終了して一からやり直しになってしまいます。
# 代わりに、Ctrl+p とタイプしてから Ctrl+q とタイプすることで終了せずにコンソールから抜けることができます。
Ctrl+p
Ctrl+q
# コンテナのidを調べます
docker ps # 今回はidは 23fd82dcc088 でした。多分実行環境によって違うかも？
docker commit 23fd82dcc088 kimh/ruby-base
```

これでCucumberテストを実行する環境が整いました。コンテナを起動してテストを一つ実行してみましょう。

```bash
docker run kimh/ruby-base /bin/bash -c "
  source /etc/profile
  cd /git/docker_demo/ci_app
  export LC_CTYPE="ja_JP.UTF-8"
  export RAILS_ENV=test
  bundle exec rake cucumber
"
```

このスクリプトでDockerはCucumberテストをkimh/ruby-baseコンテナ上で実行しました。

いよいよ最後にテストを並列実行してみましょう。考え方としては複数のコンテナを実行して、各コンテナに一つのCucumberテストを実行させます。今回は５つのコンテナを並列で実行してみましょう。

```bash
DID=""
container="kimh/ruby-base:latest"
dir="/git/docker_demo/ci_app"

for feature in {0..4}
do
  DID=$DID" "`sudo docker run -d $container /bin/bash -c "
  source /etc/profile
  cd $dir
  export LC_CTYPE="ja_JP.UTF-8"
  export RAILS_ENV=test
  bundle exec rake cucumber
  "`
done
docker wait $DID
```

このスクリプトのポイントは２つです。

１つめは、実行したコンテナのidをDIDという変数に保存していることです。この変数は各コンテナのステータスコード（テストが成功したか失敗したか）を後で知る必要があるからです。

２つめは、_docker wait_コマンドにコンテナの実行idを渡していることです。こうすると、シェルはコンテナが終了するまでブロックして、各コンテナのステータスコードを返します。

今回だとすべてのテストはパスするはずなので、五回連続で0が表示されるはずです。（0はCucumberではパスしたという意味です）

# まとめ
今回はDockerの基本的な使い方とそれを使ってのテストの並列実行をしました。各テストはコンテナ上の独立した環境で実行されます。今回の例では同じテストを５台のコンテナ上で実行したのであまり意味はありませんが、
異なる複数のテストを並列で実行するのも簡単にできます。今回はこんな感じでやりました。

```bash
DID=""
container="kimh/ruby-base:nice"
dir="/git/your_repo"

for feature in `find  ./features/`
do
  DID=$DID" "`docker run -d $container /bin/bash -c "
  source /etc/profile
  cd $dir
  export LC_CTYPE="ja_JP.UTF-8"
  export RAILS_ENV=test
  bundle exec cucumber $feature -r features/
  "`
done
docker wait $DID
```

# ちょっと残念な結果
今回のCucumberテストはとてもシンプルなテストなので、すべてのテストはとても早く終了しました。しかし、今私が実際に関わっているプロジェクトのテストを並列実行したところ、思ったよりも早くありませんでした。
50台のコンテナをLinuxの物理ホストで実行することはできましたが、それでも全部終了するのに１時間くらいかかってしまいました。詳しくは調べてみないとわかりませんが、うまくコンテナにリソースが割り当てられていないような感じでした。
なので、次はどうやってコンテナを早く実行するか調べてみようと思います。

いいアイデアが見つかり次第このブログで紹介します。
