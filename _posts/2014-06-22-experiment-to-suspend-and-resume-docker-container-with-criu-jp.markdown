---
layout: post
title: "CRIUを使ってDockerコンテナの停止/再開に挑戦"
date: 2014-06-22 21:18
comments: true
permalink: /blog/jp/criu/experiment-to-suspend-and-resume-docker-container-with-criu-jp/
categories:
- jp
- criu
---
![](/images/criu.jpeg)

### 結論: 2014/6の時点ではCRIUを使ってDockerコンテナの停止/再開はできないという少し残念な結果でこの記事は終わります。しかし、記事を読んでもらえればCRIUの面白さはわかってもらえると思います。

Dockerが急速に広まったことで、LXCがVMWareやXenなどのVMに対して持つ利点はとても明確になりました。

しかし、VMにあってLXCにはない機能が一つあります。それは、コンテナの停止/再開によるコンテナの状態の保存です。

ここで[CRIU](http://criu.org/Main_Page)の出番です。

CRIUはいわゆるCR(checkpoint/restart)ツールです。走っているプロセスを途中で止めてファイルに保存して、いつでも途中からプロセスを再開することができます。

LXCコンテナはプロセスなので、CRIUを使えばコンテナの停止/再開ができそうな気がしますが、本当にできるでしょうか？

この記事では、CRIUをインストールしてDockerコンテナの停止/再開ができるかどうか試してみます。

## CRIUをインスールする

### カーネルを再構築する
CRIUを動かすためには、CRIUが必要とするカーネルパラメータが有効になったカーネルを使わないといけません。今回はVagrat UbuntuをLXCのホストマシンとして使うのですが、既存のVagrant Boxで必要なカーネルパラメータが有効になっているものはなさそうでした。

なので、まずはカーネルの再構築から始めないといけません。カーネルの再構築と言うと敷居が高そうですが、実際はとても簡単です。

今回は[公式Ubuntu14.04 cloud image](https://cloud-images.ubuntu.com/vagrant/trusty/current/trusty-server-cloudimg-amd64-vagrant-disk1.box)を使います。

まずは、このボックスを追加します。

```sh
$ vagrant box add ubuntu14.04 https://cloud-images.ubuntu.com/vagrant/trusty/current/trusty-server-cloudimg-amd64-vagrant-disk1.box
$ vagrant init ubuntu14.04
```

 ```vagrant up``` コマンドを打つ前に、CPUとメモリーを増やしましょう。遅いマシンだと再構築にはとても時間がかかります。2コア、2048Mのメモリがあれば大丈夫だと思います。

 ```Vagrantfile``` をエディタで開いて、以下を追加してください。

```
config.vm.provider :virtualbox do |vb|
  vb.customize ["modifyvm", :id, "--memory", "2048"]
  vb.customize ["modifyvm", :id, "--cpus", 2]
end
```

終わったら、```vagrant up && vagrant ssh```してログインしてください。rootユーザになって再構築に必要なパッケージをインストールします。

```sh
$ apt-get -y update
$ apt-get -y install libncurses-dev build-essential libncurses-dev build-essential fakeroot kernel-package linux-source bc
```

インストールしたら、```/usr/src/linux-source-<kernel version>``` というディレクトリが作成されているはずです。そのディレクトリに移動します。

```sh
$ cd /usr/src/linux-source-<kernel version>
$ tar xvjf linux-source-<kernel version>.tar.bz2
$ cd ./linux-source-<kernel version>
```

次はカーネルのコンフィグファイルが必要になります。

gistにアップロードしたのを使ってもらえれば、自分でコンフィグをする必要はありません。以下のコマンドを入力してください。(カレントディレクトリをKernelのソースを展開したディレクトリに変更しておくのを忘れずに)

```sh
$ curl https://gist.githubusercontent.com/kimh/c93f42981d14a33c63c0/raw/a73af0f7f745c2538253ef153a62a8ba1a2d97be/.config -o .config
```

もし、CRIUを動かすためにどのオプションが必要か知りたい場合は[ここ](http://criu.org/Installation#Kernel_configuration)にリストがあります。

 ```.config```ファイルを準備したら、再構築の準備完了です。

**もう一度確認してください。CPUとメモリは増やしましたか？もししていないと、とても長い時間待たされることになります。**

```sh
$ export LC_CTYPE=C
$ make-kpkg clean
$ CONCURRENCY_LEVEL=4 make-kpkg --rootcmd fakeroot --initrd --revision=`date +%Y%m%d` kernel_image kernel_headers
```

ビルドが完了したら、```linux-headers-<kernel version>_amd64.deb``` と ```linux-image-<kernel version>_amd64.deb``` というファイルが ```/usr/src/``` ディレクトリに作成された思います。

さっそくインストールしましょう。

```sh
$ dpkg -i linux-headers-<kernel version>_amd64.deb
$ dpkg -i linux-image-<kernel version>_amd64.deb
$ reboot
```

再起動したらCRIUが使えるカーネルで動いているはずです。

### CRIUをソースからコンパイルする
次はCRIUをインストールします。Ubuntuは最新のパッケージを用意していないので、自分でソースからコンパイルしないといけません。

```sh
$ apt-get install bsdmainutils build-essential libprotobuf-c0-dev linux-headers-generic protobuf-c-compiler
$ mkdir /src
$ cd /src
$ curl http://download.openvz.org/criu/criu-1.3-rc2.tar.bz2 | tar -jxf-
$ make -C criu-1.3-rc2/
$ cp criu-1.3-rc2/criu /usr/local/sbin/
```

これでCRIUのインストールは完了です。ちゃんとインストールされているか確認しましょう。CRIUにはそのためのコマンドがあります。

```sh
$ criu check --ms
Warn  (tun.c:55): Skipping tun support check
Warn  (cr-check.c:259): Skipping mnt_id support check
Looks good.
```

 ```Looks good.```が表示されましたか？いくつか警告がでますが、無視して構いません。

コンテナに対して使う前に、まずは普通のプロセスを停止/再開してみましょう。以下の例は[CRIUのHOWTOページ](http://criu.org/Simple_loop)からです。

まず、ただループするだけのスクリプトを用意します。

```
$ cat > test.sh <<-EOF
#!/bin/sh
while true; do
 date
 sleep 1
done
EOF

$ chmod +x test.sh
$ ./test.sh
```

プロセスを停止するには```criu dump```コマンドを使います。

```sh
# criuを実行するためにはrootじゃないといけない
$ sudo -s
$ export PID=`pgrep -f test.sh`
$ mkdir /tmp/test
$ criu dump -t $PID --images-dir /tmp/test --shell-job
```

もし、ダンプが成功したら```/tmp/test```ディレクトリ配下に沢山のファイルができているはずです。

```sh
$ ls /tmp/test
cgroup.img         fanotify-mark.img   fs-4898.img     netlinksk.img     pstree.img         signalfd.img
core-4521.img      fanotify.img        ids-4521.img    ns-files.img      reg-files.img      sk-queues.img
core-4898.img      fdinfo-2.img        ids-4898.img    packetsk.img      remap-fpath.img    stats-dump
creds-4521.img     fdinfo-3.img        inetsk.img      pagemap-4521.img  sigacts-4521.img   tty-info.img
creds-4898.img     fifo-data.img       inotify-wd.img  pagemap-4898.img  sigacts-4898.img   tty.img
eventfd.img        fifo.img            inotify.img     pages-1.img       signal-p-4521.img  tunfile.img
eventpoll-tfd.img  filelocks-4521.img  inventory.img   pages-2.img       signal-p-4898.img  unixsk.img
eventpoll.img      filelocks-4898.img  mm-4521.img     pipes-data.img    signal-s-4521.img
ext-files.img      fs-4521.img         mm-4898.img     pipes.img         signal-s-4898.img
```

今度はプロセスを再開してみましょう。```criu restore```コマンドを使います。

```sh
$ criu restore -t $PID --images-dir /tmp/test  --shell-job
```

プロセスが問題なく再開されたら```test.sh```が```date```コマンドの出力をターミナルに出すはずです。

## CRIUでコンテナを停止/再開してみる
ここまでは大丈夫ですか？では、いよいよDockerコンテナに使ってみましょう。DockerはこのVagrantにはインストールされていないので、まずインストールしましょう。

```sh
$ apt-get install docker.io jq
$ ln -sf /usr/bin/docker.io /usr/local/bin/docker
$ sed -i '$acomplete -F _docker docker' /etc/bash_completion.d/docker.io
```

インストールしたら、簡単なコマンドをコンテナに実行させます。

```sh
$ docker run -t -i ubuntu /bin/bash
```

コンテナを停止するにはプロセスIDを知る必要があります。

```sh
$ ID=`docker ps -l -q`
$ PID=`docker inspect $ID | jq '.[0].State.Pid'`
```

いよいよ、苦労が報われる時です。コンテナを停止してみましょう！！

```sh
$ criu dump -t $PID --images-dir /tmp/docker
Error (mount.c:449): 102:./dev/console doesn't have a proper root mount
Error (cr-dump.c:1882): Dumping FAILED.
```

あれ？CRIUはエラーを吐いてしまいました。ダンプに失敗したようです。エラーメッセージをGoogleで調べてみると、以下のスレッドを見つけました。

CRIU said dumping failed. After googling the error message, I found this discussion.

[[lxc-devel] [CRIU] LXC live migrate](https://lists.linuxcontainers.org/pipermail/lxc-devel/2013-November/006326.html)
> That's container's console which is a bind mounted tty from
> the host. And since this is an external connection, CRIU doesn't dump one.

ガーン。どうやら、現状のCRIUではLXCのの停止/再開はできないみたいです。でも、[ここのページ](http://criu.org/LXC)にはCRIUを使ってLXCを停止/再開する方法が書かれていますよ。Dockerは内部でLXCを使っているので動くはずじゃ、、、

以下は同じスレッドに書かれていました。

[[lxc-devel] [CRIU] LXC live migrate](https://lists.linuxcontainers.org/pipermail/lxc-devel/2013-November/006326.html)
> AFAIK cgroups are used _inside_ containers only with recent guest templates.
> In OpenVZ we use more old ones (and more stable) so haven't meet this yet.
> And yes, cgroups are in plans for the nearest future :)

要するに、CRIUはcgroupsを2014/06の時点ではサポートしていないらしいです。でも、Dockerが使っているLXCのテンプレートはcgroupsを使っているようです。よって、CRIUではDockerコンテナの停止/再開はできないみたいです。

残念な結果です。。。

## 結論
今回の実験で、CRIU v1.3ではDockerコンテナの停止/再開はできないことがわかりました。CRIUがまだcgroupsに対応していないからです。

今回は少し残念な結果になってしまいましたが、ここまで読んでもらえた方にはCRIUの可能性がわかってもらえたかと思います。

CRIUがLXCのエコシステムにもたらす将来の可能性に対して、このプロジェクトの認知度はとても低いです。この記事を読んでくださった方は、今すぐ[Github](https://github.com/xemul/criu)でスターしてウォッチしましょう！このブログでもこれから色々CRIUについて書いて行きたいと思います。
