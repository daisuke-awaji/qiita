# ［待望リリース］もう Lambda×RDS は怖くない!　〜全てがサーバレスになる〜

Lambda から RDS に対するアクセスはコネクション数の上限に達してしまうという理由からアンチパターンとされてきました。
そのため、RDS をデータストアに選択する場合は ECS や EC2 上にアプリケーションをホストする事が一般的でした。

本日の reinvent でのリリースで衝撃のアップデートがたくさん出ましたね。EKS on Fargate や SageMaker の大幅アップデートも魅力的ですが Lambda の常識をくつがえす RDS のプロキシ機能が登場しました 🎉

https://aws.amazon.com/blogs/compute/using-amazon-rds-proxy-with-aws-lambda/

※本記事は上記のブログを参考にしています。一部文脈で引用している箇所があります。

![image](https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2019/12/03/serverless-app-rds.png)

この RDS プロキシは、データベースへの接続プールを維持します。これにより Lambda から RDS データベースへの多数の接続を管理できます。

Lambda 関数は、データベースインスタンスの代わりに RDS プロキシと通信します。スケーリング起動した Lambda 関数によって作成された多くの同時接続をスケーリングするために必要な接続プーリングを処理します。これにより、Lambda アプリケーションは関数呼び出しごとに新しい接続を作成するのではなく、既存の接続を再利用できます。

従来はアイドル接続のクリーンアップと接続プールの管理を処理するコードを用意していたのではないでしょうか。これが不要になります。劇的な進化です。関数コードは、より簡潔でシンプルで、保守が容易になります。

現在はまだプレビュー版ですがこの機能を徹底検証していきましょう。

せっかく検証するのですから従来 ECS などで一般的に使ってたフレームワークを例に上げてみましょう。今回は NestJS を Lambda にデプロイして RDS と接続してみます。

# NestJS を Lambda にデプロイする

ServerlessFramework を使用して NestJS アプリケーションを AWSLambda にデプロイしておきます。
[こちら](https://github.com/rdlabo/serverless-nestjs)のサンプルソースが参考になりました。ほぼそのまま引用させていただきます。

Lambda がデプロイできたことを確認しておきましょう。

```
$ sls deploy
Serverless: Packaging service...
Serverless: Excluding development dependencies...
Serverless: Creating Stack...
~~~~~~~~~~~~~~~~~~~~~~~~~~ 省略 ~~~~~~~~~~~~~~~~~~~~~~~~~~
endpoints:
  ANY - https://djpjh5aklf.execute-api.us-east-1.amazonaws.com/dev/
  ANY - https://djpjh5aklf.execute-api.us-east-1.amazonaws.com/dev/{proxy+}
functions:
  index: serverless-nestjs-dev-index
layers:
  None
Serverless: Run the "serverless" command to setup monitoring, troubleshooting and testing.
```

生成されたエンドポイントにアクセスします。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163591/633e096a-b8d5-de1c-2983-50c2c0b3f2e2.png)

これで準備ができました。まずは **Hello World!** と文字列を返す NestJS on Lambda をデプロイできました。
これから MySQL と接続できるアプリケーションを作っていきます。開発過程は省略しますが、以下のリポジトリに完成品をアップロードしておきます。

参考：[Nest(TypeScript)で遊んでみる 〜DB 連携編〜](https://area-b.com/blog/2018/09/16/1600/)

タスクの CRUD 操作ができるアプリケーションを用意しました。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163591/0ba279af-81b3-7e0d-5234-008db7010d2c.png)

# Secret Manger に RDS への接続情報を登録

事前に作成しておいたこちらの RDS を使用します。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163591/b30cf55a-9c7b-0e1b-900c-ef9f6e667179.png)

まずは Secret Manger コンソールで RDS への接続情報を登録するようです。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163591/2520e026-49ca-0991-4694-c17b5f50a336.png)
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163591/e96bfb60-3cde-3a5b-d010-2eebb8334de9.png)

シークレットができたら ARN をメモしておきましょう。あとで使います。

# IAM

次に、RDS プロキシがこのシークレットを読み取ることができる IAM ロールを作成します。RDS プロキシはこのシークレットを使用して、データベースへの接続プールを維持します。IAM コンソールに移動して、新しいロールを作成します。 前の手順で作成したシークレットに secretsmanager アクセス許可を提供するポリシーを追加します 。

## IAM ポリシー

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163591/e3a5166a-e381-2bf4-013c-4c28b1bbfd1c.png)

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163591/7a2e31e4-b898-d11d-1daf-bc18590dad93.png)

## IAM ロール

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163591/24a7ba0a-3167-1a72-b149-1230496f674a.png)
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163591/e06b02ff-c34d-f527-8518-e425d8d9d7cd.png)

rds-get-secret-role という名前で IAM ロールを作成しました。

# RDS Proxy

さて、ここからが本題です。
RDS のコンソールを開くと Proxies の項目があります。Lambda の接続先をこのプロキシに向けることでを使うことでコネクションプールをうまく使いまわしてくれるようです。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163591/8508c323-e689-f382-aa9f-c905f44731b7.png)

作成してみましょう。先ほど作成した IAM ロールや RDS を入力します。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163591/991c87d7-d305-9b36-fe39-bc6ac3ecb487.png)
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163591/578507bb-cbcd-ae60-7d51-09ebef77d940.png)
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163591/91c68e7d-20fe-fb44-e36b-e98279316497.png)

