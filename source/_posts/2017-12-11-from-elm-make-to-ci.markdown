---
layout: post
title: "Elmでアプリを作る: elm-makeからCIまで"
date: 2017-12-11 21:48
comments: true
categories:
---

この記事はQiita Elm Adventcalendar 2017に投稿した記事です。

## 最初に

Functional Reactive Programmingを調べていて偶然Elmに出会いました。最新版のElm0.18ではElmはFRPの概念を完全に取り払い関係なくなりましたが、
実際にElmを触ってみるとどんどんその面白さに引き込まれて行きました。もっとElmで何か作りたくなったので自分の子供が遊べるようなToy Appを作ることにしました。
このアプリを作る際にはまった点や大事な点をできるだけ丁寧に解説してみたいと思います。

## この記事の対象者

この記事ではElmについての基本知識があることを前提としています。Elmにはすでに素晴らしいチュートリアルがあるかです。もし、まだElmに触ったことがなければ https://guide.elm-laang.org/ と https://www.elm-tutorial.org/en/ にゆずります。
基本の文法はわかるけどいまいちSubscriptionや$Portがわからないなーという人には楽しんでもらえると思います。という自分もそれらの機能には今回初めて触るので、疑問に思ったとこや考え方をかければと思います。

## Airplane Toy App

この記事で解説しているアプリは https://s3-ap-northeast-1.amazonaws.com/airplane-toy-app/index.html で公開しています。 (このアプリは音が出るので注意!!)
コードは https://github.com/kimh/kids-toy-apps で公開しています。
2歳になる自分の息子に作ったアプリなのでとても単純な仕様で、クリックした場所に飛行機がいかにも子供が好きそうな音を出して移動する、というだけのアプリです。
しかしElmで実際にアプリを作るのに必要なことはほとんど使う必要があったのでとても勉強になりました。

## イベントフロー

このアプリでは以下のようなイベントフローで構成されています。

ページをロード -> Init関数が呼ばれModelが初期化される -> ユーザー (うちの息子) が画面のどこかをクリックする -> `MouseMsg`メッセージが送信される -> Subscriptionを通じて、Update関数が呼ばれる。クリックされた座標を取得して、Modelにセットする。 -> Modelに新しい座標がセットされると飛行機が動く -> 同時に `PlaySound` メッセージが送信される -> JavascriptにPortでメッセージを送信する -> Javascript側で `audio` 要素の `play` メソッドを呼ぶ

## 飛行機画像のレンダリング

飛行機の画像を扱うのに Elm Collageを使っています。Collageのライブラリには主にhttps://github.com/evancz/elm-graphics と http://package.elm-lang.org/packages/timjs/elm-collage/latest がありますが、後者です。timjs/elm-collageはevancz/elm-graphisを置き換えるものだと考えているのでこれからアプリを作るのならtimjs/elm-collageを使うことぼをお勧めします。

重要となるポイントを説明していきます。

`image` で飛行機の画像ファイルを読み込んで `shift` で初期位置まで移動させます。 `pos` は 渡されたModelから作成します。(後述)

```
plane =
    image (500, 500) "images/airplane.svg"
    |> shift pos
```

`spacer` を使っている部分は少しわかりにくいです。timjs/elm-collageでは座標が絶対的ではなく他の要素からの相対的なCollageの位置を指定するので
`plane` だけを `shift` しても画像が動いてくれませんでした。そこで `plane` の前に透明な要素である `spacer` を挟むことでうまく移動してくれるようにしました。（このあたりは自分の理解も曖昧なのでもう少し調べます）`group` は複数のCollageを並べてくれる関数です。

コメントアウトしている `debug` は有効にするとCollageのボーダーとセンターを赤線で表示してくれるのでつまった時に便利なのでいつでも有効にできるように残しています。

最後に要素を `Html Msg` 型に `svg` で変換しています。このパターンはCollageではいつも使うパターンです。

```
group [
  spacer 300 300,
  plane
]
    -- |> Collage.Layout.debug
    | > svg |
```

## アニメーション

飛行機を任意の場所に移動するために `elm-animation` を使っています。基本的な考え方はブラウザのフレーム更新毎に送られる `Tick` メッセージを捕まえてその中でモデルを更新します。

重要となるポイントを説明していきます。

モデルにはx座標、y座標、clockをもたせています。これらの値を `Tick` イベントの中で更新します。

```
type alias Model =
    { x : Animation, y : Animation, clock : Time }

model : Model
model =
    Model
        (animation 0 |> duration Time.second)
        (animation 0 |> duration Time.second)
        0
```

`Tick` メッセージは引数つきのメッセージで、引数にアニメーションの初期状態から差分の時刻が送られてきます。それをモデルの `clock` にセットしています。

