---
title: "ArgoCD使ってみた"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes"]
published: true
---

# はじめに
sweeep株式会社エンジニアの関田です。

今回はKubernetes環境でCDツールとしてよく使われるArgoCDを触ってみたので、
簡単に紹介したいと思います。


ArgoCDのドキュメントはこちらになります。
[Argo CD - Declarative GitOps CD for Kubernetes](https://argo-cd.readthedocs.io/en/stable)


# 準備
ArgoCDを使うにあたって以下の準備が必要になります。
- デプロイ先クラスタ
- マニフェストファイルを管理するGitリポジトリ

今回はMinikube上にデプロイをしていくので、初めに起動しておきます。
```
$ minikube start --driver=docker
```

# デプロイ

## ネームスペースの作成
argocdをデプロイするためのネームスペースを作成します。
```
$ kubectl create namespace argocd
```

以下ネームスペースをargocdで設定しておきます。
```
$ kubens argocd
```

## ArgoCDのデプロイ
本番運用ではArgoCDのマニフェストファイル自体の管理・運用を行なったりしますが、
今回は簡単にリモートにあるArgoCDのマニフェストを直接クラスタにデプロイします。

```
$ kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

## ブラウザで確認
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


## リポジトリの登録

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


## アプリケーションの登録

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

アプリケーションを押すとServiceやDeploymentの状態などを見ることができます。

![](/images/7afbee294aa6ce/argocd_app_detail.png)

## 自動デプロイ

最後にマニフェストファイルを編集してGithubリポジトリにpushした時の挙動を確認するため、
Pod数を3から1に変更してみます。

少し時間が経つと以下のようにマニフェストファイルの定義通りに反映されます。

![](/images/7afbee294aa6ce/argocd_app_detail_updated.png)


# おわりに
ArgoCDのインストールから自動デプロイまでを簡単に実施してみました。
分かりやすくて使いやすいUIなので特に迷わずに使える感じがしました。
ArgoCDには他にも機能がたくさんあるので、少しずつ触ってまた記事に残していこうと思います。
また、ArgoCD以外のCDツールとの比較なども試していきたいです。


最後に宣伝にはなりますが、sweeepでは一緒に働くエンジニアを募集しています！
https://corp.sweeep.ai/recruit
