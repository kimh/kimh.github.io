---
layout: post
title: Btrfs で Unprivileged で Nested な LXCコンテナ
date: 2016-01-07 01:05
comments: true
categories:
---

仕事でUnprivilegedな親LXCコンテナ上で子コンテナを動かす必要があった。つまり、コンテナの入れ子。さらにLXCのストレージバックエンドにBtrfsを使わないといけない。要件の特殊さから包括的な方法が見つからなかったので自分で書くことにした。

***Unprivileged LXCコンテナとは何か***

まずそもそもUnprivileged LXCコンテナがいまひとつ何かわかっていなかったのでまずはその理解から。

簡単に言うとroot権限を持っていないユーザが作成・起動したLXCコンテナのことをこう呼ぶらしい。LXCを使う場合、 `lxc-create` でコンテナを作成して `lxc-start` で起動するけど普通はこれはrootユーザじゃないといけない。
しかし、rootユーザでしてしまうとコンテナ上のrootユーザがホストのuid 0を持つことになってしまい、いくらchroot環境でコンテナが動いていたとしてもセキュリティ的によくない。これを避けるために一般ユーザでコンテナを動かして
コンテナ上のrootユーザをホストのrootじゃないユーザのuidにマッピングする。こうすればコンテナ上では変わらずrootになれるけどホストのrootとは違うので安全が確保できるというもの。

Unprivileged LXCを調べていると以下のようなLXCのコンフィグを必ず見ると思う。

```
lxc.id_map = u 0 100000 65536
lxc.id_map = g 0 100000 65536
```

このコンフィグでは uid (1行目) と gid (２行目) のマッピングを定義していてそれぞれの意味は、

- 0: コンテナが使えるuid/gidは0から始まり
- 10000: ホスト上の uid/gid 100000から
- 65536: ホスト上の65536個のuid/gidをコンテナに割りあてる

という意味。つまり、ホスト上の100000から165536までのuid/gidをコンテナのマッピングに使うよ、ということ。

このマッピングを定義するだけではダメで、この設定をLXCが使えるようにホスト上で設定しないとダメ。

設定は `/etc/subuid` と `/etc/subgid` に書く。

`/etc/subuid`

```
vagrant:100000:65536

```

`/etc/subgid`

```
vagrant:100000:65536
```

Unprivileged LXCを使うために必須なのは実はこれだけ。

LXCの設定か何かで `--unprivileged` みたいに指定するのかと思うのかもしれないけどそうではない。Unprivilegedなコンテナとはあくまで一般ユーザで作成・起動したコンテナというだけ。

***Btrfストレージバックエンド***

LXCは複数のストレージバックエンドに対応している。`zfs` `btrfs` `dir` などなど。どのバックエンドを使うかは要件によると思うけど、基本は高機能なファイルシステムなbtrfsとかzfsを使う方が
パフォーマンス的にいいし使える機能とかも違ってくる。今回は仕事の関係でbtrfsを使う

Btrfsをストレージバックエンドに使うとは具体的にはどういうことだろう？

これは簡単にいうと、BtrfsでフォーマットしてマウントされたディレクトリをLXCコンテナの保存先に指定するということ。具体的なやり方は後述するけど、こんな感じ。

- ホストマシンに `/dev/sdb` というディスクデバイスを追加する
- `/dev/sdb` を `/mnt` にbtrfsでマウントする
- LXCコンテナのrootfsに `/mnt` 以下のディレクトリを指定する。(Ex. `/mnt/box1`)

これもUnprivilegedと同じでbtrfsを使うという設定が特にあるわけではない。コンテナの保存先をbtrfsのディレクトリに指定するだけ。詳しくは後述する。

***Nestedコンテナ***

ここまでですでにややこしいけど、さらにもう少しだけややこしくする。LXCの特徴の一つにコンテナの入れ子のサポートがある。つまり、ホストの上でコンテナ1を動かして、コンテナ1の上でコンテナ2を動かして、コンテナ2の上で、、、ということができる。

今回の要件は一つ目のコンテナはUnprivilegedで二つ目のコンテナはPrivilegedコンテナにする。PrivilegedといってもUnprivilegedの上で動いているのでセキュリティの問題はない。

```
入れ子コンテナイメージ図

   ----------
   |コンテナ2|
   ----------
   Privileged
  -------------
  |  コンテナ1 |
  -------------
  Unprivileged
-----------------
|     ホスト1    |
-----------------
Vagrant上のLinux VM
```

***構築手順***

