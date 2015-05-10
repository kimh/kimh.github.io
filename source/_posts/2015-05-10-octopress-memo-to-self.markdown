---
layout: post
title: "octopress_memo_to_self"
date: 2015-05-10 12:55
comments: true
categories:
---

***To: ３日後には全てを忘れている自分へ***

今回、久しぶりにブログを書こうとしたらOctopressの仕組みを全く覚えていなくて、いろいろ回り道したせいで、ほぼ丸１日無駄にしてしまった。おそらく未来の自分はまた忘れるのだろうでここにメモをしておく。　

### masterとsourceブランチ
ブログはGithub Pageでホスティングしている。ブログそのものは、http://kimh.github.io/ で記事とかOctopressのコードは、https://github.com/kimh/kimh.github.io で管理している。

https://github.com/kimh/kimh.github.io には **master** と **source** の二つのブランチがある。**soruce** にはmardkdownで書かれた記事が保存されていて、デプロイするとここから静的HTMLを生成して **master** にpushする。

今回このことを覚えていなくて時間の多くを無駄にしてしまった。というのも、新しいMacに移行した際に新しくブログのレポジトリをcloneしたんだけど、中身を見たら静的HTMLしか見当たらない。というかOctopressそのものが見当たらない。
あるのはmarkdownから生成されたと思われるhtmlファイルばっかり。「なんてこった！！！markdownで書いた記事はどこか別の場所で管理していて、それをGithubにはpushしていなかったんだ！今まで書いた記事が消失してしまった。やってもたー！！！」

それで、静的HTMLからmarkdownへがんばって変換して、復旧が完了した時にふと読んだOctopressの解説記事で **source** ブランチがあることを知った。恐る恐る **source** にcheckoutするとmarkdownで書かれた記事とOctopressがあるじゃないですか！

この発見はすごくショッキングだったけど、きっと未来の自分は忘れているんだろう。ということで、メモ。

- https://github.com/kimh/kimh.github.io には ***master*** と ***source*** ブランチがある。
- ***source*** にはmarkdownで書いた記事とOctopressがある
- ***master*** にはmarkdownからOctopressが生成した静的HTMLファイルがある
- http://kimh.github.io/ は ***master*** を表示している
- 記事を編集する時は ***source*** で行う
- というか、基本 ***master*** にcheckoutすることはない

### Octopressのディレクトリ構成 (sourceブランチ)
多くの時間を無駄にしてしまった原因の一つにそもそもOctopressのファイル構成を理解していなかったことがある。幸い今回の復旧作業ではそれ学ぶことができた。

以下は、`kimh.github.io`配下の重要ディレクトリ。

`source/_posts`ディレクトリにはmarkdown記事が置いてある。ブログの編集作業の大半はここにあるファイルを編集することになる。

`source/_includes`とか`source/_layouts`にはOctopressがブログサイトを構成するためのhtmlファイルが置いてある。ブログサイトそのものを変更したければここを直接いじることになる。3rd Party Themeがインストールされるのもここ（後述）

`.themes`には3rd Party Themeを置く。置き方は、`git clone https://github.com/rwwaskk/linkedlist.git .themes/linkedlist`のようにするのが一般的な方法のよう。
`.themes`に置いたら、`rake install['name-of-theme']`でインストールできる。このコマンドを実行すると、`source`　配下にthemeが生成したHTMLやCSSやJavascriptが生成される。

### ブログサイトそのものに変更を加える
前述したように、Octopressは`source`配下にあるファイルから静的HTMLを生成して、3rd Party Themeは`source`配下にインストールされる。しかし、時にはThemeが気に入らなくて自分で変更を加えたいこともあると思う。

例えば今回の復旧作業でThemeを[Linked List](https://github.com/rwwaskk/linkedlist)インストールした状態だとブログのタイトルを site.sub_title から取得しようとしている。コードは[linkedlist/source/_layouts/home.html](https://github.com/rwwaskk/linkedlist/blob/master/source/_layouts/home.html#L10)

自分的にはここは site.title からとって欲しい。じゃー、どのファイルを編集するのかというと、一つは、`.themes/linkedlist/source/_layouts/home.html` 編集して、`rake install['name-of-theme']` で再インストールする。

もう一つは、`source/_layouts/home.html`を直接編集する。ここで一つ注意するのは、**themeを再インストールするとsource/配下は全て置き換えてしまうので自分で加えた変更も上書きされてしまう。**

なので、この方法で編集した際はなんらかの理由でthemeを再インストールする時は要注意。(じゃー、やっぱりThemeのほうを変更すればよくない？と思うかもしれないけど、themeに自分の変更を加えるのはなんかおかしいと思う。この疑問は未解決。)

### よく使うOctopressコマンド

***rake generate***

`source`にあるファイルを`public`配下に静的HTMLとして生成する。

***rake preview***

http://localhost:4000 を立ち上げて、`public`配下をブラウザからアクセスできるようにする。

***rake setup_github_pages***

Github Pageにデプロイするために一度だけ実行しないといけない。

***rake deploy***

`public`配下にあるファイルを`_deploy`ディレクトリにコピーして、**master** ブランチにpushする。少ししたら、ブログのurlからアクセスできるようになる。


***rake gen_deploy***

generateとdeployを一気に行う。とりあえず、deployしたい時はこれを実行しておけば問題ない。
