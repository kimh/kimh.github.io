---
layout: post
title: "Ruby LotusでWeb Appを作ってみる"
date: 2014-09-11 18:40
comments: true
permalink: /blog/jp/lotus/creating-web-application-with-ruby-lotus-ja/
categories:
- jp
- lotus
---

![](/images/lotus.jpeg)

***注意***  この記事はIBM製のコラボレーションソフトについてのページではありません。Rubyで書かれた[Lotus](http://lotusrb.org/)というWebフレームワークについての記事です。
## 内容
* [Lotusとは何か](#what_is_lotus)
* [なぜRailsじゃなくLotusか](#why_lotus_instead_of_rails)
* [One-fileアプリを作る](#creating_one_file_application)

<a id="what_is_lotus"></a>
## Lotusとは何か
[Lotus](http://lotusrb.org/)は新しいRubyで書かれたWebフレームワークです。比較的少人数の[lotusチーム](https://github.com/lotus)が開発しています。
2014年9月の時点ではまだ本番環境では使えませんが、簡単なアプリを作るには十分に動作します。

Lotusのプロジェクトページに書かれているスローガンを見たときに一瞬で一目惚れしました。

ページにはこう書いてあります。(かなり意訳です)

>Lotusは軽量で高速かつテストが容易なフレームワークです。Lotusはオブジェクト指向プログラミングのエッセンスを取り戻し、安定したAPI、最小限のDSLを提供してシンプルなオブジェクトに基づいたWeb開発を可能にします。

このスローガンを読んだ時、探していたものはこれだと思いました。（後で詳しく説明します）ちょうど個人でやっているプロジェクトでシンプルなAPIサーバを作ろうとしていたので早速Lotusを使ってみることにしました。
まだ情報がほとんど世の中に存在しないので色々苦労しましたが、最近は少しずつ ***Lotus Way*** がわかってきたのでこの記事で紹介しようと思います。

<a id="why_lotus_instead_of_rails"></a>
## なぜRailsではなくLotusか
最近はアプリケーションをモジュール方式で開発する機会が増えてきました。この開発方法をSOAと呼ぼうがMicroserviceと呼ぼうがなんでもいいですが、多くのプロジェクトがこの開発方式を採用するようになってきています。
[(Fylnnはお気に入りの例です)](https://github.com/flynn)

アプリをモジュール方式で開発する利点はいくつかあります。

 - テストがしやすい
 - 高いポータビリティ
 - 再利用しやすい
 - デプロイが容易

Railsを使ってこれらのことを実現するのは簡単ではありません。Railsは素晴らしいですが、フレームワークスタックは巨大で沢山の機能が最初からビルトインされています。
要するにRailsは小さいコンポーネントを沢山作るには大きすぎます。

[Sinatra](https://github.com/sinatra/sinatra)や[rails-api](https://github.com/rails-api/rails-api)が使えるのでは？と思う人もいると思います。
確かにSinatraは軽量です。ただ最近個人的にはDSLよりも ***純粋なRubyのコード*** を好むようになってきました。学習コストを低く抑えれるからです。
正直rails-apiは使ったことがないのでよくわかりません。ただ、Railsがベースなのでそこまで軽量ではないのではないかと思っています。もし、知ってる人がいたら教えてください。

[LotusのGithubページ](https://github.com/lotus)を見ればわかりますが、Lotusは複数のコンポーネントからできているので、この中の一つだけを自分のプロジェクトで使うことも可能です。
例えば、[lotus-router](https://github.com/lotus/router)だけを自分のRackアプリに組み込んでHTTPルーターとして使うことができます。

またLotusはRailsのようにフルスタックなフレームワークとして使うこともできます。LotusはRailsのいいとこ取りをしているので(例えばCoCの積極的に採用)比較的少ないコードでアプリケーションを作ることもできます。

つまり、LotusはSinatraのように軽量なアプリを作ることができる一方で、Railsのようにフルスタックなアプリも作ることができるということです。

<a id="creating_one_file_application"></a>
## One-fileアプリケーションを作る
Lotusを使ってOne-fileアプリケーションを作ってみます。LotusのGithubに掲載されている[one file application](https://github.com/lotus/lotus#one-file-application)を参考にします。

まずは、プロジェクト用のディレクトリを作成します。今後の説明はこのディレクトリをカレントディレクトリに設定しているいう前提で進めます。

```
mkdir onefileapp && cd onefileapp
```

Lotusの公式なgemはまだ配布されていません。[ここ](https://rubygems.org/gems/lotusrb)にありますが更新日は2014年1月となっています。最新のmasterブランチは
これよりずっと進んでいるので、ソースからインストールします。そのために、まずLotusレポジトリをクローンします。

```
git clone https://github.com/lotus/lotus.git
```

Gemの管理にbundlerを使います。Gemfileを作成しましょう。

```
bundle init
```

次にGemfileを編集します。`<your-path-to-lotus-repo>`をクローンしてきたLotusのレポジトリのディレクトリに適宜変更してください。

```ruby
source "https://rubygems.org"
gem 'lotusrb', :path => <your-path-to-lotus-repo>
```

Gemをインストールします。

```
bundle install --path vendor/
```

これでアプリケーションを書く準備が整いました(といっても１ファイルですが)。以下のコードを ***config.ru*** として保存してください。

***config.ru***
```ruby
require 'lotus'

module OneFile
  class Application < Lotus::Application
    configure do
      routes do
        get '/', to: 'home#index'
      end
    end
    load!
  end

  module Controllers
    module Home
      include OneFile::Controller

      action 'Index' do
        def call(params)
        end
      end
    end
  end

  module Views
    module Home
      class Index
        include OneFile::View

        def render
          "Hello, Lotus"
        end
      end
    end
  end
end

run OneFile::Application.new
```

保存したらアプリを起動してみましょう。最新のmasterブランチには `lotus server` コマンドがあるのでこれを使います。

```
bundle exec lotus server
```

起動できましたか？それではブラウザで http://localhost:2300 にアクセスしてみてください。 `Hello, Lotus` と表示されるはずです。

コードの説明をします。

まず、最初に気づくことは`Controllers`と`Views`モジュール内で定義されているクラスは継承を使っていないということです。
Lotusの哲学の一つに`モジュールをインクルードして最小限のインターフェースを実装する`というものがあります。
この哲学は開発者に本当に必要なものだけをMixinして使うことを推奨します。

次に`Application`クラスを見てみましょう。今回の例ではこのクラスはルートの設定をしているだけです。
`get '/', to: 'home#index'`とすることで`GET /`のルートは`Home::Index`コントローラを使うように設定しています。
今回はひとつだけしかルートを設定していませんが他のHTTPメソッドも同じように設定できます。

```ruby
routes do
  post   '/books',             to: 'book#create'
  put    '/books/:id',         to: 'book#update'
  delete '/books/:id',         to: 'book#destroy'

  # ワンライナーレスポンス
  get    '/ping',              to: ->(env) {[200, {}, ['pong']]}
end
```

次に`Controllers`を見てみましょう。`action`を呼んでいる以外は何もしていません。

`action`とは何でしょうか？`action`はHTTPリクエストのエンドポイントとして動き、ここでリクエストの中身を見たりレスポンスを生成したりします。
リクエストに対するビジネスロジックもここに書きます。多分、`action`はRailsのControllerとほとんど同じものだと考えていいと思います。
ここでは先ほどみたルートから呼び出される`Index` actionを定義しています。

最後に`Views`を見ましょう。このクラスの仕事はRailsのViewとは異なります。
RailsではViewはブラウザでレンダリングされるコンテンツを吐き出すコードを書く場所でした。Lotusではこれは`Template`で行います。(今回の例ではTemplateはでてきません)

LotusではViewはPresenterレイヤーとして動きます。RailsではPresenterは標準ではありません。(Draperなどのgemで使えるようになります)
Presenterの仕事はデータをControllerから受け取って抽象化してTemplateに見せることです。
このようにすることでコンテンツ描画コードをクリーンに保つことができるので最近では有名なデザインパターンです。

コードの説明に戻ります。ここでは`render`メソッドを実装して`Hello, Lotus`というメッセージを表示しています。

この例はあまり面白くないですね。ViewとControllerを連携させてデータのやりとりをさせてみましょう。

Controllerのコードを以下のように編集します。

```ruby
module Controllers
  module Home
    include OneFile::Controller
    action 'Index' do

      expose :time

      def call(params)
        @time = Time.now
      end
    end
  end
end
```

２つ新しいメソッドが出てきました。`expose`と`call`です。

ControllerからViewにデータを渡すには、exposeを使って明示的に公開する変数を指定しないといけません。
これも`本当に必要なものだけを使う`というLotusの哲学の現れです。

 `call`はHTTPリクエストのエントリーポイントとして動きます。さっきも少し触れたようにこの中にビジネスロジックやレスポンス生成のコードを書きます。

ViewをControllerからデータを受け取るように変更しましょう。

```ruby
module Views
  module Home
    class Index
      include OneFile::View

      def render
        "Current time: #{time}"
      end
    end
  end
end
```

Controllerが`@time`を`expose`で公開しているので、Viewはこのデータに`time`変数経由でアクセスすることができます。

アプリケーションを再起動してください。ブラウザで同じページにアクセスすると`Current time: 2014-09-11 23:18:30 +0900`のように表示されるはずです。

## まとめ
LotusでWebアプリを書くのはとても簡単だということがわかってもらえたでしょうか？

ええ、あなたの心の声が聞こえてきますよ。*この例は単純すぎて全然実用的ではないじゃないか* という声が、、、

確かにその通りです。ですが、それは次の記事で紹介させてください。本当はFull stackアプリケーションに作り方まで紹介する予定だったのですが
この記事を書くのに予想より手間取ってしまいました。

次の記事は、***Ruby LotusでフルスタックWeb Appを作ってみる*** みたな感じになる予定です。
