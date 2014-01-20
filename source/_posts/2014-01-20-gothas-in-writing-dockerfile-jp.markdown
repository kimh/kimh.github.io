---
layout: post
title: "Dockerfileを書く時の注意とかコツとかハックとか"
date: 2014-01-20 22:20
comments: true
categories: 
- jp
- docker
---

## Contents of This Article
#### [なぜDockerfileを使うのか？](#why_do_we_need_to_use_dockerfile)
#### [ADDとDockerfileにおいてのコンテキストを理解する](#add_and_understanding_context_in_dockerfile)
#### [ビルド時のキャッシュについて: キャッシュが有効なときと無効なとき](#build_caching_what_invalids_cache_and_not)
* [ある一行でキャッシュが使われなかったらそれ以降のすべての行でキャッシュは使われない](#cache_invalidation_at_one_instruction_invalids_cache_of_all_subsequent_instructions)
* [何もしないコマンドを追加してもキャッシュは無効になる](#cache_is_invalid_even_when_adding_commands_that_dont_do_anything)
* [コマンドと引数の間に意味のないスペースの入れてもキャッシュは無効となる](#cache_is_invalid_when_you_add_spaces_between_command_and_arguments_inside_instruction)
* [Dockerfileの行に意味のないスペースを入れてもキャッシュは有効](#cache_is_used_when_you_add_spaces_around_commands_inside_instruction)
* [冪等ではない命令でもキャッシュは効いてしまう](#cache_is_used_for_non_idempotent_instructions)
* [ADD以降にある命令はキャッシュされない (ただし、0.7.3以前のバージョンを使っている場合のみ)](#instructions_after_add_never_cached_only_versions_before_0.7.3)

#### [ コンテナをバックグラウンドで動かすハック](#hack_to_run_container_in_the_background)


<a id="why_do_we_need_to_use_dockerfile"></a>
## なぜDockerfileを使うのか？
DockerfileはYet Anotherシェルではありません。Dockerfileは特別なミッションを持っています。それは、*Dockerイメージ作成の自動化*です。

一度Dockerfileにイメージ作成の手順を記述すれば、それ以降は`docker build`コマンド一つで同じイメージを作ることができます。

Dockerfileはコンテナが何をしているかを別の人の伝える手段でもあります。Dockerfileにはイメージを作って動かすまでのすべてが書かれているので、Dockerfileを読むだけでそのコンテナのやるべき仕事がすぐにわかります。こうすれば、コンテナが何をしているかを調べるためにわざわざログインしてpsコマンドを駆使しなくてもいいというわけです。

簡単に述べましたが、これらの理由で、Dockerイメージを作るなら**必ず**Dockerfileを使ってください。しかし、Dockerfileを書くのは時々嫌になってしまうことがあるのも事実です。このポストではDockerfileを書く時に注意することわかりにくいことを解説します。この記事を読んでDockerfileに慣れてもらえればと思います。

<a id="add_and_context_in_dockerfile"></a>
## ADDとDockerfileにおいてのコンテキストを理解する
***ADD*** is the instruction to add local files to Docker image.  The basic usage is very simple. Suppose you want to add a local file called *myfile.txt* to /myfile.txt of image.
***ADD***はローカルファイルシステムのファイルやディレクトリをDockerイメージにコピーするために使います。使い方は至って簡単。もし、ローカルにある*myfile.txt*をイメージの*/myfile.txt*にコピーしたい場合を説明します。

```bash
$ ls
Dockerfile  mydir  myfile.txt
```

上記のようなディレクトリ構成の場合、Dockerfileは次のようになります。

```
ADD myfile.txt /
```

簡単ですね。しかし、*/home/vagrant/myfile.txt*を追加しようとすると失敗します。

```bash
# 以下の行がDockerfileにあるとします
# ADD /home/vagrant/myfile.txt /

$ docker build -t blog .
Uploading context 10240 bytes
Step 1 : FROM ubuntu
 ---> 8dbd9e392a96
Step 2 : ADD /home/vagrant/myfile.txt /
Error build: /home/vagrant/myfile.txt: no such file or directory
```

`no such file or directory` と言われてしまいました。確かにファイルは存在するのになぜでしょうか？理由は */home/vagrant/myfile.txt* がDockerfileのコンテキスト外だからです。DockerfileでのコンテキストとはDockerfile内の命令からアクセス可能なファイルやディレクトリの範囲のことです。そして、コンテキスト内のファイルとディレクトリしかイメージに追加することはできません。
カレントディレクトリ配下のファイルとディレクトリは自動的にコンテキストに追加されます。これは `build` コマンドを実行した時に確認できます。

```
$ docker build -t blog .
Uploading context 10240 bytes
```

ここで `build` コマンドが何をしているかというと、Dockerクライアントがカレントディレクトリ配下のファイルとディレクトリをtarballにまとめてDockerデーモンへ送信しています。なぜわざわざ送信する必要があるかというと、DockerクライアントとDockerデーモンは違うホストで動いている可能性があるからです。これが上記のコマンドを実行した時に *Uploading* と表示されている理由です。

ひとつ落とし穴があります。Dockerは自動的にカレントディレクトリ配下のファイルとディレクトリをコンテキストに追加するので、もし巨大なファイルやディレクトリを間違っておいておくと使う必要もないのにそれらのtarballを作ろうとします。

```bash
$ ls
Dockerfile  very_huge_file

$ docker build -t blog .
Uploading context xxxxxx bytes
..... # すごく時間がかかる、、、
```

ベストプラクティスとしては、イメージに追加したいファイルとディレクトリのみをbuildを実行するディレクトリに置くべきです。

<a id="build_caching_what_invalids_cache_and_not"></a>
## ビルド時のキャッシュについて: キャッシュが有効なときと無効なとき
DockerはDockerfileの各一行毎にコミットを作成していきます。行の記述を変更しない限り、Dockerは新しいイメージを作る必要がないと判断してキャッシュを使って次の行の元になるイメージを作成します。
これが初めて `docker build` を実行した時には時間がかかるのに、２回目からは一瞬でビルドが完了する理由です。

```bash
$ time docker build -t blog .
Uploading context 10.24 kB
Step 1 : FROM ubuntu
 ---> 8dbd9e392a96
Step 2 : RUN apt-get update
 ---> Running in 15705b182387
Ign http://archive.ubuntu.com precise InRelease
Hit http://archive.ubuntu.com precise Release.gpg
Hit http://archive.ubuntu.com precise Release
Hit http://archive.ubuntu.com precise/main amd64 Packages
Get:1 http://archive.ubuntu.com precise/main i386 Packages [1641 kB]
Get:2 http://archive.ubuntu.com precise/main TranslationIndex [3706 B]
Get:3 http://archive.ubuntu.com precise/main Translation-en [893 kB]
Fetched 2537 kB in 7s (351 kB/s)

 ---> a8e9f7328cc4
Successfully built a8e9f7328cc4

real	0m8.589s
user	0m0.008s
sys	0m0.012s

$ time docker build -t blog .
Uploading context 10.24 kB
Step 1 : FROM ubuntu
 ---> 8dbd9e392a96
Step 2 : RUN apt-get update
 ---> Using cache
 ---> a8e9f7328cc4
Successfully built a8e9f7328cc4

real	0m0.067s
user	0m0.012s
sys	0m0.000s
```
しかし、いつキャッシュが使われていつキャッシュが使われないのかはあまり明確ではありません。ここでは、いくつかのケースを紹介します。

<a id="cache_invalidation_at_one_instruction_invalids_cache_of_all_subsequent_instructions"></a>
#### ある一行でキャッシュが使われなかったらそれ以降のすべての行でキャッシュは使われない
これは一番基本のルールです。もし、Dockerfile内のある一行でキャッシュが使われない書き方をしていまうと、それ以降の行でキャッシュは全く使われなくなってしまいます。

```bash
# Before
From ubuntu
Run apt-get install ruby
Run echo done!

# After
From ubuntu
Run apt-get update
Run apt-get install ruby
Run echo done!
```

*Run apt-get update* という行を追加したことでベースのイメージを変更してしまったので、 それ以降の **すべての** 行で使うイメージも初めから作り直されないといけません。
Dockerfileはひとつ前の行で作られたイメージをベースにして行に書かれている命令を実行するので、これは当然の挙動だと言えます。

<a id="cache_invalidation_at_one_instruction_invalids_cache_of_all_subsequent_instructions"></a>

#### 何もしないコマンドを追加してもキャッシュは無効になる
*以下の例ではキャッシュは効きません。*

```bash
# Before
Run apt-get update

# After
Run apt-get update && true
```

`true` コマンドは実際には何もしないコマンドですが、Dockerはキャッシュを使ってはくれません。

<a id="cache_is_invalid_when_you_add_spaces_between_command_and_arguments_inside_instruction"></a>

#### コマンドと引数の間に意味のないスペースの入れてもキャッシュは無効となる
*以下の例ではキャッシュは効きません。*

```bash
# Before
Run apt-get update

# After
Run apt-get               update
```

<a id="cache_is_used_when_you_add_spaces_around_commands_inside_instruction"></a>

#### Dockerfileの行に意味のないスペースを入れてもキャッシュは有効
*以下の例ではキャッシュは有効になります。*

```bash
# Before
Run apt-get update

# After
Run                apt-get update
```

<a id="cache_is_used_for_non_idempotent_instructions"></a>
#### 冪等ではない命令でもキャッシュは効いてしまう
これはどちらかというとキャッシュの落とし穴についてです。 *冪等ではない命令* とは実行する度に結果が変わる可能性のあるコマンドを実行する行のことです。
例えば、 `apt-get update` は実行する度にアップデートされる内容が変わる可能性があるので冪等ではありません。

```bash
From ubuntu
Run apt-get update
```

上記のDockerfileを作ってイメージを作成したとします。３ヶ月後、Ubutnuがセキュリティアップデートをあるリポジトリにリリースしたので、同じDockerfileを使ってイメージの再作成をしたとします。(apt-get update がセキュリティアップデートを拾ってくれると思って)
しかし、この方法でイメージを再作成してもセキュリティアップデートはインストールされません。Dockerfileの記述自体は全く変更されていないので、たとえ `apt-get update` の実行結果が変わっていたとしてもDockerはキャッシュを使うからです。

もし、これ避けたければ、`-no-cache` オプションを使えます。

```bash
$ docker build -no-cache .
```

<a id="instructions_after_add_never_cached_only_versions_before_0.7.3"></a>
#### ADD以降にある命令はキャッシュされない (ただし、0.7.3以前のバージョンを使っている場合のみ)
もし、0.7.3以前のバージョンを使っている場合、注意してください！

```bash
From ubuntu
Add myfile /
Run apt-get update
Run apt-get install openssh-server
```

もしこのようなDockerfileだと、***Run apt-get update*** と ***Run apt-get install openssh-server*** は絶対にキャッシュされません。

この挙動は0.7.3で改善されました。ADD以降の行でも、ADDの書き方自身やADDする対象が変更されない限りキャッシュが使われます。

```bash
$ echo "Jeff Beck" > rock.you

From ubuntu
Add rock.you /
Run add rock.you

$ echo "Eric Clapton" > rock.you

From ubuntu
Add rock.you /
Run add rock.you
```

ここでは、*rock.you* ファイルの内容を変更したので、ADD以降の行ではキャッシュは使われません。

<a id="hack_to_run_container_in_the_background"></a>

## コンテナをバックグラウンドで動かすハック
もし、コンテナの起動方法をシンプルにしたければ、`docker run -d image your-command` を使ってコンテナをバックグラウンドで起動するべきです。
`docker run -i -t image your-command` の代わりに `-d` を使うことを勧める理由はコンテナの起動をたった一つのコマンドで行うことができ、かつ `Ctrl + P + Q` を入力してコンテナをターミナルから切り離す作業をしなくていいからです。

しかし、`-d` オプションには問題があります。コマンドがフォアグラウンドで実行されていないとコンテナはすぐに終了してしまいます。

apacheをサービスとして起動するコンテナを例に説明しましょう。直感的に次のようにやりたくなるでしょう。

```bash
$ docker run -d apache-server apachectl start
```

しかし、これだとコンテナは起動した瞬間に終了します。これは、 `apachectl` がapacheをデーモン化した瞬間に自身は終了してしまうからです。

Dockerはこのようなコマンドが嫌いです。Dockerはコマンドがフォアグラウンドで起動し続けることを期待しているからです。
もしそうでなければ、Dockerはアプリケーションは終了したと考えてコンテナを終了してしまいます。

この問題はapacheの実行バイナリを直接フォアグラウンドで動かすことで解決できます。

```bash
$ docker run -e APACHE_RUN_USER=www-data \
                    -e APACHE_RUN_GROUP=www-data \
                    -e APACHE_PID_FILE=/var/run/apache2.pid \
                    -e APACHE_RUN_DIR=/var/run/apache2 \
                    -e APACHE_LOCK_DIR=/var/lock/apache2 \
                    -e APACHE_LOG_DIR=/var/log/apache2 \
                    -d apache-server /usr/sbin/apache2 -D NO_DETACH -D FOREGROUND
```

ここでしていることは、`apachectl` がやっていることを手動で行ってapacheを起動しています。このやり方だとapacheはフォアグラウンドで動き続けることができます。

問題はアプリケーションによってはフォアグラウンドで起動する方法がない場合があることです。また、`apachectl` の例のようにヘルパープログラムがやってくれることを分解して手動でやらないといけないのは大変です。どうすればいいでしょう？

このような場合、`tail -f /dev/null` を実行したいコマンドに追加すればコンテナはメインのコマンドがバックグラウンドで実行されても `tail` がフォアグラウンドで起動し続けてくれるので終了しません。このテクニックをさっきのapacheの例で使ってみましょう。

```bash
$ docker run -d apache-server apachectl start && tail -f /dev/null
```

ずっと良くなりました。 `tail -f /dev/null` は無害なコマンドなのでこのテクニックはどんな場合にも使うことができるのでおすすめです。