ここからは具体的な構築の手順を説明していく。

***Vagrantでホストマシンを作成***

```
$ mkdir lxc
$ vagrant init trusty # trustyは自分でつけた名前かもしれないけど、とにかくどのBoxでもいいのでtrusty
$ vagrant up; vagrant ssh
```

LXCではカーネルのバージョンがすごく大事。最近でこそ安定してきたけどマイナーバージョンが違うと壊れてて動かないというのはよくある話。vagrantで作ったtrustyのカーネルは以下だった。

```
$ uname -a
Linux vagrant-ubuntu-trusty 3.13.0-24-generic #46-Ubuntu SMP Thu Apr 10 19:11:08 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux
```

まずはLXCとbtrfsをインストール。

LXCは1.1.5を使いたいので以下のコマンドでインストール。

```
$ sudo apt-get install -y -t trusty-backports lxc
$ vagrant@vagrant-ubuntu-trusty:~$ dpkg -l | grep lxc
ii  liblxc1        1.1.5-0ubuntu3~ubuntu14.04.1  amd64 Linux Containers userspace tools (library)
ii  lxc            1.1.5-0ubuntu3~ubuntu14.04.1  amd64 Linux Containers userspace tools
ii  lxc-templates  1.1.5-0ubuntu3~ubuntu14.04.1  amd64 Linux Containers userspace tools (templates)
ii  lxcfs          0.11-0ubuntu3~ubuntu14.04.1   amd64 FUSE based filesystem for LXC
ii  python3-lxc    1.1.5-0ubuntu3~ubuntu14.04.1  amd64 Linux Containers userspace tools (Python 3.x bindings
```

Btrfsもインストールする。

```
$ sudo apt-get install btrfs-tools
$ dpkg -l | grep btrfs
ii  btrfs-tools 3.12-1ubuntu0.1 amd64 Checksumming Copy on Write Filesystem utilities
```

***Brtfs領域を作成***

次にLXCが使うbtrfsファイルシステムを作成する。ホストマシンに`/dev/sdb`という新しいディスクを追加してこれを`/mnt`にマウントする。

ディスクを追加する方法は色々ググった結果以下をVagrantファイルに書くとできた。（ストレートには行かなかった気もするけどこのあたりの説明はめんどくさいので省く。）

```
    config.vm.provider :virtualbox do |vb|
      file_to_disk = "./tmp/disk1.vdi"
      if not File.exist?(file_to_disk) then
        vb.customize ["createhd",
                      "--filename", file_to_disk,
                      "--size", 300 * 1024]
      end
      vb.customize ['storageattach', :id,
                    '--storagectl', 'SATA Controller',
                    '--port', 1,
                    '--device', 0,
                    '--type', 'hdd',
                    '--medium', file_to_disk]
    end
```

無事に`/dev/sdb`を追加できたら`fdisk`で確認する。

```
$ sudo fdisk -l | grep sdb
Disk /dev/sdb doesn't contain a valid partition table
Disk /dev/sdb: 322.1 GB, 322122547200 bytes
```

Btrfsでフォーマットする。

```
$ sudo mkfs -t btrfs /dev/sdb

WARNING! - Btrfs v3.12 IS EXPERIMENTAL
WARNING! - see http://btrfs.wiki.kernel.org before using

Turning ON incompat feature 'extref': increased hardlink limit per file to 65536
fs created label (null) on /dev/sdb
        nodesize 16384 leafsize 16384 sectorsize 4096 size 300.00GiB
Btrfs v3.12
```

`/mnt`にマウントする。

```
$ echo '/dev/sdb /mnt               btrfs   defaults 0       1' | sudo tee -a /etc/fstab
$ sudo mount /mnt
```

最後にvagrantユーザが書き込めるように権限を変更。

```
$ sudo chown vagrant:vagrant /mnt
```

これで `/mnt` をLXCのバックエンドとして使う準備ができた。

***Unprivileged LXCの準備***

次にUnprivileged LXCを使う準備をしていく。上記で説明したようにuid/gidのマッピングの設定をする。Vagrantで作ったBoxではvagrantという一般ユーザがデフォルトなのでこのユーザでUnprivilegedコンテナを作成する。

上記でも書いたけど、Unprivilegedコンテナとはあくまでrootじゃないユーザで作るコンテナのこと。

```
$ echo 'vagrant:100000:65536' | sudo tee -a /etc/subuid
$ echo 'vagrant:100000:65536' | sudo tee -a /etc/subgid
```

