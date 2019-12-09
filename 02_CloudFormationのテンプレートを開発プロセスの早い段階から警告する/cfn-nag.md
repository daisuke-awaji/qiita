CloudFormation を使用することで、AWS のリソースを素早くプロビジョニングできます。また、yaml 形式のテンプレートファイルで記述することにより、インフラ開発者はどのリソースが作成されるのかを宣言的に管理できます。
開発者は AWS リソースをすばやく簡単に作成できますが、安全でないリソースもすばやく作成できてしまいます。「安全でない」とは、TCP ポートを世界中に公開したり、特定の IAM ユーザーにフル権限を与えてしまうことでセキュリティ的に問題があるリソースを作り出してしまうことを意味しています。

以前、CircleCi MeetUp で[CloudFormation を静的構文解析する話](https://speakerdeck.com/gawa/aws-testing-techniques-and-policy-as-a-code?slide=17)をしました。

https://github.com/stelligent/cfn_nag

# cfn-nag のインストール

```bash
gem install cfn-nag
```

# 使用方法

以下のようなテンプレートを例に取り上げます。

```template.yaml
  SecurityGroup:
    Type: 'AWS::EC2::SecurityGroup'
    Properties:
      VpcId: !Ref VpcId
      GroupDescription: security group
      SecurityGroupIngress:
        - IpProtocol: "TCP"
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
          Description: "All"
```

```bash
cloudformation$ cfn_nag template.yaml
------------------------------------------------------------
template.yaml
------------------------------------------------------------------------------------------------------------------------
| WARN W9
|
| Resources: ["SecurityGroup"]
| Line Numbers: [9]
|
| Security Groups found with ingress cidr that is not /32
------------------------------------------------------------
| WARN W2
|
| Resources: ["SecurityGroup"]
| Line Numbers: [9]
|
| Security Groups found with cidr open to world on ingress.  This should never be true on instance.  Permissible on ELB
------------------------------------------------------------
| FAIL F1000
|
| Resources: ["SecurityGroup"]
| Line Numbers: [9]
|
| Missing egress rule means all traffic is allowed outbound.  Make this explicit if it is desired configuration

Failures count: 1
Warnings count: 2
```

警告をなくす

```template.yaml
  SecurityGroup:
    Type: 'AWS::EC2::SecurityGroup'
    Metadata:
      cfn_nag:
        rules_to_suppress:
          - id: W9
            reason: "このセキュリティグループはEC2には付与しない。ELBに付与して使用する"
          - id: W5
            reason: "このセキュリティグループはEC2には付与しない。ELBに付与して使用する"
          - id: W2
            reason: "このセキュリティグループはEC2には付与しない。ELBに付与して使用する"
    Properties:
      VpcId: !Ref VpcId
      GroupDescription: security group
      SecurityGroupIngress:
        - IpProtocol: "TCP"
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
          Description: "All"
      SecurityGroupEgress:
        - IpProtocol: "TCP"
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
          Description: "All"
```
