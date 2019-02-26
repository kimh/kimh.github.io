---
layout: post
title: "CircleCI Orbsを使ってECR/ECSへ自動デプロイする"
date: 2019-02-26 11:21
comments: true
categories:
---
# モチベーション

JAWS Days 2019でCircleCI Orbsを使ってAWSと連携されるネタで登壇してきました。当日の前日Twitterで公開することを約束しちゃったので、それならしっかりしたやつ作ろうということでブログ化しました。当日のスライドは[ここ](https://speakerdeck.com/kimh/orbswoshi-tuteawshejian-dan-depuroi)。

![tweet](/images/ecr_ecs_tweet.png =450x300)
https://twitter.com/inokara/status/1098529153890963458

# やりたいこと

CircleCIとECR、ECSを連携させて変更が自動でデプロイされるようにする。めっちゃ雑だけど以下のような感じの構成。

![diagram](/images/ecs_ecr_diagram.png =800x500)

流れとしては

- リポジトリのDockerイメージをアップデート
- GitHubへプッシュ
- CircleCIでイメージをビルドしてECRへプッシュ
- CircleCIで新しいイメージを使ってECSのタスク定義を更新してECSへプッシュ
- ブラザウから変更を確認

これをするためにECR/ECS側とCircleCI側の設定が必要になる。

# ECR側の設定

まずはECR側の設定。普通にリポジトリをECRで作るだけなので特別説明が必要なところはない。ひとつあるとすると、この時に設定する `Repository name` と`URI (1234567.dkr.ecr.us-east-1.amazonaws.com/nginx みたいなやつ)` はCircleCIの設定の時に必要になるくらい。

# ECS側の設定

自分のAWS力が低すぐるってのもあるんだろうけど、ちょっとハマったのでこっちは詳しく解説してみようか。

ECSには大きく分けて3つのコンポーネントがあって、Cluster → Service → Task の順でより粒度が細かくなっていく。Taskには主に実行するコンテナに関する情報を、ServiceにはどのTaskを何個起動するかやLBと連携して負荷分散などの情報を、Clusterはそれらを実行するEC2インスタンスに関する情報をそれぞれ定義する。これらの理解については[このQiitaの記事](https://qiita.com/NewGyu/items/9597ed2eda763bd504d7)がわかりやすかった。

## Taskの作成
- Create new Task Definition: EC2を選択 (Fargateよくわからん)
- Task Definition Name*: 適当な名前をつける。この記事では `kim-app-nginx` にする
- Task Role: `ecsTaskRoleExecution` を選択。
- Network Mode: `<default>` を選択。
- Task execution role: `ecsTaskExecutionRole` を選択。
- Task memory/ CPU: 適当に。

Add containerボタンを押してコンテナイメージを入力していく。

- Container name: 適当につける
- Image: ECRで作成したリポジトリを書く。**注意するのはURI/リポジトリ:タグ** の形式で書く。この記事では例として `1234567.dkr.ecr.us-east-1.amazonaws.com/nginx:latest` としておく。
- Port mappings: Host Portに0、Container PortにそのコンテナがListenするポートを書く。nginxの場合はContainer Portは80。**Host Portを0をするところが重要。後述するけどこれをしないと複数のTaskを立てた時にポートがバッティングしてしまい自動デプロイに失敗する。**

## Clusterの作成
Serviceを作る前にまずClusterを作成する。

Create Clusterから **EC2 Linux + Networking** を選択。ここはあんまり迷うところないと思うけどContainer instance IAM roleには `ecsInstanceRole` が選択されていることを確認。Cluster nameは適当につける。このブログでは `default-kim` とする。

Clusterが作成してしばらくするとECS Instancesのところで新しいEC2インスタンスが起動される（はず。もし起動されなかったらScale ECS Instancesを押してみて）

## Load Balancerの作成

唐突にLBの話が出てきたけど、これにはちゃんと理由がある。本来であればClusterを作成したら紐づくServiceを作成する。自動デプロイなしでとりあえずECSを動かすだけならそれでいいんだけど今回はCircleCIから自動デプロイすることが最終ゴールで、これをするためにLBが必要になる。

まずは問題を説明しよう。LBなしでServiceを作って自動デプロイすると以下のようなログが出て新しいタスクが起動しなかった。

`service MYSERVICE was unable to place a task because no container instance met all of its requirements. The closest matching container-instance .... is already using a port required by your task`

エラーメッセージにあるように、新しいタスクのポートが古いタスクとコンフリクトしてしまい、起動に失敗している。これを解決するにはコンテナがephemeralポートで起動するようするしないといけない。

[このSO](https://stackoverflow.com/questions/48931823/i-cant-deploy-a-new-container-to-my-ecs-cluster-because-of-ports-in-use)を読むと[動的ポートマッピング](https://aws.amazon.com/jp/premiumsupport/knowledge-center/dynamic-port-mapping-ecs/)が必要でそれをするためにはLBを作成する必要があるらしい。

というわけで、一旦ECSの画面から離れてEC2のLoad Balancingのページに行く。Create Load BalancerからApplication Load Balancerを選択。ListenersのところでLoad Balancer ProtocolにHTTPを、Load Balancer Portは80 (nginxの待ち受けポート)を設定する。VPCはCluster作成の時に選択したVPCと同じものを選択。

次にConfigure Security GroupsでSecurity Groupを設定するんだけどここでも一つ重要ポイントが。

**選択したSGでTCPの0-65535ポートを許可しないとLBからトラフィックがコンテナまで到達しないのでタスクがちゃんと起動してくれない。** これは[公式のドキュメント](https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/create-application-load-balancer.html#alb-configure-security-groups)にも書かれているので忘れずに実施。

最後にTargetを登録する。うまくいけばClusterを作成した時に立ち上がったECSインスタンスが表示されるはず(されない場合は間違ったVPCを選んでいる？)なのでこれを選択する。これでLBの設定は完了。

## Serviceの作成

LBが無事できたらようやくServiceを作成できる。事前に作成したClusterを選択するとServicesというタブがあるのでそこから新しいServiceを作成していく。

- Launch Type: `EC2` (Fargateよくわからん)
- Task Definition: Familyに最初に作成したTaskを選択する。ここでは `kim-app-nginx` 。RevisionというのがあるけどこれはTaskをアップデートした時のバージョニングとして使われる。
- Service name: TaskのFamilyと同じにしておくとCircleCI Orbs側で渡すパラメーターが少なくて少し楽になる。今回はTaskと合わせて `kim-app-nginx` とする。
- Number Tasks: 今回は常時ひとつだけタスクが稼働しているようにするので `1` と設定。

Minimum healthy percentとMaximum healthy percentに関しては自動デプロイと絡んで重要なので解説する。この2つの設定は起動するタスクの最低・最大数を設定する。実際の数は `Number Tasks * percent` で算出。Minで設定した数だけ常時タスクが起動して、デプロイした時にMaxで設定した数になるように旧・新タスクが平行稼働する。(新タスクが完全にデプロイされたらMinの数にまた収束する)

今回はNumber Tasks 1、Min 100、Max 200にする。こうすると常時1タスクが起動していて、デプロイした時は旧・新合わせて2つのタスクが起動することでローリングアップデートすることができるようになる。

これらを設定してNext StepをクリックするとLoad balancingが出てくるので、Application Load Balancerを選ぶ。すると、さっき作成したLBが出てくるので選択する。 **もう一度言うけど、これをしないと自動デプロイできないので必ず先にLBを作成しましょう。**

次にContainer to load balanceのところでAdd to load balancerを押すと確かnginxのコンテナが表示されるはずなので、Production listener port*に80を指定。Health check pathにはちゃんとコンテナが成功レスポンスを返すパスを指定することに注意。Service discoveryは今回必要ないので無効化にすればServiceの作成は完了。

# AWS側のまとめ

ここまででAWS側の設定は全部完了した。まとめると、ECRでリポジトリを作って、ECS側でTask, Service, Clusterと自動デプロイのためにLBも作成した。LBのURLにアクセスすればnginxのレスポンスが見えるはず。

# CircleCIの設定

これでようやくCircleCIの設定をすることができる。ただ、AWSに比べれば作業量ははるかに少ない。

## 環境変数の設定

センシティブな情報をconfig.ymlに直接書いてはまずいので環境変数に入れる。今回設定する環境変数は以下。

**AWS_ACCESS_KEY_ID**
AWSのアクセスキー

**AWS_SECRET_ACCESS_KEY**
AWSのシークレットキー

**AWS_ECR_ACCOUNT_URL**
ECRのアカウントのURL。数字で始まって `amazonaws.com` で終わるところまで。今回の例だと `1234567.dkr.ecr.us-east-1.amazonaws.com` がそれにあたる。

**AWS_REGION**
AWSのリージョン

これらの環境変数はOrbsのデフォルト値として設定されているので同じ名前の環境変数にすることに注意。

# aws-ecs/aws-ecr Orbsを使う

記事が長くなってしまうので今回はOrbsそのものについては説明スキップ。すでに日本語で情報がたくさんあるのでそっちを見てください。以下の記事がおすすめ。

https://blog.tsub.me/post/introducing-to-circleci-orbs/

https://www.kaizenprogrammer.com/entry/2018/12/01/111145

https://github.com/sue445/circleci-user-community-meetup-01/blob/master/slides.md

今回は[aws-ecs](https://circleci.com/orbs/registry/orb/circleci/aws-ecs)と[aws-ecr](https://circleci.com/orbs/registry/orb/circleci/aws-ecr)というCircleCI自身がメンテしている公認Orbsを使う。

まずは `.circleci/config.yml` の完成系を貼っておく。

```
version: 2.1
orbs:
  aws-ecr: circleci/aws-ecr@1.0.0 #(1)
  aws-ecs: circleci/aws-ecs@0.0.6 #(2)
workflows:
  build-and-deploy:
    jobs:
      - aws-ecr/build_and_push_image: #(3)
          account-url: AWS_ECR_ACCOUNT_URL
          repo: 'nginx'
          tag: '${CIRCLE_SHA1}'

      - aws-ecs/deploy-service-update: #(4)
                requires:
                  - aws-ecr/build_and_push_image
                family: 'kim-app-nginx'
                cluster-name: 'default-kim5'
                container-image-name-updates: 'container=nginx,image-and-tag=${AWS_ECR_ACCOUNT_URL}/nginx:${CIRCLE_SHA1}'
```

**(1):** aws-ecr Orbをインポート

**(2):** aws-ecs Orbをインポート

**(3):** aws-ecrのOrbに `build_and_push_image` というジョブがあらかじめ用意されていてユーザーはこれに必要なパラメータを渡すだけでECRにデプロイできる。注意点としては `tag` のところで `${CIRCLE_SHA1}` をつかっている。これにはGitのコミットのSHAが入っているのでデプロイするたびに新しいタグが自動的につく。

**(4):** 同じくaws-ecsに事前定義されている `build_and_push_image` というジョブにパラメーターを渡すだけでECSにデプロイできる。超便利。 `family` にはTaskの名前を指定。 `container-image-name-updates` にはアップデートするタスクのコンテナ名と使うイメージをカンマ区切りで渡す。重要なのは `image-and-tag` のところで `${CIRCLE_SHA1}` をタグとしてつかうことで `build_and_push_image` でデプロイしたイメージを使っている。

# CircleCIまとめ

これ以降はプッシュするたびに `build_and_push_image` で新しいイメージがECRへプッシュされ `deploy-service-update` がそのイメージを使った新しいTaskを生成してServiceでそのタスクを使うようにアップートしてくれる。すると、ECSとLoad Balancerが連携して新しいインスタンスを作って古いインスタンスをDrainすることでローリングアップデート完了！

![rolling_update](/images/lb_draining.png)

登壇したスライドでも書いたんだけど、Orbsを使うことで `config.yml` を370行から20行まで圧縮できたのでOrbsの便利さが実感できる例になった。

# 全体まとめ

- ECRとECSの設定はわりと簡単
- だけど、自動デプロイするにはLBが必要で少し手間が必要
- aws-ecs/aws-ecrのOrbsは超便利。これなしでスクラッチからconfig.ymlを書くとか吐き気がする。
