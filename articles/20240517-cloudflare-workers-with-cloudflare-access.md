---
title: "【セキュアな個人開発のお供に!!】 Honoで作ったMPAに、Cloudflare Accessを使って簡単に認証をかける"
emoji: "🔐"
type: "tech"
topics: [ "cloudflare", "typescript", "hono", "cloudflareworkers", "cloudflareaccess"]
published: true
---

## 背景

個人開発者の中には、

簡単なWebアプリを作ったけど、一般公開したくない、、、でもそれなりのアクセス制限をかけるのは面倒、、、

といった場面に遭遇したことはないでしょうか？

今回はCloudflare Accessを使って、なるべく手間をかけずWebアプリにGitHub認証を実装します。

その記録をメモ。

:::message
CloudflareダッシュボードのUIは記事執筆時点から変更されている場合があります。
:::

## 前提

Cloudflare WorkersにWebアプリ(今回はHonoのMPA)がデプロイされていること。

https://zenn.dev/y640/articles/ddf7e2e9a85fd4

独自ドメインが取得されていること。

https://www.onamae.com/service/d-regist/

CloudflareのWebsitesに独自ドメインが登録されていること。

https://shukapin.com/blog/change-ns-from-onamae-com

:::message
今回はお名前.comを使用していますが、初期費用を気にしないのであれば[Cloudflare Registrar](https://www.cloudflare.com/ja-jp/products/registrar/)の使用がおすすめです。
:::

## 手順

### DNS レコードの追加

Workers作成時のエンドポイントにはCloudflare Accessでアクセス制限をかけられないため、新エンドポイントへのDNSレコードを作成します。

Websites > Home > (登録したドメインの)Overview > DNS > Recordsをクリック。

![](/images/cloudflare-workers-with-cloudflare-access/move-to-dns-records.png)

「Add record」ボタンを押下。

![](/images/cloudflare-workers-with-cloudflare-access/add-dns-records.png)

下記情報のレコードを登録する。

| Type  | Name        | Target     |
|-------|-------------|------------|
| CNAME | [Workerの名前] | [取得したドメイン] |

### 認証を実装するエンドポイントの作成

ダッシュボードから Workers & Pages > [対象のWorker] > Settings > Triggers > Add routeをクリック。

![](/images/cloudflare-workers-with-cloudflare-access/move-to-route.png)

下記情報を登録。

| Route                      | Zone       |
|----------------------------|------------|
| [Workerの名前].[取得したドメイン]`/*` | [取得したドメイン] |

:::message
すべてのエンドポイントにアクセス制限をかける場合はRouteに `/*`を使用してください。
:::

ブラウザで適当に疎通確認する。

### 認証方法にGitHubログインを追加

下記の手順を参考にZero TrustにGitHubログインを追加する。

https://developers.cloudflare.com/cloudflare-one/identity/idp-integration/github/

### エンドポイントに認証をかける

Zero Trust > Access > Applications > 「Add an application」をクリック。

![](/images/cloudflare-workers-with-cloudflare-access/move-to-access-applications.png)

Self-hostedを選択。

![](/images/cloudflare-workers-with-cloudflare-access/select-self-hosted.png)

下記情報を入力。

| 設定項目               | 値                      |
|--------------------|------------------------|
| Application name   | [Workerの名称(がわかりやすい)]   |
| Session Duration   | [なるべく短めのDuration]      |
| Application domain | [Workerの名前].[取得したドメイン] |
| Path               | *                      |


Accept all available identity providers を無効化し、Accept Identity Providers をGitHub認証のみにする。

![](/images/cloudflare-workers-with-cloudflare-access/add-identity-providers.png)

Nextをクリック。

下記情報でPolicyを作成する。

| Policy name (Required) | Action (Required) | Session duration  |
|------------------------|-------------------|-------------------|
| [任意のpolicy名]           | Allow             | [なるべく短めのDuration] |

Configure rulesで下記情報を入力。

| Selector | Value                         |
|----------|-------------------------------|
| Emails   | [アクセス許可するGitHubアカウントのメールアドレス] |

Setupセクションは任意の項目を入力して 「Add application」をクリック。

:::message
HTTP OnlyとEnable Binding Cookieは有効化したほうがいいかもしれません。
:::

## 認証がかかることを確認

認証をかけたエンドポイントにアクセスする。

![](/images/cloudflare-workers-with-cloudflare-access/signin-access.png)

GitHubログインを行う。

![](/images/cloudflare-workers-with-cloudflare-access/signin-with-github.png)

🎉🎉🎉 Yay!! 🎉🎉🎉

![](/images/cloudflare-workers-with-cloudflare-access/logined.png)

## まとめ

今回は、HonoのMPAにCloudflare Accessで簡単にGitHub認証をかける方法を紹介しました。

開発に便利なツールを自作したけど、誰でもアクセスできる状況は避けたい、、、でも個人開発で認証までかけるのは面倒、、、といった場面は多いのではないでしょうか?

Cloudflare Workers x Accessであれば、簡単にGitHub認証をかけることができ、更にWorkersにMPA(特にHono x JSXが相性良くおすすめ)をデプロイすれば、ブラウザに秘匿情報 (API keyなど)を保存せず、アプリを構築することが可能です。

## 最後に

間違い等あればご指摘お待ちしております！！

以上。