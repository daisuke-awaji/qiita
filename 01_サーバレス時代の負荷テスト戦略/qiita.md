# ローカル実行

```
$ artillery run script.ym
```

# プラグインの追加

```
$ npm install --save artillery-plugin-cloudwatch
```

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

# デプロイ

```
$ slsart deploy --stage dev
```

# 実行

```
$ slsart invoke --stage dev
```

# 片付け

```
$ slsart remove
```

# CircleCI から実行する