```
update : Msg -> Model -> (Model, Cmd Msg)
update msg model =
    case msg of
        Tick dt ->
            let
                clock = model.clock + dt
            in
                ({ model | clock = clock }, Cmd.none)
```

更新されたclockの値と飛行機のx,y座標を `animate` に渡すとその時刻での飛行機の座標が返ってきます。 `shift pos` をするとその座標に画像がレンダリングされます。これを各 `Tick` 毎に行うことで画像が移動していくように見えます。

```
view { x, y, clock } =
    let
        pos =
            ( animate clock x, animate clock y )

        plane =
            image (500, 500) "images/airplane.svg"
            |> shift pos
```

`Tick` メッセージはそのままでは送信されません。送信するためには `Subscription` で `animation-frame` という別のライブラリを使って `AnimationFrame.diffs Tick` を呼びます。Animationするためのライブラリ(`elm-animation`) とフレーム遷移を扱うライブラリが別なのは面白いポイントです。

```
subs =
    Sub.batch
        [
        AnimationFrame.diffs Tick,
        Mouse.clicks MouseMsg
        ]
```

## マウスイベント

クリックされた場所まで飛行機を移動しないといけないのでマウスイベントを捕まえてクリックされた座標を取得します。

まずマウスイベントを捕まえるために `Mouse.clicks MouseMsg` をサブスクライブします。

```
subs =
    Sub.batch
        [
        AnimationFrame.diffs Tick,
        Mouse.clicks MouseMsg
        ]
```

実際にマウスイベントを扱うには `MouseMsg` を捕まえます。`MouseMsg` は引数つきのメッセージで引数にはクリックされた座標が入っています。 このアプリ独自のロジックで `adjustment` とかを使っていますが、重要なのはクリックされた座標をFloat型に変換して `retarget` に渡すところです。`retarget` は画像がどこまで動くかの値 `to` を更新して座標で返すので、それをモデルの座標にセットします。こうすることでクリックされた位置が新しいアニメーションの `to` になります。

```
MouseMsg position ->
   let
       -- We need this so that plan's center moves to the new postion
       adjustment = -150
       posx = position.x
              |> toFloat
              |> (+) adjustment
       posy = position.y
              |> toFloat
              |> (+) adjustment
              |> (*) -1
       newX = retarget model.clock posx model.x
       newY = retarget model.clock posy model.y
   in
       ({model | x = newX, y = newY } |> update PlaySound)
```

## 音を鳴らす

このアプリを作るときに一番わかりにくかったところはクリックされたときに音を鳴らす動作です。Javascriptだと `audio` 要素を取得して `play` 関数を呼ぶだけなんですが `play` は副作用を及ぼす関数なので調べた限りではElmからは直接呼ぶことができませんでした。そこで `Port` の出番です。

Portの仕組みはとても簡単で、Javascript側でElmからのメッセージをサブスクライブして何か送られてきたら使いたいJavascriptのコードを実行するだけです。今回の例だと、Elmが音を鳴らしたいときにPortにメッセージを送り、Javascript側で `audio` 要素を取得して `play` を呼ぶだけです。

まずJavascript側から説明します。このコードは `index.html` に書かれています。

`app.ports.play.subscribe(function(val){...|})` がElmからPortにメッセージが送られてきた時に呼ばれるコールバックです。単純に `audio` 要素を取得して `play` しているところがメインです。(一度 `pause` しているのは音が終わる前にもう一度クリックされた時に最初から再生するためにです。)


```
<body background="images/sky.jpg">
    <audio id="my-audio" src="audios/flee1.mp3"></audio>
    <script type="text/javascript">
     var app = Elm.Main.fullscreen();

     // Triggered from Elm
     app.ports.play.subscribe(function(val){
         var audio = document.getElementById('my-audio');
         audio.pause();
         audio.currentTime = 0;
         audio.play();
     });
    </script>
</body>
```

次にPortにメッセージを送る側のElmコードを見てみます。Updateの中で `PlaySound` を捕まえます。 `PlaySound` がどこからくるかというと `MouseMsg` をハンドリングする際に戻り値で手動で `update` を呼んで `PlaySound` のコマンドを渡しています。

```
({model | x = newX, y = newY } |> update PlaySound)
```

`play 0` と呼ぶとJavascript側でサブスクライブしているPortに `0` という値が送られます。ここではJavascript側は何も値は必要ないのですが引数なしで送る方法がわからなかったので適当な値を渡しているだけです。

```
PlaySound ->
    (model, play 0)
```

## まとめ

大事な点はだいたいカバーしたと思います。純粋関数言語からアニメーションを扱うやり方や考え方はとても面白かったです。はじめは少しとっつきにくいかもしれませんが、慣れれば他の似たようなライブラリも簡単に理解できるようになりました。

この記事を通してElmのファンが増えてくれれば嬉しいです!