作成まではしばらく時間がかかるようです。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163591/623f1268-7dd8-77db-ffab-b81d97b1407e.png)

# Lambda の向き先を RDS から RDS Proxy に切り替える

RDS インスタンスに対して直接接続する代わりに、RDS プロキシに接続します。これを行うには、2 つのセキュリティオプションがあります。IAM 認証を使用するか、Secrets Manager に保存されているネイティブのデータベース認証情報を使用できます。IAM 認証は、機能コードに認証情報を埋め込む必要がないため、推奨されているようです。この記事では、Secrets Manager で以前に作成したデータベース資格情報を使用します。

```javascript:db.config.ts
import { TypeOrmModuleOptions } from "@nestjs/typeorm";
import { TaskEntity } from "./tasks/entities/task.entity";

export const dbConfig: TypeOrmModuleOptions = {
  type: "mysql",
  host: "rds-proxy.proxy-ch39q0fyjmuq.us-east-1.rds.amazonaws.com", // <-- DBの向き先をProxyに切り替える
  port: 3306,
  username: "user",
  password: "password",
  database: "test_db",
  entities: [TaskEntity],
  synchronize: false
};
```

```
$ npm run build && sls deploy
```

まだ Serverless では RDS Proxy をサポートしていないようでしたので Lambda のコンソールから設定してみます。セキュリティグループやサブネットなどは適宜各自の環境に合わせて作成してください。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163591/9e5f3b92-b20e-bdfb-dfd7-500ca655330d.png)
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163591/19cce4cf-1c63-01a7-49af-57ed4cfa98db.png)

接続できました 🎉
※事前に DB にはテストデータを入れてあります
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163591/9684d561-5f73-4283-4517-1114219d28d6.png)

準備に使用した SQL

```sql
CREATE TABLE `tasks` (
  `id` int(36) unsigned NOT NULL AUTO_INCREMENT,
  `overview` varchar(256) DEFAULT NULL,
  `priority` int(11) DEFAULT NULL,
  `deadline` date DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=94000001 DEFAULT CHARSET=utf8mb4;

INSERT INTO `tasks` (`id`, `overview`, `priority`, `deadline`)
VALUES
	(1, '掃除', 0, '2020-11-11'),
	(2, '洗濯', 2, '2020-12-03'),
	(3, '買い物', 0, '2020-11-28');
```

# 負荷テストを実行してみる

コネクション数が Lambda のスケールに合わせて増え続けるような挙動を取らないか確認してみましょう。

今回は負荷のために [Artillary](https://artillery.io/) を使用します。
yaml ファイルでシナリオを記述して実行する Nodejs 製の負荷テストツールです。

### Artillary のインストール

```
$ npm install -g artillery
```

### 実行

yaml ファイルを記述しなくてもワンラインで実行できる手軽さも魅力的なツールで愛用しています。
以下のようなコマンドで簡単に実行できます。30 ユーザが 300 回リクエストを送るといった内容です。

```
$ artillery quick --count 300 -n 30 https://djpjh5aklf.execute-api.us-east-1.amazonaws.com/dev/tasks
```

実行された Lambda を確認します。Invocations が 9000 回を記録しています。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163591/8d04655d-9cf7-1306-9030-dcbc1697770f.png)

一方で RDS のコネクション数はなんと 43 になっていました。すごい。
ちなみに MySQL の現在のコネクション数は `show status like 'Threads_connected'` で確認できます。

| 負荷テスト開始前 | 最大リクエスト時 |
| ---------------- | ---------------- |
| 18               | 43               |

# RDS Proxy を使わない場合はどうなるか

アプリケーションの向き先を RDS 本体に直接接続するように変更してみます。
この状態でもう一度負荷テストを行うとどうなるでしょうか。

```
import { TypeOrmModuleOptions } from '@nestjs/typeorm';
import { TaskEntity } from './tasks/entities/task.entity';

export const dbConfig: TypeOrmModuleOptions = {
  type: 'mysql',
  host: 'aurora.cluster-ch39q0fyjmuq.us-east-1.rds.amazonaws.com', // <-- RDS 本体に向ける
  port: 3306,
  username: 'user',
  password: 'password',
  database: 'test_db',
  entities: [TaskEntity],
  synchronize: false,
};
```

### 実行

コネクション数が 124 まで膨れ上がってしまいました。
やはりプロダクションロードで普通に Lambda+RDS の組み合わせはやってはいけないアンチパターンになりそうですね。RDS Proxy の威力を改めて感じることができました。

```
$ artillery quick --count 300 -n 10 https://djpjh5aklf.execute-api.us-east-1.amazonaws.com/dev/tasks
```

| 負荷テスト開始前 | 最大リクエスト時 |
| ---------------- | ---------------- |
| 18               | 124              |

# まとめ

RDS プロキシを使用することで、データベースへの接続プールを保持することが確認できました。これで API やユーザリクエストを受けるようなワークロードでも Lambda から RDS への多数の接続を管理できます。とてつもなく強力なアップデートを体感できました。

次は RDS がインスタンスを意識することなく水平にスケールするようになったとき、完全にサーバレスなクラウドが完成しますね。待ち遠しいです。