このコマンドの意味は最初の説明をみてください。

次にvagrantユーザが使うLXCのコンフィグを設定する。主なファイルは `~/.config/lxc/default.conf` と `~/.config/lxc/lxc.conf`。

`~/.config/lxc/default.conf`

```
lxc.id_map = u 0 100000 65536
lxc.id_map = g 0 100000 65536
lxc.network.type = veth
lxc.network.link = lxcbr0
lxc.network.flags = up
lxc.network.hwaddr = 00:16:3e:xx:xx:xx
```

`~/.config/lxc/lxc.conf`

```
lxc.lxcpath = /mnt
```

`lxc.lxcpath` はLXCがコンテナとコンテナのコンフィグを保存する場所を指定する。今回はbtrfsでマウントしたディレクトリを使いたいので `/mnt` と指定する。

コンフィグがちゃんと設定されているかどうかは `lxc-config` コマンドを使ってできる。

確認可能なコンフィグの一覧は `-l` オプションで見れる。

```
$ lxc-config -l
lxc.default_config
lxc.lxcpath
lxc.bdev.lvm.vg
lxc.bdev.lvm.thin_pool
lxc.bdev.zfs.root
lxc.cgroup.use
lxc.cgroup.pattern

```

LXCにはコンフィグがたくさんあるのになぜこれだけしか確認できないのかは不明。コンフィグの値を確認するには設定の名前を渡す。

```
$ lxc-config lxc.lxcpath
/mnt
```

最後にもう一つ。Unprivilegedコンテナ上のrootは本当のrootではないので本来ならデバイスにアクセスできない。これで一番問題になるのはネットワークの設定。ネットワークインターフェースはコンテナから色々カスタマイズしたい
ことが普通だから。この問題の解決するために `lxc-user-nic` というツールがLXCについている。Unprivilegedコンテナ上でネットワークの操作をするために `/etc/lxc/lxc-usernet` に以下の設定を入れる。


```
vagrant veth lxcbr0 10
```

このコンフィグはvagrantユーザがlxcbr0というブリッジデバイスに `veth` ネットワークインターフェイスを１０個まで作成できるという意味。

***コンテナの作成・起動***

ここまででUnprivilegedなコンテナを作成する準備は整った。以下のコマンドでコンテナをダウンロードする。

```
$ lxc-create -t download -n box1 -B btrfs -- -d ubuntu -r trusty -a amd64
```

これで起動もできる状態になったけど今回はコンテナを入れ子にしたいので作成したコンテナのコンフィグを変更する。

コンンフィグは `/mnt/box1/config` にある。以下の設定を追加。

```
lxc.mount.auto = cgroup
lxc.aa_profile = lxc-container-default-with-nesting
```

`lxc.mount.auto = cgroup`は調べてないのでよくわからない。おそらく、`cgroupfs` を自動マウントするということだけど、 `cgroupfs` が説明できるほど理解していないので省略。

`lxc.aa_profile = lxc-container-default-with-nesting` はコンテナの入れ子をAppArmorで許可する設定。これがないと入れ子はできない。

これですべての準備は整った。以下のコマンドでコンテナを起動する。

```
lxc-start -n box1 -d --logfile test.log --logpriority DEBUG
```

`--logfile test.log --logpriority DEBUG` はなんらかの問題でコンテナが起動できない時にデバッグで便利なので指定しておくといい。

自分の環境では `lxc-start: utils.c: setproctitle: 1455 Invalid argument - setting cmdline failed` というエラーが出たけど無害みたいなので無視。

無事に起動できたら `lxc-attacth` でログインする。

```
$ lxc-attach -n box1
```

これでUnprivilegedコンテナを起動することができた。

***コンテナの入れ子***

最後にもう一つの目標であるコンテナの入れ子をやって見る。つまり、`box1` にログインして `n1` という名前でさらにコンテナを起動する。

これをするために、少し変な感じがするけどbox1上でLXCをインストールしないといけない。

```
$ apt-get update
$ apt-get install lxc
```

`n1` はPrivilegedなコンテナ(といってもUnprivileged上のPrivilegedなので実際にはUnprivilegedと同じ)でいいのでLXCをインストールすればすぐに作成できる。

```
$ lxc-create -n n1 -t ubuntu-cloud
```

でコンテナをダウンロードして

```
$ lxc-start -n n1 -d
```

で起動するだけ。問題なく起動すれば `n1` にまたログインして普通のコンテナとして使うことができるはず。


