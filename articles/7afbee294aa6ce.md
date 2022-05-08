---
title: "ArgoCD使ってみた"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes"]
published: false
---

# はじめに
sweeep株式会社エンジニアの関田です。

今回はKubernetes環境でCDツールとしてよく使われるArgoCDを触ってみたので、
簡単に紹介したいと思います。


## 準備
ArgoCDを使うにあたって以下の準備が必要になります。
- デプロイ先クラスタ
- マニフェストファイルを管理するGitリポジトリ

今回はMinikube上にデプロイをしていくので、初めに起動しておきます。
```
$ minikube start --driver=docker
```

## インストール

### ネームスペースの作成
argocdをデプロイするためのネームスペースを作成します。
```
$ kubectl create namespace argocd
```

以下ネームスペースをargocdで設定しておきます。
```
$ kubens argocd
```

### ArgoCDのデプロイ
本番運用ではArgoCDのマニフェストファイル自体の管理・運用を行なったりしますが、
今回は簡単にリモートにあるArgoCDのマニフェストを直接クラスタにデプロイします。

```
$ kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### ブラウザで確認
次にデプロイしたArgoCDをブラウザ上で確認していきます。

まず初めにデフォルトで設定されているユーザのパスワードを控えておきます。
```
$ kubectl get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

次に、外部からアクセスできるようにサービスタイプをLoadBalancerに変更します。
```
$ kubectl patch svc argocd-server -p '{"spec": {"type": "LoadBalancer"}}'
```

minikubeを使っている場合は以下のコマンドでホストからアクセスできるようになります。
```
$ minikube tunnel
```

http://localhost にアクセスして以下の情報を入力してログインを行います。
- ユーザ名: admin
- パスワード: 先ほど控えたパスワード


![](/images/7afbee294aa6ce/argocd_top.png)

ログインを行うと以下のようなダッシュボード画面が表示されます。
これから

![](/images/7afbee294aa6ce/argocd_dashboard.png)


### リポジトリの登録

ArgoCDでは監視するマニフェストファイルがあるリポジトリとそのパス、デプロイ先クラスタとネームスペースなどを1つの組みとして、アプリケーションという単位で管理することができます。

まずはリポジトリを登録していきます。
User Info -> CONNECT REPO USING SSH

![](/images/7afbee294aa6ce/argocd_connect_repo.png)

するとNameやProjectなどの入力画面が表示されますが、まだアプリケーションを登録していないので、
ここではRepository URLとSSH private key dataのみを入力して、CONNECTを押します。

![](/images/7afbee294aa6ce/argocd_connect_repo_input.png)

:::message alert
パスフレーズを設定しているSSHキーは対応していないため、あらかじめパスフレーズを解除しておく必要があります。
:::


### アプリケーションの登録

次にアプリケーションを登録していきます。
Manage your applications -> NEW APP

![](/images/7afbee294aa6ce/argocd_manage_app.png)

NEW APPを押した後の設定項目では以下のように入力します。

![](/images/7afbee294aa6ce/argocd_create_app_input_general.png)

| 項目 | 値 | 説明 |
| ---- | ---- | ---- |
| Application Name | sample | アプリケーション名 |
| Project | default | アプリケーションをグルーピングするため |
| SYNC POLICY | Automatic (SELF HEAL) | 同期方法（Gitで定義したマニフェストファイルを強制的にクラスタへ反映させる） |


![](/images/7afbee294aa6ce/argocd_create_app_input_source.png)

| 項目 | 値 | 説明 |
| ---- | ---- | ---- |
| Repository URL | プライベートリポジトリURL | 同期対象のリポジトリURL |
| Path | fastapi/ | 同期対象のマニフェストファイルが保存されているディレクトリパス |


![](/images/7afbee294aa6ce/argocd_create_app_input_destination.png)

| 項目 | 値 | 説明 |
| ---- | ---- | ---- |
| Cluster URL | ArgoCDがデプロイされているクラスタ | デプロイ先のクラスタURL |
| Namespace | default | デプロイ先のネームスペース |

入力後CREATEを押してアプリケーションを作成します。
同期方法をAutomaticに設定したので自動的にデプロイが開始し、最終的に以下のような画面になります。

![](/images/7afbee294aa6ce/argocd_first_app.png)


## 番外編

```bash
$ argocd login --insecure localhost:80
$ argocd repo add git@github.com:${USER}/${REPOSITORY_NAME}.git --ssh-private-key-path ~/.ssh/${PRIVATE_KEY}
```

argocd repo add git@github.com:yu-sekita/sample-gitops-manifests-private.git --ssh-private-key-path ~/.ssh/github_key