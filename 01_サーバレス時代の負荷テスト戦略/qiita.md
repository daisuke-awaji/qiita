# サーバレス時代の負荷テスト戦略

![img](https://github.com/Nordstrom/serverless-artillery-workshop/raw/master/Images/serverless-artillery-diagram.png)

<!-- 仮想マシン(VM)のスペックの決定、ワークロードに応じた VM 数の調整、障害や災害に備えた可用性の確保などの作業さえもクラウドに任せ、開発者はアプリケーションの開発に専念できるサーバレス。サーバレスはクラウドのあるべき姿を体現しています。素晴らしい技術です。

しかし、その一方で制限がある条件下でのアプリケーション開発は決して容易ではありません。
例えばサーバレスを代表とする Lambda を WebAPI としてユーザリクエストを受ける構成がプロダクション環境での性能に耐えうるかどうか、開発プロセスの早い段階で判断したいものです。もし仮にどうしてもプロダクション環境での性能を満たせなかった場合は Fargate に移植するなどの選択肢をとるべきです。 -->

負荷テストに対する考え方は時代とともに変化してきました。従来はサーバスペックやシステムの限界性能を測るという考え方でしたが、クラウドネイティブなシステムではそれに加えて、システムの弾力性（スケールアウトのしやすさ）も考慮する必要があります。

本記事では負荷テストによって、システムの弾力性をどのように評価するかの方法についてツールの具体的な使用方法も交えて説明します。システムの弾力性を評価するために、プロダクション環境でのユーザからのリクエストを想定したロードテストを検討します。

ロードテストでは以下の項目を検証します。

### ドリップテスト

ドリップテストは通常、数日間にわたって行われます。通常のバックグラウンド負荷レベルをシミュレートします。遅延またはエラー率の増加が見られる処理を特定します。

### スラムテスト

スラムテストは、トラフィックの突然のスパイクをシミュレートします。これにより、オートスケーリングが適切に処理するのが従来困難であったトラフィックに直面したときのシステムの動作を確認できます。

### ランプテスト

ランプテストでは、キャンペーン期間中などを想定した日々トラフィックが増加する負荷をシミュレートします。システムが正常にスケールアウトすることを確認します。

# 負荷テストツールの選定

## [Apache ab](https://httpd.apache.org/docs/2.4/programs/ab.html)

Apache の ab コマンドはコマンドラインから Web リクエストの負荷をかけることができるシンプルなツールです。手軽に導入できますが、単一のサーバからしか実行できないため、大量のリクエストを送ることができません。[こちら](https://qiita.com/flexfirm/items/ac5a2f53cfa933a37192)の記事が参考になります。

## [Apache JMeter](https://jmeter.apache.org/)

![](https://jmeter.apache.org/images/logo.svg)

Apache JMeter は言わずと知れた負荷テストツールです。マスタスレーブ構成を構築し、マスタとなるサーバにはスレーブの情報（ip アドレスなど）を登録しておきます。GUI でシナリオを作り、マスタからスレーブに指示を送ることでロードテストを実行します。スレーブを増やすことで大量のリクエスト数を送ることができます。
JMeter は確かに多機能で操作性も良く、手に馴染んでいるツールですが負荷をかけるためのサーバを構築したり、GUI からシナリオを作成する手間があります。

## [Locust](https://locust.io/)

<img width="50%" alt="locust" src="https://i.imgur.com/ekk3ENY.png">

Locust は Python 製負荷テストツールです。
JMeter 同様、GUI が提供されており、マスタスレーブ構成を取り大量リクエストを生成できます。またテストシナリオを Python で記述でき、柔軟に対応できます。

Locust のシナリオの記述は容易ですが、 JMeter 同様サーバを構築する必要があります。
アプリケーションがサーバレスで構築できるようになったのに負荷テストのためにサーバを立てなければいけないというのは残念なところです。

# 宣言的な負荷テストツールをサーバレスで実行するという選択

最近では [JAMStack](https://jamstack.org/) な構成を採用する場合が増え、バックエンドは Web API として提供することが増えてきました。
Web API だけを対象とするならば、画面の表示を想定せずシンプルなテストシナリオを作成できます。
つまり、テストしたい API のエンドポイントとヘッダー、ボディなどを宣言的に羅列しておくだけ負荷テストを実行できると幸せです。

## [Artillery](https://artillery.io/)

Artillery は yaml ファイルに宣言的にシナリオを記述しておき、シンプルなインタフェースで負荷をかけることができる Nodejs 製のツールです。

### ローカル実行

npm コマンドでインストールします。

```
$ npm install -g artillery
```

ab コマンドのようにワンライナーが用意されています。
以下のコマンドは、10 人の仮想ユーザーを作成し、それぞれが 20 回の HTTP GET リクエストを https://artillery.io/ に送信することを意味しています。

```
$ artillery quick --count 10 -n 20 https://artillery.io/
```

実際には以下のように `script.yml` にテストシナリオを記述して使用します。

```script.yml
config:
  target: 'https://artillery.io'
  phases:
    - duration: 60
      arrivalRate: 20
  defaults:
    headers:
      x-my-service-auth: '987401838271002188298567'
scenarios:
  - flow:
    - get:
        url: "/docs"

```

HTTP 経由で通信する https://artillery.io のエンドポイントのテストシナリオです。 `phases` には、20 人の新しい仮想ユーザーで 60 秒負荷テストを実行することを記述しています。リクエストは平均で 1 秒ごとに到達します。

以下のコマンドでテストシナリオを実行します。

```
$ artillery run script.yml
```

以下のように結果が出力されます。

```
Complete report @ 2019-01-02T17:32:36.653Z
  Scenarios launched:  300
  Scenarios completed: 300
  Requests completed:  600
  RPS sent: 18.86
  Request latency:
    min: 52.1
    max: 11005.7
    median: 408.2
    p95: 1727.4
    p99: 3144
  Scenario counts:
    0: 300 (100%)
  Codes:
    200: 300
    302: 300
```

| 項目                | 説明                                                                                        |
| ------------------- | ------------------------------------------------------------------------------------------- |
| Scenarios launched  | 直前の 10 秒間に作成された仮想ユーザーの数（または合計）                                    |
| Scenarios completed | 直前の 10 秒間（またはテスト全体）でシナリオを完了した仮想ユーザーの数                      |
| Requests completed  | 送信された HTTP リクエストとレスポンスまたは WebSocket メッセージの数                       |
| RPS sent            | 直前の 10 秒間（またはテスト全体）に完了した 1 秒あたりのリクエストの平均数                 |
| Request latency     | p99: 500 は 100 リクエスト中 99 リクエストが完了するまでに 500 ミリ秒以下かかったことを意味 |
| Codes               | 受信した HTTP 応答コードの内訳                                                              |

# Lambda を使用して大量のリクエストを生み出す

Artillery だけでは単一のサーバ上から実行するため大量リクエストを生成できませんが、大量にリクエストを生成する方法があります。Artillery をサーバレスな実行環境に乗せてスケールさせる [serverless-artillery](https://github.com/Nordstrom/serverless-artillery) が公開されています。

同様のコンセプトを持ったツールに[Goad](https://goad.io/)があります。Goad は、Go で構築された AWS Lambda 搭載の高度に分散された負荷テストツールです。ここでは名前だけの紹介に留めます。

serverless-artillery は Artillery を serverless framework を介してデプロイします。低コストかつ短時間で大量リクエストを生成する負荷テスト環境を構築できます。

### デプロイ

```
$ slsart deploy --stage dev
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163591/c7c3970b-d264-0aa6-04d7-703f47413400.png)

### 実行

```
$ slsart invoke --stage dev
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163591/5b8fc9e7-2fc3-6364-a198-88f6bed0d58f.png)

### 片付け

```
$ slsart remove
```

# プラグインの追加

## artillery-plugin-cloudwatch

artillery-plugin-cloudwatch を追加することで、テスト結果を AWS CloudWatch に記録できます。
他にも DataDog 用のプラグインなどが用意されています。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163591/9e679bb6-ad5a-e44d-8633-34ac95aca488.png)

```
$ npm install --save artillery-plugin-cloudwatch
```

以下に serverless.yml の一部を抜粋する。

```
service: serverless-artillery

provider:
  name: aws
  runtime: nodejs8.10
  region: ap-northeast-1
  iamRoleStatements:
    - Effect: "Allow"
      Action:
        - "lambda:InvokeFunction"
      Resource:
        "Fn::Join":
          - ":"
          - - "arn:aws:lambda"
            - Ref: "AWS::Region"
            - Ref: "AWS::AccountId"
            - "function"
            - "${self:service}-${opt:stage, self:provider.stage}-loadGenerator*" # must match function name
    - Effect: "Allow"
      Action:
        - "sns:Publish"
      Resource:
        Ref: monitoringAlerts
    - Effect: "Allow"
      Action:
        - "cloudwatch:PutMetricData"
      Resource:
        - "*"
```

# 継続的に負荷テストをするという考え方

## CircleCI から実行する

## 負荷にランダム性を持たせる

CircleCI から継続的に負荷テストを実行できるようになったので開発フェーズにおいて常にある程度の負荷をかけ続けることができるようになりました。通常のバックグラウンド負荷レベルをシミュレートするドリップテストが継続的に実行できていますが、ある程度ユーザリクエストにゆらぎ（ランダム性）を持たせておくとより多くのケースをテストできます。つまり、`phases` の項目を動的に変更してみましょう。

```
config:
  target: https://gatjk9gwc4.execute-api.ap-northeast-1.amazonaws.com/dev/
  phases:
    - duration: 10              # この部分を
      arrivalRate: 1000000      # ランダムに変化させる
scenarios:
  - flow:
      - get:
          url: "/"
```

シナリオをランダムに変更する簡単なツールを作成します。

```generateSenario.js
function rand(min, max) {
  min = Math.ceil(min);
  max = Math.floor(max);
  return Math.floor(Math.random() * (max - min)) + min; //The maximum is exclusive and the minimum is inclusive
}

function generatePhases(phases, duration, arrivalRate) {
  const generatedPhases = [];
  for (let i = 0; i < rand(phases.min, phases.max); i++) {
    generatedPhases.push({
      duration: rand(duration.min, duration.max),
      arrivalRate: rand(arrivalRate.min, arrivalRate.max)
    });
  }
  return generatedPhases;
}

function generateScript() {
  fs = require("fs");
  yaml = require("js-yaml");
  const script = yaml.safeLoad(fs.readFileSync("./script.yml", "utf-8"));
  const phases = {
    min: 1,
    max: 50
  };
  const duration = {
    min: 10,
    max: 100
  };
  const arrivalRate = {
    min: 10,
    max: 10000
  };
  script.config.phases = generatePhases(phases, duration, arrivalRate);
  fs.writeFileSync("./converted.yml", yaml.safeDump(script));
}

generateScript();
```

シンプルに node コマンドで実行しても良いですが、以下のように npm スクリプトで実行できるようにしておくと CircleCI の設定を記述する際に見通しがよくなります。

```
$ node generateSenario.js
```

```
$ npm run generate:senario
```

実行結果は以下のようになります。このようにしてランダムに生成された `script.yml` を CI から実行します。今回のサンプルソースではスパイクがかかるようなケースを生成しませんでした。ランダムに生成する値をうまく調整することで、通常時の負荷を想定したドリップテストとスパイクアクセスを想定したスラムテストを同時に継続的に実行できます。

```
config:
  target: 'https://gatjk9gwc4.execute-api.ap-northeast-1.amazonaws.com/dev/'
  phases:
    - duration: 78
      arrivalRate: 6184
    - duration: 94
      arrivalRate: 4583
    - duration: 73
      arrivalRate: 6991
    - duration: 82
      arrivalRate: 2664
    - duration: 92
      arrivalRate: 5239
    - duration: 33
      arrivalRate: 1596
    - duration: 51
      arrivalRate: 9621
    - duration: 94
      arrivalRate: 4155
    - duration: 51
      arrivalRate: 3145
    - duration: 36
      arrivalRate: 802
    - duration: 90
      arrivalRate: 6286
    - duration: 46
      arrivalRate: 5219
    - duration: 29
      arrivalRate: 8020
scenarios:
  - flow:
      - get:
          url: /
```
