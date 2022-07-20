---
title: "Kyverno使ってみた"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---

# はじめに
sweeep株式会社エンジニアの関田です。

今回はKubernetes向けに作られたポリシーエンジンであるkyvernoを使ってみたので
簡単に紹介しようと思います。

kyvernoのドキュメントはこちらになります。

[kyverno Documentation](https://kyverno.io/docs/)

## ポリシーとは

ポリシーとはリソースを作成する際のルールのことで、例えば以下のような例があります。
- imageのタグにlatestを使用させない
- namespaceにdefaultを設定させない
- podの特定のタグを必須にさせる

また、他のポリシーエンジンとしてはOpen Policy Agentがあり、以下のような特徴があります。
- 特別なRego言語で記述
- 汎用的なツール
- Kubernetesに適応するにはGatekeeperというツールと組み合わせる

# Kyvernoのインストール

[Installation](https://kyverno.io/docs/installation/)

今回はHelmを使ってインストールしていきます。

```bash
# リポジトリに登録
$ helm repo add kyverno https://kyverno.github.io/kyverno/
# キャッシュアップデート
$ helm repo update
# デプロイ
$ helm install kyverno kyverno/kyverno -n kyverno
$ helm install kyverno-policies kyverno/kyverno-policies -n kyverno
```

インストールしたらClusterPolicyというリソースが作成されるので確認していきます。

```bash
$ kubectl get cpol
NAME                             BACKGROUND   ACTION   READY
disallow-capabilities            true         audit    true
disallow-host-namespaces         true         audit    true
disallow-host-path               true         audit    true
disallow-host-ports              true         audit    true
disallow-host-process            true         audit    true
disallow-privileged-containers   true         audit    true
disallow-proc-mount              true         audit    true
disallow-selinux                 true         audit    true
restrict-apparmor-profiles       true         audit    true
restrict-seccomp                 true         audit    true
restrict-sysctls                 true         audit    true
```

上記で作成されたポリシーの内容は下記サイトに書かれています。
また、その他のポリシーがサンプルとして記載されています。

[kyverno policies](https://kyverno.io/policies/?policytypes=validate)

# ポリシーの追加

次に新しいポリシーを作成してデプロイをしていきます。

今回はpodに特定のタグが設定されているか、されていなければリソースの作成をさせないようにしたいと思います。

## validationFailureAction

ポリシーに従っていないリソースの作成をさせないようにするには`validationFailureAction`を設定する必要があります。

```yaml
  validationFailureAction: enforce
```

`validationFailureAction`の設定値と説明は以下のようになっています。

| 値 | 説明 |
| ---- | ---- |
| enforce | 違反した場合リソースの作成はされない |
| audit | 違反した場合リソースの作成はされる。`ClusterPolicyReport` または `PolicyReport`というリソースも作成される |

## background

既にデプロイされているリソースに対してチェックするかどうかを設定するには`background`にtrueを設定します。

```yaml
  background: true
```

## rules

ポリシーリソースの構造は以下のようになっています。

```yaml
spec:
    rules:
    - name: rule1
      match: 
      validate: 
    - name: rule2
      match: 
      validate: 
```

複数のルールが定義でき、ルールごとにmatchとvalidateが記載されます。
validateの他にmutate, generateなどが設定できますが、
今回はvalidateを使っていきます。

```yaml
  rules:
  - name: check-for-labels
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "The label `app.kubernetes.io/name` is required."
      pattern:
        metadata:
          labels:
            pod-label: "?*"
```

## デプロイ

今までの設定をファイルに反映させると全体として以下のようになります。

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels    
spec:
  validationFailureAction: enforce
  background: true
  rules:
  - name: check-for-labels
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "The label `app.kubernetes.io/name` is required."
      pattern:
        metadata:
          labels:
            pod-label: "?*"
```

デプロイは`kubectl apply`で実行できます。

```bash
$ kubectl apply require-labels.yml
```

作成されたか確認していきます。

```bash
$ kubectl get cpol
NAME                             BACKGROUND   ACTION   READY
disallow-capabilities            true         audit    true
disallow-host-namespaces         true         audit    true
disallow-host-path               true         audit    true
disallow-host-ports              true         audit    true
disallow-host-process            true         audit    true
disallow-privileged-containers   true         audit    true
disallow-proc-mount              true         audit    true
require-labels                   true         audit    true  # <- 追加されている
disallow-selinux                 true         audit    true
restrict-apparmor-profiles       true         audit    true
restrict-seccomp                 true         audit    true
restrict-sysctls                 true         audit    true
```

## 確認

ポリシーをデプロイできたので、実際にポリシー違反しているリソースと違反していないリソースをデプロイして、
挙動を確認してみます。

サンプルに使うマニフェストは以下になります。
metadataには`pod-label`を設定していないので、エラーになるはずです。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fastapi-deployment
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: fastapi-deployment
    spec:
      containers:
      - image: yusekita/sample-fastapi:test
        imagePullPolicy: Always
        name: sample-fastapi-container
```

デプロイしてみます。
```bash
$ kubectl apply -f fastapi-deployment.yaml
Error from server: error when creating "fastapi-deployment.yaml": admission webhook "validate.kyverno.svc-fail" denied the request:

resource Deployment/default/fastapi-deployment was blocked due to the following policies

require-labels:
  autogen-check-for-labels: 'validation error: The label `app.kubernetes.io/name`
    is required. Rule autogen-check-for-labels failed at path /spec/template/metadata/labels/pod-label/'
```

require-labelsのポリシー違反ということが分かるようにログが出力されました。


次は以下のようにポリシー違反にならないようにマニフェストを修正してデプロイをしてみます。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      pod-label: fastapi-deployment # 修正
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        pod-label: fastapi-deployment # 修正
    spec:
      containers:
      - image: yusekita/sample-fastapi:test
        imagePullPolicy: Always
        name: sample-fastapi-container
```

```bash
$ kubectl apply -f fastapi-deployment.yaml
deployment.apps/fastapi-deployment created
```

ポリシー違反にならずに正常にデプロイができました。


# おわりに
サービスが増え、マニフェストファイルが増えても、
デプロイされるクラスタ側にこのツールを導入するだけでルールに沿って作られているかの
チェックが一括でできるようになるのかなと思い、とても便利なツールだなと思いました。
またKubernetes用に作られているので、導入する上でのハードルはかなり低く感じました。


最後に宣伝にはなりますが、sweeepでは一緒に働くエンジニアを募集しています！
https://corp.sweeep.ai/recruit