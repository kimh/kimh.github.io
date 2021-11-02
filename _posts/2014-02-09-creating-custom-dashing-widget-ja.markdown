---
layout: post
title: "Dashingのカスタムウィジェットを作成する"
date: 2014-02-09 21:05
comments: true
permalink: /blog/jp/dashing/creating-custom-dashing-widget-ja/
categories:
- jp
- dashing
---

![Dashing Preview](/images/dashing_preview.png)

## Dashingの紹介
[Dashing](https://github.com/Shopify/dashing)というダッシュボード作成のフレームワークをご存知ですか？ Dashingを使えば、上のようなダッシュボードを簡単に作ることができます。いいですよね？欲しいですよね？


この記事では、自分で一からウィジェットを作成する方法を解説していきます。Dashingの基本的な使い方は、[公式ページ](http://shopify.github.io/dashing/) を見ればすぐわかると思うのでここでは割愛です。

## 目次
#### [サンプルダッシュボードの作成](#creating_sample_dashboard)
#### [ウィジェットに必要なファイル](#files_for_widget)
#### [ジョブファイルを作成](#creating_job_file)
#### [ジョブのコードを書く](#writing_job_code)
#### [ウィジェットのコードを書く](#writing_widget_code)
#### [ウィジェットを使う](#using_widget)
#### [ウィジェットを公開する](#publishing_widget)


<a id="creating_sample_dashboard"></a>
## サンプルダッシュボードの作成
このチュートリアルで使うサンプルのダッシュボードをまず作成しましょう。サンプルダッシュボードは **My Dashboard** という名前にして、**Fizz Buzz** ウィジェットを作成しましょう。
FizzBuzzウィジェットは一定間隔で数字をインクリメントしていって、３で割り切れる数字の場合は**Fizz**と、５で割り切れる場合は**Buzz**と、どちらでも割り切れる場合は、**FizzBuzz**と表示するウィジェットです。

![Widget Preview](/images/fizzbuzz.png)

新しいダッシュボードを作成するのは以下のコマンドを実行するだけです。

```bash
$ dashing new my_dashboard
      create  my_dashboard
      create  my_dashboard/.gitignore
      create  my_dashboard/Gemfile
      create  my_dashboard/README.md
      .... 省略 ....
      create  my_dashboard/widgets/text/text.coffee
      create  my_dashboard/widgets/text/text.html
      create  my_dashboard/widgets/text/text.scss
```

これでカレントディレクトリに`my_dashboard`というディレクトリが作成されます。このディレクトリがダッシュボードのベースとなるので、以降の作業はすべて`my_dashboard`をrootディレクトリとして読んでください。

<a id="files_for_widget"></a>
## ウィジェットに必要なファイル
Dashingはフレームワークなので、ルール従って開発すれば簡単に自分のウィジェットを作成することができます。ルールの一つに必要なファイルを用意することと、Dashingの命名規則に従ってそれらのファイル/ディレクトリ名をつけることがあります。

新たにウィジェットを作成するために必要なファイルは主に２つに分かれます。

ひとつはウィジェットのメインとなるウィジェットファイルで**/widgets** ディレクトリ配下に作られます。

もう一つはウィジェットにデータを渡すためのジョブファイルで、**/jobs**ディレクトリ配下に作られます。

まずは、ウィジェットファイルを作成しましょう。

### ウィジェットファイルを作成
テンプレートとなるファイルをコマンドで作成します。

```bash
$ dashing generate widget fizz_buzz
      create  widgets
      create  widgets/fizz_buzz/fizz_buzz.coffee
      create  widgets/fizz_buzz/fizz_buzz.html
      create  widgets/fizz_buzz/fizz_buzz.scss
```

簡単に各ファイルの役割を説明すると

`fizz_buzz.html` はウィジェットのhtmlそのものです。ジョブから渡されたデータをこのhtml内でレンダリングします。

`fizz_buzz.scss` はウィジェットのスタイルを書く場所です。

`fizz_buzz.coffee` はウィジェットで使うjavascriptを書くための場所です。ファイル名からもわかる通り、単なるJSファイルではなくcoffee script形式で書かないといけません。(私は普通のJSで書かせてほしいのですが、、、)

ウィジェットの表示に効果をつけたり、ウィジェットの初期化動作などを書くことができます。

このファイルは見た目のそこまでこだわらなければ、ほぼ空っぽでも構いません。これはDashingのいいところの一つだと思います。つまり、javascirptを全然知らなくても、それなりのウィジェットを作ることができるからです。

<a id="creating_job_file"></a>
## ジョブファイルを作成
次にジョブファイルを作成します。

```bash
$ dashing generate job fizz_buzz
      create  jobs
      create  jobs/fizz_buzz.rb
```

`fizz_buzz.rb` はウィジェットに渡すためのデータを作るコードを書くファイルです。ジョブファイルの基本はcronのように定期的に実行される[Rufus Scheduler](https://github.com/jmettraux/rufus-scheduler)で、
メインのコードをrufusによっての定期実行されるブロックの中に書いていきます。

```ruby
SCHEDULER.every '2s' do
  # ここにメインのコードを書く
  # 2sなのでウィジェットのデータが２秒毎に更新されることになる
end
```

<a id="writing_job_code"></a>
## ジョブのコードを書く
まずはウィジェットにデータを提供するためのジョブのコーディングから始めましょう。以下のようなコードになります。

**fizz_buzz.rb** @ [Gist](https://gist.github.com/kimh/8899670#file-fizz_buzz-rb)
```ruby
class FizzBuzz
  FIZZ=3
  BUZZ=5

  def initialize
    @current_num = 1
  end

  def fizzbuzz
    if fizzbuzz?
      out = "fizzbuzz"
    elsif fizz?
      out = "fizz"
    elsif buzz?
      out = "buzz"
    else
      out = @current_num
    end
    go_next
    out
  end

  private

  def go_next
    @current_num+=1
  end

  def fizz?
    (@current_num % 3) == 0
  end

  def buzz?
    (@current_num % 5) == 0
  end

  def fizzbuzz?
    (@current_num % (FIZZ*BUZZ)) == 0
  end

end

fb = FizzBuzz.new

SCHEDULER.every '5s', :first_in => 0 do
  send_event('fizz_buzz', { value: fb.fizzbuzz })
end
```

1 ~ 43行は普通のRubyで書いたFizzBuzzクラスとそれのインスタンスを作っているだけです。これらのコードは、`SCHEDULER`の外に書かれているので定期実行はされず、`dashing start`で
ダッシュボードを起動した時の一度だけ実行されるので一度だけ初期化したい処理 (例えば、ステータスを保存する変数とか)を書きます。

46行目の`send_event`がジョブファイルのキモです。

`send_event`は一つ目の引数にウィジェットの名前を指定します。この名前は、後述するレイアウトファイルに書くウィジェットのhtmlの`data-id`属性で指定する名前とマッチしないといけません。

二つめの引数には、ウィジェットに渡すデータをハッシュ形式で指定します。ハッシュのキーは`fizz_buzz.html`内の`data-bind`属性で指定している名前と同じにします。

**fizz_buzz.html**
```
<div data-bind="value"></div>
```

`send_event`で送ったデータは`data-bind="value"`属性をもっている`div`タグの内容として挿入されます。

つまり、`send_event('fizz_buzz', { value: "fizzbuzz" })` とすると、`fizz_buzz.html` は以下のように自動で変更されます。

```
<div data-bind="value">fizzbuzz</div>
```

この自動でhtmlの内容が更新される仕組みは**batman.js**の[data binding](http://batmanjs.org/docs/api/batman.view_bindings.html)を使って実現しています。

<a id="writing_widget_code"></a>
## ウィジェットのコードを書く
ジョブファイルでデータを更新できる準備は整ったので今度はそのデータをウィジェットとして表示するコードを書きます。

まずは、ウィジェットそのもののhtmlを担当するファイルです。今回はチュートリアルなのでシンプルにしています。

**fizz_buzz.html** @ [Gist](https://gist.github.com/kimh/8899670#file-fizz_buzz-html)
```
<h1 class="title" data-bind="title"></h1>
<div data-bind="value"></div>
```

。1行目はウィジェットのタイトルを表示します。このタイトルは後述するレイアウトファイルの`data-title`属性で指定した値が表示されます。

2行目は前述したように、`send_event`から送られてくるデータを表示します。

次にウィジェットのスタイルを担当するファイルです。

**fizz_buzz.scss** @ [Gist](https://gist.github.com/kimh/8899670#file-fizz_buzz-scss)
```
.widget-fizz-buzz {
  background-color: #444;
}
```

ここでは、ウィジェットの背景色を指定しています。セレクタは`.widget`で始めてその後に自分のウィジェットの名前をつなげた名前にします。そして、ブロック内にそのウィジェットに関わるスタイルを書いていきます。

**ここで少し命名規則に注意です。** rubyのジョブファイルではウィジェットの名前はアンダースコア区切りの`fizz_buzz`でしたが、こっちはCSSファイルなので、ウィジェットの名前の単語をつなげるのは`-`にしないといけません。

最後にウィジェットにエフェクトをつけるcoffee scriptです。何もエフェクトを使わなければここはテンプレートのままでも構いませんが、せっかくなので少しだけエフェクトをつけてみます。

**fizz_buzz.coffee** @ [Gist](https://gist.github.com/kimh/8899670#file-fizz_buzz-coffee)

```ruby
class Dashing.FizzBuzz extends Dashing.Widget

  ready: ->
    # ここは初期化時に実行したいエフェクトを書く

  onData: (data) ->
    $(@node).fadeOut().fadeIn()
```

`$(@node)`でウィジェット全体のDOMを取得することができるので、これでデータが更新される度にエフェクトが効きます。

<a id="using_widget"></a>
## ウィジェットを使ってみる
これでFizzBuzzウィジェットは出来上がりました。あとはウィジェットを使うだけです。まずはレイアウトファイルを作成します。ここではレイアウトファイルの名前は`fizz_buzz_display`としましたが、なんでも構いません。

```bash
$ dashing generate dashboard fizz_buzz_display
      exist  dashboards
      create  dashboards/fizz_buzz_display.erb
```

レイアウトファイルができたら以下のように編集します。

**fizz_buzz_display.erb** @ [Gist](https://gist.github.com/kimh/8899670#file-fizz_buzz_display-erb)
```
<div class="gridster">
  <ul>
    <li data-row="1" data-col="1" data-sizex="1" data-sizey="1">
      <div data-id="fizz_buzz" data-view="FizzBuzz" data-title="Fizz Buzz"></div>
    </li>
  </ul>
</div>
```

`data-id`属性が一番重要な部分です。ここで表示するウィジェットを指定しています。`fizz_buzz.rb`内の`send_event`の第一引数でデータを送るウィジェットを指定しましたが、それはここの名前を指定しています。

`data-view`属性はウィジェットのcoffee scriptのクラスと一致しないといけません。coffee scriptはマッチする`data-view`を`@node`にマップします。

`data-title`属性で指定したタイトルが`fizz_buzz.html`内の`data-bind="title"`を持つ要素の内容として表示されます。ここでは"Fizz Buzz"と指定しているので

```
<h1 class="title" data-bind="title">Fizz Buzz</h1>
```
となります。

これでFizzBuzzウィジェットが表示されるはずです。`dashing start`でダッシュボードを起動して、`http://localhost:3030/fizz_buzz_display`にアクセスしてみてください。ちゃんと表示されたでしょうか？

今回は、チュートリアル用に個別で`fizz_buzz_display`を作りましたが、FizzBuzzウィジェットは他のダッシュボードでも使えます。使いたければ、

```
<li data-row="1" data-col="1" data-sizex="1" data-sizey="1">
  <div data-id="fizz_buzz" data-view="FizzBuzz" data-title="Fizz Buzz"></div>
</li>
```

をレイアウトファイルに書くだけです。

<a id="publishing_widget"></a>
## ウィジェットを公開する
最後に作成したウィジェットは[公開して使ってもらいましょう。](https://github.com/Shopify/dashing/wiki/Additional-Widgets)

DashingはウィジェットをGistで管理しています。自分のGistページで新たにGistを作成して、ウィジェットに必要なファイル名と同じ名前でGistの内容に各ファイルのコードを書きます。

今回で言えば、以下の４つのファイルを一つのGistに作成します。

- fizz_buzz.html
- fizz_buzz.css
- fizz_buzz.coffee
- fizz_buzz.rb

Gistを作成したら、インストールできるか確認してください。

```bash
$ dashing install GIST_ID
```

`GIST_ID`は作成したGistのIDです。また必須ではありませんが、自分のウィジェットの使い方をGistにREADMEとして書いておくのが定番のようです。

正しく動けば、[Additional Widgets](https://github.com/Shopify/dashing/wiki/Additional-Widgets)のWikiを修正して登録するのを忘れずに。
